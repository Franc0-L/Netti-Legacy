# Operaciones: Git, Entorno y Despliegue — Netti

## 1. Git

| Rama | Uso |
|------|-----|
| `main` | Producción |
| `feat/*`, `fix/*`, `chore/*` | Cambios cortos con preview |

El contenido no toca Git: vive en Postgres y se escribe desde el panel. No hay rama de contenido.

**Previews**: cada PR usa un **proyecto de Neon separado**, no una rama del proyecto de producción. Las ramas comparten las 100 CU-horas y el tope de 10; el PR número 11 fallaría. Un segundo proyecto Free (hay 100) evita compartir cuota y evita que un preview con migraciones rompa producción.

**CI** en cada PR y en `main`:

```bash
npm ci
npm run lint        # eslint .
npm run typecheck   # tsc --noEmit
npm run test        # vitest run, con Postgres 17 en service container
npm run build
```

`next build` ya no corre ESLint: si no se agrega `npm run lint`, un error de lint llega a producción.

---

## 2. Entorno

### 2.1 Servicios y costo real

| Componente | Servicio | Costo |
|------------|----------|-------|
| Base de datos | Neon Free | $0 |
| Imágenes | Vercel Blob | $0 en Hobby dentro de los límites |
| Dominio de imágenes | Dominio propio en Cloudflare | **~USD 10/año** |
| Hosting | Vercel Hobby | $0, solo uso no comercial |
| CI | GitHub Actions | $0 en repo público |
| Monitoreo | Servicio de uptime + tracker de errores | $0 en planes gratuitos |

**El "costo cero" es real en el plan Hobby.** Solo hay que respetar los límites de Blob:

| Recurso | Incluido | Consecuencia de excederlo |
|---------|----------|---------------------------|
| Storage | 1 GB | Blob deja de estar accesible |
| Transferencia | 10 GB/mes | Blob deja de estar accesible |
| Operaciones simples | 10.000/mes | Blob deja de estar accesible |
| Operaciones avanzadas | 2.000/mes | Blob deja de estar accesible |

No hay factura sorpresa: **no se cobra el exceso, se bloquea el almacenamiento hasta 30 días** y hay que esperar el ciclo. El límite más ajustado son las operaciones avanzadas: subir una imagen con tres variantes son cuatro `put` (cuatro operaciones). Da para ~500 imágenes nuevas al mes, muy por encima de un blog de un autor. Se mide y se evita resubir la misma imagen.

**Vercel Hobby es no comercial.** Publicidad o afiliados obligan a Pro o a mover el hosting a Cloudflare Pages.

### 2.2 Variables

| Variable | Requerida | Descripción |
|----------|-----------|-------------|
| `DATABASE_URL` | Sí | Cadena **pooled** de Neon, para la aplicación |
| `DATABASE_URL_UNPOOLED` | Sí | Cadena directa, para migraciones y `pg_dump` |
| `ADMIN_PASSWORD_HASH` | Sí | `scrypt:131072:8:1:<saltB64>:<hashB64>` |
| `IP_HASH_SECRET` | Sí | Secreto del HMAC que hashea IPs |
| `PUBLIC_SITE_URL` | Sí | Origen de producción |
| `PREVIEW_ORIGIN_REGEX` | No | Patrón de origen permitido en previews, anclado al dominio del proyecto (no aceptar subdominios arbitrarios) |
| `IMAGES_PUBLIC_BASE_URL` | Sí | URL pública de Blob (`https://<store>.public.blob.vercel-storage.com`) |
| `CRON_SECRET` | Sí | Bearer del endpoint de revalidación |
| `AGE_PUBLIC_KEY` | Sí | Clave pública `age` para cifrar el dump de backup |
**Custodia de la clave `age`**: la clave pública (`AGE_PUBLIC_KEY`) se usa para cifrar el backup en GitHub Actions. La clave privada (`AGE_PRIVATE_KEY`) se conserva fuera del repo y fuera de GitHub Actions (por ejemplo, en el gestor de contraseñas del autor o un secreto de entorno protegido), y se usa solo para la restauración mensual de prueba. Si se pierde, el backup cifrado no se puede recuperar.
| `BLOB_STORE_ID` | No | ID del store de Vercel Blob (OIDC en Vercel suele proveerlo automáticamente; confirmar si el código lo usa realmente) |
| `BLOB_READ_WRITE_TOKEN` | No | Token de lectura/escritura para desarrollo local y CI; en producción se usa OIDC |
| `SITE` | Sí | Origen público usado por el cron de revalidación (`https://netti.vercel.app`). |

