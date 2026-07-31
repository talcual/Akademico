# Módulo 6: Manipulación del DOM

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Comprender el modelo de objetos del documento (DOM) y el BOM.
- Seleccionar y modificar nodos del DOM con `querySelector` y APIs relacionadas.
- Manejar eventos del navegador, incluyendo delegación.
- Construir y validar formularios accesibles.
- Renderizar contenido dinámicamente con JavaScript puro.
- Crear componentes reutilizables sin frameworks.

---

## 1. DOM (Document Object Model)

El DOM es una **representación en árbol** del documento HTML que el navegador carga en memoria. Permite a JavaScript acceder y modificar la estructura, el contenido y los estilos de una página.

### Tipos de nodos

```js
// Tipos principales de Node
document.nodeType // 9 (Document)
element.nodeType  // 1 (Element)
text.nodeType     // 3 (Text)
comment.nodeType  // 8 (Comment)
```

### Árbol del DOM

```
Document
 └── html
     ├── head
     │   ├── title
     │   └── meta
     └── body
         ├── header
         │   ├── h1
         │   └── nav
         └── main
             ├── article
             └── aside
```

### Selección del nodo raíz

```js
document;       // documento
document.documentElement; // <html>
document.head;  // <head>
document.body;  // <body>
```

### Crear y modificar nodos

```js
const div = document.createElement("div");
div.id = "nuevo";
div.className = "card";
div.textContent = "Hola mundo";
div.innerHTML = "<strong>HTML</strong>";

document.body.appendChild(div);
```

### Tipos de relaciones

```js
elemento.parentElement;
elemento.children;       // solo elementos
elemento.childNodes;     // todos los nodos (incluye texto)
elemento.firstElementChild;
elemento.lastElementChild;
elemento.previousElementSibling;
elemento.nextElementSibling;
```

### Insertar, mover, reemplazar, eliminar

```js
const padre = document.querySelector("#lista");
const nuevo = document.createElement("li");

padre.appendChild(nuevo);                // al final
padre.prepend(nuevo);                    // al inicio
padre.insertBefore(nuevo, hijoExistente);// antes de otro nodo

// Insertar HTML adyacente
nuevo.insertAdjacentHTML("beforebegin", "<p>Antes</p>");
nuevo.insertAdjacentHTML("afterend",   "<p>Después</p>");
nuevo.insertAdjacentHTML("afterbegin", "<span>inicio</span>");
nuevo.insertAdjacentHTML("beforeend",  "<span>fin</span>");

padre.replaceChild(nuevoNuevo, viejo);
padre.removeChild(viejo);

// Moderno (sin padre)
nuevo.remove();

// Fragment (optimización)
const frag = document.createDocumentFragment();
for (let i = 0; i < 5; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  frag.appendChild(li);
}
padre.appendChild(frag); // 1 solo reflow
```

### Modificar atributos

```js
const img = document.querySelector("img");

img.getAttribute("alt");
img.setAttribute("alt", "descripción");
img.removeAttribute("alt");
img.hasAttribute("alt");
img.toggleAttribute("loading");

// Como propiedades (más rápido)
img.src = "nuevo.jpg";
img.alt  = "texto";
img.id   = "main";

// Dataset (data-*)
img.dataset.id = "1";  // <img data-id="1">
console.log(img.dataset.id);
```

### Clases

```js
el.classList.add("activo");
el.classList.remove("oculto");
el.classList.toggle("seleccionado");
el.classList.replace("antiguo", "nuevo");
el.classList.contains("activo");
el.className = "a b c"; // reemplaza todo
```

### Estilos en línea

```js
el.style.backgroundColor = "#0a7d54";
el.style.cssText = "color: red; padding: 1rem;";

// Variables CSS
el.style.setProperty("--size", "200px");
getComputedStyle(el).getPropertyValue("--size");
```

### Texto vs HTML

