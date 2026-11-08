# Stack Tecnológico y Arquitectura — Netti

Tecnologías, organización de las piezas y registro de decisiones. Revisado contra las versiones de noviembre de 2026.

## 1. Stack

| Capa | Tecnología |
|------|-----------|
| Framework | Next.js 16.4.5 (App Router, TypeScript estricto, Turbopack) |
| Estilos | Tailwind CSS v4 con tokens en CSS |
| Base de datos | Neon (PostgreSQL serverless), plan Free |
| Acceso a datos | Drizzle ORM sobre `pg` (node-postgres) |
| Mutaciones | Server Actions |
| Endpoints HTTP | Route Handlers solo para `upload`, `revalidate`, `health` y `ready` |
| Imágenes | Vercel Blob + `sharp` (variantes y re-encodeo) |
| Markdown | `remark` + `remark-gfm` + `rehype` + `rehype-sanitize` |
| Validación | Zod 4 |
| Auth | Cookie de sesión propia + tabla `sessions` |
| Rate limiting | Tabla `login_attempts` en Postgres |
| Caché | Cache Components (`cacheComponents: true`) |
| Hosting | Vercel Hobby, región `gru1` |
| DB región | Neon `sa-east-1` |
| CI | GitHub Actions: eslint, typecheck, test, build |

### 1.1 Next.js 16

- 16.4.5 es Active LTS; 15.5.29 quedó en Maintenance LTS.
- `middleware.ts` pasó a llamarse `proxy.ts`.
- **`next lint` fue eliminado.** `next build` ya no corre ESLint; el script es `eslint .`.
- **Cache Components es opt-in**: hace falta `cacheComponents: true` en `next.config.ts`. Pasa a ser el modelo por defecto en Next 17.
- El `proxy.ts` corre en Node. No se usa edge runtime.

### 1.2 Neon

**Por qué Neon y no Supabase**: PostgreSQL puro, sin PostgREST ni RLS que configurar, sin pausa manual del proyecto y sin tarjeta. El branching existe para experimentos, pero los previews usan un proyecto aparte para no compartir cuota.

**Límites del plan Free** (verificar en la consola antes de contratar):

| Recurso | Límite |
|---------|--------|
| Cómputo | 100 CU-horas por proyecto al mes |
| Storage | 1 GB por proyecto |
| Ramas | 10 |
| Transferencia | 5 GB |
| Suspensión por inactividad | 5 minutos |
| Cómputo mínimo | 0,25 CU |
| Ventana de restauración | ~6 horas |
| Historial de restauración | 6 horas, hasta 1 GB de cambios |

Algunos sitios de terceros citan 0,5 GB y 50 CU-horas; la documentación oficial de Neon (noviembre de 2026) confirma **1 GB y 100 CU-horas por proyecto**. El almacenamiento total de la cuenta no debe exceder 20 GB entre todos los proyectos Free. Estos valores se confirman en la consola antes de desplegar.

**El riesgo real**: al agotar las 100 CU-horas, Neon **suspende el cómputo hasta el mes siguiente**. No degrada: se cae. Con 0,25 CU de mínimo, 100 CU-horas equivalen a unas 400 horas despierto, o sea ~55 % del mes.

**El riesgo real**: al agotar las 100 CU-horas, Neon **suspende el cómputo hasta el mes siguiente**. No degrada: se cae. Con 0,25 CU de mínimo, 100 CU-horas equivalen a unas 400 horas despierto, o sea ~55 % del mes.

Por eso el diseño no puede permitirse que las visitas públicas despierten la base:

1. Los artículos se prerenderizan. La lectura pública sale de caché o de ISR, no de Postgres.
2. `cacheLife('max')` y invalidación on-demand al publicar.
3. El health check público **no** toca la base.
4. Un 404 de una ruta desconocida **no** consulta `redirects`; solo una ruta de artículo válida sin fila lo hace.
5. `/buscar`, "Juego al azar" y las estadísticas de portada leen de datos cacheados.
6. Las previews usan **otro proyecto** de Neon, no ramas del de producción (las ramas comparten la cuota de CU-horas y el tope de 10).