**El separador del hash es `:`, no `$`.** Next expande `$VAR` en los archivos `.env`; un hash con `$` queda corrupto y el login falla sin explicación.

**Previews**: `PUBLIC_SITE_URL` es fijo, así que el panel en un preview rechazaría el login con `403`. La verificación de origen compara el `Origin` normalizado contra `PUBLIC_SITE_URL` **o** contra `PREVIEW_ORIGIN_REGEX` (patrón anclado al dominio de Vercel, por ejemplo `https://.*-netti\.vercel\.app`). No se acepta cualquier `VERCEL_BRANCH_URL` que llegue: el host permitido se valida contra el proyecto de Vercel configurado.

**Dos URLs de base**: el pooler no sirve para migraciones ni para `pg_dump`, y `CREATE INDEX CONCURRENTLY` falla a través de él.

### 2.3 `.env.local`

```env
DATABASE_URL=postgresql://...@...-pooler.sa-east-1.aws.neon.tech/netti?sslmode=require
DATABASE_URL_UNPOOLED=postgresql://...@...sa-east-1.aws.neon.tech/netti?sslmode=require

ADMIN_PASSWORD_HASH=scrypt:131072:8:1:
IP_HASH_SECRET=
CRON_SECRET=
SITE=https://netti.vercel.app

PUBLIC_SITE_URL=http://localhost:3000
PREVIEW_ORIGIN_REGEX=https://.*-netti\.vercel\.app
IMAGES_PUBLIC_BASE_URL=https://ce0rcu23vrrdzqap.public.blob.vercel-storage.com
AGE_PUBLIC_KEY=age1...

BLOB_STORE_ID=
BLOB_READ_WRITE_TOKEN=
```

Generar el hash:

```bash
node -e "
const c = require('crypto');
const p = process.argv[1];
const s = c.randomBytes(16);
c.scrypt(p, s, 64, { N: 2**17, r: 8, p: 1, maxmem: 256*1024*1024 }, (err, h) => {
  if (err) throw err;
  console.log(['scrypt', 2**17, 8, 1, s.toString('base64'), h.toString('base64')].join(':'));
});
" "tu-contraseña"
```

`scryptSync` se evita a propósito: bloquea el event loop y consume CPU activa. N=2^17 necesita `maxmem` explícito; sin él, Node falla con `ERR_CRYPTO_INVALID_SCRYPT_PARAMS`.

El formato del hash debe coincidir **exactamente** entre lo que genera este comando y lo que parsea `lib/auth.ts`: `scrypt:<N>:<r>:<p>:<saltB64>:<hashB64>`. El primer número es el N real (131072), no el exponente (17) ni el valor por defecto de Node (16384).

### 2.3.1 Versión exacta de las dependencias

- `next` se fija **sin rango**: `"next": "16.4.5"`, no `"^16.4.5"`. Un despliegue tiene que ser reproducible.
- `package-lock.json` se commitea y el CI usa `npm ci`.
- Node se fija con `.nvmrc` y `engines` (24.x LTS).
- Actualizaciones de seguridad: Renovate o Dependabot abre el PR, el CI corre, se revisa el advisory y se despliega. No se actualiza a ciegas ni se queda en una versión con vulnerabilidades conocidas por comodidad.
- Active LTS no significa "seguro por defecto": los advisories de Next 16.4.x se revisan antes de cada bump.

### 2.4 Migraciones

```bash
npm run db:generate   # SQL desde db/schema.ts
npm run db:migrate    # aplica contra DATABASE_URL_UNPOOLED
npm run db:studio
```

Se revisan en el PR y se ejecutan **manualmente**, nunca en el build de Vercel.

### 2.5 Backup

Neon Free no incluye backups automáticos. La ventana de restauración de ~6 horas complementa, no reemplaza.

**Destino del backup:** GitHub Actions artifacts. Son gratuitos en repos públicos, cuentan contra los 500 MB de almacenamiento de Actions de GitHub Free, y expiran tras 90 días. Para Netti (volumen bajo), alcanzan. No hay tarjeta ni proveedor extra.

Workflow nocturno:

