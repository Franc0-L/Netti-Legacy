# Modelo de Contenido y Validación — Netti

Define el esquema PostgreSQL (Neon), las restricciones que viven en la base y las tablas auxiliares.

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
| `storage_orphans` | Registro de compensación entre Blob y Postgres |
| `cron_runs` | Cursor del job de revalidación programada |

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
declare
  cycle_found boolean;
begin
  -- Serializa las modificaciones del grafo de redirecciones para evitar ciclos
  -- creados por transacciones concurrentes (ej. A→B y B→A al mismo tiempo).
  perform pg_advisory_xact_lock(hashtext('redirects_graph'));

  -- Evita recursión: el UPDATE de abajo vuelve a disparar este trigger
  if pg_trigger_depth() > 1 then
    return new;
  end if;

  if new.from_path = new.to_path then
    raise exception 'La redirección apunta a sí misma.' using errcode = '23514';
  end if;

  -- Rutas internas absolutas: empiezan por '/' seguido de un carácter no '/'.
  -- Rechaza '//' (open redirect), barras invertidas, caracteres de control,
  -- query strings y fragmentos. El contrato es solo de paths internos.
  if new.from_path !~ '^/[^/][^ ]*$'
     or new.to_path !~ '^/[^/][^ ]*$'
     or new.from_path like '%//%'
     or new.to_path like '%//%'
     or position(chr(92) in new.from_path) > 0
     or position(chr(92) in new.to_path) > 0
     or new.from_path like '%?%'
     or new.to_path like '%?%'
     or new.from_path like '%#%'
     or new.to_path like '%#%'
     or new.from_path ~ '[[:cntrl:]]'
     or new.to_path ~ '[[:cntrl:]]' then
    raise exception 'Las rutas de redirección deben ser internas absolutas.' using errcode = '23514';
  end if;

  -- Detección de ciclos mediante CTE recursiva: seguir la cadena desde new.to_path
  -- hasta encontrar new.from_path. Si se llega, hay un ciclo de cualquier longitud.
  -- Usa UNION (no UNION ALL) para garantizar terminación incluso con ciclos preexistentes.
  select exists (
    with recursive chain(to_path) as (
      select to_path from redirects where from_path = new.to_path
      union
      select r.to_path from chain c join redirects r on r.from_path = c.to_path
    )
    select 1 from chain where to_path = new.from_path
  ) into cycle_found;

  if cycle_found then
    raise exception 'La redirección crearía un ciclo.' using errcode = '23514';
  end if;

  -- Colapsa cadenas: si X→A existía y ahora A→B, X pasa a apuntar a B.
  -- El filtro evita crear un auto-redirect al colapsar sobre la fila que se está insertando.
  -- El ciclo directo A→B / B→A ya fue rechazado por el chequeo de ciclos arriba,
  -- así que no hace falta un DELETE separado de la fila inversa.
  update redirects
     set to_path = new.to_path
   where to_path = new.from_path
     and from_path <> new.to_path;
  return new;
end;
$$;
```

El trigger de redirecciones se ejecuta `BEFORE INSERT OR UPDATE` y colapsa cadenas y rechaza ciclos. Sin él, renombrar A→B y luego B→A deja un bucle infinito que el navegador corta y el SEO castiga.

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
  cover_image_url  text,        -- URL pública derivada de cover_media_id; caché derivado
  cover_alt        text,
  cover_media_id   integer,     -- FK agregada con ALTER TABLE tras crear media
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  version          integer not null default 1,
  constraint games_cover_alt_required
    check (status <> 'published' or (cover_image_url is null and cover_media_id is null)
           or char_length(btrim(coalesce(cover_alt, ''))) > 0),
  constraint games_published_requires_cover
    check (status <> 'published' or cover_image_url is not null),
  constraint games_not_trashed
    check (status <> 'trashed')
);

create unique index games_external_idx
  on games (external_source, external_id)
  where external_id is not null;

create index games_tags_idx on games using gin (platforms, genres);

create trigger games_set_updated_at
  before update on games
  for each row
  when (
    to_jsonb(old) - 'updated_at' - 'version'
    is distinct from
    to_jsonb(new) - 'updated_at' - 'version'
  )
  execute function set_updated_at();
```