**Driver**: `pg` con un pool chico y `attachDatabasePool` de `@vercel/functions` para liberar conexiones en Fluid compute. `max` se calibra: empezar con **1–3** por instancia y medir, no fijar 5 por intuición. El pooler de Neon también tiene un límite de conexiones, y varias instancias concurrentes multiplican los pools. No se usa `neon-http`: no soporta transacciones interactivas, y publicar un cambio de slug necesita una transacción. `@neondatabase/serverless` queda para edge, que no se usa.

**Dos URLs**: `DATABASE_URL` pooled para la aplicación y `DATABASE_URL_UNPOOLED` para migraciones, `pg_dump` y `CREATE INDEX CONCURRENTLY`.

### 1.3 Caché y publicación

```ts
// lib/content.ts
export async function getArticle(kind, slug) {
  "use cache";
  cacheTag(`article:${kind}:${slug}`, `kind:${kind}`, "articles");
  cacheLife("max");
  return db.query.articles.findFirst({ where: visibleArticle });
}
```

**La diferencia que importa**:

| API | Comportamiento | Dónde |
|-----|----------------|-------|
| `revalidateTag(tag, "max")` | Marca obsoleto y sirve lo viejo una vez (SWR) | Route Handlers, cron |
| `revalidateTag(tag, { expire: 0 })` | Expira ya; la próxima request es un miss bloqueante | Route Handlers |
| `updateTag(tag)` | Expira ya y garantiza lectura de lo escrito | **Solo Server Actions** |

Las mutaciones del panel son Server Actions, así que usan `updateTag`. El autor ve su cambio publicado al volver al sitio, sin esperar la segunda visita.

**Caché: lo que no está garantizado.** `"use cache"` es almacenamiento en memoria. En entornos serverless no es seguro que persista entre invocaciones ni que se comparta entre instancias; `cacheLife("max")` no convierte la caché local en almacenamiento remoto compartido. Para una caché distribuida real se necesita `use cache: remote` con un handler, que tiene coste y latencia asociados.

Por tanto, la frase de `README.md` —"la lectura pública no despierta la base"— no está demostrada solo por usar Cache Components. Se debe:

1. Prerenderizar las páginas públicas que se conozcan durante el build cuando sea viable.
2. Separar el shell estático de las consultas realmente dinámicas.
3. Probar explícitamente `use cache` en el despliegue real (Fase 0).
4. Si el contenido depende de caché compartida en runtime, evaluar `use cache: remote`, estudiando su costo y latencia.
5. Mantener una alternativa de lectura estática que no dependa de una conexión a Neon para las páginas ya generadas.

Este punto es un bloqueante de la Fase 0: sin esta prueba, no apruebo la arquitectura de producción.

**Invalidación granular**: una función pura `affectedTags(before, after)` devuelve los tags de la entrada, del listado del `kind`, de las etiquetas **viejas y nuevas**, y `home` solo si cambió `status`, `published_at`, `featured` o la portada. Corregir una tilde no invalida la portada.

**Inventario de proyecciones.** Un tag por entrada no alcanza si el contenido aparece en seis lugares. La tabla es chica y explícita:

| Proyección | Tags |
|------------|------|
| Detalle | `article:<kind>:<slug>`, `kind:<kind>`, `articles` |
| Listado por colección | `kind:<kind>`, `articles` |
| Portada | `home` |
| Etiquetas | `tag:<tag>` (viejas y nuevas) |
| Ficha de juego | `game:<slug>`, `kind:game` |
| Sitemap y RSS | `articles`, `kind:<kind>` |
| Redirecciones | `redirects` |

`affectedTags` es la implementación, no la documentación: la tabla es la fuente. Los tests de integración publican, cambian slug, borran y restauran, y comprueban las siete proyecciones.

**Programados**: `published_at` futuro no se ve porque el predicado público exige `published_at <= now()`. Un job cada 30 minutos llama a un Route Handler protegido que revalida lo que cruzó la hora. Si falla, el artículo aparece en la siguiente revalidación: retraso, no fuga.

**Quién corre ese job**: GitHub Actions es el valor por defecto. Los Cron Jobs de Vercel en Hobby corren como máximo **una vez por día**, y los Schedules —que sí permiten cada 15 minutos— siguen en beta y tienen 5.000 ejecuciones por período. GitHub Actions es gratis, estable y no depende de un producto beta. Si Schedules sale de beta, se migra sin tocar código: el endpoint es el mismo.

