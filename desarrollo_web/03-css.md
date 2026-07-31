# Módulo 3: CSS

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Aplicar selectores CSS con precisión y entender la cascada y especificidad.
- Modelar cajas (box model) y controlar el flujo de elementos.
- Crear layouts modernos con Flexbox y Grid.
- Diseñar interfaces responsivas con media queries y unidades relativas.
- Usar variables, animaciones, transiciones y transformaciones.
- Organizar CSS escalable con metodologías como BEM.

---

## 1. Selectores

CSS permite seleccionar elementos del DOM de múltiples formas.

### Selectores básicos

```css
/* Universal */
* { box-sizing: border-box; }

/* De tipo */
h1 { color: navy; }

/* De clase */
.destacado { background: yellow; }

/* De id */
#principal { padding: 2rem; }

/* De atributo */
input[type="text"] { border: 1px solid #ccc; }
a[href^="https"] { color: green; }                /* empieza con */
a[href$=".pdf"] { color: red; }                    /* termina con */
img[alt*="logo"] { border: 2px solid blue; }       /* contiene */

/* Agrupados */
h1, h2, h3 { font-family: sans-serif; }
```

### Selectores combinadores

```css
/* Descendiente: cualquier nivel */
article p { line-height: 1.6; }

/* Hijo directo: solo nivel inmediato */
ul > li { list-style: square; }

/* Hermano adyacente: el siguiente */
h1 + p { font-size: 1.2rem; }

/* Hermano general: todos los siguientes */
h1 ~ p { color: gray; }
```

### Pseudoclases y pseudoelementos

(Se detallan más adelante, pero como ejemplo:)

```css
a:hover { text-decoration: underline; }
p::first-line { font-variant: small-caps; }
```

### Selectores avanzados (CSS Selectors Level 4)

```css
/* :is() y :where() agrupan selectores */
:is(h1, h2, h3) { color: #0a7d54; }
:where(h1, h2, h3) { color: gray; } /* especificidad 0 */

/* :has() selector de padres */
.card:has(img) { padding: 0; }

/* :not() */
button:not([disabled]) { cursor: pointer; }

/* :nth-child() y :nth-of-type() */
li:nth-child(odd)  { background: #eee; }
li:nth-child(3n+1) { background: #ddd; } /* 1, 4, 7... */
tr:nth-of-type(even) { background: #f9f9f9; }
```

### Actividad práctica

- Dada una lista de 10 productos, aplica diferentes estilos usando selectores de atributo, hijos directos y `nth-child`.

---

## 2. Cascada

La **cascada** es el algoritmo que decide qué regla CSS aplicar cuando hay conflictos. Considera tres factores (en orden):

1. **Origen e importancia** (`!important` > estilos del autor > del usuario > del navegador).
2. **Especificidad**.
3. **Orden de aparición** (última regla gana si todo lo demás es igual).

### Reglas prácticas

```css
/* Más cercano al elemento gana */
<div style="color: red">  <!-- inline gana -->
```

### Ejemplo de cascada

```css
p { color: blue; }
p { color: red; } /* Este gana por orden */
```

### `!important` (úsalo con moderación)

```css
button { background: blue !important; }
```

> ⚠️ Evitar `!important`. Si lo necesitas, suele indicar un problema de arquitectura.

### Actividad práctica

- Crea un HTML con varios párrafos y aplica reglas en conflicto. Predice cuál ganará y compruébalo.

---

## 3. Especificidad

Es el **peso** de un selector. Gana el de mayor peso.

### Cálculo

Se representa con cuatro valores `(a, b, c, d)`:

| Selector                          | a | b | c | d | Total            |
| --------------------------------- |---|---|---|---|------------------|
| `*`                               | 0 | 0 | 0 | 0 | 0,0,0,0          |
| `h1`                              | 0 | 0 | 0 | 1 | 0,0,0,1          |
| `.clase`                          | 0 | 0 | 1 | 0 | 0,0,1,0          |
| `#id`                             | 0 | 1 | 0 | 0 | 0,1,0,0          |
| `h1.clase`                        | 0 | 0 | 1 | 1 | 0,0,1,1          |
| `style="..."` (inline)            | 1 | 0 | 0 | 0 | 1,0,0,0          |
| `!important`                      |   |   |   |   | gana a todo      |

