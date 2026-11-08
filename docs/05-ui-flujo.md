# Diseño UI/UX y Flujo Editorial — Netti

## 1. Paleta

| Token | Valor | Uso |
|-------|-------|-----|
| `--accent` | `#a78bfa` | Botones, enlaces, foco, selección |
| `--accent-strong` | `#c9bbff` | Hover |
| `--accent-deep` | `#5b21b6` | Bordes, sombras |
| `--on-accent` | `#1a0b2e` | Texto sobre morado |
| `--bg` | `#141218` | Fondo |
| `--surface` | `#1e1b24` | Tarjetas |
| `--surface-raised` | `#272232` | Modales |
| `--text` | `#f4effc` | Texto |
| `--muted` | `#b6adc9` | Secundario |
| `--border` | `#352d45` | Bordes |
| `--red` | `#f16c79` | Errores, borrar |
| `--gold` | `#e6bd55` | Puntuación, destacados |
| `--cyan` | `#5ac7e8` | Juegos |
| `--green` | `#53bd83` | Guías |

Tokens en `app/globals.css` como variables CSS; Tailwind v4 los consume con `@theme`. Ningún color literal suelto en componentes.

Contraste: `--text` sobre `--bg` > 12:1, `--muted` > 7:1, `--accent` > 7:1. Ningún texto de interfaz baja de 4.5:1.

## 2. Tipografía

Quicksand auto-hospedada (`public/fonts/`), pesos 300–700. Cuerpo de artículo 1.125 rem, interlineado 1.7, ancho máximo 68 caracteres. Monospace del sistema para código.

## 3. Sitio público

- Header: "Netti." + Reseñas, Noticias, Guías, Juegos, Buscar.
- Inicio: destacado, últimos por colección, estadísticas cacheadas, "Juego al azar" (cacheado, nunca consulta la base por visita).
- Detalle: título, meta, cuerpo, etiquetas, puntuación, relacionados.
- 404 real; si el slug no existe se consulta `redirects` y se responde 308.

## 4. Panel (pantalla completa)

```
┌──────────────────────────────────────────────────────────────┐
│ Netti.   [Juegos] [Reseñas] [Noticias] [Guías]     [Salir]   │
├───────────────┬──────────────────────────────────────────────┤
│ Buscar…       │  Título del artículo        ● Borrador       │
│               │  [Vista previa] [Guardar] [Publicar]         │
│  Artículo     │                                              │
│  Artículo     │  Título          [__________________]        │
│  Artículo     │  Slug            [__________________] 🔒     │
│               │  Descripción     [__________________] 42/240 │
│  ─────────    │                                              │
│  Papelera     │  Cuerpo                                      │
│               │  [ área grande de Markdown            ]      │
│               │                                              │
│               │  Metadatos        Portada        Historial   │
└───────────────┴──────────────────────────────────────────────┘
```

- Sidebar con búsqueda, lista por colección y Papelera.
- Formulario siempre visible.
- Vista previa en modal con **la plantilla real**, mismo pipeline de sanitizado que el sitio. Si el cliente renderizara distinto, la vista previa mentiría.
- Estado visible en todo momento: borrador, publicado o en papelera. "Programado" no es un estado: es publicado con fecha futura, y se muestra como "Se publica el 12/12 a las 09:00".

## 5. Principios

1. **Nunca se pierde trabajo.** Autoguardado en `localStorage` cada 5 segundos, con aviso al salir. Si llega `401` (sesión vencida), el panel reautentica y restaura el borrador local **validándolo con Zod**: el esquema puede haber cambiado.
2. **Guardar no publica.** En un borrador los botones son "Guardar borrador" y "Publicar". En una entrada publicada son "Guardar cambios" y "Despublicar".
3. **Sin redirecciones silenciosas.** Cada acción deja un mensaje explícito.
4. **Conflictos visibles.** Si `version` no coincide, se muestra el estado actual y se ofrece recargar. No se pisa trabajo.
5. **Slug automático y bloqueado.** Se genera del título, editable hasta la primera publicación. Después, cambiarlo es explícito y crea una redirección 308.
6. **Atajos**: `Ctrl/Cmd+S` guarda, `Ctrl/Cmd+Shift+S` publica, `Ctrl/Cmd+P` abre vista previa, `Esc` cierra el modal.
7. **Confirmaciones destructivas**: borrar pide confirmación con el título; purgar exige escribir el slug.
8. **Responsivo** desde 1024 px. En móvil se consulta y corrige, no se escribe una reseña larga.

## 6. Accesibilidad

- `<label>` asociado en todo input; el placeholder no es etiqueta.
- Botones de icono con `aria-label`.
- Foco visible de 2 px con `--accent`.
- Modal con trampa de foco, `Esc` y foco devuelto al disparador.
- Errores con `aria-live="polite"` y `aria-describedby` en el campo.
- Nada depende solo del color para indicar estado.

## 7. Flujo