**Cardinalidad de tags**: solo entidades y grupos (`articles`, `kind:review`, `article:review:slug`, `tag:analisis`). Nunca un tag derivado de entrada del usuario (`search:<consulta>`, `article:<id-de-visita>`): crecerían sin límite y el usuario público controlaría qué se invalida.

### 1.4 Datos y modelo

Ver `content.md`. Resumen:

- `games` + `articles(kind)` + `media` + `media_variants` + `article_revisions` + `sessions` + `login_attempts` + `redirects`.
- Sin `users`, sin `article_bodies`, sin estado `scheduled`.
- `CHECK` condicionales: un borrador puede estar vacío; una publicación no.
- `previous_status` y `trashed_at` para restaurar.
- `version` entera con trigger que solo incrementa si algo cambió.
- Índice único parcial `(kind, slug) WHERE status <> 'trashed'`.

### 1.5 Imágenes

`POST /api/upload` con `multipart/form-data`, máximo 2 MB (el cliente redimensiona antes a WebP).

El servidor valida tamaño real, MIME, magic numbers, dimensiones y megapíxeles; después `sharp` re-encodea a WebP, descarta EXIF (incluido GPS), limita a 2560 px y genera variantes de 480/960/1600 px. Se suben con `@vercel/blob` (`put`) y se guardan en `media` y `media_variants`. GIF queda fuera: la salida es estática y aceptar animaciones para perder los fotogramas sería peor que rechazarlas.

El re-encodeo reduce la superficie de un archivo manipulado y elimina metadatos; no es una garantía absoluta contra vulnerabilidades del decodificador. Por eso `sharp` se mantiene actualizado y la entrada se valida antes de decodificar.

**Por qué Blob y no R2**: R2 es gratis en uso, pero el dominio propio para servirlo cuesta ~USD 10/año y R2 puede exigir tarjeta. Vercel Blob entra en Hobby sin tarjeta y sirve desde `*.public.blob.vercel-storage.com`, sin dominio propio. El requisito de costo cero real manda.

**Límites que hay que respetar** (Hobby, verificados):

| Recurso | Incluido | Consecuencia de excederlo |
|---------|----------|---------------------------|
| Storage | 1 GB | Blob deja de estar accesible |
| Transferencia | 10 GB/mes | Blob deja de estar accesible |
| Operaciones simples | 10.000/mes | Blob deja de estar accesible |
| Operaciones avanzadas | 2.000/mes | Blob deja de estar accesible |

No hay factura sorpresa: **no se cobra el exceso, se bloquea el almacenamiento hasta 30 días**. El límite operativo más ajustado es el de operaciones avanzadas: subir una imagen con tres variantes son cuatro `put` (cuatro operaciones). Da para ~500 imágenes nuevas al mes, muy por encima de un blog de un autor. Aun así, se mide y se evita resubir la misma imagen.

`cacheControlMaxAge: 31536000` en cada `put` (el máximo útil; por defecto es un mes). La clave lleva UUID, así que el contenido de una URL nunca cambia y la caché larga es segura.

**Por qué server-side y no subida directa desde el navegador**: la subida directa no permite validar los bytes reales ni re-encodear con `sharp`. Con 2 MB el cuerpo entra en el límite de 4,5 MB de Vercel y no infla como base64.

**Sin `next/image` por defecto**: la cuota de transformaciones de Hobby es de 5.000 al mes y cada combinación de ancho, formato y calidad cuenta. Se usa un `loader` propio sobre las variantes pregeneradas.

**Costo**: cero. La contrapartida es que el sitio queda atado a Vercel para las imágenes; `lib/storage.ts` aísla `put`/`del` para poder migrar a R2 o a otro proveedor si algún día se justifica.

### 1.6 Autenticación