```js
el.textContent = "Texto seguro (sin HTML)";
el.innerHTML   = "<strong>HTML</strong>"; // ⚠️ posible XSS

// Saneado básico
const limpio = document.createTextNode(textoUsuario);
padre.appendChild(limpio);
```

### `MutationObserver`

```js
const obs = new MutationObserver((mutations) => {
  console.log("Cambios:", mutations);
});
obs.observe(document.body, { childList: true, subtree: true });
```

### Actividad práctica

- Crea una función `construirCard({titulo, texto, img})` que devuelva un elemento DOM totalmente armado.

---

## 2. BOM (Browser Object Model)

Es el conjunto de objetos que expone el navegador fuera del documento.

### `window`

```js
window.innerWidth;       // viewport
window.innerHeight;
window.scrollY;          // scroll vertical
window.scrollX;
window.location.href;    // URL completa
window.location.pathname;
window.location.search;  // query string
window.history.length;
window.navigator.userAgent;
window.localStorage;
window.sessionStorage;
window.document;
```

### `location`

```js
location.href = "https://example.com";
location.reload();
location.assign("/nueva");
location.replace("/otra"); // sin guardar historial
const params = new URLSearchParams(location.search);
params.get("id");
```

### `history`

```js
history.back();
history.forward();
history.go(-2);

history.pushState({ id: 1 }, "", "/ruta"); // SPA
history.replaceState({ id: 2 }, "", "/otra");
window.addEventListener("popstate", (e) => {
  console.log("estado:", e.state);
});
```

### `navigator`

```js
navigator.userAgent;
navigator.language;        // "es-ES"
navigator.languages;
navigator.onLine;          // boolean
navigator.cookieEnabled;
navigator.platform;
navigator.clipboard.writeText("texto"); // portapapeles
navigator.share({ title: "...", url: "..." });
navigator.geolocation.getCurrentPosition(...);
navigator.mediaDevices.getUserMedia({ video: true });
```

### `screen`

```js
screen.width;
screen.height;
screen.availWidth;
screen.orientation.type;
```

### Diálogos nativos

```js
alert("Hola");
const ok = confirm("¿Continuar?");
const txt = prompt("Tu nombre");
```

### Temporizadores

```js
const id = setTimeout(() => console.log("t"), 1000);
clearTimeout(id);

const i = setInterval(() => console.log("tick"), 1000);
clearInterval(i);
```

### `requestAnimationFrame`

```js
let raf;
function frame(t) {
  console.log(t);
  raf = requestAnimationFrame(frame);
}
raf = requestAnimationFrame(frame);
cancelAnimationFrame(raf);
```

### Actividad práctica

- Crea un componente que lea y muestre los parámetros de la URL (`?utm=...`, `?id=...`).

---

## 3. QuerySelector y QuerySelectorAll

API moderna para seleccionar elementos con selectores CSS.

### Básicos

```js
const header = document.querySelector("header");
const main   = document.querySelector("main");
const item   = document.querySelector(".item");
const link   = document.querySelector('a[href^="https"]');
const id     = document.querySelector("#titulo");
const primero = document.querySelector("ul li:first-child");

// Todos
const todos = document.querySelectorAll(".item");
```

### Recorrer resultados

```js
const items = document.querySelectorAll(".item");

items.forEach(el => el.classList.add("visible"));

// Convertir a array
const arr = [...items];
arr.map(el => el.id);

// Filtrar
const visibles = [...items].filter(el =>
  el.getBoundingClientRect().top < window.innerHeight
);
```

### Otros métodos legacy (útiles en HTMLCollection viva)

```js
document.getElementById("id");
document.getElementsByClassName("item");
document.getElementsByTagName("li");
document.getElementsByName("email");
```

### `closest()` y `matches()`

```js
item.closest(".card");    // padre (o sí mismo) que coincida
item.matches(".destacado"); // boolean

// Caso típico (delegación de eventos)
btn.closest("li").remove();
```

### `root`

```js
const root = document.querySelector(":root"); // <html>
```

