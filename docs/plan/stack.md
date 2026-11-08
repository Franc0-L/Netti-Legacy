# Stack Tecnológico y Arquitectura — Netti

Tecnologías, organización de las piezas y registro de decisiones. Revisado contra las versiones de noviembre de 2026.

## 1. Stack

| Capa | Tecnología |
|------|-----------|
| Framework | Next.js 16.4.x (App Router, TypeScript estricto, Turbopack) |
| Estilos | Tailwind CSS v4 con tokens en CSS |
| Base de datos | Neon (PostgreSQL serverless), plan Free |
| Acceso a datos | Drizzle ORM sobre `pg` (node-postgres) |
| Mutaciones | Server Actions |
| Endpoints HTTP | Route Handlers solo para `upload` y revalidación programada |
| Imágenes | Cloudflare R2 + `sharp` (variantes y re-encodeo) |
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

**Por qué Neon y no Supabase**: PostgreSQL puro, sin PostgREST ni RLS que configurar, sin pausa manual del proyecto, branching para previews y sin tarjeta.

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

**El riesgo real**: al agotar las 100 CU-horas, Neon **suspende el cómputo hasta el mes siguiente**. No degrada: se cae. Con 0,25 CU de mínimo, 100 CU-horas equivalen a unas 400 horas despierto, o sea ~55 % del mes.

Por eso el diseño no puede permitirse que las visitas públicas despierten la base:

1. Los artículos se prerenderizan. La lectura pública sale de caché o de ISR, no de Postgres.
2. `cacheLife('max')` y invalidación on-demand al publicar.
3. El health check público **no** toca la base.
4. Un 404 de una ruta desconocida **no** consulta `redirects`; solo una ruta de artículo válida sin fila lo hace.
5. `/buscar`, "Juego al azar" y las estadísticas de portada leen de datos cacheados.
6. Las previews usan **otro proyecto** de Neon, no ramas del de producción (las ramas comparten la cuota de CU-horas y el tope de 10).

**Driver**: `pg` con un pool chico (`max: 5`) y `attachDatabasePool` de `@vercel/functions` para liberar conexiones en Fluid compute. No se usa `neon-http`: no soporta transacciones interactivas, y publicar un cambio de slug necesita una transacción. `@neondatabase/serverless` queda para edge, que no se usa.

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

**Invalidación granular**: una función pura `affectedTags(before, after)` devuelve los tags de la entrada, del listado del `kind`, de las etiquetas **viejas y nuevas**, y `home` solo si cambió `status`, `published_at`, `featured` o la portada. Corregir una tilde no invalida la portada.

**Programados**: `published_at` futuro no se ve porque el predicado público exige `published_at <= now()`. Una GitHub Action cada 30 minutos llama a un Route Handler protegido que revalida lo que cruzó la hora. Si falla, el artículo aparece en la siguiente revalidación: retraso, no fuga.

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

El servidor valida tamaño, MIME, magic numbers, dimensiones y megapíxeles; después `sharp` re-encodea (elimina EXIF y polyglots) y genera variantes de 480/960/1600 px. Se guardan en `media` y `media_variants`.

**Por qué server-side y no URL prefirmada**: R2 no soporta presigned POST con `content-length-range`, así que una URL prefirmada no puede imponer un tope. Además obliga a CORS y no permite validar los bytes reales. Con 2 MB el cuerpo entra en el límite de 4,5 MB de Vercel y no infla como base64.

**Sin `next/image` por defecto**: la cuota de transformaciones de Hobby es de 5.000 al mes y cada combinación de ancho, formato y calidad cuenta. Se usa un `loader` propio sobre las variantes pregeneradas.

**Costo**: servir R2 con un dominio propio cuesta ~USD 10/año. `r2.dev` es gratis pero limitado y no apto para producción. **Esto rompe el "costo cero" estricto** y hay que aceptarlo explícitamente o cambiar a Vercel Blob.

### 1.6 Autenticación

- Sin tabla `users`. Contraseña como hash `scrypt` en `ADMIN_PASSWORD_HASH`, formato `scrypt:16384:8:1:<saltB64>:<hashB64>`. **Separador `:`, no `$`**: Next expande `$VAR` en los archivos `.env` y corrompería el hash.
- `crypto.scrypt` **asíncrono** (no `scryptSync`, que bloquea el event loop). Parámetros según OWASP: N=2^17, r=8, p=1, con `maxmem` explícito.
- `timingSafeEqual` solo después de comparar longitudes: lanza si los buffers difieren.
- Token aleatorio de 256 bits; en `sessions` se guarda su hash SHA-256. Sin firma: la entropía alcanza y la firma no agrega nada.
- Cookie `HttpOnly`, `Secure`, `SameSite=Lax`. 8 horas deslizantes **con tope absoluto de 30 días**.
- `last_used_at` se actualiza como máximo cada 5 minutos.
- Rate limit **por IP**, nunca global: un límite global permite que cualquiera te bloquee el acceso. Backoff progresivo en `login_attempts`.
- `ip_hash` es HMAC con un secreto, no SHA-256 plano (una IPv4 se revierte por fuerza bruta).
- "Cerrar todas las sesiones" = `TRUNCATE sessions`.
- `proxy.ts` redirige; `lib/auth.ts` verifica en cada lectura y escritura. El proxy no es una barrera de seguridad.
- Mutaciones: `SameSite=Lax` + `Origin`/`Referer` contra una allowlist (incluye `VERCEL_BRANCH_URL` en previews) + `Content-Type: application/json`. Se rechaza si faltan ambos headers.

