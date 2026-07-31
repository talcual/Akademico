# Módulo 2: HTML y HTML5

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Construir páginas web semánticas y bien estructuradas con HTML5.
- Utilizar correctamente las etiquetas básicas de marcado.
- Diseñar formularios accesibles y validados en el cliente.
- Incorporar contenido multimedia: imágenes, audio, video, iframes, canvas y SVG.
- Aplicar atributos ARIA y buenas prácticas de accesibilidad.
- Conocer las bases del SEO on-page.
- Comprender el concepto y uso de los Web Components.

---

## 1. Introducción a HTML

**HTML (HyperText Markup Language)** es el lenguaje de marcado estándar para crear páginas web. No es un lenguaje de programación: describe la **estructura** del contenido mediante **etiquetas** (*tags*).

### Historia breve

- **HTML 1.0** (1993): Tim Berners-Lee.
- **HTML 4.01** (1999): separación clara entre contenido y presentación.
- **XHTML 1.0** (2000): XML más estricto.
- **HTML5** (2014, recomendación W3C): introduce semántica, multimedia nativa, APIs modernas.
- **HTML Living Standard**: WHATWG mantiene una versión "viva" que evoluciona continuamente.

### ¿Qué necesitas?

- Un **editor de texto** (VS Code, Sublime, Atom, Vim).
- Un **navegador** moderno.
- (Opcional) Un servidor local: `python -m http.server`, `npx serve`, Live Server de VS Code.

### Tu primer archivo HTML

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Mi primera página</title>
</head>
<body>
  <h1>Hola, mundo</h1>
  <p>Esta es mi primera página web.</p>
</body>
</html>
```

Guárdalo como `index.html` y ábrelo en el navegador.

### Actividad práctica

- Crea un archivo `index.html` con tu nombre y un párrafo de presentación. Visualízalo en el navegador.

---

## 2. Anatomía de una página

Toda página HTML5 tiene una estructura mínima obligatoria:

```html
<!DOCTYPE html>                       <!-- Declara HTML5 -->
<html lang="es">                      <!-- Elemento raíz -->
<head>                                <!-- Metadatos (no se ven) -->
  <meta charset="UTF-8">              <!-- Codificación -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0"> <!-- Responsive -->
  <meta name="description" content="Aprende HTML5 desde cero">
  <meta name="author" content="Luis Toscano">
  <meta name="keywords" content="HTML5, web, curso">
  <meta name="theme-color" content="#0a7d54">
  <title>Título de la pestaña</title> <!-- Título de la pestaña -->
  <link rel="icon" href="/favicon.ico"> <!-- Ícono -->
  <link rel="stylesheet" href="styles.css">
  <script src="app.js" defer></script>
</head>
<body>
  <!-- Contenido visible -->
</body>
</html>
```

### Cabeceras meta importantes

- `charset`: define la codificación (UTF-8 siempre).
- `viewport`: indispensable en diseño responsive.
- `description`: resumen que muestran los buscadores.
- `robots`: indica a los crawlers si indexar (`index`, `noindex`).
- `og:*` y `twitter:*`: para compartir en redes sociales.
- `theme-color`: color de la barra en móviles.

### Actividad práctica

- Crea una página con `title`, `description`, `viewport` y un ícono SVG como favicon.

---

## 3. Etiquetas básicas

Las etiquetas se escriben entre ángulos: `<etiqueta>`. La mayoría tiene **apertura** y **cierre**: `<p>...</p>`. Algunas son **autocerradas** (`<img>`, `<br>`, `<hr>`, `<input>`).

### Atributos comunes

- `id`: identificador único.
- `class`: una o varias clases.
- `style`: estilos en línea (evitar).
- `title`: tooltip.
- `data-*`: atributos personalizados para datos.
- `lang`: idioma del contenido.
- `hidden`: oculta el elemento.
- `tabindex`: control del orden de tabulación.
- `contenteditable`: hacer contenido editable.

```html
<p id="intro" class="lead" data-author="luis" hidden>
  Párrafo de ejemplo
</p>
```

### Comentarios

```html
<!-- Esto es un comentario, no se muestra -->
```

### Texto plano y escape

```html
<p>El símbolo &lt; representa "menor que".</p>
<p>&copy; 2025 - Aprende HTML5</p>
```

### Actividad práctica

- Crea una página con varios párrafos, aplica `id`, `class` y atributos `data-*`.

---

## 4. Encabezados

HTML ofrece seis niveles de encabezado, de mayor a menor importancia:

```html
<h1>Título principal de la página</h1>
<h2>Sección importante</h2>
<h3>Subsección</h3>
<h4>Sub-subsección</h4>
<h5>Detalle menor</h5>
<h6>Detalle muy menor</h6>
```

### Buenas prácticas

- Una sola `<h1>` por página (representa el tema principal).
- No uses encabezados solo por tamaño; usa CSS para eso.
- Mantén la jerarquía: no saltes de `h1` a `h4`.

```html
<header>
  <h1>Aprende HTML5</h1>
  <p>El lenguaje de la web</p>