- Sin tabla `users`. Contraseña como hash `scrypt` en `ADMIN_PASSWORD_HASH`, formato `scrypt:131072:8:1:<saltB64>:<hashB64>`. El primer número es **N real** (2^17), no una constante de formato. **Separador `:`, no `$`**: Next expande `$VAR` en los archivos `.env` y corrompería el hash.
- `crypto.scrypt` **asíncrono** (no `scryptSync`, que bloquea el event loop). Parámetros según OWASP: N=2^17, r=8, p=1, con `maxmem` explícito. Cada verificación consume ~128 MiB: el rate limit por IP se aplica **antes** de derivar, y se limita la concurrencia con un semáforo local. Si en Vercel no entra con holgura, se baja N con el motivo documentado en un ADR, no en silencio.
- `timingSafeEqual` solo después de comparar longitudes: lanza si los buffers difieren.
- Token aleatorio de 256 bits; en `sessions` se guarda su hash SHA-256. Sin firma: la entropía alcanza y la firma no agrega nada. El token se genera **en el login**; no existe sesión anónima que se reutilice.
- Cookie `HttpOnly`, `Secure`, `SameSite=Lax`. 8 horas deslizantes **con tope absoluto de 30 días**.
- `last_used_at` se actualiza como máximo cada 5 minutos.
- Las sesiones vencidas se purgan con la misma consulta que valida o con un job de baja frecuencia. Una tabla de sesiones que solo crece es una fuga silenciosa.
- Rate limit **por IP**, nunca global: un límite global permite que cualquiera te bloquee el acceso. Backoff progresivo en `login_attempts`. La respuesta de error es genérica: no distingue "no existe" de "contraseña incorrecta", aunque con un solo autor tampoco revela mucho.
- `ip_hash` es HMAC con un secreto, no SHA-256 plano (una IPv4 se revierte por fuerza bruta). Retención de 7 días; el secreto se puede rotar sin perder nada porque los intentos viejos ya no importan.
- "Cerrar todas las sesiones" = `TRUNCATE sessions`.
- `proxy.ts` redirige; `lib/auth.ts` verifica en cada lectura y escritura. El proxy no es una barrera de seguridad.
- Mutaciones: `SameSite=Lax` + `Origin`/`Referer` comparados por **origen completo** (esquema y puerto, no sufijo) contra una allowlist. En previews, el host permitido se valida contra el proyecto de Vercel, no se acepta cualquier `VERCEL_BRANCH_URL` que llegue.

  **No exigir `Content-Type: application/json` en Server Actions.** Ese header es propio de APIs JSON convencionales y no se aplica de forma universal al protocolo de Server Actions de Next.js; imponerlo podría rechazar peticiones legítimas del panel (por ejemplo, enviadas desde un formulario). Next.js ya incorpora comprobación de `Origin` contra `Host`/`X-Forwarded-Host`. Se mantienen controles de origen para operaciones sensibles y controles de Content-Type específicos en los Route Handlers que realmente reciben JSON.

  `SameSite=Lax` y la comprobación de origen son la defensa; la CSP y el saneado de Markdown son capas distintas, no sustitutos.

### 1.7 Markdown y salida

- Pipeline: `remark-parse` → `remark-gfm` → `remark-rehype` → `rehype-sanitize` → `rehype-stringify`.
- Allowlist explícita; sin `style`, sin `on*`, sin `iframe` genérico.
- **La vista previa usa el mismo pipeline que el sitio público.** Si el cliente renderiza distinto, divergen y la vista previa miente.
- **JSON-LD**: escapar `<` como `\u003c` antes de inyectar en `<script type="application/ld+json">`. Un título con `</script>` sería XSS almacenado. El escape vive en una sola función `safeJsonLd(data)`, con tests para `</script><script>alert(1)</script>`; no se repite a mano en cada plantilla.
- **Tipo por colección**: `Review` para reseñas (con `reviewRating.bestRating: 10`, `author`, `itemReviewed` como `VideoGame`), `NewsArticle` para noticias, `Article` para guías y `VideoGame` para fichas. Todos con `inLanguage: "es"` y `dateModified`. No se promete resultado enriquecido: depende del buscador.
- **Imágenes**: `srcset` con variantes pregeneradas; `width` y `height` siempre presentes para evitar CLS.
- **RSS**: escapar `]]>` en CDATA, con una función `safeCdata(text)` probada con ese caso. Feed general y por colección.
- `source_url` se renderiza con `rel="noopener noreferrer nofollow"`. Nunca se hace `fetch` de esa URL desde el servidor.

### 1.8 Seguridad de la respuesta