1. Instalar `postgresql-client-17` desde PGDG (el runner no lo trae y `pg_dump` falla si el cliente es más viejo que el servidor).
2. `pg_dump --format=custom --no-owner --exclude-table-data=sessions --exclude-table-data=login_attempts` contra `DATABASE_URL_UNPOOLED`.
3. Cifrar con `age`.
4. Subir como artifact de GitHub Actions (`actions/upload-artifact`).
5. Retención de 90 días (límite de GitHub); suficiente para la política de 30 días.
6. Verificar el dump **antes** de darlo por bueno (ver checklist).

**Límite conocido:** 500 MB de almacenamiento de Actions compartido en la cuenta. GitHub Free también incluye 2.000 minutos mensuales de runners privados. Si los dumps crecen demasiado, habrá que rotar más agresivamente o buscar otro destino gratuito. No hay factura sorpresa: se deja de poder subir, no se paga el exceso. Estos límites se verifican en la cuenta/plan actual y se confirman con un artifact de prueba; no se asumen como permanentes.

**Checklist de integridad del backup.** Un dump que termina con código 0 puede estar incompleto o no ser restaurable:

- [ ] El archivo existe, pesa más que un mínimo razonable y `pg_restore --list` lo abre sin error.
- [ ] Incluye esquema, datos, tipos, funciones, triggers, índices y restricciones (el formato custom los lleva todos; verificarlo con `--list`).
- [ ] Incluye las extensiones usadas (`pg_trgm`, `unaccent`, `pgcrypto` si se usa) y falla la restauración si no están disponibles.
- [ ] Excluye `sessions` y `login_attempts`.
- [ ] Los conteos de `articles`, `games` y `media` coinciden con los de producción al momento del dump (guardados en un archivo junto al dump).
- [ ] La restauración de prueba mensual inserta datos y ejecuta una consulta pública real, no solo `count(*)`.

**El backup de Postgres no incluye Blob.** Son dos recuperaciones distintas:

| Qué | Cómo se recupera |
|-----|------------------|
| Artículos, juegos, metadatos de medios | `pg_dump` |
| Objetos y variantes de imagen | Durabilidad de Blob + inventario verificable |

`media.storage_key` es el inventario: un job de baja frecuencia compara las claves de la tabla con las del store y reporta faltantes y huérfanos. Si un objeto se borró por error, no se puede regenerar; por eso la purga es mark-and-sweep y nunca borra antes de confirmar la base.

**Heartbeat**: si el workflow no corre, nadie se entera. Un chequeo externo (healthchecks.io, gratis) avisa cuando el dump falta. Los workflows programados de GitHub se desactivan tras 60 días sin actividad en repos públicos.

**Restauración probada mensualmente**: un job levanta Postgres 17 en un service container, restaura el dump y verifica conteos de filas. Un backup que nunca se restauró no es un backup.

### 2.6 Revalidación programada

GitHub Action cada 30 minutos:

```bash
curl -fsS -X POST "$SITE/api/revalidate" -H "Authorization: Bearer $CRON_SECRET"
```

**Por qué no Vercel Cron.** Hay dos productos y ninguno sirve tal cual hoy:

| Producto | Hobby | Problema |
|----------|-------|----------|
| Cron Jobs (legacy) | 100 jobs por proyecto, pero **mínimo una vez por día** | No permite cada 30 min |
| Schedules (beta) | Cada 15 min con jitter, 5.000 ejecuciones por período | Beta; no se apuesta la publicación programada a un producto que puede cambiar |

GitHub Actions es gratis, estable y no depende de un producto beta. Si Schedules sale de beta con condiciones aceptables, se migra sin tocar código: el endpoint es el mismo.

El endpoint debe ser **idempotente**: revalida lo que ya cruzó `published_at` y no hace nada si no hay pendientes. Registra `last_run_at` y `last_activated_at` para que el panel y el monitor puedan distinguir "no había nada" de "el job dejó de correr". Si el job falla, el artículo aparece en la siguiente ejecución: retraso acotado, no fuga de contenido.

### 2.7 Despliegue

1. Importar el repo en Vercel; framework Next.js; sin output directory manual.
2. Proyecto Neon de producción y proyecto separado de previews.
3. Blob store creado en Vercel Storage; CORS no hace falta (la subida pasa por el servidor).
4. Cargar variables en Production, Preview y Development.
5. `npm run db:migrate` contra producción, una vez.
6. Verificar: `/`, `/api/health`, login, subida, publicación, programación, backup y restauración. `/api/ready` se prueba aparte, una vez, no en el monitor.