### `querySelector` vs `getElementById`

- `getElementById` es ligeramente más rápido y devuelve referencia directa.
- `querySelector` es más flexible pero menos performante en bucles grandes.

### Ámbitos

```js
// Buscar dentro de un subárbol
const lista = document.querySelector("#lista");
const item  = lista.querySelector(".item");
```

### Actividad práctica

- Construye un helper `$$(selector, root = document)` que devuelva un array.

---

## 4. Eventos (ampliación desde el DOM)

Ya cubiertos en el Módulo 5, pero aquí añadimos lo esencial para manipular el DOM.

### Eventos clásicos

```js
element.addEventListener("click", (e) => {
  console.log(e.target.textContent);
});
```

### Eventos de mouse

```js
"click", "dblclick",
"mousedown", "mouseup",
"mousemove",
"mouseenter", "mouseleave",
"mouseover", "mouseout",
"contextmenu"
```

### Eventos de teclado

```js
input.addEventListener("keydown", (e) => {
  if (e.key === "Enter") submit();
  if (e.ctrlKey && e.key === "k") openCommandK();
});

input.addEventListener("keyup", (e) => {});
input.addEventListener("input", (e) => {
  console.log(e.target.value);
});
```

### Eventos de formulario

```js
form.addEventListener("submit", onSubmit);
input.addEventListener("input",   onChange);
input.addEventListener("change",  onCommit);
input.addEventListener("focus",   onFocus);
input.addEventListener("blur",    onBlur);
select.addEventListener("change", onSelect);
```

### Eventos de UI

```js
window.addEventListener("resize",   onResize);
window.addEventListener("scroll",   onScroll);
window.addEventListener("load",     onLoad);
document.addEventListener("DOMContentLoaded", onReady);
window.addEventListener("beforeunload", (e) => {
  e.preventDefault();
  e.returnValue = "";
});
```

### Eventos del portapapeles

```js
el.addEventListener("copy",  e => console.log("copió"));
el.addEventListener("paste", e => console.log("pegó"));
```

### `passive` y `once`

```js
btn.addEventListener("click", handler, { once: true });
window.addEventListener("scroll", handler, { passive: true });
```

### Actividad práctica

- Crea un tooltip que se muestre al hacer `mouseenter` en un botón y se oculte al salir.

---

## 5. Delegación de eventos

Consiste en **escuchar eventos en un ancestro** y reaccionar según el `target`. Reduce el número de listeners y funciona con contenido dinámico.

### Caso básico

```html
<ul id="lista">
  <li>Item 1</li>
  <li>Item 2</li>
  <li>Item 3</li>
</ul>
```

```js
const lista = document.querySelector("#lista");

lista.addEventListener("click", (e) => {
  if (e.target.matches("li")) {
    e.target.classList.toggle("done");
  }
});
```

### Caso dinámico (elementos creados después)

```js
// Los nuevos <li> también responderán sin re-asignar listeners
lista.addEventListener("click", (e) => {
  const li = e.target.closest("li");
  if (!li) return;
  console.log("Click en:", li.textContent);
});

const nuevo = document.createElement("li");
nuevo.textContent = "Item 4";
lista.appendChild(nuevo); // funciona automáticamente
```

### Botones de eliminar en una lista

```html
<ul id="todo"></ul>
<template id="tpl">
  <li>
    <span class="text"></span>
    <button class="delete">×</button>
  </li>
</template>
```

```js
const ul = document.querySelector("#todo");

ul.addEventListener("click", (e) => {
  const btn = e.target.closest("button.delete");
  if (!btn) return;
  btn.closest("li").remove();
});

function addTask(text) {
  const tpl = document.querySelector("#tpl").content.cloneNode(true);
  tpl.querySelector(".text").textContent = text;
  ul.appendChild(tpl);
}
```

### Buenas prácticas

- Usa `closest()` para encontrar el elemento relevante.
- Salir temprano si `e.target` no coincide.
- Usa `e.currentTarget` si necesitas el contenedor con el listener.

