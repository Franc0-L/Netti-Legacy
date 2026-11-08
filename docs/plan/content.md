# Modelo de Contenido y Validación — Netti

Define el esquema PostgreSQL (Neon), las restricciones que viven en la base y los esquemas Zod que validan la entrada.

**Principio**: la base impone lo que puede imponer. Zod da mensajes al editor; la base es la última línea. Un borrador es válido con casi todo vacío; una publicación, no.

---

## 1. Tablas

| Tabla | Función |
|-------|---------|
| `games` | Catálogo de juegos (entidad propia, referenciada por las reseñas) |
| `articles` | Reseñas, noticias y guías. Una tabla con `kind`, no tres |
| `media` | Imágenes subidas, con dimensiones y variantes |
| `article_revisions` | Snapshots por guardado, para deshacer |
| `sessions` | Sesiones del panel |
| `login_attempts` | Rate limiting del login |
| `redirects` | Redirecciones por cambio de slug |

### Por qué una sola tabla `articles`

Cuatro tablas separadas obligan a repetir en cada una: el predicado de visibilidad, el índice de búsqueda, la papelera, el sitemap, el RSS y la validación de slug. Basta olvidar `published_at <= now()` en una consulta para filtrar borradores.

Con `articles(kind)` hay **un** lugar donde se define qué es visible, **un** índice full-text y **un** namespace de slugs por `kind` (`/resenas/x` y `/guias/x` pueden coexistir). La integridad referencial se conserva con `game_id` nullable y un `CHECK` que lo exige solo cuando `kind = 'review'`.

`games` queda aparte porque es una entidad de catálogo, no un artículo, y la comparten varias reseñas.

No hay tabla `users`: un solo autor, contraseña como hash en el entorno.

---

## 2. Esquema SQL

### 2.0 Tipos y funciones

```sql
create type article_kind as enum ('review', 'news', 'guide');
create type content_status as enum ('draft', 'published', 'trashed');

-- No existe 'scheduled': programar es publicar con published_at futuro.

create or replace function set_updated_at()
returns trigger
language plpgsql
as $$
begin
  new.updated_at := now();
  new.version := old.version + 1;
  return new;
end;
$$;

create or replace function validate_tags()
returns trigger
language plpgsql
as $$
begin
  if exists (
    select 1 from unnest(new.tags) as t
    where char_length(btrim(t)) = 0 or char_length(t) > 30
  ) then
    raise exception 'Cada etiqueta debe tener entre 1 y 30 caracteres.'
      using errcode = '23514';
  end if;
  if cardinality(new.tags) > 5 then
    raise exception 'Máximo 5 etiquetas.' using errcode = '23514';
  end if;
  return new;
end;
$$;

create or replace function validate_redirect()
returns trigger
language plpgsql
as $$
begin
  -- Evita recursión: el UPDATE de abajo vuelve a disparar este trigger
  if pg_trigger_depth() > 1 then
    return new;
  end if;

  if new.from_path = new.to_path then
    raise exception 'La redirección apunta a sí misma.' using errcode = '23514';
  end if;
  -- Evita el ciclo directo A→B / B→A
  delete from redirects
   where from_path = new.to_path and to_path = new.from_path;
  -- Colapsa cadenas: si X→A existía y ahora A→B, X pasa a apuntar a B
  update redirects set to_path = new.to_path where to_path = new.from_path;
  return new;
end;
$$;
```

El trigger de redirecciones se ejecuta `BEFORE INSERT OR UPDATE` y limpia ciclos y cadenas. Sin él, renombrar A→B y luego B→A deja un bucle infinito que el navegador corta y el SEO castiga.

### 2.1 `games`

```sql
create table games (
  id               integer generated always as identity primary key,
  slug             text not null unique
                     check (slug ~ '^[a-z0-9]+(?:-[a-z0-9]+)*$'),
  title            text not null check (char_length(title) between 3 and 120),
  description      text not null default '' check (char_length(description) <= 240),
  body             text not null default '',
  developer        text,
  publisher        text,
  release_date     date,
  platforms        text[] not null default '{}',
  genres           text[] not null default '{}',
  external_source  text check (external_source in ('rawg', 'igdb', 'manual')),
  external_id      text,
  status           content_status not null default 'draft',
  cover_image_url  text,
  cover_alt        text,
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  version          integer not null default 1,
  constraint games_cover_alt_required
    check (cover_image_url is null or char_length(btrim(coalesce(cover_alt, ''))) > 0),
  constraint games_published_requires_cover
    check (status <> 'published' or cover_image_url is not null)
);

create unique index games_external_idx
  on games (external_source, external_id)
  where external_id is not null;

create index games_tags_idx on games using gin (platforms, genres);

create trigger games_set_updated_at
  before update on games
  for each row
  when (old.* is distinct from new.*)
  execute function set_updated_at();
```