### Ejemplo

```css
#header .nav a.active { /* (0,1,2,1) */
  color: red;
}
.nav a {                  /* (0,0,1,2) */
  color: blue;
}
```

### Truco: specificity calculator

Herramientas como [specificity.keegan.st](https://specificity.keegan.st/) permiten calcularla.

### Actividad práctica

- Resuelve 5 conflictos de selectores y predice el resultado. Verifica en el navegador.

---

## 4. Box Model

Todo elemento HTML se representa como una **caja rectangular** con cuatro áreas:

```
┌─────────────── margin ───────────────┐
│  ┌──────────── border ───────────┐  │
│  │  ┌────────── padding ────────┐  │  │
│  │  │          content          │  │  │
│  │  └────────────────────────────┘  │  │
│  └─────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Propiedades

```css
.caja {
  width: 200px;
  height: 100px;
  padding: 20px;
  border: 5px solid #333;
  margin: 30px;
  box-sizing: border-box; /* recomendado */
}
```

### `box-sizing`

```css
/* content-box (default): width = solo contenido */
div { box-sizing: content-box; }

/* border-box: width incluye padding y border */
div { box-sizing: border-box; }

/* Truco universal */
*, *::before, *::after { box-sizing: border-box; }
```

### Margin collapse

Los márgenes verticales **colapsan** (se fusionan) entre elementos adyacentes o entre padre e hijo sin padding/border.

```html
<p style="margin-bottom: 30px">A</p>
<p style="margin-top: 20px">B</p>
<!-- Margen real entre ellos: 30px (no 50) -->
```

Soluciones:
- Usar `padding` en el padre.
- Usar `display: flex` o `grid` en el padre.
- Aplicar `overflow: hidden` al padre.
- Definir `margin` solo arriba o solo abajo.

### Centrar una caja

```css
.caja {
  width: 600px;
  margin-left: auto;
  margin-right: auto;
}
```

### Centrar con `margin: auto` y flex

```css
.padre {
  display: flex;
  justify-content: center;
}
```

### Actividad práctica

- Crea tres cajas y observa la diferencia entre `content-box` y `border-box` con DevTools.

---

## 5. Display

La propiedad `display` define cómo se renderiza un elemento.

### Valores principales

```css
div      { display: block; }
span    { display: inline; }
img     { display: inline-block; }
.caja   { display: flex; }
.grid   { display: grid; }
.none   { display: none; }      /* no se renderiza */
.invisible { visibility: hidden; } /* ocupa espacio */
.absolute { position: absolute; }
```

### block vs inline vs inline-block

| Propiedad      | block        | inline          | inline-block    |
| -------------- | ------------ | --------------- | --------------- |
| Salto de línea | Sí           | No              | No              |
| width/height   | Sí           | No              | Sí              |
| margin vertical| Sí           | No              | Sí              |
| padding vertical| Sí (afecta) | No (no afecta)  | Sí (afecta)     |

### Display: contents

```css
.wrapper { display: contents; }
/* El wrapper "desaparece" del layout, sus hijos actúan como si estuvieran sueltos. */
```

### `display: none` vs `visibility: hidden` vs `opacity: 0`

```css
.oculto1 { display: none; }       /* no ocupa espacio */
.oculto2 { visibility: hidden; }  /* ocupa espacio, invisible */
.oculto3 { opacity: 0; }          /* ocupa espacio, transparente, sigue interactivo */
```

### Actividad práctica

- Convierte una lista horizontal usando `inline`, `inline-block` y `flex` para ver las diferencias.

---

## 6. Position

La propiedad `position` define cómo se posiciona un elemento en el flujo del documento.

### Valores

```css
.static   { position: static; }    /* default, sigue el flujo */
.relative { position: relative; }  /* se posiciona respecto a su lugar original */
.absolute { position: absolute; }  /* respecto al ancestro posicionado más cercano */
.fixed    { position: fixed; }     /* respecto al viewport */
.sticky   { position: sticky; }    /* híbrido entre relative y fixed */
```

### Propiedades de offset

```css
.caja {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
}
```

### `relative` como contexto

```css
.padre { position: relative; }
.hijo  { position: absolute; top: 10px; left: 10px; }
```

### `sticky` ejemplo (header pegajoso)

```css
header {
  position: sticky;
  top: 0;
  background: white;
  z-index: 10;
}
```

### `z-index`

```css
.modal   { position: fixed; z-index: 1000; }
.tooltip { position: absolute; z-index: 10; }
```

> Solo funciona con `position` distinta de `static`, o con `flex`/`grid` items y `opacity < 1`.

### Actividad práctica

- Construye un tooltip con `position: absolute` respecto a un botón posicionado como `relative`.

---

## 7. Float

**Float** fue pensado originalmente para rodear texto con imágenes. Hoy se usa poco para layout (reemplazado por Flexbox/Grid), pero sigue siendo útil para eso.

```css
img {
  float: left;
  margin: 0 1rem 1rem 0;
}
.texto { /* el texto fluye alrededor */ }
```

### Clearfix

```css
.clearfix::after {
  content: "";
  display: table;
  clear: both;
}
```

### Layout clásico con float

```css
.columna { float: left; width: 50%; }
.fila::after { content: ""; display: table; clear: both; }
```

> ⚠️ Hoy en día prefiere Flexbox o Grid.

### Actividad práctica

- Crea una galería de imágenes con `float: left` y `clearfix`. Después reemplázala con CSS Grid.

---

## 8. Overflow

Controla qué pasa cuando el contenido **excede** el contenedor.

```css
.box {
  overflow: visible;   /* default, se ve por fuera */
  overflow: hidden;    /* recorta */
  overflow: scroll;    /* siempre muestra scroll */
  overflow: auto;      /* scroll solo si es necesario */
  overflow-x: auto;    /* solo horizontal */
  overflow-y: hidden;  /* solo vertical */
}
```

### `text-overflow: ellipsis`

```css
.truncado {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  width: 200px;
}
```

### Múltiples líneas con `-webkit-line-clamp`

```css
.multilinea {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
```

### Actividad práctica

- Crea una tarjeta de producto con título truncado a 2 líneas y descripción truncada a 3.

---

## 9. Flexbox

**Flexbox** es un modelo de layout unidimensional (fila o columna) perfecto para alinear y distribuir espacio entre items.

### Conceptos

- **Contenedor flex** (`display: flex`).
- **Eje principal (main axis)** y **eje cruzado (cross axis)**.
- **Items flex** (hijos directos).

```css
.contenedor {
  display: flex;
  flex-direction: row;        /* row | row-reverse | column | column-reverse */
  flex-wrap: wrap;             /* nowrap | wrap | wrap-reverse */
  justify-content: center;    /* alineación en main axis */
  align-items: center;         /* alineación en cross axis */
  align-content: center;       /* alineación de filas (cuando hay wrap) */
  gap: 1rem;                   /* espacio entre items */
}
```

### `justify-content`

```css
flex-start    /* al inicio */
flex-end      /* al final */
center        /* centrado */
space-between /* primero y último al borde, resto distribuido */
space-around  /* espacio igual alrededor */
space-evenly  /* espacio completamente igual */
```

### `align-items`

```css
stretch      /* estira al alto del contenedor */
flex-start   /* arriba */
flex-end     /* abajo */
center       /* centrado */
baseline     /* en la línea base del texto */
```

### Propiedades de los items

```css
.item {
  flex-grow: 1;        /* capacidad de crecer */
  flex-shrink: 1;      /* capacidad de encoger */
  flex-basis: 200px;   /* tamaño base */
  flex: 1;             /* shorthand: grow shrink basis */
  align-self: center;  /* sobreescribe align-items del padre */
  order: 2;            /* cambia el orden visual */
}
```

### Centrado perfecto

```css
.padre {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}
```

### Ejemplo: barra de navegación

```html
<nav class="navbar">
  <img src="logo.svg" alt="Logo">
  <ul class="links">
    <li><a href="/">Inicio</a></li>
    <li><a href="/blog">Blog</a></li>
    <li><a href="/contacto">Contacto</a></li>
  </ul>
  <button class="cta">Login</button>
</nav>
```

```css
.navbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 2rem;
  background: #0a7d54;
  color: white;
}
.links {
  display: flex;
  gap: 1.5rem;
  list-style: none;
}
.cta {
  margin-left: auto; /* empuja a la derecha */
}
```

### Ejemplo: card layout

```css
.cards {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}
.card {
  flex: 1 1 280px; /* grow shrink basis */
}
```

### Actividad práctica

- Crea una galería de tarjetas responsiva con `flex-wrap` y `flex: 1 1 280px`.

---

## 10. Grid

**CSS Grid** es un sistema de layout **bidimensional** (filas y columnas).

### Conceptos

- **Contenedor grid** (`display: grid`).
- **Tracks**: filas (`grid-template-rows`) y columnas (`grid-template-columns`).
- **Celdas**: intersección de fila y columna.
- **Áreas**: conjunto de celdas con nombre.
- **Gaps**: espacio entre tracks.

```css
.grid {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
  grid-template-rows: auto 1fr auto;
  gap: 1rem 2rem;
  height: 100vh;
}
```

### Unidades `fr`

```css
grid-template-columns: 1fr 2fr 1fr; /* distribución proporcional */
grid-template-columns: repeat(3, 1fr);
grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); /* responsivo */
```

### `auto-fit` vs `auto-fill`

```css
/* auto-fill: mantiene tracks vacíos */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