### Actividad práctica

- Implementa una tabla con botones "editar" y "eliminar" por fila, usando delegación.

---

## 6. Formularios

### Crear y leer

```html
<form id="form">
  <input name="email" type="email">
  <input name="pwd" type="password">
  <button>Enviar</button>
</form>
```

```js
const form = document.querySelector("#form");

form.addEventListener("submit", (e) => {
  e.preventDefault();
  const data = new FormData(form);
  const obj  = Object.fromEntries(data);
  console.log(obj);
});
```

### `FormData`

```js
const fd = new FormData();
fd.append("nombre", "Ana");
fd.append("tags", "html");
fd.append("tags", "css");
fd.get("nombre");
fd.getAll("tags");
fd.has("email");
fd.delete("pwd");
fd.set("nombre", "Luis");

// Enviar a servidor
await fetch("/api", { method: "POST", body: fd });

// Subir archivo
fd.append("avatar", file);
```

### `reset()` programático

```js
form.reset();
```

### Acceso directo a inputs

```js
form.elements.email.value;
form.elements["email"].value;
```

### Mostrar/Ocultar contraseñas

```html
<input type="password" id="pwd">
<button type="button" id="toggle">👁</button>
```

```js
document.querySelector("#toggle").addEventListener("click", () => {
  const input = document.querySelector("#pwd");
  input.type = input.type === "password" ? "text" : "password";
});
```

### Contador de caracteres

```html
<textarea maxlength="500" id="msg"></textarea>
<span id="contador">0/500</span>
```

```js
const msg = document.querySelector("#msg");
const counter = document.querySelector("#contador");
msg.addEventListener("input", () => {
  counter.textContent = `${msg.value.length}/500`;
});
```

### Select personalizado (parte visual)

```html
<select id="country">
  <option value="">Selecciona</option>
  <option value="ve">Venezuela</option>
  <option value="mx">México</option>
</select>
```

```js
const sel = document.querySelector("#country");
sel.addEventListener("change", (e) => {
  console.log("Seleccionado:", e.target.value);
});
```

### Actividad práctica

- Construye un formulario completo con inputs, textarea, select, file y un preview de la imagen seleccionada.

---

## 7. Validaciones

JS permite validar **antes** y **después** del envío, complementando la validación HTML5.

### Estados de validación

```js
const input = document.querySelector("#email");

input.checkValidity();      // boolean
input.reportValidity();     // muestra mensaje y devuelve boolean
input.validity.valid;       // boolean
input.validity.valueMissing;
input.validity.typeMismatch;
input.validity.patternMismatch;
input.validity.tooShort;
input.validity.rangeUnderflow;
input.validity.rangeOverflow;
input.validity.customError;
input.validationMessage;
```

### Validación custom

```js
input.addEventListener("input", (e) => {
  if (e.target.value.includes("@admin")) {
    e.target.setCustomValidity("No se permite @admin");
  } else {
    e.target.setCustomValidity("");
  }
});
```

### Restricciones visuales

```css
input:invalid { border-color: red; }
input:valid   { border-color: green; }
input:focus:invalid { outline: 2px solid red; }
```

### Mensaje personalizado

```js
input.addEventListener("invalid", (e) => {
  e.target.setCustomValidity("Por favor, completa este campo correctamente");
});
```

### Validador reusable

