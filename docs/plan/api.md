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

---

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
      suggestedSlug?: string;
    };
```

Los componentes leen `result.ok` y muestran `message` o `fieldErrors`. No hay códigos HTTP que interpretar, salvo en los dos Route Handlers.

---

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

La cookie se emite con `cookies().set()` dentro de la acción: `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, `Max-Age` de 8 horas.

### `logout()`

Borra la fila de `sessions` y la cookie. Idempotente.

### `logoutAll()`

`TRUNCATE sessions`. Cierra todas las sesiones, incluida la actual.

---

## 4. Contenido

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
}): Promise<ActionResult<{ version: number; slug: string; status: ContentStatus }>>;
```

Ejecuta `UPDATE ... WHERE id = $id AND version = $version RETURNING version`. Si no devuelve fila, hace un `SELECT` para distinguir:

| Situación | `code` | Extra |
|-----------|--------|-------|
| No existe | `not_found` | — |
| Versión desactualizada | `conflict` | `current` con el estado actual |
| Slug ocupado | `conflict` | `fieldErrors.slug` |

Antes del `UPDATE` inserta un snapshot en `article_revisions` con la versión anterior.

**Editar una entrada publicada**: guardar cambios **no la despublica**. Mantiene `status = 'published'` y revalida. Para sacarla del aire hay una acción explícita, `unpublishArticle`.

### `unpublishArticle({ id, version })`

`published` → `draft`. Conserva `published_at` para no perder la fecha original. Revalida.

### `trashArticle({ id, version })`

Guarda `previous_status` y `trashed_at`, pone `status = 'trashed'`. Libera el slug (índice parcial). No borra imágenes.

### `restoreArticle({ id })`

Vuelve a `previous_status` y limpia `trashed_at`. Si el slug está tomado, el `UPDATE` falla con `23505` y devuelve:

```json
{ "ok": false, "code": "conflict", "message": "El slug ya está en uso.", "suggestedSlug": "mi-articulo-2" }
```

El panel acepta el sufijo y reintenta. **No hay pre-chequeo**: sería una condición de carrera.

### `purgeArticle({ id, confirmSlug })`

Borrado definitivo. Exige escribir el slug para confirmar. Elimina revisión, filas de `media` no referenciadas y sus objetos en R2. Las imágenes referenciadas por otro artículo no se tocan.

### `searchGames({ q, limit })`

Lectura para el autocompletado. Usa `pg_trgm`, no `tsvector` (los prefijos de dos letras no funcionan con full-text).

---

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
3. `Content-Length <= 2 MB` → `413` antes de leer el cuerpo.
4. MIME en la allowlist → `400`.
5. Magic numbers → `400`.
6. Dimensiones decodificadas y megapíxeles ≤ 20 → `400`.
7. `sharp`: re-encodeo a WebP, lado mayor ≤ 2560 px, sin EXIF.
8. Variantes 480 / 960 / 1600 px.

Respuesta:

```json
{
  "ok": true,
  "media": {
    "id": 12,
    "url": "https://img.netti.example/articles/8f3c1a2e.webp",
    "width": 1600,
    "height": 900,
    "bytes": 184320,
    "variants": [
      { "width": 480, "url": "https://img.netti.example/articles/8f3c1a2e-480.webp" },
      { "width": 960, "url": "https://img.netti.example/articles/8f3c1a2e-960.webp" },
      { "width": 1600, "url": "https://img.netti.example/articles/8f3c1a2e-1600.webp" }
    ]
  }
}
```

| Código | Cuerpo |
|--------|--------|
| 400 | `{ "error": "Formato no admitido. Usá JPEG, PNG, WebP, GIF o AVIF." }` |
| 400 | `{ "error": "El archivo no coincide con su tipo declarado." }` |
| 400 | `{ "error": "La imagen supera el tamaño de píxeles permitido." }` |
| 413 | `{ "error": "La imagen no puede superar los 2 MB." }` |
| 500 | `{ "error": "No se pudo guardar la imagen." }` |

La clave del objeto es `articles/<uuid>.webp` o `games/<uuid>.webp`: nunca el nombre original ni el slug. Se guarda con `Cache-Control: public, max-age=31536000, immutable`.

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

### 5.3 `GET /api/health`

Responde `200` con `{ "ok": true }` **sin tocar Postgres**. Sirve para monitoreo de uptime; un health que consulta la base la despierta y consume CU-horas.

---

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
Content-Security-Policy: default-src 'self'; img-src 'self' https://img.netti.example; ...
```

En `/admin/*` la CSP usa nonces y esas rutas se renderizan dinámicamente.