</header>

<main>
  <article>
    <h2>Encabezados</h2>
    <p>Los encabezados dan estructura al documento.</p>
    <h3>Niveles</h3>
    <p>Hay seis niveles disponibles.</p>
  </article>
</main>
```

### Actividad práctica

- Escribe un artículo corto (3 secciones) usando correctamente la jerarquía `h1 → h2 → h3`.

---

## 5. Párrafos

```html
<p>Este es un párrafo de texto. El navegador
lo renderiza como un bloque separado del resto.</p>
<p>Otro párrafo independiente.</p>
```

### Formato en línea

```html
<p>Texto en <strong>negrita semántica</strong> y <b>estilística</b>.</p>
<p>Texto en <em>énfasis</em> y <i>itálica técnica</i>.</p>
<p>Texto <u>subrayado</u> y <s>tachado</s>.</p>
<p>Texto <mark>resaltado</mark>, <small>pequeño</small> y <sup>superíndice</sup>.</p>
<p>Las siglas <abbr title="HyperText Markup Language">HTML</abbr>.</p>
<p>La constante <code>const PI = 3.14</code>.</p>
<p>La variable <var>x</var> es igual a <samp>5</samp>.</p>
<p>Una cita: <q>Hola mundo</q>.</p>
<p>Entrada del usuario: <kbd>Ctrl + S</kbd>.</p>
```

### Bloques especiales

```html
<blockquote cite="https://example.com">
  <p>El único modo de hacer un gran trabajo es amar lo que haces.</p>
  <cite>Steve Jobs</cite>
</blockquote>

<pre><code>function saludar() {
  console.log("Hola");
}</code></pre>
```

### Salto de línea y línea horizontal

```html
<p>Primera línea<br>Segunda línea</p>
<hr>
<p>Separado por una línea horizontal</p>
```

### Actividad práctica

- Crea una biografía tuya con párrafos, énfasis, citas y bloques `<pre><code>`.

---

## 6. Enlaces

```html
<!-- Enlace externo -->
<a href="https://developer.mozilla.org" target="_blank" rel="noopener noreferrer">
  MDN Web Docs
</a>

<!-- Enlace interno -->
<a href="#seccion-2">Ir a la sección 2</a>
<a href="/contacto.html">Contacto</a>
<a href="mailto:luis@example.com">Enviar correo</a>
<a href="tel:+584141234567">Llamar</a>
<a href="cv.pdf" download>Descargar CV</a>
```

### Atributos clave

- `href`: destino del enlace.
- `target="_blank"`: abre en nueva pestaña.
- `rel="noopener noreferrer"`: seguridad y privacidad.
- `download`: indica descarga.
- `aria-label`: etiqueta accesible.

### Anclas (enlaces a secciones)

```html
<h2 id="seccion-2">Sección 2</h2>
<p>...</p>

<a href="#seccion-2">Saltar a la sección 2</a>
```

### Estados de un enlace (CSS)

```css
a:link    { color: blue; }
a:visited { color: purple; }
a:hover   { color: red; }
a:focus   { outline: 2px solid orange; }
a:active  { color: green; }
```

### Actividad práctica

- Crea un índice de navegación con anclas internas a cada sección de tu página.

---

## 7. Imágenes

```html
<img src="gato.jpg"
     alt="Un gato atigrado mirando por la ventana"
     width="600" height="400"
     loading="lazy"
     decoding="async"
     srcset="gato-400.jpg 400w, gato-800.jpg 800w, gato-1200.jpg 1200w"
     sizes="(max-width: 600px) 100vw, 50vw">
```

### Atributos clave

- `src`: URL de la imagen (obligatorio).
- `alt`: texto alternativo para accesibilidad y SEO.
- `width` y `height`: evitan layout shift (CLS).
- `loading="lazy"`: carga diferida.
- `decoding="async"`: decodificación asíncrona.
- `srcset` + `sizes`: imágenes responsivas.
- `fetchpriority="high"`: priorizar la carga.

### Formatos de imagen

- **JPEG**: fotos, muchos colores.
- **PNG**: transparencia, logos.
- **GIF**: animaciones simples.
- **WebP**: alternativa moderna, mejor compresión.
- **AVIF**: aún mejor compresión, soporte creciente.
- **SVG**: vectores, ideal para iconos y logos.

### Figura con pie

```html
<figure>
  <img src="paisaje.jpg" alt="Paisaje montañoso al atardecer">
  <figcaption>Atardecer en los Andes</figcaption>