| Cabecera | Valor |
|----------|-------|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` (sin `preload` hasta tener el dominio definitivo) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `X-Frame-Options` | `DENY` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` |
| `Content-Security-Policy` | Público: `'self'` + hashes. Admin: nonces, renderizado dinámico |

**Decisión sobre CSP**: nonces obligan a renderizar dinámicamente cada página, lo que choca con ISR y con el ahorro de CU-horas. Se aplica CSP con hashes en lo público y nonces solo en `/admin`. No queda "pendiente": queda decidido.

### 1.10 SEO y legal mínimo

- `robots.ts`: permitir el sitio público, `disallow: ['/admin', '/api']` y `sitemap`.
- `/admin/*` responde con `X-Robots-Tag: noindex, nofollow` además de la regla de `robots.txt`. Un `robots.txt` no impide indexar una URL enlazada; la cabecera sí.
- `sitemap.ts` incluye solo contenido visible (`visibleArticle`), con `lastmod` tomado de `updated_at`, y fichas de juegos publicadas.
- `not-found.tsx`, `error.tsx` y `global-error.tsx` con la misma identidad visual. Un 500 no debe mostrar la pantalla por defecto de Next.
- **Atribución de imágenes**: si `media.rights` no es `own`, se muestra `credit` y, cuando exista, enlace a `source_url`. Preferir material de prensa de la desarrolladora antes que capturas de terceros.
- **Afiliados**: si algún día hay enlaces de afiliado, `rel="sponsored nofollow"` y aviso visible. Además, Vercel Hobby deja de ser válido: es hosting no comercial.
- Aviso legal y privacidad antes de añadir analítica o newsletter. Sin cuentas de lectores, el alcance es chico, pero no inexistente.

### 1.9 Qué se descarta

- CMS externos y embebidos: limitan la personalización del panel.
- Contenido en Markdown dentro del repositorio: exige rebuild y no permite escritura inmediata.
- Repositorio de contenido separado.
- Base64 en JSON para imágenes.
- URL prefirmada para subir.
- Tabla `users`, `article_bodies` y estado `scheduled`.
- `next lint` (eliminado en Next 16).
- Edge runtime.
- `neon-http` (sin transacciones interactivas).

---

## 2. Arquitectura

### 2.1 Estructura

```
netti/
├── app/
│   ├── (public)/
│   │   ├── page.tsx
│   │   ├── resenas/[slug]/
│   │   ├── noticias/[slug]/
│   │   ├── guias/[slug]/
│   │   ├── juegos/[slug]/
│   │   ├── etiquetas/[tag]/
│   │   └── buscar/
│   ├── admin/
│   │   ├── login/
│   │   ├── panel/
│   │   └── layout.tsx
│   ├── api/
│   │   ├── upload/route.ts        # multipart, único endpoint de escritura
│   │   ├── revalidate/route.ts    # cron protegido, idempotente
│   │   ├── health/route.ts        # liveness, sin DB
│   │   └── ready/route.ts         # readiness, con DB (baja frecuencia)
│   ├── actions/
│   │   ├── auth.ts                # login, logout
│   │   └── content.ts             # crear, editar, borrar, restaurar, purgar
│   ├── sitemap.ts
│   ├── rss.xml/route.ts
│   └── globals.css
├── components/
├── db/
│   ├── schema.ts
│   ├── index.ts                   # pg + Drizzle + attachDatabasePool
│   └── migrations/
├── lib/
│   ├── auth.ts
│   ├── env.ts
│   ├── content.ts                 # visibleArticle, "use cache", cacheTag
│   ├── tags.ts                    # affectedTags(before, after)
│   ├── markdown.ts
│   ├── images.ts                  # sharp + variantes
│   ├── storage.ts                 # put/delete (Blob)
│   ├── validation.ts              # makeArticleSchema(baseUrl)
│   ├── storage.ts                 # put/delete (Blob)
│   └── seo.ts                     # safeJsonLd, safeCdata
├── public/fonts/
├── proxy.ts
├── next.config.ts
├── drizzle.config.ts
└── package.json
```

### 2.2 Lectura pública

1. La ruta pide `getArticle(kind, slug)`.
2. Si hay entrada en caché, se sirve.
3. Si no, Drizzle consulta Neon y el resultado queda cacheado con `cacheLife('max')`.
4. Markdown → HTML sanitizado.
5. SEO, canonical, JSON-LD.