/* auto-fit: colapsa los vacíos y estira los items */
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
```

### Posicionar items

```css
.item1 { grid-column: 1 / 3; }   /* de la línea 1 a la 3 */
.item2 { grid-row: 2 / 4; }
.item3 { grid-area: header; }

.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main { grid-area: main; }
.footer { grid-area: footer; }

.grid {
  grid-template-areas:
    "header header header"
    "sidebar main main"
    "footer  footer  footer";
}
```

### `place-items` y `place-content`

```css
.grid {
  place-items: center;    /* align-items + justify-items */
  place-content: center;  /* align-content + justify-content */
}
```

### Ejemplo: layout completo

```html
<div class="layout">
  <header>Header</header>
  <aside>Sidebar</aside>
  <main>Main</main>
  <footer>Footer</footer>
</div>
```

```css
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 250px 1fr;
  grid-template-rows: 80px 1fr 60px;
  min-height: 100vh;
  gap: 1rem;
}
header { grid-area: header; background: #0a7d54; }
aside  { grid-area: sidebar; background: #f0f0f0; }
main   { grid-area: main; }
footer { grid-area: footer; background: #333; color: white; }
```

### Grid + Flex (combinación)

Grid para layout macro, Flex dentro de cada celda.

### Actividad práctica

- Reproduce la página de un periódico: header, sidebar, contenido en columnas, footer.

---

## 11. Responsive Design

Diseño que se adapta a distintos tamaños de pantalla y dispositivos.

### Principios

1. **Mobile first**: diseñar primero para móviles.
2. **Unidades relativas**: `%`, `em`, `rem`, `vw`, `vh`, `clamp()`, `min()`, `max()`.
3. **Imágenes flexibles**: `max-width: 100%`.
4. **Media queries** para breakpoints.

### Unidades

```css
html { font-size: 100%; } /* base 16px */
h1 { font-size: 2rem; }  /* 32px */
h2 { font-size: 1.5rem; } /* 24px */

.caja { width: 90%; max-width: 1200px; }

h1 {
  font-size: clamp(1.5rem, 2.5vw + 1rem, 3rem);
}

img { max-width: 100%; height: auto; display: block; }
```

### Viewport meta tag

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

### Actividad práctica

- Crea una página con tipografía que escale con `clamp()` y un layout que cambie a 1, 2 o 3 columnas según el ancho.

---

## 12. Media Queries

Aplican estilos según las condiciones del dispositivo.

### Sintaxis

```css
/* Mobile first */
.caja { padding: 1rem; }

@media (min-width: 600px) {
  .caja { padding: 2rem; }
}

@media (min-width: 1024px) {
  .caja { padding: 3rem; }
}
```

### Tipos de medios

```css
@media screen { ... }
@media print { ... }
@media all { ... }
```

### Características

```css
/* Ancho */
@media (min-width: 768px) { ... }
@media (max-width: 767px) { ... }
@media (min-width: 768px) and (max-width: 1023px) { ... }

/* Alto */
@media (min-height: 600px) { ... }

/* Orientación */
@media (orientation: landscape) { ... }
@media (orientation: portrait) { ... }

/* Relación de píxeles (Retina) */
@media (-webkit-min-device-pixel-ratio: 2), (min-resolution: 192dpi) { ... }

/* Modo oscuro */
@media (prefers-color-scheme: dark) { ... }

/* Movimiento reducido */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}

/* Puntero fino/grueso */
@media (pointer: coarse) { ... } /* táctil */
@media (hover: hover) { ... }    /* mouse */
```

### Operadores lógicos

```css
@media (min-width: 600px) and (max-width: 1024px) { ... }
@media (max-width: 600px), (orientation: portrait) { ... }
@media not all and (monochrome) { ... }
```

### Estilos para impresión

```css
@media print {
  body { font-size: 12pt; color: black; background: white; }
  .no-print { display: none; }
  a::after { content: " (" attr(href) ")"; }
}
```

### Actividad práctica

- Diseña una página que tenga 3 breakpoints (móvil, tablet, escritorio) y que cambie la disposición del menú.

---

## 13. Variables CSS (Custom Properties)

Permiten definir valores reutilizables y dinámicos.

### Definición y uso

```css
:root {
  --color-primario: #0a7d54;
  --color-secundario: #ff6b35;
  --fuente-base: 'Inter', sans-serif;
  --radio: 8px;
  --espaciado: 1rem;
}

body {
  font-family: var(--fuente-base);
  color: var(--color-primario);
  padding: var(--espaciado);
}

button {
  background: var(--color-primario);
  border-radius: var(--radio);
}
```

### Valor de respaldo

```css
color: var(--color-texto, black);
```

### Variables con scope local

```css
.card {
  --color-fondo: white;
  background: var(--color-fondo);
}
.card.dark {
  --color-fondo: #222;
}
```

### Temas dinámicos

```css
:root { --bg: white; --text: black; }
:root[data-theme="dark"] { --bg: #111; --text: #eee; }

body { background: var(--bg); color: var(--text); }
```

```html
<html data-theme="light">
  <button onclick="document.documentElement.dataset.theme =
                   document.documentElement.dataset.theme === 'dark'
                     ? 'light' : 'dark'">
    Cambiar tema
  </button>
</html>
```

### Manipulación con JS

```html
<div id="box" style="--size: 100px; width: var(--size); height: var(--size);"></div>
<button onclick="box.style.setProperty('--size', '200px')">Crecer</button>
```

### `calc()` con variables

```css
:root { --base: 1rem; }
h1 { font-size: calc(var(--base) * 2); }
```

### Actividad práctica

- Implementa un toggle de tema claro/oscuro usando variables CSS y un botón.

---

## 14. Animaciones

CSS permite crear animaciones con `@keyframes` y la propiedad `animation`.

### Sintaxis

```css
@keyframes mover {
  from { transform: translateX(0); }
  to   { transform: translateX(200px); }
}

.caja {
  animation: mover 2s ease-in-out infinite alternate;
}
```

### Propiedades

```css
.animado {
  animation-name: mover;
  animation-duration: 2s;
  animation-timing-function: ease;
  animation-delay: 0.5s;
  animation-iteration-count: infinite;
  animation-direction: alternate;
  animation-fill-mode: forwards;
  animation-play-state: running;
}
```

### Múltiples animaciones

```css
.animado {
  animation: mover 2s ease, girar 4s linear infinite;
}
```

### Keyframes intermedios

```css
@keyframes pulso {
  0%   { transform: scale(1);   opacity: 1; }
  50%  { transform: scale(1.1); opacity: 0.7; }
  100% { transform: scale(1);   opacity: 1; }
}
```

### Loading spinner

```css
@keyframes spin {
  to { transform: rotate(360deg); }
}
.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #ccc;
  border-top-color: #0a7d54;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}
```

### Accesibilidad

```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}
```

### Actividad práctica

- Crea un loader (spinner) y un botón que al hacer click ejecute una animación de "shake" si falla la validación.

---

## 15. Transiciones

Suavizan el cambio de una propiedad cuando su valor cambia.

### Sintaxis

```css
.boton {
  background: #0a7d54;
  transition: background 0.3s ease, transform 0.2s ease;
}
.boton:hover {
  background: #054a32;
  transform: translateY(-2px);
}
```

### Propiedades

```css
.boton {
  transition-property: background, transform;
  transition-duration: 0.3s, 0.2s;
  transition-timing-function: ease, ease-out;
  transition-delay: 0s, 0.1s;
  transition: all 0.3s ease; /* shorthand */
}
```

### Funciones de timing

- `linear`, `ease`, `ease-in`, `ease-out`, `ease-in-out`
- `cubic-bezier(x1, y1, x2, y2)` personalizada
- `steps(n, start|end)` (efecto de pasos)

### Qué se puede y qué no

- ✅ Se animan: `transform`, `opacity`, `color`, `background-color`, `box-shadow`, `filter`, etc.
- ❌ No se animan (sin truco): `display`, `height: auto`, `top` con `position: static`.

### Truco para animar altura

```css
.card {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.4s ease;
}
.card.open { max-height: 500px; }
```

### Actividad práctica

- Crea un menú desplegable y un botón "hover" con transición suave.

---

## 16. Transform

Aplica transformaciones 2D/3D a un elemento.

### Funciones 2D

```css
.caja {
  transform: translate(50px, 100px);     /* mover */
  transform: rotate(45deg);              /* rotar */
  transform: scale(1.5);                 /* escalar (1.5x) */
  transform: scaleX(0.5) scaleY(2);      /* ejes individuales */
  transform: skewX(20deg);               /* inclinar */
}

.caja {
  transform: translate(50px, 50px) rotate(45deg) scale(1.2);
}
```

### `transform-origin`

```css
.caja {
  transform-origin: center;   /* default */
  transform-origin: top left;
  transform-origin: 0% 100%;
}
```

### Funciones 3D

```css
.caja {
  transform: perspective(500px) rotateY(45deg);
  transform-style: preserve-3d; /* para hijos */
}
```

### `backface-visibility`

```css
.tarjeta {
  backface-visibility: hidden;
}
```

### Card flip

```html
<div class="flip">
  <div class="flip-inner">
    <div class="flip-front">Frente</div>
    <div class="flip-back">Reverso</div>
  </div>
</div>
```

```css
.flip { perspective: 1000px; width: 200px; height: 300px; }
.flip-inner {
  position: relative;
  width: 100%; height: 100%;
  transition: transform 0.6s;
  transform-style: preserve-3d;
}
.flip:hover .flip-inner { transform: rotateY(180deg); }
.flip-front, .flip-back {
  position: absolute; inset: 0;
  backface-visibility: hidden;
}
.flip-back { transform: rotateY(180deg); }
```

### `will-change` (optimización)

```css
.animado { will-change: transform, opacity; }
```

> Úsalo solo cuando sepas que algo va a animar.

### Actividad práctica

- Crea un card flip 3D al hacer hover.

---

## 17. Pseudo-clases

Seleccionan elementos según estado o posición.

### Estructurales

```css
:first-child       /* primer hijo */
:last-child        /* último hijo */
:nth-child(n)      /* enésimo hijo */
:nth-last-child(n) /* enésimo desde el final */
:only-child        /* hijo único */
:first-of-type     /* primer elemento de su tipo */
:last-of-type      /* último de su tipo */
:nth-of-type(n)    /* enésimo de su tipo */
:only-of-type      /* único de su tipo */
:empty             /* sin hijos ni texto */
:root              /* elemento raíz del documento */
```

### De estado

```css
:hover     /* cursor encima */
:focus     /* con foco (input, link) */
:active    /* siendo pulsado */
:visited   /* enlace visitado */
:link      /* enlace no visitado */
:target    /* destino del ancla actual */
:checked   /* checkbox/radio/option marcado */
:disabled  /* deshabilitado */
:enabled   /* habilitado */
:required  /* campo obligatorio */
:optional  /* campo no obligatorio */
:valid     /* válido */
:invalid   /* inválido */
:read-only /* solo lectura */
:placeholder-shown /* mostrando placeholder */
:focus-within /* el elemento o un descendiente tiene foco */
:focus-visible /* foco visible por teclado */
```

### Ejemplos prácticos

```css
/* Filas alternas */
tr:nth-child(odd)  { background: #f4f4f4; }
tr:nth-child(even) { background: white; }

/* Primer y último elemento de una lista */
li:first-child { border-top-left-radius: 8px; }
li:last-child  { border-bottom-right-radius: 8px; }

/* Input con foco */
input:focus {
  outline: none;
  border-color: #0a7d54;
  box-shadow: 0 0 0 3px rgba(10,125,84,0.2);
}

/* Botón deshabilitado */
button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

### Actividad práctica

- Estiliza un formulario con estados `:focus`, `:invalid`, `:disabled` y aplica zebra striping a una tabla.

---

## 18. Pseudo-elementos

Permiten estilizar **partes específicas** de un elemento. Sintaxis `::` (doble).

```css
::before      /* crea contenido antes del contenido */
::after       /* crea contenido después del contenido */
::first-line  /* primera línea */
::first-letter/* primera letra */
::placeholder /* placeholder del input */
::selection   /* selección de texto */
::marker      /* marcador de li */
::backdrop    /* detrás de un dialog */
```

### `::before` y `::after`

```css
.icono-flecha::after {
  content: " →";
  color: gray;
}

.quote::before {
  content: "“";
  font-size: 3rem;
  color: #0a7d54;
}
.quote::after {
  content: "”";
  font-size: 3rem;
  color: #0a7d54;
}
```

### Iconos decorativos

```html
<a href="#" class="externo">Sitio externo</a>
```

```css
.externo::after {
  content: " ↗";
}
```

### `::selection`

```css
::selection {
  background: #0a7d54;
  color: white;
}
```

### `::placeholder`

```css
input::placeholder {
  color: #999;
  font-style: italic;
}
```

### `::marker` (viñetas de lista)

```css
li::marker {
  color: #0a7d54;
  font-weight: bold;
}
```

### Decoración con `::before` y `::after`

```css
.titulo-decorado {
  position: relative;
  display: inline-block;
}
.titulo-decorado::before,
.titulo-decorado::after {
  content: "";
  display: inline-block;
  width: 20px;
  height: 2px;
  background: #0a7d54;
  vertical-align: middle;
  margin: 0 0.5rem;
}
```

### Actividad práctica

- Decora los títulos de tu página con `::before` y `::after`. Cambia el color de selección.

---

## 19. Organización del CSS

Mantener CSS ordenado es clave en proyectos grandes.

### Buenas prácticas

1. **Un archivo base** (`reset.css` o `normalize.css`).
2. **Variables y tokens** en `:root`.
3. **Estructura por capas**:
   - `1-reset.css`
   - `2-variables.css`
   - `3-base.css` (tipografía, body, etc.)
   - `4-layout.css` (header, main, footer)
   - `5-components/` (button.css, card.css, modal.css)
   - `6-pages/` (home.css, blog.css)
   - `7-utilities.css` (`.text-center`, `.mt-1`, etc.)
4. **Mobile first**.
5. **Evitar selectores muy específicos** (no más de 2-3 niveles).
6. **No usar `id` para estilos** (clases son reutilizables).
7. **Comentar secciones**.

### Estructura de carpetas recomendada

```
styles/
├── base/
│   ├── reset.css
│   ├── typography.css
│   └── variables.css
├── layout/
│   ├── header.css
│   ├── grid.css
│   └── footer.css
├── components/
│   ├── button.css
│   ├── card.css
│   └── modal.css
├── pages/
│   ├── home.css
│   └── blog.css
└── main.css   /* importa todos */
```

### `@import` (no recomendado en producción)

```css
@import url("base/reset.css");
@import url("base/variables.css");
```

Mejor usar un bundler (Vite, PostCSS, Sass).

### Actividad práctica

- Reorganiza el CSS de un proyecto pequeño siguiendo la estructura por capas.

---

## 20. Metodologías: BEM

**BEM** (Block Element Modifier) es una convención de nombres que mejora la legibilidad y reduce conflictos.

### Conceptos

- **Block**: componente independiente (`header`, `card`, `button`).
- **Element**: parte de un bloque (`card__title`, `card__image`).
- **Modifier**: variación o estado (`card--dark`, `button--disabled`).

### Convenciones

```css
.bloque {}
.bloque__elemento {}
.bloque--modificador {}
.bloque__elemento--modificador {}
```

### Ejemplo: tarjeta

```html
<article class="card card--featured">
  <img class="card__image" src="..." alt="...">
  <div class="card__body">
    <h3 class="card__title">Título</h3>
    <p class="card__text">Texto de la tarjeta...</p>
    <button class="card__button card__button--primary">Ver más</button>
  </div>
</article>
```

```css
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}
.card--featured {
  border-color: #0a7d54;
  box-shadow: 0 4px 10px rgba(0,0,0,.1);
}
.card__image {
  width: 100%;
  display: block;
}
.card__body { padding: 1rem; }
.card__title { margin: 0 0 .5rem; }
.card__text { color: #555; }
.card__button {
  display: inline-block;
  padding: .5rem 1rem;
  border: 0;
  border-radius: 4px;
  background: #ccc;
  cursor: pointer;
}
.card__button--primary {
  background: #0a7d54;
  color: white;
}
```

### Ventajas de BEM

- **Legibilidad**: el HTML describe su estructura.
- **Aislamiento**: cada bloque es independiente.
- **Menos conflictos de nombres**.
- **Facilidad de refactor**.

### Alternativas

- **OOCSS** (Object-Oriented CSS).
- **SMACSS** (Scalable and Modular Architecture).
- **ITCSS** (Inverted Triangle).
- **Atomic CSS** (Tailwind, UnoCSS).
- **CSS Modules** (con bundler).
- **CSS-in-JS** (Styled Components, Emotion).

### Actividad práctica

- Convierte los estilos de tu proyecto a BEM y verifica que no haya colisiones.

---

## Recursos complementarios

### Lecturas

- [MDN CSS](https://developer.mozilla.org/es/docs/Web/CSS)
- [CSS Tricks](https://css-tricks.com/)
- [web.dev - Learn CSS](https://web.dev/learn/css/)
- [Specificity Calculator](https://specificity.keegan.st/)

### Herramientas

- [Can I Use](https://caniuse.com/)
- [CSS Validator](https://jigsaw.w3.org/css-validator/)
- [Flexbox Froggy](https://flexboxfroggy.com/) (juego)
- [Grid Garden](https://cssgridgarden.com/) (juego)
- [CSS Gradient](https://cssgradient.io/)

### Cursos interactivos

- [Codecademy - Learn CSS](https://www.codecademy.com/learn/learn-css)
- [freeCodeCamp - Responsive Web Design](https://www.freecodecamp.org/)

---

## Evaluación sugerida

1. **Cuestionario (20%)**: especificidad, cascada, box model.
2. **Layout Flexbox (20%)**: navbar y cards responsivas.
3. **Layout Grid (20%)**: maquetar una página completa con áreas.
4. **Animaciones y transiciones (20%)**: landing con interactividad.
5. **Refactor con BEM (20%)**: tomar un CSS existente y renombrarlo.

---

> **Nota para el estudiante**: CSS es un lenguaje poderoso y muy profundo. Domina Flexbox y Grid antes de saltar a frameworks como Tailwind o Bootstrap: te dará fundamentos sólidos.