### 1.7 Markdown y salida

- Pipeline: `remark-parse` → `remark-gfm` → `remark-rehype` → `rehype-sanitize` → `rehype-stringify`.
- Allowlist explícita; sin `style`, sin `on*`, sin `iframe` genérico.
- **La vista previa usa el mismo pipeline que el sitio público.** Si el cliente renderiza distinto, divergen y la vista previa miente.
- **JSON-LD**: escapar `<` como `\u003c` antes de inyectar en `<script type="application/ld+json">`. Un título con `</script>` sería XSS almacenado.
- **RSS**: escapar `]]>` en CDATA.
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
│   │   └── revalidate/route.ts    # cron protegido
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
│   ├── storage.ts                 # put/delete (R2 o Blob)
│   ├── validation.ts              # makeArticleSchema(baseUrl)
│   └── r2.ts
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

Si el artículo no existe, **recién ahí** se consulta el mapa de `redirects` (cacheado bajo el tag `redirects`) y se responde 308. Nunca en el proxy ni en cada 404.

### 2.3 Escritura

1. Server Action → `requireSession()` → verifica `Origin` → Zod → `articleSchema`.
2. El servidor completa `published_at` si hace falta.
3. `UPDATE ... WHERE id AND version RETURNING version`.
4. Snapshot en `article_revisions`.
5. `updateTag` sobre los tags de `affectedTags()`.
6. Devuelve `{ ok, version }` o `{ ok: false, errors }`.

### 2.4 Límites intencionales

- Sin comentarios ni cuentas de lectores.
- Nada se escribe al disco local.
- Sin API pública.
- Sin duplicar lógica entre Server Actions y Route Handlers.

---

## 3. Decisiones

### D-001: Next.js 16

16.4.x, no 15: 15 está en Maintenance LTS, `next lint` desapareció y Cache Components es el modelo nuevo. **Aceptada.**

### D-002: Neon (PostgreSQL)

Sin pausa manual, sin RLS que configurar, branching y Postgres puro. El riesgo de CU-horas se mitiga no consultando la base en la lectura pública. **Aceptada.**

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

### D-009: R2 + subida por servidor

`multipart/form-data` ≤ 2 MB, validación de bytes y `sharp`. **Aceptada** (con el costo del dominio explícito).

### D-010: Tailwind v4

**Aceptada.**

### D-011: Markdown sanitizado

`remark` + `rehype` + `rehype-sanitize`, mismo pipeline en sitio y vista previa. **Aceptada.**

### D-012: Backups obligatorios desde la Fase 3

`pg_dump` nocturno con cliente 17, contra la URL directa, cifrado y con restauración automatizada. **Aceptada.**

### D-013: Una tabla `articles` con `kind`

Reemplaza la idea de tablas separadas. Un predicado de visibilidad, un índice FTS, una papelera, un sitemap. **Aceptada** (revisión de la ronda anterior).

### D-014: Server Actions para mutaciones

`updateTag` resuelve la lectura inmediata y la verificación de origen. Route Handlers solo para `upload` y el cron. **Aceptada.**

### D-015: Cache Components explícito

`cacheComponents: true`, `cacheLife('max')`, invalidación granular con `affectedTags`. **Aceptada.**

### D-016: `scheduled` derivado

No es un estado: es `published_at` futuro con `status = 'published'`. **Aceptada.**

### D-017: Imágenes pregeneradas

`sharp` + variantes + loader propio, no la optimización de Vercel. **Aceptada.**

### D-018: Rate limit en Postgres

Sin Upstash. Una tabla, backoff por IP. **Aceptada.**

### D-019: CSP con hashes en público, nonces en admin

**Aceptada.**

### D-020: Costo real declarado

Dominio de imágenes ~USD 10/año y posible método de pago en R2. No es cero absoluto. **Aceptada con reserva**: si no se acepta, se evalúa Vercel Blob.

### Decisiones diferidas

- **D-F1**: export a Markdown como portabilidad (distinto del backup).
- **D-F2**: paginación server-side.
- **D-F3**: analítica sin cookies.
- **D-F4**: newsletter.
- **D-F5**: OG dinámicas.
