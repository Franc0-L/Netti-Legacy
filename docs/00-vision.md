# Documentación Maestra — Netti

Plan de reconstrucción de Netti. Es la referencia para cualquier persona o agente que trabaje en el proyecto.

## Qué hay aquí

| Archivo | Qué explica |
|---------|-------------|
| `00-vision.md` | Visión, alcance, reglas e índice |
| `01.1-stack-basico.md` | Stack, Next.js 16, Neon |
| `01.2.1-stack-caché-modelo.md` | Caché, arquitectura de lectura, contratos de lib/ |
| `01.2.2-stack-imagenes-auth.md` | Imágenes (Vercel Blob), auth, Markdown, seguridad, decisiones D-001 a D-022 |
| `02.1-modelo-tablas.md` | Esquema SQL: tipos, juegos, articles |
| `02.2.1-modelo-busqueda-media.md` | Búsqueda FTS, media, article_revisions |
| `02.2.2-modelo-sesiones-zod.md` | sessions, login_attempts, redirects, storage_orphans, cron_runs (esquemas SQL) |
| `03-validacion-zod.md` | makeArticleSchema, makeGameSchema, makeMediaMetadataSchema |
| `04.1-api-acciones-contenido.md` | Server Actions: auth, create/update/publish/trash/restore/purge |
| `04.2-api-endpoints-cabeceras.md` | Route Handlers (upload/revalidate/health/ready), cabeceras |
| `05-ui-flujo.md` | Paleta, panel, flujo editorial |
| `06.1-operaciones-entorno.md` | Git, entorno, variables, migraciones |
| `06.2-operaciones-backup-hoja.md` | Backups, despliegue, hoja de ruta |
| `07-reconciliacion-y-cron.md` | Contrato Storage, reconciliación de Blob, cursor del cron |

## Qué es Netti

Blog editorial de videojuegos en español, de un solo autor, con reseñas (puntuación 1–10), noticias, guías y fichas de juegos. Tiene un sitio público de lectura y un panel editorial propio a pantalla completa. No usa CMS externo.

## Cómo usar estos documentos

1. Para entender el contexto y el stack: `01.1-stack-basico.md`.
2. Para modelar datos: `02.1-modelo-tablas.md`.
3. Para validación: `03-validacion-zod.md`.
4. Para escribir Server Actions o endpoints: `04.1-api-acciones-contenido.md`.
5. Para construir interfaz: `05-ui-flujo.md`.
6. Para configurar el repo y desplegar: `06.1-operaciones-entorno.md`.
7. Para empezar a programar: `06.1-operaciones-entorno.md`, **Fase 0**.

## Reglas

- **La mejor herramienta gana**, aunque obligue a rehacer algo. No se defiende una decisión por haberla tomado antes.
- **Un solo repositorio.** El contenido vive en Postgres, no en archivos.
- **La lectura pública no despierta la base.** Es un **objetivo de Fase 0**, no una garantía ya probada: `use cache` no asegura persistencia compartida entre instancias serverless. Se verifica con `x-vercel-cache` y consumo de CU-h antes de considerar la arquitectura lista para producción.
- **La base impone lo que puede imponer.** `CHECK`, índices parciales y claves foráneas, además de Zod.
- **Backups obligatorios** desde la Fase 3, con restauración probada.
- **Morado** (`#a78bfa`, `#c9bbff`, `#5b21b6`) en sitio y panel.
- **Panel a pantalla completa**, vista previa como acción aparte.
- **Decisiones difíciles de revertir**: un ADR corto en el momento. El resto se decide programando; no se documenta antes de existir.

## Cómo se lee una decisión

Cada afirmación de infraestructura lleva su grado de certeza. Sin esto, una hipótesis termina tratada como propiedad confirmada.

| Estado | Significado |
|--------|-------------|
| **Decidida** | Se eligió y no se reabre sin un argumento nuevo. |
| **Validación: pendiente** | Depende de un comportamiento que todavía no se midió. Se confirma en la Fase 0. |
| **Diferida** | Se sabe que hace falta, pero no en la primera versión. |
| **Descartada** | Se evaluó y se rechazó, con el motivo escrito. |

## Decisiones y validaciones pendientes

| # | Decisión | Estado | Qué se valida en Fase 0 |
|---|----------|--------|-------------------------|
| 1 | **Vercel Blob + subida por servidor.** `lib/storage.ts` aísla `put`/`del` para poder migrar si algún día se justifica. | Decidida | Subida real de 2 MB, cuatro objetos, compensación si falla `INSERT` en media. |
| 2 | **`scrypt` con N=2^17 en Vercel.** Consume ~128 MiB por verificación. Si el spike muestra problemas de memoria o latencia, se baja N con un ADR que lo justifique. | Decidida con validación pendiente | Latencia p50/p95, memoria máxima, concurrencia de verificaciones. |
| 3 | **Cron de revalidación.** GitHub Actions hoy; Vercel Schedules cuando salga de beta con condiciones aceptables. | Decidida con validación pendiente | Endpoint idempotente, `last_success_at`, retraso acotado, medición de CU-h por despertar (Fase 0). |
| 4 | **Caché pública con `"use cache"` + `cacheLife("max")`.** | Validación: pendiente | `x-vercel-cache` en visitas repetidas y lectura sin consultas a Neon. |
| 5 | **CSP con hashes en lo público, nonces en `/admin`.** | Validación: pendiente | Verificar que los scripts inline de Next.js admitan hashes estáticos; si no, evaluar nonces en lo público. |
| 6 | **Códigos 404 y 308 reales con `cacheComponents: true`.** | Validación: pendiente | Verificar que 404 y 308 no queden atrapados en Suspense ni en `loading.tsx`; que salgan como respuesta HTTP real. |
| 7 | **Lecturas de Blob: ¿operaciones simples o avanzadas?** | Validación: pendiente | Confirmar si `list` y las lecturas con CDN miss cuentan como operaciones simples (10.000/mes) o avanzadas (2.000/mes). |

## Estado

Documentación revisada contra las versiones de noviembre de 2026 y contra varias revisiones arquitectónicas independientes. No hay código todavía.

**Próximo paso**: Fase 0 de `06.1-operaciones-entorno.md`. Es un spike de tres días para medir consumo de Neon, invalidación de caché, `scrypt`, subida real a Blob y backup restaurado, antes de escribir el panel. Si esos números no cierran, se corrige el diseño con evidencia y no se sigue documentando.