```js
class FormValidator {
  constructor(form, rules) {
    this.form = form;
    this.rules = rules;
    form.addEventListener("submit", (e) => {
      e.preventDefault();
      if (this.validateAll()) this.onSuccess(this.getData());
      else this.onError(this.errors);
    });
  }

  getData() {
    return Object.fromEntries(new FormData(this.form));
  }

  validateAll() {
    this.errors = {};
    Object.entries(this.rules).forEach(([field, validators]) => {
      const value = this.form.elements[field]?.value;
      for (const v of validators) {
        const err = v(value, this.getData());
        if (err) {
          this.errors[field] = err;
          this.form.elements[field].setCustomValidity(err);
          break;
        }
      }
    });
    return Object.keys(this.errors).length === 0;
  }

  onSuccess(_) {}
  onError(_) {}
}

new FormValidator(form, {
  email: [v => (!v ? "Requerido" : null),
          v => (!v.includes("@") ? "Inválido" : null)],
  pwd:   [v => (v?.length < 8 ? "Mínimo 8" : null)],
}).onSuccess = data => console.log(data);
```

### Confirmación previa (al salir con cambios)

```js
let cambios = false;
form.addEventListener("input", () => (cambios = true));
window.addEventListener("beforeunload", (e) => {
  if (cambios) {
    e.preventDefault();
    e.returnValue = "";
  }
});
```

### Actividad práctica

- Implementa validación de email, contraseña (8+ caracteres, mayúscula y número), y coincidencia de confirmación.

---

## 8. Renderizado dinámico

### Patrón: render template

```html
<ul id="users"></ul>
<template id="user-tpl">
  <li class="user">
    <strong class="name"></strong>
    <span class="email"></span>
  </li>
</template>
```

```js
function renderUsers(users) {
  const ul = document.querySelector("#users");
  const tpl = document.querySelector("#user-tpl");
  ul.innerHTML = "";

  const frag = document.createDocumentFragment();
  users.forEach(u => {
    const node = tpl.content.cloneNode(true);
    node.querySelector(".name").textContent = u.name;
    node.querySelector(".email").textContent = u.email;
    frag.appendChild(node);
  });
  ul.appendChild(frag);
}
```

### Render funcional

```js
function el(tag, props = {}, children = []) {
  const node = document.createElement(tag);
  for (const [k, v] of Object.entries(props)) {
    if (k === "class") node.className = v;
    else if (k === "style" && typeof v === "object") Object.assign(node.style, v);
    else if (k.startsWith("on") && typeof v === "function") {
      node.addEventListener(k.slice(2).toLowerCase(), v);
    } else if (k in node) {
      node[k] = v;
    } else {
      node.setAttribute(k, v);
    }
  }
  children.forEach(c => {
    if (typeof c === "string") node.appendChild(document.createTextNode(c));
    else if (c instanceof Node) node.appendChild(c);
  });
  return node;
}

const card = el("div", { class: "card", "data-id": 1 }, [
  el("h3", {}, ["Producto"]),
  el("p",  {}, ["Descripción"]),
  el("button", {
    onclick: () => alert("click"),
    class: "btn"
  }, ["Comprar"]),
]);
document.body.appendChild(card);
```

### Render reactivo (mini)

```js
function createState(initial) {
  let value = initial;
  const subs = new Set();
  return {
    get: () => value,
    set: (newValue) => {
      value = newValue;
      subs.forEach(fn => fn(value));
    },
    subscribe: (fn) => subs.add(fn),
  };
}

const count = createState(0);

count.subscribe(v => {
  document.querySelector("#count").textContent = v;
});

document.querySelector("#inc").onclick = () => count.set(count.get() + 1);
```

### Render incremental con `IntersectionObserver`

```js
const obs = new IntersectionObserver(async (entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      const data = await fetch(`/api/items?skip=${skip}`).then(r => r.json());
      renderItems(data);
      skip += 10;
    }
  }
});
obs.observe(document.querySelector("#sentinel"));
```

### Buenas prácticas de render

- Usa `DocumentFragment` para inserciones múltiples.
- Reemplaza contenido en un solo `replaceChildren()`.
- Evita `innerHTML` con datos no confiables.
- Usa `textContent` en lugar de `innerHTML` cuando sea posible.
- Cachea referencias a nodos frecuentemente usados.

### Actividad práctica

- Implementa un feed que renderice una lista de usuarios con scroll infinito.

---

## 9. Componentización con JavaScript puro

