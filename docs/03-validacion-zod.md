# Validación con Zod — Netti

Los esquemas viven en `lib/validation.ts`. **No se importan directamente en el cliente**: se construyen con fábricas sin parámetros, porque las URLs de imágenes se resuelven en el servidor desde `media`, no se reciben desde el cliente.

## 1. `imageUrlSchema` — eliminado

No existe `imageUrlSchema` como función pública. Las URLs de imagen no entran desde el cliente: solo `coverMediaId` y `ogMediaId` (IDs numéricos de `media`) son aceptados. El servidor resuelve las URLs públicas desde la tabla `media`.

## 2. Esquema de artículo (`makeArticleSchema`)

Todo lo que sigue vive dentro de `makeArticleSchema()`, que construye el esquema de validación de entradas del cliente. No recibe parámetros.

```ts
export const makeArticleSchema = () => {
  // Campos comunes a las tres colecciones. Un borrador puede dejarlos vacíos.
  const baseArticle = z.object({
    slug: z
      .string()
      .regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/, "Solo minúsculas, números y guiones.")
      .or(z.literal("")),
    title: z.string().trim().max(120).default(""),
    description: z.string().trim().max(240).default(""),
    body: z.string().default(""),
    author: z.string().trim().max(60).nullable().optional(),
    tags: z.array(z.string().trim().min(1).max(30)).max(5).default([]),
    coverMediaId: z.number().int().positive().nullable().optional(),
    coverAlt: z.string().trim().max(200).nullable().optional(),
    status: z.enum(["draft", "published"]).default("draft"),
    publishedAt: z
      .iso
      .datetime({ offset: true })
      .nullable()
      .optional()
      .transform((val) => (val ? new Date(val) : null)),
    featured: z.boolean().default(false),
    seoTitle: z.string().trim().max(70).nullable().optional(),
    seoDescription: z.string().trim().max(160).nullable().optional(),
    ogMediaId: z.number().int().positive().nullable().optional(),
  });

  const reviewShape = baseArticle.extend({
    kind: z.literal("review"),
    gameId: z.number().int().positive().nullable().optional(),
    score: z.number().int().min(1).max(10).nullable().optional(),
    scoreReason: z.string().trim().max(2000).nullable().optional(),
    verdict: z.string().trim().max(500).nullable().optional(),
  });

  const newsShape = baseArticle.extend({
    kind: z.literal("news"),
    category: z.string().trim().max(40).nullable().optional(),
    sourceName: z.string().trim().max(120).nullable().optional(),
    sourceUrl: z
      .string()
      .regex(/^https?:\/\//i, "La fuente debe empezar por http:// o https://.")
      .nullable()
      .optional(),
  });

  const guideShape = baseArticle.extend({ kind: z.literal("guide") });

  const articleSchema = z
    .discriminatedUnion("kind", [reviewShape, newsShape, guideShape])
    .superRefine((value, ctx) => {
      const publishing = value.status === "published";
      if (!publishing) return;

      if (value.title.trim().length < 3)
        ctx.addIssue({ code: "custom", path: ["title"], message: "El título necesita al menos 3 caracteres." });
      if (!value.slug)
        ctx.addIssue({ code: "custom", path: ["slug"], message: "Una publicación necesita slug." });
      if (value.body.trim().length < 10)
        ctx.addIssue({ code: "custom", path: ["body"], message: "El cuerpo necesita al menos 10 caracteres." });
      if (value.coverMediaId && !value.coverAlt?.trim())
        ctx.addIssue({ code: "custom", path: ["coverAlt"], message: "La portada necesita texto alternativo." });
      if (!value.publishedAt)
        ctx.addIssue({ code: "custom", path: ["publishedAt"], message: "Una publicación necesita fecha." });

      if (value.kind === "review") {
        if (!value.gameId)
          ctx.addIssue({ code: "custom", path: ["gameId"], message: "Elegí un juego." });
        if (value.score == null)
          ctx.addIssue({ code: "custom", path: ["score"], message: "Falta la puntuación." });
        if (!value.scoreReason || value.scoreReason.trim().length < 20)
          ctx.addIssue({ code: "custom", path: ["scoreReason"], message: "Justificá la puntuación (mínimo 20)." });
        if (!value.verdict || value.verdict.trim().length < 10)
          ctx.addIssue({ code: "custom", path: ["verdict"], message: "El veredicto necesita al menos 10 caracteres." });
      }
    });

  return articleSchema;
};

export type ArticleInput = z.input<ReturnType<typeof makeArticleSchema>>;
export type ParsedArticle = z.output<ReturnType<typeof makeArticleSchema>>;
```

