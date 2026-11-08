# Reconciliación de Blob y Cursor del Cron — Netti

## 1. Contrato de almacenamiento (`lib/storage.ts`)

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
  list(args: {
    prefix: string;
    cursor?: string;
    limit?: number;
  }): Promise<{
    objects: Array<{ key: string; bytes: number; uploadedAt: Date }>;
    nextCursor: string | null;
  }>;
}
```

(`@vercel/blob` expone `list` con `prefix` y `cursor`; confirmar en Fase 0 si `list` cuenta como operación avanzada.)

- `put` usa claves exactas (`addRandomSuffix: false`) para que el `key` sea estable y el `url` predecible.
- `del` es idempotente; si falla alguna variante, se registra en `storage_orphans` para reconciliación posterior.
- Credenciales: OIDC en Vercel (`VERCEL_OIDC_TOKEN` + `BLOB_STORE_ID`), `BLOB_READ_WRITE_TOKEN` en desarrollo local y CI.

## 2. Metadatos de imágenes — segundo paso

El upload solo recibe el archivo y devuelve los datos de la imagen. La UI exige `alt` y permite crédito y procedencia. Después de subir, el autor completa los metadatos en un segundo paso:

```ts
export const makeMediaMetadataSchema = () =>
  z.object({
    mediaId: z.number().int().positive(),
    alt: z.string().trim().min(1).max(200),
    credit: z.string().trim().max(120).nullable().optional(),
    rights: z.enum(["own", "press", "third_party"]),
    sourceUrl: z.string().max(500).regex(/^https?:\/\//i).nullable().optional(),
  }).superRefine((v, ctx) => {
    if (v.rights !== "own" && !v.credit?.trim())
      ctx.addIssue({ code: "custom", path: ["credit"], message: "Indicá el crédito." });
  });

export async function updateMediaMetadata(args: {
  mediaId: number;
  alt: string;
  credit?: string | null;
  rights: "own" | "press" | "third_party";
  sourceUrl?: string | null;
}): Promise<ActionResult<{ mediaId: number }>>;
```

La acción rechaza con `conflict` si `pending_delete_at` no es nulo. Después invalida los artículos que usan el medio: `cover_media_id = $1 OR og_media_id = $1 OR body ~ ('media:' || $1 || '([^0-9]|$)')`, más juegos con esa portada. Sin esto, el `alt` y el crédito quedan viejos en caché.

Faltan además `listMedia` (la UI habla de "biblioteca") y `deleteMedia` (solo si no está referenciado; marca `pending_delete_at`). No generes variantes por encima del ancho de la fuente (una imagen de 800 px no necesita variantes de 960/1600); eso baja las operaciones por subida.

## 3. Reconciliación de Blob

Los documentos dicen que un job compara `media.storage_key` con el store y reporta huérfanos. Pero no definen cómo obtiene todas las claves de Blob, cómo distingue un objeto recién subido de un objeto abandonado, cómo utiliza `storage_orphans`, qué ocurre cuando la base tiene una fila y Blob ya no tiene el objeto, o cuándo se considera reparado un registro.

**Algoritmo `reconcileStorage()`**, un endpoint protegido con `CRON_SECRET`, acotado en tiempo y reanudable:

1. **Sweep de marcados**: `media` con `pending_delete_at < now() - 24 h`. Re-verificar referencias con regex con borde (`media:1` no debe matchear `media:12`). Si hay referencia, limpiar la marca. Si no, `storage.del(clave + variantes)`; y solo si Blob confirma, borrar las filas. Si falla, registrar en `storage_orphans (action='del', pending)` y dejar la marca.
2. **Huérfanos en el store**: `list` de `articles/` y `games/` contra `media.storage_key ∪ media_variants.storage_key`. Borrar lo que no está en DB **y** tenga `uploadedAt < now() - 1 h` (para no pisar subidas en curso); registrarlo como `repaired`.
3. **Faltantes** (en DB, no en el store): `storage_orphans` con `status='orphaned'` y alerta. No borrar filas solas.
4. **Reintentos** de `storage_orphans` pendientes: los de `put` se resuelven si la clave ya está en `media`.
5. **Subidas abandonadas**: `media` de más de 7 días sin referencias → marcar `pending_delete_at`.
6. `last_success_at` solo si 1–5 terminaron sin excepción. Correrlo dentro del workflow del backup nocturno (misma despertada de Neon).

## 3. Cursor del cron (`POST /api/revalidate`)

```
0. Bearer con timingSafeEqual → 401
1. t0  = select now()                          -- reloj de la base
2. cur = cron_runs.last_success_at ?? t0 - 24 h -- primera ejecución: ventana conservadora
3. update cron_runs set last_run_at = t0
4. due = articles where status='published' and published_at > cur and published_at <= t0
         order by published_at limit 200        -- usa articles_pending_activation_idx
5. por fila: affectedTags(null, estado(fila)) → revalidateTag(tag, { expire: 0 })
6. cursor = (due.length == 200) ? published_at de la última : t0
7. update cron_runs set last_success_at = cursor   -- solo acá, nunca antes
8. return { revalidated, cursor }
```

- **No hace falta lock.** Dos ejecuciones solapadas revalidan dos veces; es idempotente. No usar `pg_try_advisory_lock` de sesión: el pooler de Neon (modo transacción) lo rompe; solo `xact` es seguro.
- **`expire: 0`, no `"max"`.** Con `"max"` el primer visitante tras la hora todavía ve el 404 de antes (stale-while-revalidate).
- Una entrada despublicada o a papelera entre medio queda fuera por el filtro de `status`.