# Contrato de Acciones y Endpoints — Netti

El panel no necesita una API REST: tiene un solo cliente escrito por la misma persona. Las mutaciones son **Server Actions**; los Route Handlers quedan solo donde hace falta HTTP real (subida de archivos y cron).

## 1. Por qué Server Actions

| Necesidad | Server Action | Route Handler |
|-----------|---------------|---------------|
| Lectura inmediata tras escribir | `updateTag` | `revalidateTag(tag, { expire: 0 })` |
| Verificación de origen | Resuelta contra el host | Manual |
| Tipado de punta a punta | Nativo | Duplicado |
| Subida de archivos de 2 MB | Límite de body chico | Sin problema |

Con Server Actions desaparece el contrato de 300 líneas que antes describía `/api/content`. El panel importa la acción y recibe un resultado tipado.

**Todas las acciones de escritura** empiezan con `requireSession()` y `assertSameOrigin()`. Ninguna confía en el proxy.

## 2. Resultado uniforme

```ts
export type ActionResult<T = undefined> =
  | { ok: true; data: T }
  | {
      ok: false;
      code:
        | "unauthorized"
        | "forbidden"
        | "invalid"
        | "conflict"
        | "not_found"
        | "rate_limited"
        | "too_large"
        | "server";
      message: string;
      fieldErrors?: Record<string, string[]>;
      current?: unknown;
      retryAfter?: number;
    };
```

Los componentes leen `result.ok` y muestran `message` o `fieldErrors`. No hay códigos HTTP que interpretar, salvo en los Route Handlers.

## 3. Autenticación

### `login({ password })`

```ts
"use server";
export async function login(input: { password: string }): Promise<ActionResult<{ expiresAt: string }>>;
```

Orden de ejecución: `assertSameOrigin()` → rate limit por IP → `scrypt` asíncrono → `timingSafeEqual` → crear sesión → setear cookie.

| Resultado | `code` | `message` |
|-----------|--------|-----------|
| Cookie emitida | — (`ok: true`) | — |
| Contraseña incorrecta | `unauthorized` | "La contraseña no es correcta." |
| Demasiados intentos | `rate_limited` | "Demasiados intentos. Probá en unos minutos." con `retryAfter` |
| Origen ajeno | `forbidden` | "Origen no permitido." |
| Falta configuración | `server` | "El panel no está configurado." |

La cookie se emite dentro de la acción con `cookies()` (async en Next 16):

```ts
const cookieStore = await cookies();
cookieStore.set("session", token, {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/",
  maxAge: 8 * 60 * 60,
});
```

`HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, `Max-Age` de 8 horas.

### `logout()`

Borra la fila de `sessions` y la cookie. Idempotente.

### `logoutAll()`

`TRUNCATE sessions`. Cierra todas las sesiones, incluida la actual.

## 4. Contenido

### 4.0 Una escritura, una transacción

Actualizar el artículo e insertar su revisión son **una sola transacción**. Si se hacen por separado, una desconexión entre ambas deja el artículo guardado sin snapshot o un snapshot de algo que nunca se guardó. La invalidación de caché va **después del `COMMIT`**: invalidar antes de confirmar publica un estado que puede revertirse.

```ts
await db.transaction(async (tx) => {
  const [row] = await tx
    .update(articles)
    .set(values)
    .where(and(eq(articles.id, id), eq(articles.version, version)))
    .returning({ version: articles.version, slug: articles.slug, status: articles.status });

  if (!row) throw new StaleVersion();

  await tx.insert(articleRevisions).values({
    articleId: id,
    version: row.version,
    snapshot: row, // fila completa de RETURNING *, sin search_vector
  });
});
```

El trigger `BEFORE UPDATE` incrementa `version` y fija `updated_at`. `RETURNING` ve los valores **después** del trigger, así que devuelve la versión nueva sin un `SELECT` extra y sin incrementar dos veces. Si la fila no cambió (misma versión devuelta), no se inserta una revisión duplicada. Si `rowCount = 0`, se consulta la fila para distinguir `not_found` de `conflict`.

### `createArticle(input)`

```ts
async function createArticle(input: ArticleInput): Promise<ActionResult<{ id: number; version: number; slug: string }>>;
```

- `status: "draft"` → guarda lo que haya. Un borrador puede estar vacío salvo `kind`.
- `status: "published"` → valida completitud; si falta `publishedAt`, el servidor pone `now()`.

| `code` | Cuándo |
|--------|--------|
| `invalid` | Zod falló; `fieldErrors` trae el detalle por campo |
| `conflict` | El slug ya existe para ese `kind` |
| `server` | Error de base |

### `updateArticle({ id, version, input })`

```ts
async function updateArticle(args: {
  id: number;
  version: number;
  input: ArticleInput;
}): Promise<ActionResult<{ version: number; slug: string; status: ContentStatus; oldSlug?: string }>>;
```

Ejecuta la transacción de §4.0: `UPDATE ... WHERE id = $id AND version = $version RETURNING version` + `INSERT` en `article_revisions`. Si no devuelve fila, hace un `SELECT` para distinguir:

| Situación | `code` | Extra |
|-----------|--------|-------|
| No existe | `not_found` | — |
| Versión desactualizada | `conflict` | `current` con el estado actual |
| Slug ocupado | `conflict` | `fieldErrors.slug` |

**Cambio de slug**: si `input.slug` difiere del slug actual, dentro de la misma transacción se inserta una redirección. Las rutas públicas usan prefijos por colección (definidos en `lib/routes.ts`):

```ts
await tx
  .deleteFrom(redirects)
  .where(eq(redirects.from_path, articlePath(kind, input.slug)));