`release_date` es `date` completa, no solo año: las noticias suelen anunciar lanzamientos futuros y el año solo no alcanza.

`games` no tiene `previous_status` ni `trashed_at` en la v1: se borra físicamente (ya protegido por `ON DELETE RESTRICT` si tiene reseñas publicadas).

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
  cover_image_url  text,        -- URL pública derivada de cover_media_id; caché derivado
  cover_alt        text,
  cover_media_id   integer,     -- FK agregada con ALTER TABLE tras crear media
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
  category         text,
  source_name      text,
  source_url       text,

  -- SEO
  seo_title        text,
  seo_description  text,
  og_image_url     text,        -- URL pública derivada de og_media_id; caché derivado
  og_media_id      integer,     -- FK agregada con ALTER TABLE tras crear media

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
    check (status <> 'published' or (cover_image_url is null and cover_media_id is null)
           or char_length(btrim(coalesce(cover_alt, ''))) > 0),

  -- Estados
  constraint articles_published_requires_date
    check (status <> 'published' or published_at is not null),
  constraint articles_trashed_requires_date
    check (status <> 'trashed' or trashed_at is not null),
  constraint articles_trashed_requires_previous
    check (status <> 'trashed' or previous_status is not null),

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
      and score is not null
      and score between 1 and 10
      and char_length(btrim(coalesce(score_reason, ''))) >= 20
      and char_length(btrim(coalesce(verdict, ''))) >= 10
    )),
  constraint articles_news_fields_only
    check (kind = 'news' or (category is null and source_name is null and source_url is null)),
  constraint articles_news_category_len
    check (category is null or char_length(category) <= 40)
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

-- Índice para consultar publicaciones por fecha. Incluye todas las publicadas
-- (pasadas y futuras); el job de revalidación filtra las que ya cruzaron la hora.
create index articles_pending_activation_idx
  on articles (published_at)
  where status = 'published';

create trigger articles_validate_tags
  before insert or update on articles
  for each row execute function validate_tags();

create trigger articles_set_updated_at
  before update on articles
  for each row
  when (
    to_jsonb(old) - 'updated_at' - 'version'
    is distinct from
    to_jsonb(new) - 'updated_at' - 'version'
  )
  execute function set_updated_at();
```

**Qué permite un borrador**: `title` vacío, `slug` vacío, `body` vacío, sin portada, sin puntuación, sin fecha. Lo único obligatorio es `kind` y `status`. Todo lo demás se exige recién al publicar, mediante los `CHECK` con `status <> 'published' or ...`.

**`previous_status` y `trashed_at`** existen porque "restaurar devuelve el estado anterior" necesita dónde guardarlo. Sin esas columnas, restaurar solo puede adivinar.

**`WHEN` sobre `to_jsonb(old)` menos `updated_at` y `version`** evita dos cosas: que un `UPDATE` que no cambia nada incremente `version`, y que el propio trigger, al escribir `updated_at`, se considere un cambio. Comparar `OLD.*` a secas incluiría esos dos campos de mantenimiento; excluirlos por nombre en JSONB no obliga a reescribir el trigger cuando se agrega una columna.

### 2.3 Búsqueda full-text

```sql
create extension if not exists unaccent;
create extension if not exists pg_trgm;

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

```sql
create index games_title_trgm_idx on games using gin (title gin_trgm_ops);
```

### 2.4 `media`

```sql
create table media (
  id            integer generated always as identity primary key,
  storage_key   text not null unique,
  public_url    text not null,
  width         integer not null check (width > 0),
  height        integer not null check (height > 0),
  bytes         integer not null check (bytes > 0),
  content_type  text not null default 'image/webp'
                  check (content_type = 'image/webp'),
  alt           text,
  credit        text,
  rights        text not null default 'own'
                  check (rights in ('own', 'press', 'third_party')),
  source_url    text,
  -- Procedencia: dónde se subió. No implica propiedad exclusiva: el cuerpo
  -- puede referenciar el mismo medio desde otro artículo.
  article_id    integer references articles (id) on delete set null,
  game_id       integer references games (id) on delete set null,
  pending_delete_at timestamptz,
  created_at    timestamptz not null default now(),
  constraint media_single_owner
    check (num_nonnulls(article_id, game_id) <= 1),
  constraint media_rights_require_credit
    check (rights = 'own' or char_length(btrim(coalesce(credit, ''))) > 0),
  constraint media_source_url_valid
    check (source_url is null or source_url ~* '^https?://')
);

create index media_pending_delete_idx
  on media (pending_delete_at)
  where pending_delete_at is not null;
```