`release_date` es `date` completa, no solo año: las noticias suelen anunciar lanzamientos futuros y el año solo no alcanza.

### 2.2 `articles`

```sql
create table articles (
  id               integer generated always as identity primary key,
  kind             article_kind not null,
  slug             text not null default '',
  title            text not null default '',
  description      text not null default '',
  body             text not null default '',
  published_at     timestamptz,
  author           text,
  tags             text[] not null default '{}',
  cover_image_url  text,
  cover_alt        text,
  status           content_status not null default 'draft',
  previous_status  content_status,
  trashed_at       timestamptz,
  featured         boolean not null default false,
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  version          integer not null default 1,

  -- Reseñas
  game_id          integer references games (id) on delete restrict,
  score            smallint check (score between 1 and 10),
  score_reason     text,
  verdict          text,

  -- Noticias
  category         text not null default 'Noticias',
  source_name      text,
  source_url       text,

  -- SEO
  seo_title        text,
  seo_description  text,
  og_image_url     text,

  -- Formato
  constraint articles_title_len
    check (char_length(title) <= 120),
  constraint articles_description_len
    check (char_length(description) <= 240),
  constraint articles_seo_description_len
    check (seo_description is null or char_length(seo_description) <= 160),
  constraint articles_slug_format
    check (slug = '' or slug ~ '^[a-z0-9]+(?:-[a-z0-9]+)*$'),
  constraint articles_source_url
    check (source_url is null or source_url ~* '^https?://'),
  constraint articles_cover_alt_required
    check (cover_image_url is null or char_length(btrim(coalesce(cover_alt, ''))) > 0),

  -- Estados
  constraint articles_published_requires_date
    check (status <> 'published' or published_at is not null),
  constraint articles_trashed_requires_date
    check (status <> 'trashed' or trashed_at is not null),

  -- Publicación completa
  constraint articles_publish_title
    check (status <> 'published' or char_length(btrim(title)) between 3 and 120),
  constraint articles_publish_slug
    check (status <> 'published' or slug <> ''),
  constraint articles_publish_body
    check (status <> 'published' or char_length(btrim(body)) >= 10),

  -- Campos por tipo
  constraint articles_review_fields_only
    check (kind = 'review' or (game_id is null and score is null
           and score_reason is null and verdict is null)),
  constraint articles_review_complete
    check (kind <> 'review' or status <> 'published' or (
      game_id is not null
      and score between 1 and 10
      and char_length(btrim(coalesce(score_reason, ''))) >= 20
      and char_length(btrim(coalesce(verdict, ''))) >= 10
    )),
  constraint articles_news_fields_only
    check (kind = 'news' or (source_name is null and source_url is null))
);

create unique index articles_kind_slug_active
  on articles (kind, slug)
  where status <> 'trashed' and slug <> '';

create index articles_visible_idx
  on articles (kind, published_at desc)
  where status = 'published';

create index articles_featured_idx
  on articles (published_at desc)
  where featured and status = 'published';

create index articles_tags_idx on articles using gin (tags);

create trigger articles_validate_tags
  before insert or update on articles
  for each row execute function validate_tags();

create trigger articles_set_updated_at
  before update on articles
  for each row
  when (old.* is distinct from new.*)
  execute function set_updated_at();
```

**Qué permite un borrador**: `title` vacío, `slug` vacío, `body` vacío, sin portada, sin puntuación, sin fecha. Lo único obligatorio es `kind` y `status`. Todo lo demás se exige recién al publicar, mediante los `CHECK` con `status <> 'published' or ...`.

**`previous_status` y `trashed_at`** existen porque "restaurar devuelve el estado anterior" necesita dónde guardarlo. Sin esas columnas, restaurar solo puede adivinar.

**`WHEN (OLD.* IS DISTINCT FROM NEW.*)`** evita que un `UPDATE` que no cambia nada (un backfill, un toque de mantenimiento) incremente `version` y provoque un `409` falso en el formulario abierto.