### 7.1 Login

1. `/admin/login`, contraseña.
2. Rate limit por IP antes de verificar; si está bloqueado, se muestra el tiempo restante.
3. Sesión de 8 horas deslizante, tope absoluto de 30 días.
4. Redirección a `/admin/panel`.

### 7.2 Panel

Listado por colección con estado, versión y última modificación. Acciones: Editar, Borrar. Buscador full-text en español, sin tildes.

### 7.3 Crear

1. "Nuevo" abre un formulario vacío en `draft`.
2. Se elige la colección.
3. Título genera el slug.
4. Cuerpo Markdown con barra mínima: negrita, itálica, enlace, H2, cita, imagen, lista.
5. Metadatos según colección. En reseñas, el juego se busca con autocompletado (`pg_trgm`). Crear la ficha desde la reseña queda para después del primer lanzamiento: en la v1, la ficha se crea antes desde Juegos.
6. **Portada e imágenes del cuerpo**: se eligen de la biblioteca de medios o se suben arrastrando. **El cliente redimensiona a WebP antes de subir**, porque un PNG de 1080p supera los 2 MB. Cada imagen exige texto alternativo y permite crédito y procedencia (`own`, `press`, `third_party`). Las imágenes del cuerpo no se pegan como URL: se insertan desde el selector y el Markdown guarda `![alt](media:12)`, que el render resuelve con `srcset` y dimensiones.
7. Guardar borrador o publicar. Los errores aparecen junto al campo.

### 7.4 Editar

Se carga la entrada con su `version`. Al guardar se envía esa versión. Si otro proceso la cambió, llega `conflict` con el estado actual y se ofrece recargar.

**Una entrada publicada no se despublica al guardar.** "Guardar cambios" mantiene el estado y revalida; "Despublicar" es una acción aparte.

### 7.5 Programar

Se elige una fecha futura y se publica. La entrada queda `published` con `published_at` futuro y no se ve hasta la hora. El job de revalidación la activa.

La interfaz dice **"Se publica el 12/12 a las 09:00"** y no promete precisión al segundo: el job de revalidación se ejecuta periódicamente (cada 30 minutos por defecto, pero GitHub Actions puede retrasar o descartar ejecuciones bajo carga). Si el job se atrasa, la entrada aparece igual en la siguiente ejecución exitosa; nunca se muestra antes de tiempo porque la consulta exige `published_at <= now()`.

### 7.6 Vista previa

Modal con la plantilla real: cabecera, ancho, tipografía, JSON-LD. `Esc` cierra y devuelve el foco. Usa **el mismo pipeline de sanitizado y el mismo resolvedor de `media:`** que el sitio público; si el cliente renderizara distinto, la vista previa mentiría.

### 7.7 Borrar y papelera

1. Confirmación con el título.
2. Pasa a `trashed` guardando el estado anterior y la fecha. El slug queda libre.
3. Papelera: recuperar o eliminar definitivamente.
4. **Recuperar con slug ocupado**: el servidor reintenta con `-2`, `-3`… hasta 10 veces y devuelve el slug definitivo. El panel no lo pide antes ni lo elige: muestra "Se recuperó como `mi-articulo-2`". Si el slug original lo ocupa otro artículo, **no se crea redirección** desde él (apropiaría la URL del artículo nuevo).
5. Purgar exige escribir el slug y sigue el orden mark-and-sweep: primero se marca y se borra la fila, después el objeto en Blob. Si Blob falla, la imagen queda pendiente y se reintenta; el artículo ya no está.

### 7.8 Historial

Lista de revisiones con fecha y versión. Se puede ver el snapshot y restaurar uno anterior (crea una versión nueva, no reescribe el pasado). Se conservan las últimas 20. Restaurar un snapshot es una escritura normal: exige la `version` actual y pasa por Zod como cualquier otra.

### 7.9 Cerrar sesión

Borra la sesión en el servidor y la cookie. "Cerrar todas las sesiones" invalida todos los dispositivos.

## 8. Mensajes

| Situación | Texto |
|-----------|-------|
| Borrador | "Borrador guardado." |
| Publicado | "Publicado. Ya está visible en el sitio." |
| Programado | "Se va a publicar el 12/12 a las 09:00." |
| Conflicto | "La publicación cambió mientras editabas. Recargá para ver la última versión." |
| Slug ocupado | "Ese slug ya está en uso. Probá con otro." |
| Sesión vencida | "Tu sesión venció. Volvé a entrar; tu borrador está guardado en este navegador." |
| Imagen grande | "La imagen no puede superar los 2 MB." |
| Imagen animada | "Las imágenes animadas no están soportadas." |
| Formato inválido | "Formato no admitido. Usá JPEG, PNG, WebP o AVIF." |
| Recuperado con sufijo | "Se recuperó como «mi-articulo-2»." |
| Sin conexión | "No se pudo guardar. Tu borrador sigue en este navegador." |