</figure>
```

### Mapa de imagen (avanzado)

```html
<img src="mapa.png" alt="Mapa del mundo" usemap="#mapa-mundi">
<map name="mapa-mundi">
  <area shape="rect" coords="0,0,100,100" href="americas.html" alt="Américas">
  <area shape="circle" coords="200,200,50" href="europa.html" alt="Europa">
</map>
```

### Actividad práctica

- Inserta una imagen responsiva con `srcset` y `sizes` en tu página, usando `figure` y `figcaption`.

---

## 8. Listas

### Listas desordenadas

```html
<ul>
  <li>HTML</li>
  <li>CSS</li>
  <li>JavaScript</li>
</ul>
```

### Listas ordenadas

```html
<ol type="1" start="1" reversed>
  <li>Instalar editor</li>
  <li>Crear archivo</li>
  <li>Abrir en navegador</li>
</ol>

<ol type="A">
  <li>Opción A</li>
  <li>Opción B</li>
</ol>
```

### Listas de definición

```html
<dl>
  <dt>HTML</dt>
  <dd>Lenguaje de marcado para estructurar la web.</dd>
  <dt>CSS</dt>
  <dd>Lenguaje de estilos para diseñar la web.</dd>
</dl>
```

### Listas anidadas

```html
<ul>
  <li>Frontend
    <ul>
      <li>HTML</li>
      <li>CSS</li>
      <li>JavaScript</li>
    </ul>
  </li>
  <li>Backend
    <ul>
      <li>Node.js</li>
      <li>Python</li>
    </ul>
  </li>
</ul>
```

### Lista de tareas (custom checkbox)

```html
<ul style="list-style: none;">
  <li><input type="checkbox" checked> Crear estructura</li>
  <li><input type="checkbox"> Estilar con CSS</li>
  <li><input type="checkbox"> Programar con JS</li>
</ul>
```

### Actividad práctica

- Crea una lista de compras con sublistas por categoría (frutas, verduras, etc.).

---

## 9. Tablas

```html
<table>
  <caption>Estudiantes del curso</caption>
  <thead>
    <tr>
      <th scope="col">#</th>
      <th scope="col">Nombre</th>
      <th scope="col">Edad</th>
      <th scope="col">Ciudad</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">1</th>
      <td>Ana Pérez</td>
      <td>22</td>
      <td>Caracas</td>
    </tr>
    <tr>
      <th scope="row">2</th>
      <td>Luis Gómez</td>
      <td>25</td>
      <td>Maracaibo</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <td colspan="4">Total: 2 estudiantes</td>
    </tr>
  </tfoot>