### 2.8 Scripts

| Script | Comando |
|--------|---------|
| `dev` | `next dev` |
| `build` | `next build` |
| `start` | `next start` |
| `lint` | `eslint .` |
| `typecheck` | `tsc --noEmit` |
| `test` | `vitest run` |
| `db:generate` | `drizzle-kit generate` |
| `db:migrate` | `drizzle-kit migrate` |
| `db:studio` | `drizzle-kit studio` |

### 2.9 Monitoreo

- **Uptime**: ping a `/api/health`, que no toca la base. Nunca a `/api/ready`.
- **Readiness**: `/api/ready` se consulta a mano o desde un job de baja frecuencia. Distingue "el sitio responde" de "la base está disponible".
- **Errores**: Sentry u otro tracker. Los logs de Vercel Hobby tienen retención corta y no sirven como historial.
- **CU-horas de Neon**: revisión mensual en la consola. Es el primer límite que se agota.
- **Backups**: heartbeat externo; si el dump no llegó, alerta.
- **Revalidación programada**: `last_run_at` visible; si se atrasa más de una hora, alerta.
- **Degradación**: si Neon suspende el cómputo por cuota, el sitio cacheado sigue sirviendo y el panel deja de funcionar. El monitor tiene que avisar antes.

### 2.10 Runbook mínimo

Cada procedimiento empieza por confirmar el síntoma, no por reiniciar cosas.

| Síntoma | Qué comprobar | Qué hacer |
|---------|---------------|-----------|
| El sitio público no carga | `/api/health` | Si responde, el problema es de datos o de caché; si no responde, revisar el despliegue en Vercel. |
| El panel no entra | `/api/ready`, CU-horas en Neon | Cuota agotada → esperar el ciclo o pasar a plan pago; no hay atajo. |
| Una publicación no aparece | `published_at`, `last_run_at` del cron | Revalidar el tag a mano desde el panel. |
| Se borró una imagen por error | `media.storage_key` y el store | Restaurar desde el original del autor; el backup de Postgres no la contiene. |
| Un backup falló | Heartbeat y logs del workflow | Ejecutarlo a mano antes de investigar; luego corregir la causa. |
| Se publicó contenido incorrecto | `article_revisions` | Restaurar el snapshot anterior (crea una versión nueva). |
| Neon suspendido por cuota | Consola de Neon | El sitio cacheado sigue; el panel no. Documentar la fecha y planificar el cambio de plan. |

**Rotación de secretos**: `IP_HASH_SECRET` y `CRON_SECRET` se rotan sin downtime (el segundo, actualizando la Action). `BLOB_READ_WRITE_TOKEN` se rota creando un token nuevo en Vercel, desplegando y revocando el viejo. `ADMIN_PASSWORD_HASH` se regenera y se despliega; las sesiones abiertas siguen hasta expirar o se cierran con `logoutAll`.

### 2.11 Pruebas

| Nivel | Qué cubre | Dónde |
|-------|-----------|-------|
| Unitarias | Slug, Zod (borrador vs publicación), `affectedTags`, `safeJsonLd`, `safeCdata` | Vitest |
| Integración | Publicar, editar, despublicar, papelera, restaurar con sufijo, purga, redirecciones | Vitest + Postgres 17 en service container |
| Seguridad | Corpus XSS para Markdown y JSON-LD; orígenes ajenos en Server Actions | Vitest |
| E2E | Login → crear → guardar borrador → publicar → ver en el sitio | Playwright, contra un preview |

El service container usa Postgres 17 con las mismas extensiones (`unaccent`, `pg_trgm`). Los tests de integración corren las migraciones reales, no un esquema paralelo.

Un test que no puede fallar no prueba nada: los casos de papelera y restauración verifican el estado final y las filas de `article_revisions`, no solo que la acción devuelva `ok`.

---

## 3. Hoja de ruta

### Fase 0 — Spike de verificación (2–3 días)

No es opcional: hay tres supuestos que pueden invalidar el diseño.