**Medición de caché.** `x-vercel-cache` evidencia caché de respuesta de ruta/CDN, no necesariamente que la lectura de datos no haya consultado Neon. Para validar que la lectura pública no despierta la base se mide por separado:
- Generación estática / ISR: `x-vercel-cache` y ausencia de CU-h en Neon.
- Lectura de datos con `use cache` runtime: CU-h de Neon antes y después de visitas repetidas.

Si el artículo no existe, **recién ahí** se consulta el mapa de `redirects` (cacheado bajo el tag `redirects`) y se responde 308. Nunca en el proxy ni en cualquier URL arbitraria.

El resolver es común a todas las rutas de contenido. Una URL que cambia de colección (`/resenas/juego-x` → `/juegos/juego-x`) se redirige igual, porque el mapa se consulta después de que la ruta de contenido no encontró la fila, no después de que el artículo exista en la colección anterior.

### 2.3 Escritura

1. Server Action → `requireSession()` → verifica `Origin` → Zod → `articleSchema`.
2. El servidor completa `published_at` si hace falta.
3. Dentro de **una transacción**: `UPDATE ... WHERE id AND version RETURNING version` + `INSERT` del snapshot en `article_revisions`.
4. Después del `COMMIT`: `updateTag` sobre los tags de `affectedTags()`.
5. Devuelve `{ ok, version }` o `{ ok: false, errors, current? }`.

Invalidar antes de confirmar publica un estado que todavía puede revertirse.

### 2.4 Límites intencionales

- Sin comentarios ni cuentas de lectores.
- Nada se escribe al disco local.
- Sin API pública.
- Sin duplicar lógica entre Server Actions y Route Handlers.

### 2.5 Contratos de `lib/`

Estos contratos están listados aquí para que la implementación no parta de cero:

**`lib/storage.ts`** — abstrae el proveedor de archivos (hoy Vercel Blob):

```ts
export interface StoredObject {
  key: string;          // articles/<uuid>.webp, nunca slug ni nombre original
  url: string;          // pública, derivada de la respuesta de Blob
  bytes: number;
}

export interface Storage {
  put(
    key: string,
    body: ReadableStream | Blob | ArrayBuffer,
    options: {
      contentType: string;
      cacheControlMaxAge: number;
    },
  ): Promise<StoredObject>;
  del(keys: string[]): Promise<void>;
}
```

- `put` usa claves exactas (`addRandomSuffix: false`) para que el `key` sea estable y el `url` predecible.
- `del` es idempotente; si falla alguna variante, se registra en `storage_orphans` para reconciliación posterior.
- Credenciales: OIDC en Vercel (`VERCEL_OIDC_TOKEN` + `BLOB_STORE_ID`), `BLOB_READ_WRITE_TOKEN` en desarrollo local y CI.

**`lib/seo.ts`**:

```ts
export function safeJsonLd(value: unknown): string;
export function safeCdata(value: string): string;
```

- `safeJsonLd`: `JSON.stringify(value)` y sustitución de `<`, `>`, `&` por secuencias `\uXXXX` antes de inyectar en `<script type="application/ld+json">`. Fallá controlado si hay referencias cíclicas; nunca emite texto parcial.
- `safeCdata`: rechaza o escapa `]]>` en CDATA de RSS.

**`lib/validation.ts`**:

```ts
export function makeArticleSchema(imagesBaseUrl: string): z.ZodType<ArticleInput, any, any>;
export type ArticleInput = z.infer<ReturnType<typeof makeArticleSchema>>;
export function isUniqueViolation(error: unknown, constraintName: string): boolean;
export function requireSession(): Promise<Session>;
export function assertSameOrigin(origin?: string): void;
```

**`lib/content.ts`**:

```ts
export function visibleArticle(now?: Date): SQL;
export function affectedTags(before: ArticleTagState | null, after: ArticleTagState | null): string[];
```

--- 

## 3. Decisiones

Cada decisión tiene dos planos: **qué se eligió** y **cuánto está probado**. Las que dependen de infraestructura no verificada llevan `Validación: pendiente` y se confirman en la Fase 0. Presentar una hipótesis como propiedad confirmada es justamente lo que hay que evitar.

### D-001: Next.js 16.4.5