`coverMediaId` y `ogMediaId` son los únicos campos de entrada para imágenes; el servidor resuelve las URLs públicas desde `media`. `coverImageUrl` y `ogImageUrl` en la base son caché derivado, no entrada del cliente.

El `kind` es un literal por variante, así que `discriminatedUnion` estrecha el tipo: en `value.kind === "review"`, `gameId` y `score` existen en el tipo. Los campos de reseña son nullable para que un borrador incompleto sea válido; la exigencia aparece al publicar, en el mismo lugar que los `CHECK` de la base.

**Pendiente:** `makeGameSchema` para el CRUD de fichas de juegos (slug, portada, estado, plataformas, géneros). Se define antes de crear ese CRUD; no bloquea el MVP de reseñas.

`sourceUrl` usa la misma regla que el `CHECK` (`~*`), insensible a mayúsculas. Si Zod fuera más laxo, `HTTPS://x` pasaría la validación y fallaría en la base con un `500`.

### 3. `makeGameSchema`

La base exige `slug` y `title` desde el insert (los `CHECK` de `games` no admiten vacío), así que un juego borrador no puede estar vacío (a diferencia de `articles`).

```ts
// lib/validation.ts
const SLUG_RE = /^[a-z0-9]+(?:-[a-z0-9]+)*$/;
const label = (max: number) => z.string().trim().min(1).max(max);

export const makeGameSchema = () =>
  z
    .object({
      slug: z.string().regex(SLUG_RE, "Solo minúsculas, números y guiones."),
      title: z.string().trim().min(3).max(120),
      description: z.string().trim().max(240).default(""),
      body: z.string().default(""),
      developer: z.string().trim().max(120).nullable().optional(),
      publisher: z.string().trim().max(120).nullable().optional(),
      releaseDate: z.iso.date().nullable().optional(),
      platforms: z.array(label(30)).max(12).default([]),
      genres: z.array(label(30)).max(8).default([]),
      externalSource: z.enum(["rawg", "igdb", "manual"]).nullable().optional(),
      externalId: z.string().trim().min(1).max(64).nullable().optional(),
      status: z.enum(["draft", "published"]).default("draft"),
      coverMediaId: z.number().int().positive().nullable().optional(),
      coverAlt: z.string().trim().max(200).nullable().optional(),
    })
    .superRefine((v, ctx) => {
      if (v.coverMediaId && !v.coverAlt?.trim())
        ctx.addIssue({ code: "custom", path: ["coverAlt"], message: "La portada necesita texto alternativo." });
      if (v.status === "published" && !v.coverMediaId)
        ctx.addIssue({ code: "custom", path: ["coverMediaId"], message: "Una ficha publicada necesita portada." });
      if (!!v.externalSource !== !!v.externalId)
        ctx.addIssue({ code: "custom", path: ["externalId"], message: "Fuente e id externo van juntos." });
    });

export type GameInput = z.input<ReturnType<typeof makeGameSchema>>;
export type ParsedGame = z.output<ReturnType<typeof makeGameSchema>>;
```

### 3.1 `makeMediaMetadataSchema`

Segundo paso después de la subida de imagen. El servidor valida que `mediaId` exista y no esté marcado para purga.

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
```

### 3.2 Constantes de subida

```ts
export const UPLOAD_MAX_BYTES = 2 * 1024 * 1024;
export const MAX_MEGAPIXELS = 20;
export const ALLOWED = ["image/jpeg", "image/png", "image/webp", "image/avif"] as const;
```

El servidor valida, en este orden: tamaño declarado (como pista), tamaño real, MIME, magic numbers, dimensiones decodificadas y megapíxeles. Después re-encodea con `sharp`, que descarta EXIF (incluido GPS) y las estructuras del archivo original, y genera las variantes. `sharp` reduce la superficie de un archivo manipulado; no es una garantía absoluta frente a vulnerabilidades del decodificador, así que la dependencia se mantiene actualizada.

**GIF queda fuera.** La salida es WebP estático y `sharp` descartaría los fotogramas en silencio. Aceptar el formato para luego romper la animación es peor que rechazarlo con un mensaje claro. Si algún día se necesitan animaciones, se decide y se implementa entonces.

### 3.3 Versión de Zod

Se fija Zod 4 en `package.json`. Si se usara Zod 3, el `superRefine` externo sigue siendo la forma correcta; no cambiar la estructura al migrar.

En Zod 4, el *input* de `z.iso.datetime()` es `unknown`, así que `ArticleInput` pierde el tipo de `publishedAt`. Por eso se usa `.transform((val) => (val ? new Date(val) : null))` para devolver `Date | null`.