Patrón para construir componentes reutilizables sin frameworks.

### Componente "función" simple

```js
function Card({ title, body, onClick }) {
  const root = document.createElement("div");
  root.className = "card";
  root.innerHTML = `
    <h3>${title}</h3>
    <p>${body}</p>
    <button class="close">×</button>
  `;
  root.querySelector(".close").addEventListener("click", () => {
    root.remove();
    onClick?.();
  });
  return root;
}

const c = Card({
  title: "Hola",
  body: "Contenido",
  onClick: () => console.log("cerrado"),
});
document.body.appendChild(c);
```

### Clase con Shadow DOM

```js
class AppCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open" });
  }

  static get observedAttributes() {
    return ["title", "body"];
  }

  attributeChangedCallback() {
    this.render();
  }

  connectedCallback() {
    this.render();
  }

  render() {
    this.shadowRoot.innerHTML = `
      <style>
        :host { display: block; border: 1px solid #ddd;
                border-radius: 8px; padding: 1rem; }
        h3 { margin: 0 0 .5rem; color: #0a7d54; }
      </style>
      <h3>${this.getAttribute("title") || "Sin título"}</h3>
      <p>${this.getAttribute("body") || ""}</p>
      <slot></slot>
    `;
  }
}

customElements.define("app-card", AppCard);
```

```html
<app-card title="Hola" body="Mundo">
  Contenido proyectado
</app-card>
```

### Componente con estado

```js
class Counter extends HTMLElement {
  constructor() {
    super();
    this.count = 0;
    this.attachShadow({ mode: "open" });
  }

  connectedCallback() {
    this.render();
  }

  render() {
    this.shadowRoot.innerHTML = `
      <style>
        :host { display: inline-flex; gap: .5rem; align-items: center;
                font-family: sans-serif; }
        button { padding: .25rem .5rem; }
      </style>
      <button>-</button>
      <span>${this.count}</span>
      <button>+</button>
    `;
    const [minus, plus] = this.shadowRoot.querySelectorAll("button");
    minus.onclick = () => { this.count--; this.render(); };
    plus.onclick  = () => { this.count++; this.render(); };
  }
}
customElements.define("app-counter", Counter);
```

### Sistema de props con `properties`

```js
class UserCard extends HTMLElement {
  static get properties() {
    return { user: { type: Object, attribute: false } };
  }

  set user(value) {
    this._user = value;
    this.render();
  }
  get user() { return this._user; }

  connectedCallback() { this.render(); }

  render() {
    if (!this._user) return;
    this.innerHTML = `
      <div class="card">
        <h3>${this._user.name}</h3>
        <p>${this._user.email}</p>
      </div>
    `;
  }
}
customElements.define("user-card", UserCard);

const card = document.querySelector("user-card");
card.user = { name: "Ana", email: "ana@x.com" };
```

### Componente "Modal" reutilizable

```js
class AppModal extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `
      <dialog>
        <h2><slot name="title">Título</slot></h2>
        <div><slot></slot></div>
        <button class="close">Cerrar</button>
      </dialog>
    `;
    this.querySelector("dialog").addEventListener("close", () => {
      this.dispatchEvent(new CustomEvent("closed"));
    });
    this.querySelector(".close").onclick = () => {
      this.querySelector("dialog").close();
    };
  }

  open() {
    this.querySelector("dialog").showModal();
  }
  close() {
    this.querySelector("dialog").close();
  }
}
customElements.define("app-modal", AppModal);
```

```html
<app-modal id="m">
  <span slot="title">Hola</span>
  <p>Contenido del modal</p>
</app-modal>
<button onclick="document.querySelector('#m').open()">Abrir</button>
```

### Sistema de plantillas globales

```js
const Components = {
  card: ({ title, body }) => `
    <article class="card">
      <h3>${title}</h3>
      <p>${body}</p>
    </article>
  `,
  button: ({ label, onClick }) => `
    <button class="btn">${label}</button>
  `,
};

function renderComponent(name, props) {
  const html = Components[name](props);
  const wrapper = document.createElement("div");
  wrapper.innerHTML = html.trim();
  const el = wrapper.firstElementChild;
  if (props.onClick) el.addEventListener("click", props.onClick);
  return el;
}

document.body.appendChild(
  renderComponent("card", { title: "Hola", body: "Mundo" })
);
```

### Ciclo de vida

- `constructor`: inicialización.
- `connectedCallback`: insertado en DOM.
- `disconnectedCallback`: removido.
- `attributeChangedCallback`: cambio de atributo observado.
- `adoptedCallback`: movido a otro documento.

### Actividad práctica

- Implementa `<todo-list>` que permita agregar, marcar y eliminar tareas, con Shadow DOM y CustomEvents.

---

## Ejemplo integrador

```html
<!-- index.html -->
<div id="app">
  <form id="todo-form">
    <input id="todo-input" placeholder="Nueva tarea" maxlength="80" required>
    <button>Añadir</button>
  </form>
  <ul id="todo-list"></ul>
  <p id="contador"></p>
</div>

<template id="todo-tpl">
  <li class="todo">
    <input type="checkbox" class="toggle">
    <span class="text"></span>
    <button class="delete">×</button>
  </li>
</template>
```

```js
// app.js
const form = document.querySelector("#todo-form");
const input = document.querySelector("#todo-input");
const list = document.querySelector("#todo-list");
const contador = document.querySelector("#contador");
const tpl = document.querySelector("#todo-tpl");

const KEY = "todos";
let todos = JSON.parse(localStorage.getItem(KEY) || "[]");

const guardar = () => localStorage.setItem(KEY, JSON.stringify(todos));

const render = () => {
  list.innerHTML = "";
  const frag = document.createDocumentFragment();
  todos.forEach((todo, i) => {
    const node = tpl.content.cloneNode(true);
    node.querySelector(".text").textContent = todo.text;
    const check = node.querySelector(".toggle");
    check.checked = todo.done;
    check.addEventListener("change", () => {
      todos[i].done = !todos[i].done;
      guardar();
      render();
    });
    node.querySelector(".delete").addEventListener("click", () => {
      todos.splice(i, 1);
      guardar();
      render();
    });
    frag.appendChild(node);
  });
  list.appendChild(frag);
  contador.textContent = `${todos.length} tareas`;
};

form.addEventListener("submit", (e) => {
  e.preventDefault();
  const texto = input.value.trim();
  if (!texto) return;
  todos.push({ text: texto, done: false });
  input.value = "";
  guardar();
  render();
});

input.addEventListener("input", () => {
  input.setCustomValidity(input.value.trim() ? "" : "Vacío");
});

render();
```

---

## Recursos complementarios

### Lecturas

- [MDN - DOM](https://developer.mozilla.org/es/docs/Web/API/Document_Object_Model)
- [MDN - Element](https://developer.mozilla.org/es/docs/Web/API/Element)
- [JavaScript.info - DOM](https://javascript.info/dom-nodes)

### Práctica

- [JavaScript30](https://javascript30.com/)
- [Frontend Mentor](https://www.frontendmentor.io/)

### Herramientas

- [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/)
- [axe DevTools](https://www.deque.com/axe/devtools/)
- [Performance Insights (Chrome)](https://developer.chrome.com/docs/devtools/performance-insights/)

---

## Evaluación sugerida

1. **Render dinámico (30%)**: lista con template + delegación de eventos.
2. **Validación de formulario (25%)**: reglas custom + mensajes.
3. **Web Component (30%)**: Custom Element con Shadow DOM.
4. **Refactor de código (15%)**: convertir innerHTML manual a una función `el()`.

---

> **Nota para el estudiante**: entender el DOM a fondo te permite sobrevivir sin frameworks y, cuando uses uno (React, Vue, Svelte), comprenderás mejor por qué hace lo que hace.