await tx.insert(redirects).values({
  from_path: articlePath(kind, oldSlug),
  to_path: articlePath(kind, input.slug),
  status_code: 308,
});
```

La función `articlePath(kind, slug)` vive en `lib/routes.ts`, no en `app/actions/`, porque las funciones `"use server"` solo pueden exportar funciones async.

El código de estado es siempre `308`, nunca `301`, para que el navegador preserve el método y el cuerpo si es necesario. Si el `INSERT` en `redirects` falla (porque el destino ya existe), se captura y se trata como conflicto de slug.

**Cambio de estado**: `updateArticle` no permite cambiar `status` ni `kind`. Si el `input.status` o `input.kind` difieren del actual, devuelve `conflict`. Para publicar usa `publishArticle`; para despublicar usa `unpublishArticle`; el `kind` se define al crear y no se modifica después.

**Editar una entrada publicada**: guardar cambios **no la despublica**. Mantiene `status = 'published'` y revalida.

### `publishArticle({ id, version })`

`draft` → `published`. Valida completitud del contenido; si falta `publishedAt`, el servidor pone `now()`. Si `published_at` es futuro, queda programado.

```ts
async function publishArticle(args: {
  id: number;
  version: number;
}): Promise<ActionResult<{ version: number; slug: string; status: "published" }>>;
```

Invalidación: `article:<kind>:<slug>`, `kind:<kind>`, `articles`, `home`.

### `unpublishArticle({ id, version })`

`published` → `draft`. Conserva `published_at` para no perder la fecha original. Revalida.

```ts
async function unpublishArticle(args: {
  id: number;
  version: number;
}): Promise<ActionResult<{ version: number; status: "draft" }>>;
```

Invalidación: `article:<kind>:<slug>`, `kind:<kind>`, `articles`, `home`.

### `trashArticle({ id, version })`

`published` o `draft` → `trashed`. Guarda `previous_status` y `trashed_at`, pone `status = 'trashed'`. Libera el slug (índice parcial). No borra imágenes.

```ts
async function trashArticle(args: {
  id: number;
  version: number;
}): Promise<ActionResult<{ version: number; status: "trashed" }>>;
```

Invalidación: `article:<kind>:<slug>`, `kind:<kind>`, `articles`, `home` si estaba publicado o destacado, `tag:<t>` para cada etiqueta previa.

### `restoreArticle({ id, version })`

Vuelve a `previous_status` y limpia `trashed_at`. **No hay pre-chequeo**: comprobar si el slug está libre antes del `UPDATE` es una condición de carrera. Se intenta el `UPDATE` y se captura la violación de unicidad.

```ts
async function restoreArticle(args: {
  id: number;
  version: number;
}): Promise<ActionResult<{ version: number; status: ContentStatus; slug: string }>>;
```

Dos detalles que un `catch (23505)` genérico no resuelve:

1. **Identificar la restricción.** Solo se trata como conflicto de slug si el error corresponde a `articles_kind_slug_active`. Un `23505` de otra restricción es un error real y no debe convertirse en "elegí otro slug".
2. **Reintentar hasta encontrar un sufijo libre.** Se prueba primero el slug original. Si está ocupado, se prueban `-2`, `-3`, … hasta un límite (10); si ninguno sirve, se devuelve `conflict` sin sugerencia.

Cada intento debe ejecutarse en una **transacción independiente** o usar un `SAVEPOINT` con rollback ante el conflicto. Si una violación de unicidad ocurre dentro de una transacción PostgreSQL, esa transacción queda abortada y no se puede continuar con otro `UPDATE` dentro de ella.

```ts
// Intento 1: slug original
try {
  return await restoreWithSlug(id, slug); // UPDATE ... RETURNING
} catch (error) {
  if (!isUniqueViolation(error, "articles_kind_slug_active")) throw error;
}

