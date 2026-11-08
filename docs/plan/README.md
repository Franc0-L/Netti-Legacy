# Documentación Maestra — Netti

Plan de reconstrucción de Netti. Es la referencia para cualquier persona o agente que trabaje en el proyecto.

## Qué hay aquí

| Archivo | Qué explica |
|---------|-------------|
| `README.md` | Visión, alcance, reglas e índice. |
| `stack.md` | Stack, arquitectura y decisiones D-001 a D-022. |
| `content.md` | Esquema SQL, restricciones y validación con Zod. |
| `ui.md` | Paleta, panel y flujo editorial. |
| `api.md` | Server Actions y los Route Handlers (`upload`, `revalidate`, `health`, `ready`). |
| `ops.md` | Git, entorno, backups, despliegue y hoja de ruta. |

## Qué es Netti

Blog editorial de videojuegos en español, de un solo autor, con reseñas (puntuación 1–10), noticias, guías y fichas de juegos. Tiene un sitio público de lectura y un panel editorial propio a pantalla completa. No usa CMS externo.

## Cómo usar estos documentos

1. Para entender el contexto y el stack: `stack.md`.
2. Para modelar datos: `content.md`.
3. Para construir interfaz: `ui.md`.
4. Para escribir Server Actions o endpoints: `api.md`.
5. Para configurar el repo y desplegar: `ops.md`.
6. Para empezar a programar: `ops.md`, **Fase 0**.

## Reglas

- **La mejor herramienta gana**, aunque obligue a rehacer algo. No se defiende una decisión por haberla tomado antes.
- **Un solo repositorio.** El contenido vive en Postgres, no en archivos.
- **La lectura pública no despierta la base.** Es un requisito operativo, no una optimización: Neon Free suspende el cómputo al agotar las CU-horas.
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

## Decisiones abiertas

1. **Vercel Blob + subida por servidor.** Decidida: costo cero real en Hobby, sin tarjeta ni dominio propio. `lib/storage.ts` aísla `put`/`del` para poder migrar si algún día se justifica.
2. **`scrypt` con N=2^17 en Vercel.** Consume ~128 MiB por verificación. Si el spike muestra problemas de memoria o latencia, se baja N con un ADR que lo justifique.
3. **Cron de revalidación.** GitHub Actions hoy; Vercel Schedules cuando salga de beta con condiciones aceptables.

## Estado

Documentación revisada contra las versiones de noviembre de 2026 y contra tres revisiones arquitectónicas independientes. No hay código todavía.

**Próximo paso**: Fase 0 de `ops.md`. Es un spike de tres días para medir consumo de Neon, invalidación de caché, `scrypt`, subida real a Blob y backup restaurado, antes de escribir el panel. Si esos números no cierran, se corrige el diseño con evidencia y no se sigue documentando.