</table>
```

### Atributos importantes

- `scope`: indica si la cabecera es de columna (`col`), fila (`row`), grupo de columnas (`colgroup`) o grupo de filas (`rowgroup`).
- `colspan` / `rowspan`: combinación de celdas.
- `<caption>`: título accesible.
- `<thead>`, `<tbody>`, `<tfoot>`: estructura semántica.

### Estilos básicos (CSS)

```css
table { border-collapse: collapse; width: 100%; }
th, td { border: 1px solid #ccc; padding: 0.5rem; text-align: left; }
thead { background: #0a7d54; color: white; }
```

### Actividad práctica

- Crea una tabla con datos de 5 productos: nombre, precio, stock. Usa `caption`, `thead`, `tbody` y `tfoot`.

---

## 10. Formularios

Los formularios permiten al usuario enviar datos al servidor.

### Estructura básica

```html
<form action="/registro" method="POST" enctype="multipart/form-data" novalidate>
  <fieldset>
    <legend>Registro de usuario</legend>

    <label for="nombre">Nombre</label>
    <input type="text" id="nombre" name="nombre" required minlength="3" maxlength="50" placeholder="Tu nombre">

    <label for="email">Email</label>
    <input type="email" id="email" name="email" required autocomplete="email">

    <label for="password">Contraseña</label>
    <input type="password" id="password" name="password" required minlength="8" autocomplete="new-password">

    <label for="edad">Edad</label>
    <input type="number" id="edad" name="edad" min="18" max="99" value="18">

    <label for="nacimiento">Fecha de nacimiento</label>
    <input type="date" id="nacimiento" name="nacimiento">

    <label for="hora">Hora de contacto</label>
    <input type="time" id="hora" name="hora">

    <label for="color">Color favorito</label>
    <input type="color" id="color" name="color" value="#0a7d54">

    <label for="rango">Satisfacción</label>
    <input type="range" id="rango" name="rango" min="0" max="10" step="1" value="5">

    <label for="telefono">Teléfono</label>
    <input type="tel" id="telefono" name="telefono" pattern="[0-9]{10}" placeholder="04141234567">

    <label for="sitio">Sitio web</label>
    <input type="url" id="sitio" name="sitio" placeholder="https://example.com">

    <label for="busqueda">Búsqueda</label>
    <input type="search" id="busqueda" name="busqueda">

    <label for="cv">Sube tu CV (PDF, máx 2MB)</label>
    <input type="file" id="cv" name="cv" accept="application/pdf" >

    <label for="pais">País</label>
    <select id="pais" name="pais" required>
      <option value="">Selecciona...</option>
      <optgroup label="América">
        <option value="ve">Venezuela</option>
        <option value="co">Colombia</option>
        <option value="mx">México</option>
      </optgroup>
      <optgroup label="Europa">
        <option value="es">España</option>
        <option value="fr">Francia</option>
      </optgroup>
    </select>

    <fieldset>
      <legend>Género</legend>
      <label><input type="radio" name="genero" value="m"> Masculino</label>
      <label><input type="radio" name="genero" value="f"> Femenino</label>
      <label><input type="radio" name="genero" value="o"> Otro</label>
    </fieldset>

    <fieldset>
      <legend>Intereses</legend>
      <label><input type="checkbox" name="intereses" value="html"> HTML</label>
      <label><input type="checkbox" name="intereses" value="css"> CSS</label>
      <label><input type="checkbox" name="intereses" value="js"> JavaScript</label>
    </fieldset>

    <label for="comentarios">Comentarios</label>
    <textarea id="comentarios" name="comentarios" rows="5" maxlength="500" placeholder="Cuéntanos..."></textarea>

    <label>
      <input type="checkbox" name="terminos" required>
      Acepto los <a href="/terminos">términos</a>
    </label>

    <button type="submit">Enviar</button>
    <button type="reset">Limpiar</button>
  </fieldset>
</form>
```

### Tipos de `<input>`

- `text`, `password`, `email`, `url`, `tel`, `search`
- `number`, `range`, `date`, `time`, `month`, `week`, `datetime-local`
- `color`, `file`, `checkbox`, `radio`, `hidden`
- `submit`, `reset`, `button`, `image`

### Validación HTML5

- `required`: campo obligatorio.
- `min`, `max`, `minlength`, `maxlength`, `pattern`, `step`.
- Tipos que ya validan: `email`, `url`, `number`, `date`.

### `:valid`, `:invalid`, `:focus`

```css
input:invalid { border-color: red; }
input:valid   { border-color: green; }
input:focus   { outline: 2px solid blue; }
```

### `<datalist>` (autocompletado)

```html
<label for="navegador">Navegador favorito</label>
<input list="navegadores" id="navegador" name="navegador">
<datalist id="navegadores">
  <option value="Chrome">
  <option value="Firefox">
  <option value="Safari">
  <option value="Edge">
</datalist>
```

### `<output>` para resultados dinámicos

```html
<form oninput="resultado.value=parseInt(a.value)+parseInt(b.value)">
  <input type="number" id="a" value="0"> +
  <input type="number" id="b" value="0"> =
  <output name="resultado" for="a b">0</output>
</form>
```

### Actividad práctica

- Construye un formulario de registro completo con todos los tipos de input y validación HTML5.

---

## 11. HTML Semántico

HTML5 introduce etiquetas con **significado**, no solo presentación. Mejoran accesibilidad, SEO y mantenimiento.

### Estructura semántica de un documento

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Mi sitio</title>
</head>
<body>
  <header>
    <a href="/" aria-label="Inicio">
      <img src="logo.svg" alt="Logo del sitio">
    </a>
    <nav aria-label="Navegación principal">
      <ul>
        <li><a href="/">Inicio</a></li>
        <li><a href="/blog">Blog</a></li>
        <li><a href="/contacto">Contacto</a></li>
      </ul>
    </nav>
  </header>

  <main>
    <article>
      <header>
        <h1>Título del artículo</h1>
        <p>Por <a href="/autor/luis">Luis</a> · <time datetime="2025-01-15">15 de enero de 2025</time></p>
      </header>
      <p>Contenido del artículo...</p>
      <section>
        <h2>Sección interna</h2>
        <p>Más contenido...</p>
      </section>
    </article>

    <aside aria-label="Relacionado">
      <h3>Artículos relacionados</h3>
      <ul>
        <li><a href="#">HTML básico</a></li>
        <li><a href="#">CSS Grid</a></li>
      </ul>
    </aside>
  </main>

  <footer>
    <p>&copy; 2025 - Aprende HTML5</p>
    <address>Contacto: <a href="mailto:info@example.com">info@example.com</a></address>
  </footer>
</body>
</html>
```

### Etiquetas clave

| Etiqueta      | Propósito                                    |
| ------------- | -------------------------------------------- |
| `<header>`    | Cabecera de página o de sección              |
| `<nav>`       | Bloque de navegación                         |
| `<main>`      | Contenido principal (único por página)       |
| `<article>`   | Contenido independiente y reusable           |
| `<section>`   | Sección temática genérica                    |
| `<aside>`     | Contenido relacionado o tangencial           |
| `<footer>`    | Pie de página o de sección                   |
| `<figure>`    | Contenido ilustrativo (img, code, video)     |
| `<figcaption>`| Leyenda de un figure                         |
| `<address>`   | Información de contacto                      |
| `<time>`      | Fecha u hora legible por máquinas            |
| `<mark>`      | Texto resaltado                              |
| `<details>`   | Widget expandible                            |
| `<summary>`   | Resumen del details                          |
| `<dialog>`    | Cuadro de diálogo modal                      |

### `<details>` y `<summary>`

```html
<details>
  <summary>¿Qué es HTML5?</summary>
  <p>Es la quinta versión del lenguaje de marcado para la web.</p>
</details>

<details open>
  <summary>Ver respuesta</summary>
  <p>JavaScript.</p>
</details>
```

### `<dialog>` modal nativo

```html
<dialog id="miDialogo">
  <h2>Confirmar</h2>
  <p>¿Deseas continuar?</p>
  <form method="dialog">
    <button value="si">Sí</button>
    <button value="no">No</button>
  </form>
</dialog>

<button onclick="document.getElementById('miDialogo').showModal()">
  Abrir diálogo
</button>
```

### Actividad práctica

- Convierte un sitio existente en una versión semántica usando `header`, `nav`, `main`, `article`, `aside`, `footer`.

---

## 12. SEO básico

**SEO (Search Engine Optimization)** son las técnicas para mejorar la visibilidad de una página en buscadores.

### Factores on-page esenciales

- `<title>` único y descriptivo (50-60 caracteres).
- `<meta name="description">` atractiva (140-160 caracteres).
- Una sola `<h1>` por página con la palabra clave principal.
- URLs amigables (`/blog/html-semantico` en vez de `?id=42`).
- Texto `alt` descriptivo en todas las imágenes.
- Datos estructurados (Schema.org).
- Open Graph y Twitter Cards para redes.

### Ejemplo de metadatos completos

```html
<head>
  <title>Aprende HTML5 - Guía completa desde cero</title>
  <meta name="description" content="Curso gratuito de HTML5 con ejemplos prácticos, etiquetas semánticas, formularios y accesibilidad.">
  <link rel="canonical" href="https://misitio.com/curso/html5">

  <!-- Open Graph -->
  <meta property="og:type" content="article">
  <meta property="og:title" content="Aprende HTML5">
  <meta property="og:description" content="Curso gratuito con ejemplos prácticos.">
  <meta property="og:image" content="https://misitio.com/img/html5.png">
  <meta property="og:url" content="https://misitio.com/curso/html5">
  <meta property="og:site_name" content="Aprende Web">

  <!-- Twitter -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:title" content="Aprende HTML5">
  <meta name="twitter:description" content="Curso gratuito con ejemplos prácticos.">
  <meta name="twitter:image" content="https://misitio.com/img/html5.png">
</head>
```

### Datos estructurados (JSON-LD)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Aprende HTML5",
  "author": {
    "@type": "Person",
    "name": "Luis Toscano"
  },
  "datePublished": "2025-01-15",
  "image": "https://misitio.com/img/html5.png"
}
</script>
```

### Sitemap y robots

- `sitemap.xml`: lista de URLs de tu sitio.
- `robots.txt`: indica a los crawlers qué pueden indexar.

```txt
# robots.txt
User-agent: *
Allow: /
Sitemap: https://misitio.com/sitemap.xml
```

```xml
<!-- sitemap.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://misitio.com/</loc>
    <lastmod>2025-01-15</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
</urlset>
```

### Actividad práctica

- Audita una página con Lighthouse (categoría SEO) y aplica mejoras sugeridas.

---

## 13. Accesibilidad (ARIA)

La accesibilidad web garantiza que **todas las personas**, incluyendo aquellas con discapacidades, puedan usar la web.

### Pautas WCAG

- **POUR**: Perceptible, Operable, Understandable, Robust.
- Niveles de conformidad: **A**, **AA**, **AAA**.

### Buenas prácticas básicas

- Todo `<img>` debe tener `alt` (vacío `alt=""` si es decorativa).
- Todo campo de formulario debe tener un `<label>` asociado.
- Jerarquía de encabezados coherente.
- Contraste de color suficiente (mínimo 4.5:1 para texto normal).
- Navegación completa con teclado.
- Foco visible (`outline`).
- No depender solo del color para transmitir información.

### Roles ARIA

ARIA (**Accessible Rich Internet Applications**) añade semántica cuando HTML no es suficiente.

```html
<!-- Rol de botón en un div -->
<div role="button" tabindex="0" aria-pressed="false" id="toggle">Activar</div>

<!-- Región importante -->
<div role="alert" aria-live="assertive">Error: campo requerido</div>

<!-- Estado expandido -->
<button aria-expanded="false" aria-controls="menu">Menú</button>
<ul id="menu" hidden>...</ul>

<!-- Etiqueta accesible -->
<button aria-label="Cerrar ventana">×</button>

<!-- Campo solo para lectores de pantalla -->
<span class="sr-only" aria-hidden="true">Texto visual</span>
```

### Atributos ARIA clave

- `role`: define el rol (`navigation`, `button`, `dialog`, `alert`, `status`).
- `aria-label`: etiqueta corta.
- `aria-labelledby`: referencia a un elemento que etiqueta.
- `aria-describedby`: referencia a una descripción.
- `aria-hidden`: oculta del árbol de accesibilidad.
- `aria-live`: indica contenido dinámico (`polite`, `assertive`).
- `aria-expanded`: estado expandido/colapsado.
- `aria-current`: indica el elemento actual (`page`, `step`).
- `aria-required`, `aria-invalid`, `aria-disabled`.

### Clase "screen reader only"

```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

### Saltar al contenido principal

```html
<a href="#main" class="sr-only sr-only-focusable">Saltar al contenido</a>
```

```css
.sr-only-focusable:focus {
  position: static;
  width: auto; height: auto;
  clip: auto;
}
```

### Formularios accesibles

```html
<label for="email">Email</label>
<input type="email" id="email" required aria-describedby="ayuda-email">
<small id="ayuda-email">Nunca compartiremos tu correo.</small>

<input type="email" id="email" aria-invalid="true" aria-errormessage="error-email">
<span id="error-email" role="alert">Email no válido</span>
```

### Actividad práctica

- Instala la extensión **WAVE** o **axe DevTools** y audita una página.
- Navega una página completa usando solo el teclado (Tab, Enter, Esc).

---

## 14. Multimedia

### Imágenes responsivas avanzadas

```html
<picture>
  <source media="(min-width: 1024px)" srcset="banner-desktop.webp" type="image/webp">
  <source media="(min-width: 768px)" srcset="banner-tablet.webp" type="image/webp">
  <source media="(max-width: 767px)" srcset="banner-mobile.webp" type="image/webp">
  <img src="banner-fallback.jpg" alt="Banner principal" loading="eager">
</picture>
```

### `<picture>` vs `srcset`

- `<picture>` permite **cambiar la imagen** según condiciones (art direction).
- `srcset` permite al navegador **elegir el tamaño** más adecuado (resolution switching).

### Actividad práctica

- Crea una galería con `<picture>` que muestre distintas imágenes en móvil y escritorio.

---

## 15. Audio y Video

### Audio

```html
<audio controls preload="metadata" autoplay muted loop>
  <source src="audio.mp3" type="audio/mpeg">
  <source src="audio.ogg" type="audio/ogg">
  Tu navegador no soporta audio.
</audio>
```

Atributos:
- `controls`: muestra los controles.
- `autoplay`, `loop`, `muted`.
- `preload`: `none`, `metadata`, `auto`.

### Video

```html
<video controls width="640" height="360"
       poster="poster.jpg"
       preload="metadata"
       playsinline>
  <source src="video.mp4" type="video/mp4">
  <source src="video.webm" type="video/webm">
  <track kind="subtitles" src="subs-es.vtt" srclang="es" label="Español" default>
  <track kind="captions" src="captions-es.vtt" srclang="es" label="Español (CC)">
  Tu navegador no soporta video.
</video>
```

Atributos:
- `poster`: imagen previa.
- `playsinline`: evita fullscreen automático en iOS.
- `disablePictureInPicture`.
- `crossorigin`: junto con CORS.

### Subtítulos (WebVTT)

```vtt
WEBVTT

00:00:00.000 --> 00:00:05.000
Hola, bienvenidos al curso de HTML5.

00:00:05.000 --> 00:00:10.000
Hoy veremos multimedia.
```

### API de Media (JS)

```html
<video id="miVideo" src="video.mp4" controls></video>
<button onclick="document.getElementById('miVideo').play()">Play</button>
<button onclick="document.getElementById('miVideo').pause()">Pause</button>
<button onclick="document.getElementById('miVideo').currentTime = 0">Reiniciar</button>
```

### Actividad práctica

- Inserta un video con subtítulos en español e inglés.

---

## 16. Iframes

Un `<iframe>` incrusta otra página dentro de la actual.

```html
<iframe
  src="https://www.openstreetmap.org/export/embed.html?bbox=-0.1,51.5,-0.06,51.52"
  width="600" height="450"
  title="Mapa de Londres"
  loading="lazy"
  referrerpolicy="no-referrer-when-downgrade"
  sandbox="allow-scripts allow-same-origin"
  allow="geolocation; fullscreen"
  allowfullscreen>
</iframe>
```

### Atributos importantes

- `src`: URL a incrustar.
- `sandbox`: restricciones de seguridad.
- `allow`: permisos (cámara, micrófono, fullscreen, etc.).
- `referrerpolicy`.
- `loading="lazy"`.

### YouTube embebido

```html
<iframe
  width="560" height="315"
  src="https://www.youtube.com/embed/dQw4w9WgXcQ"
  title="Video de YouTube"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen>
</iframe>
```

### Seguridad con `sandbox`

```html
<iframe src="contenido.html" sandbox></iframe>
<iframe src="contenido.html" sandbox="allow-scripts allow-forms"></iframe>
```

### Actividad práctica

- Embebe un video de YouTube, un mapa de Google Maps y un PDF (usando `<embed>` o `<object>`).

---

## 17. Canvas

`<canvas>` es un lienzo 2D/3D donde se dibuja con JavaScript.

```html
<canvas id="miCanvas" width="400" height="200">
  Tu navegador no soporta canvas.
</canvas>
<script>
  const canvas = document.getElementById('miCanvas');
  const ctx = canvas.getContext('2d');

  // Fondo
  ctx.fillStyle = '#0a7d54';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Texto
  ctx.fillStyle = 'white';
  ctx.font = 'bold 24px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText('Hola Canvas', canvas.width / 2, canvas.height / 2);

  // Círculo
  ctx.beginPath();
  ctx.arc(100, 100, 40, 0, Math.PI * 2);
  ctx.fillStyle = 'orange';
  ctx.fill();
  ctx.strokeStyle = 'white';
  ctx.lineWidth = 3;
  ctx.stroke();

  // Línea
  ctx.beginPath();
  ctx.moveTo(50, 150);
  ctx.lineTo(350, 150);
  ctx.strokeStyle = 'white';
  ctx.lineWidth = 2;
  ctx.stroke();

  // Gradiente
  const grad = ctx.createLinearGradient(0, 0, 400, 0);
  grad.addColorStop(0, 'red');
  grad.addColorStop(1, 'blue');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 170, 400, 30);
</script>
```

### Animación con `requestAnimationFrame`

```html
<canvas id="anim" width="200" height="200"></canvas>
<script>
  const c = document.getElementById('anim');
  const ctx = c.getContext('2d');
  let x = 0;

  function frame() {
    ctx.clearRect(0, 0, c.width, c.height);
    ctx.fillStyle = 'tomato';
    ctx.beginPath();
    ctx.arc(x, 100, 20, 0, Math.PI * 2);
    ctx.fill();
    x = (x + 2) % c.width;
    requestAnimationFrame(frame);
  }
  frame();
</script>
```

### WebGL (3D)

```js
const gl = canvas.getContext('webgl');
// Inicialización de shaders, buffers, etc.
```

### Actividad práctica

- Crea un canvas que dibuje un reloj analógico en tiempo real.

---

## 18. SVG (Scalable Vector Graphics)

SVG es un formato XML para gráficos vectoriales escalables.

### SVG inline

```html
<svg width="200" height="200" viewBox="0 0 200 200" xmlns="http://www.w3.org/2000/svg">
  <circle cx="100" cy="100" r="80" fill="#0a7d54" />
  <rect x="60" y="60" width="80" height="80" fill="orange" />
  <path d="M 20 180 Q 100 50 180 180" stroke="white" stroke-width="4" fill="transparent" />
  <text x="100" y="105" text-anchor="middle" fill="white" font-size="20" font-weight="bold">SVG</text>
</svg>
```

### Formas básicas

- `<circle>`, `<rect>`, `<ellipse>`, `<line>`, `<polyline>`, `<polygon>`, `<path>`.
- `<text>` para texto.
- `<g>` para agrupar.
- `<defs>` y `<use>` para reutilizar.

### Icono SVG (componente reusable)

```html
<svg viewBox="0 0 24 24" width="24" height="24" aria-hidden="true">
  <path d="M12 2L2 22h20L12 2zm0 6l6.5 12h-13L12 8z" fill="currentColor" />
</svg>
```

### SVG como `<img>`

```html
<img src="icono.svg" alt="Icono" width="32" height="32">
```

### Animación con CSS o SMIL

```html
<svg width="100" height="100">
  <circle cx="50" cy="50" r="20" fill="red">
    <animate attributeName="r" values="10;30;10" dur="2s" repeatCount="indefinite" />
  </circle>
</svg>
```

### Ventajas de SVG

- Escalable sin pérdida de calidad.
- Se puede animar con CSS/JS.
- Accesible (`<title>`, `<desc>`, `role`).
- Se puede estilizar con CSS (`fill`, `stroke`).
- Bajo peso para iconos.

### Actividad práctica

- Crea un ícono de un corazón en SVG, luego coloréalo con CSS usando `currentColor`.

---

## 19. Web Components (Introducción)

Los **Web Components** son un conjunto de APIs nativas que permiten crear **elementos HTML personalizados, reutilizables y encapsulados**.

### Pilares

1. **Custom Elements**: definir nuevas etiquetas.
2. **Shadow DOM**: encapsular estilos y marcado.
3. **HTML Templates**: `<template>` y `<slot>`.

### Custom Element básico

```html
<template id="plantilla-usuario">
  <style>
    .card {
      border: 1px solid #ccc;
      border-radius: 8px;
      padding: 1rem;
      max-width: 300px;
      font-family: sans-serif;
    }
    .nombre { font-weight: bold; color: #0a7d54; }
  </style>
  <div class="card">
    <p class="nombre"></p>
    <p class="email"></p>
  </div>
</template>

<user-card nombre="Ana" email="ana@example.com"></user-card>
<user-card nombre="Luis" email="luis@example.com"></user-card>

<script>
  class UserCard extends HTMLElement {
    constructor() {
      super();
      const shadow = this.attachShadow({ mode: 'open' });
      const template = document.getElementById('plantilla-usuario');
      shadow.appendChild(template.content.cloneNode(true));

      shadow.querySelector('.nombre').textContent = this.getAttribute('nombre');
      shadow.querySelector('.email').textContent = this.getAttribute('email');
    }
  }
  customElements.define('user-card', UserCard);
</script>
```

### Con `<slot>`

```html
<template id="plantilla-card">
  <style>
    :host {
      display: block;
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 1rem;
      max-width: 320px;
      box-shadow: 0 2px 6px rgba(0,0,0,.1);
    }
    h3 { margin: 0 0 .5rem; color: #0a7d54; }
  </style>
  <h3><slot name="titulo">Título por defecto</slot></h3>
  <div><slot>Contenido por defecto</slot></div>
</template>

<mi-card>
  <span slot="titulo">Hola mundo</span>
  <p>Este es el contenido de la tarjeta.</p>
</mi-card>

<script>
  class MiCard extends HTMLElement {
    constructor() {
      super();
      const shadow = this.attachShadow({ mode: 'open' });
      shadow.appendChild(document.getElementById('plantilla-card').content.cloneNode(true));
    }
  }
  customElements.define('mi-card', MiCard);
</script>
```

### Ciclo de vida

- `connectedCallback()`: cuando se añade al DOM.
- `disconnectedCallback()`: cuando se quita.
- `attributeChangedCallback(name, old, new)`: cuando cambia un atributo observado.
- `observedAttributes`: lista de atributos a observar.

```js
class Reloj extends HTMLElement {
  static get observedAttributes() { return ['formato']; }
  connectedCallback() {
    this._timer = setInterval(() => this.render(), 1000);
  }
  disconnectedCallback() {
    clearInterval(this._timer);
  }
  attributeChangedCallback() { this.render(); }
  render() {
    const formato = this.getAttribute('formato') || '24h';
    const ahora = new Date();
    this.textContent = ahora.toLocaleTimeString('es', { hour12: formato === '12h' });
  }
}
customElements.define('mi-reloj', Reloj);

<mi-reloj></mi-reloj>
<mi-reloj formato="12h"></mi-reloj>
```

### Librerías populares

- **Lit**: pequeño y reactivo (https://lit.dev).
- **Stencil**: por el equipo de Ionic.
- **FAST** (Microsoft).
- **Hybrids**: basado en funciones puras.

### Actividad práctica

- Crea un componente `<contador-clicks>` que tenga un botón y muestre el número de clicks. Usa Shadow DOM.

---

## Recursos complementarios

### Lecturas

- [MDN - HTML](https://developer.mozilla.org/es/docs/Web/HTML)
- [HTML5 Doctor](http://html5doctor.com/)
- [W3C HTML Living Standard](https://html.spec.whatwg.org/)
- [WebAIM - Accesibilidad](https://webaim.org/)

### Herramientas

- [W3C HTML Validator](https://validator.w3.org/)
- [Can I Use](https://caniuse.com/)
- [WAVE Accesibilidad](https://wave.webaim.org/)
- [axe DevTools](https://www.deque.com/axe/devtools/)
- [Schema.org](https://schema.org/)

### Videos

- "HTML5 Crash Course" - Traversy Media.
- "Accesibilidad web" - Manz.dev.
- "Web Components" - fireship.

---

## Evaluación sugerida

1. **Cuestionario HTML (20%)**: sintaxis, etiquetas, atributos.
2. **Página personal (30%)**: sitio estático con HTML semántico, formulario y multimedia.
3. **Auditoría de accesibilidad (20%)**: entregar reporte de WAVE/axe con correcciones aplicadas.
4. **Web Component (30%)**: crear un componente reutilizable con Shadow DOM y slots.

---

> **Nota para el estudiante**: HTML es la base de todo. Una estructura semántica correcta no solo mejora el SEO y la accesibilidad: hace que tu CSS y JavaScript sean más fáciles de mantener.