- [ ] Next **16.4.5 fijado** (sin `^`) con `cacheComponents: true`, `pg` + `attachDatabasePool` y una ruta de reseña prerenderizada leyendo Drizzle.
- [ ] Desplegar a Vercel y medir: `x-vercel-cache` en visitas repetidas y consumo de CU-horas en Neon tras 24 horas.
- [ ] Verificar `updateTag` desde una Server Action: publicar y ver el cambio en la primera visita.
- [ ] Medir `scrypt` con N=2^17 en la función: latencia y memoria. Si no entra, bajar N con criterio documentado, no en silencio.
- [ ] Subir una imagen real de 2 MB de punta a punta contra un store de Blob real.
- [ ] Confirmar que no hay factura y que Blob funciona dentro de los límites de Hobby.
- [ ] `pg_dump` con cliente 17 contra la URL directa y restaurar en un contenedor.
- [ ] Presupuesto de conexiones: empezar con `max: 1–3` por instancia y medir; `max: 5` no es un valor universal.

**Listo cuando**: hay números reales de consumo y de invalidación. Si la lectura pública despierta Neon, se corrige antes de escribir el panel.

### Fase 1 — Cimientos (1 semana)

- [ ] Next 16 + TypeScript estricto + Tailwind v4 + `cacheComponents`.
- [ ] Tokens morados, Quicksand, layout público.
- [ ] `lib/env.ts` con Zod; falla al arrancar si falta algo.
- [ ] Esquema completo y primera migración.
- [ ] CI con eslint, typecheck, test y build.
- [ ] Despliegue y `/api/health`.

### Fase 2 — Autenticación (1 semana)

- [ ] `login`, `logout`, `logoutAll` como Server Actions.
- [ ] `sessions` con hash, expiración deslizante y tope absoluto.
- [ ] `login_attempts` con backoff por IP.
- [ ] `proxy.ts` + verificación en `lib/auth.ts`.
- [ ] Página de login y aviso de sesión vencida.

### Fase 3 — Contenido y lectura pública (2 semanas)

- [ ] `visibleArticle`, `getArticle` con `"use cache"`, `cacheTag`, `cacheLife('max')`.
- [ ] Rutas `/resenas`, `/noticias`, `/guias`, `/juegos`, `[slug]`.
- [ ] Markdown sanitizado + JSON-LD escapado.
- [ ] Sitemap con `lastmod`, RSS general y por colección, redirects on-demand.
- [ ] `robots.ts`, `noindex` en `/admin`, `not-found`/`error`/`global-error`.
- [ ] **Backup y restauración probada.**

### Fase 4 — Panel (2 semanas)

- [ ] Server Actions de contenido con `version` y snapshots **en la misma transacción**.
- [ ] Layout a pantalla completa, borrador/publicar/despublicar.
- [ ] Autoguardado validado con Zod al restaurar, sin guardar credenciales ni tokens.
- [ ] `affectedTags()` y `updateTag` después del `COMMIT`.
- [ ] Programación por `published_at` + endpoint de revalidación idempotente.

### Fase 5 — Imágenes (1 semana)

- [ ] `/api/upload` con `sharp`, variantes y `media`.
- [ ] Inserción en el cuerpo desde el editor.
- [ ] Vista previa con el mismo pipeline.
- [ ] Loader propio y `srcset`.

### Fase 6 — Papelera y revisiones (1 semana)

- [ ] Trash/restore por `id`, purga con confirmación.
- [ ] Historial de revisiones y deshacer.
- [ ] Redirecciones por cambio de slug con el trigger de ciclos.

### Fase 7 — Pulido (1 semana)

- [ ] Búsqueda con `unaccent` y `websearch_to_tsquery`.
- [ ] Etiquetas, relacionados, breadcrumbs.
- [ ] CSP final, revisión de cabeceras.
- [ ] Tests unitarios, de integración, de seguridad y E2E en verde.
- [ ] Atribución de imágenes y aviso legal/privacidad si se añade analítica.
- [ ] Analítica sin cookies, monitoreo, runbook de restauración.

### Métricas

- [ ] Lectura pública sin consultas a Neon en régimen normal.
- [ ] Consumo de CU-horas por debajo del 40 % del límite.
- [ ] Publicar se ve en la primera visita.
- [ ] Restauración desde backup verificada en el mes.
- [ ] Ninguna mutación sin sesión ni con origen ajeno.
- [ ] Cero dependencias de CMS externo.