Sin `media`, la purga de la papelera no sabe qué imágenes borrar y las subidas huérfanas se acumulan. `credit`, `rights` y `source_url` documentan procedencia: una carátula disponible públicamente no es una carátula libre de derechos.

`article_id` registra **dónde se subió**, no una propiedad exclusiva. Antes de purgar un medio, se comprueba que ningún otro artículo lo referencie: `cover_image_url`, `og_image_url` y las apariciones de `media:<id>` en el cuerpo. Es una consulta de texto sobre `articles.body`, acotada a los medios candidatos, no un escaneo de toda la tabla.

**Borrado consistente.** Postgres y Blob no comparten transacción, así que el orden importa:

1. En una transacción: recolectar los ids de medios que solo referencia este artículo, borrar la fila del artículo (la FK `ON DELETE SET NULL` libera `media.article_id` y `articles.cover_media_id`/`og_media_id`), y recién entonces marcar `pending_delete_at` en los medios huérfanos (las revisiones caen por `ON DELETE CASCADE`).
2. Fuera de la transacción: borrar en Blob el objeto y sus variantes.
3. Solo si Blob confirma: borrar las filas de `media` y `media_variants`.

Si Blob falla, la fila queda marcada y se reintenta. Nunca al revés: borrar el objeto antes de confirmar la base deja referencias rotas que no se pueden reparar. Las imágenes referenciadas por otro artículo no se marcan.

**Imágenes dentro del cuerpo.** No se pegan URLs. El editor inserta `![texto alternativo](media:12)` y un plugin de remark resuelve el id contra `media`/`media_variants`, genera `srcset` y dimensiones. Así la purga sabe qué medios referencia el cuerpo, se evita CLS y una URL mal copiada no rompe la imagen.

**Borrado de medios con FKs circulares.** `games` y `articles` declaran FK hacia `media` (`cover_media_id`, `og_media_id`), y `media` declara FK hacia `games` y `articles` (`article_id`, `game_id`). El orden de creación es:

1. `games` y `articles` (sin FK a `media`).
2. `media` (con FK a `games` y `articles`).
3. `ALTER TABLE` para agregar las FK `cover_media_id` y `og_media_id` en `games` y `articles`.

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
    check (from_path ~ '^/[^/][^ ]*$' and to_path ~ '^/[^/][^ ]*$'),
  constraint redirects_not_self
    check (from_path <> to_path)
);

create trigger redirects_validate
  before insert or update on redirects
  for each row execute function validate_redirect();
```

Se consulta **solo cuando el artículo no existe**, con el mapa cacheado bajo el tag `redirects`. No se consulta en cada request ni en el proxy.

### 2.9 `storage_orphans`

```sql
create table storage_orphans (
  id          bigint generated always as identity primary key,
  storage_key text not null,
  action      text not null check (action in ('put', 'del')),
  status      text not null default 'pending' check (status in ('pending', 'repaired', 'orphaned')),
  created_at  timestamptz not null default now(),
  repaired_at timestamptz
);

create index storage_orphans_pending_idx
  on storage_orphans (status)
  where status = 'pending';
```

Registro de compensación para subidas/borrados fallidos entre Blob y Postgres. Si falla el `INSERT` en `media` tras subir objetos, o falla un `del` de Blob, se anota aquí para que un job de baja frecuencia repare o reporte el desajuste.

### 2.10 `cron_runs`

```sql
create table cron_runs (
  job_name         text not null primary key,
  last_success_at  timestamptz,
  last_run_at      timestamptz not null default now()
);

insert into cron_runs (job_name) values ('revalidate_scheduled') on conflict do nothing;
```

Cursor de ejecución del job de revalidación programada. Permite recuperar publicaciones pendientes si una ejecución se retrasa o falla.