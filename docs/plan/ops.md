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
| Imágenes | Cloudflare R2 Free | $0 de uso; puede pedir tarjeta para activar |
| Dominio de imágenes | Dominio propio en Cloudflare | **~USD 10/año** |
| Hosting | Vercel Hobby | $0, solo uso no comercial |
| CI | GitHub Actions | $0 en repo público |
| Monitoreo | Servicio de uptime + tracker de errores | $0 en planes gratuitos |

**El "costo cero" no es absoluto.** Dos excepciones hay que aceptarlas o cambiar de proveedor:

1. El dominio propio para servir R2. `r2.dev` es gratis, está limitado y no es para producción.
2. El método de pago que R2 puede exigir para activarse. Cloudflare no ofrece tope de gasto duro.

Si ninguna de las dos es aceptable, la alternativa es **Vercel Blob**: no requiere cuenta extra ni tarjeta, pero factura egress y ata el proyecto a Vercel. El adaptador `lib/storage.ts` permite cambiar sin tocar el resto.

**Vercel Hobby es no comercial.** Publicidad o afiliados obligan a Pro o a mover el hosting a Cloudflare Pages.

### 2.2 Variables

| Variable | Requerida | Descripción |
|----------|-----------|-------------|
| `DATABASE_URL` | Sí | Cadena **pooled** de Neon, para la aplicación |
| `DATABASE_URL_UNPOOLED` | Sí | Cadena directa, para migraciones y `pg_dump` |
| `ADMIN_PASSWORD_HASH` | Sí | `scrypt:16384:8:1:<saltB64>:<hashB64>` |
| `IP_HASH_SECRET` | Sí | Secreto del HMAC que hashea IPs |
| `PUBLIC_SITE_URL` | Sí | Origen de producción |
| `PREVIEW_ORIGIN_REGEX` | No | Orígenes permitidos en previews |
| `IMAGES_PUBLIC_BASE_URL` | Sí | Dominio público de imágenes |
| `CRON_SECRET` | Sí | Bearer del endpoint de revalidación |
| `R2_ACCOUNT_ID` | Sí | Cuenta de Cloudflare |
| `R2_ACCESS_KEY_ID` | Sí | Token con alcance solo al bucket de imágenes |
| `R2_SECRET_ACCESS_KEY` | Sí | Ídem |
| `R2_BUCKET` | Sí | Bucket de imágenes |
| `BACKUP_R2_*` | Sí | Token **distinto**, solo al bucket de backups |

**El separador del hash es `:`, no `$`.** Next expande `$VAR` en los archivos `.env`; un hash con `$` queda corrupto y el login falla sin explicación.

**Previews**: `PUBLIC_SITE_URL` es fijo, así que el panel en un preview rechazaría el login con `403`. La verificación de origen compara contra `PUBLIC_SITE_URL` **más** `PREVIEW_ORIGIN_REGEX` (o `VERCEL_BRANCH_URL`).

**Dos URLs de base**: el pooler no sirve para migraciones ni para `pg_dump`, y `CREATE INDEX CONCURRENTLY` falla a través de él.

### 2.3 `.env.local`