### 2.3 Búsqueda full-text

```sql
create text search configuration es_unaccent (copy = spanish);

alter text search configuration es_unaccent
  alter mapping for hword, hword_part, word
  with unaccent, spanish_stem;

alter table articles add column search_vector tsvector
  generated always as (
    setweight(to_tsvector('es_unaccent'::regconfig, coalesce(title, '')), 'A') ||
    setweight(to_tsvector('es_unaccent'::regconfig, coalesce(description, '')), 'B') ||
    setweight(to_tsvector('es_unaccent'::regconfig, coalesce(body, '')), 'C')
  ) stored;

create index articles_search_idx on articles using gin (search_vector);
```

- **`unaccent`**: sin él, "resena" no encuentra "reseña", y en LATAM se busca sin tildes.
- **`setweight`**: el título pesa más que el cuerpo, así los resultados relevantes suben.
- **`::regconfig` literal**: `to_tsvector(text, text)` es `STABLE` y no se puede usar en una columna generada; la versión con `regconfig` sí es `IMMUTABLE`.
- Consulta: `websearch_to_tsquery('es_unaccent', $1)`, que tolera frases entre comillas y `OR` sin romper con entrada arbitraria.
- Autocompletado de juegos: `pg_trgm` con `gin (title gin_trgm_ops)` y `similarity()`. `tsvector` no sirve para prefijos de dos letras.

### 2.4 `media`

```sql
create table media (
  id            integer generated always as identity primary key,
  storage_key   text not null unique,
  public_url    text not null,
  width         integer not null check (width > 0),
  height        integer not null check (height > 0),
  bytes         integer not null check (bytes > 0),
  content_type  text not null check (content_type in
                  ('image/jpeg','image/png','image/webp','image/gif','image/avif')),
  alt           text,
  credit        text,
  article_id    integer references articles (id) on delete set null,
  game_id       integer references games (id) on delete set null,
  created_at    timestamptz not null default now(),
  constraint media_single_owner
    check (num_nonnulls(article_id, game_id) <= 1)
);

create table media_variants (
  media_id   integer not null references media (id) on delete cascade,
  width      integer not null,
  format     text not null check (format in ('webp','avif')),
  storage_key text not null unique,
  bytes      integer not null,
  primary key (media_id, width, format)
);
```

Sin `media`, la purga de la papelera no sabe qué imágenes borrar y las subidas huérfanas se acumulan. `credit` cubre el requisito legal de atribuir capturas de prensa.

### 2.5 `article_revisions`

```sql
create table article_revisions (
  id          bigint generated always as identity primary key,
  article_id  integer not null references articles (id) on delete cascade,
  version     integer not null,
  snapshot    jsonb not null,
  created_at  timestamptz not null default now(),
  unique (article_id, version)
);

create index article_revisions_created_idx
  on article_revisions (article_id, created_at desc);
```

Se guarda un snapshot por cada escritura desde el panel. Se conservan las últimas 20 por artículo; una purga periódica borra el resto.

### 2.6 `sessions`

```sql
create table sessions (
  id            text primary key,          -- SHA-256 hex del token, nunca el token
  created_at    timestamptz not null default now(),
  expires_at    timestamptz not null,
  last_used_at  timestamptz not null default now(),
  absolute_expires_at timestamptz not null,
  user_agent    text check (char_length(user_agent) <= 300),
  ip_hash       text
);

create index sessions_expires_idx on sessions (expires_at);
```

`absolute_expires_at` (30 días) impide que la renovación deslizante mantenga una sesión viva para siempre. `last_used_at` se actualiza como máximo una vez cada 5 minutos, no en cada request.

### 2.7 `login_attempts`

```sql
create table login_attempts (
  id           bigint generated always as identity primary key,
  ip_hash      text not null,
  attempted_at timestamptz not null default now(),
  success      boolean not null default false
);

create index login_attempts_ip_idx
  on login_attempts (ip_hash, attempted_at desc);
```

Se consulta antes de verificar la contraseña. Purga semanal de filas con más de 7 días.

### 2.8 `redirects`