// Intento 2 a 10: sufijos -2 hasta -10
for (let attempt = 2; attempt <= 10; attempt++) {
  const candidate = `${slug}-${attempt}`;
  try {
    return await restoreWithSlug(id, candidate); // UPDATE ... RETURNING en transacción nueva o savepoint
  } catch (error) {
    if (!isUniqueViolation(error, "articles_kind_slug_active")) throw error;
  }
}

return { ok: false, code: "conflict", message: "No se pudo liberar el slug." };
```

Cada reintento es un `UPDATE`; el que gana es el que la base acepta. El resultado exitoso incluye el slug definitivo para que el panel lo muestre.

**No crear redirección desde el slug original.** Si el slug original ya lo ocupa otro artículo (porque se creó tras el borrado), no se crea ninguna redirección. Crearla apropiaría la URL del artículo nuevo. El panel muestra el slug definitivo y el autor decide si actualiza enlaces internos manualmente.

Invalidación: `article:<kind>:<slug-original>`, `article:<kind>:<slug-definitivo>`, `kind:<kind>`, `articles`, `home` si el estado restaurado es publicado o destacado.

### `purgeArticle({ id, version, confirmSlug })`

Borrado definitivo. Exige escribir el slug para confirmar.

```ts
async function purgeArticle(args: {
  id: number;
  version: number;
  confirmSlug: string;
}): Promise<ActionResult<{ version: number }>>;
```

**Orden mark-and-sweep** (Postgres y Blob no comparten transacción):

1. En una transacción: recolectar los ids de medios que solo referencia este artículo, borrar la fila del artículo (la FK `ON DELETE SET NULL` libera `media.article_id` y `articles.cover_media_id`/`og_media_id`), y recién entonces marcar `pending_delete_at` en los medios huérfanos (las revisiones caen por `ON DELETE CASCADE`).
2. Fuera de la transacción: borrar en Blob el objeto y sus variantes.
3. Solo si Blob confirma: borrar las filas de `media` y `media_variants`.

Si el paso 2 falla, la fila queda marcada y un barrido de baja frecuencia reintenta. Nunca se borra el objeto antes de confirmar la base: eso deja referencias rotas que no se pueden reparar. Las imágenes referenciadas por otro artículo no se marcan.

Invalidación: `article:<kind>:<slug>`, `kind:<kind>`, `articles`, `home` si estaba publicado o destacado, `tag:<t>` para cada etiqueta previa, `redirects` si el artículo tenía redirecciones activas.

### `searchGames({ q, limit })`

Lectura para el autocompletado. Usa `pg_trgm`, no `tsvector` (los prefijos de dos letras no funcionan con full-text).

## 5. Route Handlers

### 5.1 `POST /api/upload`

Es el único lugar donde el navegador manda bytes.

```http
POST /api/upload
Content-Type: multipart/form-data