16.4.5, no 15: 15 está en Maintenance LTS, `next lint` desapareció y Cache Components es el modelo nuevo. **Aceptada.**

### D-002: Neon (PostgreSQL)

Sin pausa manual, sin RLS que configurar, branching y Postgres puro. El riesgo de CU-horas se mitiga no consultando la base en la lectura pública. **Aceptada.** `Validación: pendiente` — el consumo real se mide en la Fase 0.

### D-003: Un solo repositorio

Código y documentación en un repo; el contenido en Postgres. **Aceptada.**

### D-004: `main` estable + ramas cortas

Previews con **proyecto de Neon separado**, no ramas del de producción. CI obligatorio. **Aceptada.**

### D-005: Sesión propia

Un autor, hash en entorno, token aleatorio hasheado, `SameSite=Lax`, tope absoluto de 30 días. **Aceptada.**

### D-006: Panel a pantalla completa

Sidebar + formulario, vista previa en modal con la plantilla real. **Aceptada.**

### D-007: Paleta morada

`#a78bfa`, `#c9bbff`, `#5b21b6`. **Aceptada.**

### D-008: Sin comentarios ni cuentas

**Aceptada.**

### D-009: Vercel Blob + subida por servidor

`multipart/form-data` ≤ 2 MB, validación de bytes y `sharp`. **Aceptada.** Blob es gratis en Hobby sin tarjeta ni dominio propio; la contrapartida es el límite de 1 GB storage y 10 GB transferencia mensuales. `Validación: pendiente` — se comprueba antes de la Fase 5.

### D-010: Tailwind v4

**Aceptada.**

### D-011: Markdown sanitizado

`remark` + `rehype` + `rehype-sanitize`, mismo pipeline en sitio y vista previa. **Aceptada.**

### D-012: Backups obligatorios desde la Fase 3

`pg_dump` nocturno con cliente 17, contra la URL directa, cifrado y con restauración automatizada. **Aceptada.**

### D-013: Una tabla `articles` con `kind`

Reemplaza la idea de tablas separadas. Un predicado de visibilidad, un índice FTS, una papelera, un sitemap. **Aceptada** (revisión de la ronda anterior).

### D-014: Server Actions para mutaciones

`updateTag` resuelve la lectura inmediata y la verificación de origen. Route Handlers solo para `upload`, revalidación programada, liveness y readiness. **Aceptada.**

### D-015: Cache Components explícito

`cacheComponents: true`, `cacheLife('max')`, invalidación granular con `affectedTags`. **Aceptada.** `Validación: pendiente` — se mide `x-vercel-cache` en la Fase 0.

### D-016: `scheduled` derivado

No es un estado: es `published_at` futuro con `status = 'published'`. **Aceptada.**

### D-017: Imágenes pregeneradas

`sharp` + variantes + loader propio, no la optimización de Vercel. **Aceptada.**

### D-018: Rate limit en Postgres

Sin Upstash. Una tabla, backoff por IP. **Aceptada.**

### D-019: CSP con hashes en público, nonces en admin

**Aceptada.**

### D-020: Costo real declarado

Cero en Hobby. Blob entra en Hobby sin tarjeta; no hay dominio propio que pagar. El único "costo" posible es quedar bloqueado si se superan los límites (1 GB storage, 10 GB transferencia, 2.000 operaciones avanzadas/mes), pero no se genera deuda. **Aceptada.**

### D-021: Escritura atómica y borrado mark-and-sweep

Artículo y revisión en una transacción; invalidación de caché después del `COMMIT`. Medios: marcar → borrar en Blob → borrar filas. **Aceptada.** Postgres y Blob no comparten transacción; cualquier orden que borre el objeto antes de confirmar la base deja referencias irrecuperables.

### D-022: Revalidación programada fuera de Vercel

GitHub Actions cada 30 minutos contra un endpoint idempotente. Cron Jobs de Vercel (Hobby) no baja de una ejecución diaria; Schedules sigue en beta. **Aceptada.**

### Decisiones diferidas

- **D-F1**: export a Markdown como portabilidad (distinto del backup).
- **D-F2**: paginación server-side.
- **D-F3**: analítica sin cookies.
- **D-F4**: newsletter.
- **D-F5**: OG dinámicas.