```sql
create table redirects (
  id           integer generated always as identity primary key,
  from_path    text not null unique,
  to_path      text not null,
  status_code  smallint not null default 301 check (status_code in (301, 302, 308)),
  created_at   timestamptz not null default now(),
  constraint redirects_absolute_paths
    check (from_path ~ '^/[^/]' and to_path ~ '^/[^/]'),
  constraint redirects_not_self
    check (from_path <> to_path)
);

create trigger redirects_validate
  before insert or update on redirects
  for each row execute function validate_redirect();
```

Se consulta **solo cuando el artículo no existe**, con el mapa cacheado bajo el tag `redirects`. No se consulta en cada request ni en el proxy.

---

## 3. Validación Zod

Los esquemas viven en `lib/validation.ts`. **No se importan directamente en el cliente**: se construyen con una fábrica que recibe la URL pública de imágenes, porque `process.env` no existe en el navegador.

```ts
// lib/validation.ts
export function imageUrlSchema(imagesBaseUrl: string) {
  return z
    .string()
    .url("La portada debe ser una URL válida.")
    .refine((value) => new URL(value).origin === new URL(imagesBaseUrl).origin, {
      message: "La portada debe estar alojada en el almacenamiento de Netti.",
    })
    .refine((value) => !/\.svg$/i.test(new URL(value).pathname), {
      message: "No se admiten SVG.",
    })
    .nullable()
    .optional();
}
```

Comparar `origin` y no `startsWith` evita que `https://img.netti.example.evil.com/x.png` pase la validación.

### 3.1 Esquema de artículo

Todo lo que sigue vive dentro de `makeArticleSchema(imagesBaseUrl)`, que recibe la URL del almacenamiento y construye `cover` una sola vez. El servidor la llama con `env.IMAGES_PUBLIC_BASE_URL`; el cliente, con el valor que expone el layout. Así no hay ningún `process.env` en código compartido.

```ts
const cover = imageUrlSchema(imagesBaseUrl);

const baseArticle = z.object({
  kind: z.enum(["review", "news", "guide"]),
  slug: z
    .string()
    .regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/, "Solo minúsculas, números y guiones.")
    .or(z.literal("")),
  title: z.string().trim().max(120).default(""),
  description: z.string().trim().max(240).default(""),
  body: z.string().default(""),
  author: z.string().trim().max(60).nullable().optional(),
  tags: z.array(z.string().trim().min(1).max(30)).max(5).default([]),
  coverImageUrl: cover,
  coverAlt: z.string().trim().max(200).nullable().optional(),
  status: z.enum(["draft", "published", "trashed"]).default("draft"),
  publishedAt: z.coerce.date().nullable().optional(),
  featured: z.boolean().default(false),
  seoTitle: z.string().trim().max(70).nullable().optional(),
  seoDescription: z.string().trim().max(160).nullable().optional(),
  ogImageUrl: cover,
});
```

Las variantes por `kind` se derivan de `baseArticle` y el resultado se devuelve desde la fábrica. Los miembros de `discriminatedUnion` van **sin refinar**; las reglas condicionales se aplican después con `superRefine`, porque un `discriminatedUnion` no acepta miembros refinados:

```ts
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
    if (value.coverImageUrl && !value.coverAlt?.trim())
      ctx.addIssue({ code: "custom", path: ["coverAlt"], message: "La portada necesita texto alternativo." });
    if (!value.publishedAt)
      ctx.addIssue({ code: "custom", path: ["publishedAt"], message: "Una publicación necesita fecha." });

    if (value.kind === "review") {
      if (!value.gameId)
        ctx.addIssue({ code: "custom", path: ["gameId"], message: "Elegí un juego." });
      if (value.score == null)
        ctx.addIssue({ code: "custom", path: ["score"], message: "Falta la puntuación." });
      if (value.scoreReason.trim().length < 20)
        ctx.addIssue({ code: "custom", path: ["scoreReason"], message: "Justificá la puntuación (mínimo 20)." });
      if (value.verdict.trim().length < 10)
        ctx.addIssue({ code: "custom", path: ["verdict"], message: "El veredicto necesita al menos 10 caracteres." });
    }
  });
```

**Versión de Zod**: se fija Zod 4 en `package.json`. Si se usara Zod 3, el `superRefine` externo sigue siendo la forma correcta; no cambiar la estructura al migrar.

### 3.2 Subida

```ts
export const UPLOAD_MAX_BYTES = 2 * 1024 * 1024;
export const MAX_MEGAPIXELS = 20;
export const ALLOWED = ["image/jpeg","image/png","image/webp","image/gif","image/avif"] as const;
```