file=<binario>
kind=review            # opcional
```

Validaciones, en orden:

1. Sesión válida → `401`.
2. Origen permitido → `403`.
3. `Content-Length <= 2 MB + margen` → `413` antes de leer el cuerpo. Es una optimización, **no la defensa**: el header puede faltar o mentir.
4. Leer el `File` y comprobar el **tamaño real** ≤ 2 MB y que el cuerpo total no supere el presupuesto (2 MB + 64 KB de formulario).
5. MIME declarado en la allowlist → `400`.
6. Magic numbers del contenido real → `400`.
7. Dimensiones decodificadas y megapíxeles ≤ 20 → `400`.
8. GIF (estático o animado) → `400`. La salida es WebP estático y `sharp` perdería los fotogramas o el formato original en silencio; se rechaza con un mensaje explícito en vez de degradar la imagen.
9. `sharp`: re-encodeo a WebP, lado mayor ≤ 2560 px, sin EXIF.
10. Variantes 480 / 960 / 1600 px.

La respuesta:

```json
{
  "ok": true,
  "media": {
    "id": 12,
    "url": "https://ce0rcu23vrrdzqap.public.blob.vercel-storage.com/articles/8f3c1a2e.webp",
    "width": 1600,
    "height": 900,
    "bytes": 184320,
    "variants": [
      { "width": 480, "url": "https://ce0rcu23vrrdzqap.public.blob.vercel-storage.com/articles/8f3c1a2e-480.webp" },
      { "width": 960, "url": "https://ce0rcu23vrrdzqap.public.blob.vercel-storage.com/articles/8f3c1a2e-960.webp" },
      { "width": 1600, "url": "https://ce0rcu23vrrdzqap.public.blob.vercel-storage.com/articles/8f3c1a2e-1600.webp" }
    ]
  }
}
```

| Código | Cuerpo |
|--------|--------|
| 400 | `{ "error": "Formato no admitido. Usá JPEG, PNG, WebP o AVIF." }` |
| 400 | `{ "error": "El archivo no coincide con su tipo declarado." }` |
| 400 | `{ "error": "Las imágenes animadas no están soportadas." }` |
| 400 | `{ "error": "La imagen supera el tamaño de píxeles permitido." }` |
| 413 | `{ "error": "La imagen no puede superar los 2 MB." }` |
| 500 | `{ "error": "No se pudo guardar la imagen." }` |

La subida pasa por `@vercel/blob` (`put`) desde el servidor. El token se resuelve con OIDC en Vercel; en desarrollo local se usa `BLOB_READ_WRITE_TOKEN`.

La clave del objeto es `articles/<uuid>.webp` o `games/<uuid>.webp`: nunca el nombre original ni el slug. Se guarda con `cacheControlMaxAge: 31536000` (caché larga; el contenido nunca cambia porque la clave lleva UUID).

**Compensación de subida.** Si falla el `INSERT` en `media` después de subir a Blob, se intenta borrar los objetos de Blob y se registra la clave en un mecanismo de reconciliación para limpieza posterior. No hay transacción distribuida.

### 5.2 `POST /api/revalidate`

Lo llama la GitHub Action cada 30 minutos. No lo usa el panel.

```http
POST /api/revalidate
Authorization: Bearer $CRON_SECRET
```

```json
{ "revalidated": ["article:review:mi-resena", "kind:review", "home"] }
```

Si `CRON_SECRET` falta o no coincide: `401`. Sin este endpoint, un artículo programado se ve tarde o no se ve.

### 5.3 `GET /api/health` y `GET /api/ready`

Son dos comprobaciones distintas y no intercambiables.

| Endpoint | Qué prueba | Toca Postgres | Frecuencia |
|----------|-----------|---------------|------------|
| `/api/health` | *Liveness*: el proceso responde | No | Cada 1–5 min (uptime monitor) |
| `/api/ready` | *Readiness*: DB y almacenamiento alcanzables | Sí (`select 1`) | Cada 30–60 min, o solo desde el runbook |

`/api/health` devuelve `200` con `{ "ok": true }` y `Cache-Control: no-store`. Un health que consulta la base la despierta y consume CU-horas: eso es exactamente lo que este diseño evita.

`/api/ready` no se expone al monitor público. Requiere `Authorization: Bearer $CRON_SECRET` y se llama a mano cuando algo falla, o desde un job de baja frecuencia que además registra el resultado. Devuelve `503` con el componente que falló:

```json
{ "ok": false, "db": "unreachable", "storage": "ok" }
```

## 6. Cabeceras

En `/api/*`:

```
Cache-Control: no-store
X-Content-Type-Options: nosniff
```

En el resto del sitio, desde `next.config.ts`:

```
Strict-Transport-Security: max-age=63072000; includeSubDomains
Referrer-Policy: strict-origin-when-cross-origin
X-Frame-Options: DENY
Permissions-Policy: camera=(), microphone=(), geolocation=()
Content-Security-Policy: default-src 'self'; img-src 'self' https://<IMAGES_PUBLIC_BASE_URL_HOST>; ...
```

En `/admin/*` la CSP usa nonces y esas rutas se renderizan dinámicamente.