```env
DATABASE_URL=postgresql://...@...-pooler.sa-east-1.aws.neon.tech/netti?sslmode=require
DATABASE_URL_UNPOOLED=postgresql://...@...sa-east-1.aws.neon.tech/netti?sslmode=require

ADMIN_PASSWORD_HASH=scrypt:16384:8:1:
IP_HASH_SECRET=
CRON_SECRET=

PUBLIC_SITE_URL=http://localhost:3000
IMAGES_PUBLIC_BASE_URL=https://img.netti.example

R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=netti-images
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

`scryptSync` se evita a propósito: bloquea el event loop y consume CPU activa.

### 2.4 Migraciones

```bash
npm run db:generate   # SQL desde db/schema.ts
npm run db:migrate    # aplica contra DATABASE_URL_UNPOOLED
npm run db:studio
```

Se revisan en el PR y se ejecutan **manualmente**, nunca en el build de Vercel.

### 2.5 Backup

Neon Free no incluye backups automáticos. La ventana de restauración de ~6 horas complementa, no reemplaza.

Workflow nocturno:

1. Instalar `postgresql-client-17` desde PGDG (el runner no lo trae y `pg_dump` falla si el cliente es más viejo que el servidor).
2. `pg_dump --format=custom --no-owner --exclude-table-data=sessions --exclude-table-data=login_attempts` contra `DATABASE_URL_UNPOOLED`.
3. Cifrar con `age`.
4. Subir a un bucket de backups con un token distinto y alcance mínimo.
5. Retención de 30 días.

**Heartbeat**: si el workflow no corre, nadie se entera. Un chequeo externo (healthchecks.io, gratis) avisa cuando el dump falta. Los workflows programados de GitHub se desactivan tras 60 días sin actividad en repos públicos.

**Restauración probada mensualmente**: un job levanta Postgres 17 en un service container, restaura el dump y verifica conteos de filas. Un backup que nunca se restauró no es un backup.

### 2.6 Revalidación programada

GitHub Action cada 30 minutos:

```bash
curl -fsS -X POST "$SITE/api/revalidate" -H "Authorization: Bearer $CRON_SECRET"
```

Vercel Hobby no sirve para esto: sus cron corren como máximo una vez por día y son dos por proyecto. El endpoint revalida solo los artículos cuyo `published_at` cruzó la hora.

### 2.7 Despliegue

1. Importar el repo en Vercel; framework Next.js; sin output directory manual.
2. Proyecto Neon de producción y proyecto separado de previews.
3. Bucket R2 + dominio propio; CORS no hace falta (la subida pasa por el servidor).
4. Cargar variables en Production, Preview y Development.
5. `npm run db:migrate` contra producción, una vez.
6. Verificar: `/`, `/api/health`, login, subida, publicación, programación, backup y restauración.

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

- **Uptime**: ping a una página pública cacheada o a `/api/health` (que no toca la base).
- **Errores**: Sentry u otro tracker. Los logs de Vercel Hobby tienen retención corta y no sirven como historial.
- **CU-horas de Neon**: revisión mensual en la consola. Es el primer límite que se agota.
- **Degradación**: si Neon suspende el cómputo por cuota, el sitio cacheado sigue sirviendo y el panel deja de funcionar. El monitor tiene que avisar antes.

---

## 3. Hoja de ruta

### Fase 0 — Spike de verificación (2–3 días)

No es opcional: hay tres supuestos que pueden invalidar el diseño.

- [ ] Next 16.4 con `cacheComponents: true`, `pg` + `attachDatabasePool` y una ruta de reseña prerenderizada leyendo Drizzle.
- [ ] Desplegar a Vercel y medir: `x-vercel-cache` en visitas repetidas y consumo de CU-horas en Neon tras 24 horas.
- [ ] Verificar `updateTag` desde una Server Action: publicar y ver el cambio en la primera visita.
- [ ] Subir una imagen real de 2 MB de punta a punta.
- [ ] `pg_dump` con cliente 17 contra la URL directa y restaurar en un contenedor.

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
- [ ] Sitemap, RSS, redirects on-demand.
- [ ] **Backup y restauración probada.**

### Fase 4 — Panel (2 semanas)

- [ ] Server Actions de contenido con `version` y snapshots.
- [ ] Layout a pantalla completa, borrador/publicar/despublicar.
- [ ] Autoguardado validado con Zod al restaurar.
- [ ] `affectedTags()` y `updateTag`.
- [ ] Programación por `published_at` + endpoint de revalidación.

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
- [ ] Analítica sin cookies, monitoreo, runbook de restauración.

### Métricas

- [ ] Lectura pública sin consultas a Neon en régimen normal.
- [ ] Consumo de CU-horas por debajo del 40 % del límite.
- [ ] Publicar se ve en la primera visita.
- [ ] Restauración desde backup verificada en el mes.
- [ ] Ninguna mutación sin sesión ni con origen ajeno.
- [ ] Cero dependencias de CMS externo.