El servidor valida, en este orden: tamaño declarado, MIME, magic numbers, dimensiones decodificadas y megapíxeles. Después re-encodea con `sharp` (elimina EXIF y polyglots) y genera las variantes.

---

## 4. Estados y transiciones

| Desde | Acción | Hacia | Notas |
|-------|--------|-------|-------|
| — | Crear | `draft` | Todo vacío salvo `kind` |
| `draft` | Publicar | `published` | Valida completitud; si `published_at` es futuro, queda programado |
| `draft` | Programar | `published` | Igual, con `published_at` futuro |
| `published` | Guardar cambios | `published` | Revalida; no despublica |
| `published` | Despublicar | `draft` | `published_at` se conserva |
| `draft` / `published` | Borrar | `trashed` | Guarda `previous_status` y `trashed_at` |
| `trashed` | Restaurar | `previous_status` | Si el slug está tomado, se propone sufijo |
| `trashed` | Purgar | — | Borra fila, revisiones y medios propios |

**No existe `scheduled`.** Un artículo `published` con `published_at` futuro no es visible porque la consulta pública exige `published_at <= now()`. Eso elimina un valor del enum, un `CHECK` y la dependencia de un cron que en Vercel Hobby solo corre una vez al día.

Lo que sí necesita un cron es **invalidar la caché** cuando llega la hora. Se resuelve con una GitHub Action cada 30 minutos que llama a un endpoint protegido y revalida lo que cruzó `published_at`. Si el cron falla, el artículo se ve igual en la siguiente revalidación; el peor caso es un retraso, no una fuga.

### Predicado único de visibilidad

```ts
export const visibleArticle = and(
  eq(articles.status, "published"),
  lte(articles.publishedAt, new Date()),
);
```

Toda consulta pública usa este helper. No se escribe la condición a mano en ningún lado.

---

## 5. Imágenes

- Almacenamiento: R2, servido por un dominio propio.
- Clave: `articles/<uuid>.<ext>` o `games/<uuid>.<ext>`. **Nunca** el nombre del archivo del usuario ni el slug, que cambian.
- Al subir: `sharp` re-encodea a WebP, limita a 2560 px de lado mayor, elimina EXIF (incluido GPS) y genera variantes de 480, 960 y 1600 px.
- `Cache-Control: public, max-age=31536000, immutable` en cada `PutObject`.
- Se inserta una fila en `media` y una por variante en `media_variants`.
- El render usa un `loader` propio que elige la variante según `srcset`. No se delega a `next/image` por defecto: la cuota de transformaciones de Hobby es de 5.000 al mes y cada combinación de ancho, formato y calidad cuenta.
- SVG rechazado. AVIF acepta las marcas `avif`, `avis` y `mif1` con `avif` compatible.
- Tope de 2 MB en la subida, pero el cliente **redimensiona antes** a WebP: una captura PNG de 1080p supera los 2 MB con facilidad.
- Borrar un artículo no borra sus imágenes (papelera). La purga definitiva sí, y solo las que no estén referenciadas en otro artículo.

---

## 6. Integridad y concurrencia

- **Edición**: `UPDATE ... WHERE id = $1 AND version = $2 RETURNING version`. Si no devuelve fila, se hace un `SELECT` para distinguir `404` de `409`. Nunca read-modify-write sin condición.
- **Slug**: el índice parcial `(kind, slug) WHERE status <> 'trashed' AND slug <> ''` garantiza unicidad aunque dos peticiones lleguen juntas. No se hace pre-chequeo.
- **Restaurar con slug ocupado**: se intenta el `UPDATE` y se captura `23505` para proponer un sufijo. Un pre-chequeo tendría condición de carrera.
- **Cambio de slug**: misma transacción, `UPDATE` del artículo + `INSERT` en `redirects`. El trigger colapsa cadenas y rompe ciclos.
- **Juego con reseñas**: `ON DELETE RESTRICT` cubre el borrado físico. El soft delete de un juego con reseñas publicadas se bloquea en la capa de servicio: la base no puede expresar "no lo marques trashed si otro tiene una fila publicada que lo referencia" sin un trigger, y un trigger ahí es más frágil que la comprobación explícita.
- **Publicar sin fecha**: el servidor completa `published_at = now()` antes del `INSERT`/`UPDATE`. Así el `CHECK` y Zod nunca se contradicen.
