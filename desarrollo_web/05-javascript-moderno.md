# Módulo 5: JavaScript Moderno

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Organizar código en módulos ES6 usando `import` y `export`.
- Modelar entidades con clases, herencia y campos privados.
- Manejar asincronía con promesas y `async/await`.
- Realizar peticiones HTTP con la Fetch API y consumir JSON.
- Persistir datos en el navegador con `localStorage`, `sessionStorage` y cookies.
- Capturar y manejar errores con `try/catch`.
- Trabajar con el DOM a través de eventos.
- Comprender el Event Loop, microtasks y macrotasks.

---

## 1. Módulos ES6

Los módulos permiten dividir el código en archivos separados, con **scope propio** y posibilidad de **importar/exportar** partes.

### Tipos de módulos

- **ES Modules (ESM)**: estándar oficial, usado en navegadores y en Node.js con `"type": "module"` o `.mjs`.
- **CommonJS**: `require`/`module.exports`, original de Node.js.
- **AMD**: histórico, usado en RequireJS.

### En HTML

```html
<script type="module" src="app.js"></script>
```

### Activar ESM en Node.js

```json
// package.json
{
  "type": "module"
}
```

```js
// archivo.mjs o con type: module
import { sumar } from "./math.js";
```

### Características

- Carga **estática y perezosa**.
- Cada módulo se ejecuta **una sola vez**.
- Usa **modo estricto** por defecto.
- Variables no contaminan el scope global.
- Soporta **top-level await**.

### Actividad práctica

- Crea dos archivos `saludo.js` y `app.js`, importa una función del primero y ejecútala desde el segundo.

---

## 2. Import / Export

### Export nombrado (named exports)

```js
// math.js
export const sumar = (a, b) => a + b;
export const restar = (a, b) => a - b;

const PI = 3.1416;
const E = 2.718;
export { PI, E };
```

```js
// app.js
import { sumar, restar, PI } from "./math.js";
console.log(sumar(2, 3)); // 5
```

### Renombrar al importar

```js
import { sumar as add, restar as subtract } from "./math.js";
```

### Importar todo como namespace

```js
import * as math from "./math.js";
math.sumar(2, 3); // 5
math.PI;          // 3.1416
```

### Export por defecto (default)

Un módulo puede tener **solo un export default**.

```js
// user.js
export default class User {
  constructor(nombre) { this.nombre = nombre; }
  saludar() { return `Hola, soy ${this.nombre}`; }
}
```

```js
// app.js
import User from "./user.js"; // sin llaves
const u = new User("Ana");
```

### Mezcla de default + nombrado

```js
// logger.js
export default function log(msg) { console.log(msg); }
export const LEVEL = "info";
```

```js
import log, { LEVEL } from "./logger.js";
```

### Re-exports (barrel file)

```js
// index.js (carpeta components)
export { default as Button } from "./Button.js";
export { default as Card }   from "./Card.js";
export { default as Modal }  from "./Modal.js";
```

```js
import { Button, Card, Modal } from "./components/index.js";
```

### Import dinámico (`import()`)

```js
const modulo = await import("./modulo-pesado.js");
modulo.hacerAlgo();
```

Útil para code-splitting y carga condicional.

```js
button.addEventListener("click", async () => {
  const { Chart } = await import("./chart.js");
  new Chart(datos);
});
```

### Top-level await (ES2022)

```js
// Solo en módulos
const respuesta = await fetch("/api/config");
const config = await respuesta.json();
```

### Actividad práctica

- Crea una carpeta `utils/` con archivos `math.js`, `string.js` y un `index.js` que reexporte todo. Úsalo desde `app.js`.

---

## 3. Clases

Azúcar sintáctica sobre los prototipos, introducida en ES6.

### Declaración

```js
class Persona {
  constructor(nombre, edad) {
    this.nombre = nombre;
    this.edad = edad;
  }

  saludar() {
    return `Hola, soy ${this.nombre}`;
  }

  despedir() {
    return `Adiós de ${this.nombre}`;
  }

  static especie() {
    return "Humano";
  }
}

const p = new Persona("Ana", 25);
p.saludar();           // "Hola, soy Ana"
Persona.especie();     // "Humano"
```

### Getters y setters

```js
class Cuenta {
  #saldo = 0;

  get saldo() { return this.#saldo; }
  set saldo(valor) {
    if (valor < 0) throw new Error("Saldo negativo");
    this.#saldo = valor;
  }
}
const c = new Cuenta();
c.saldo = 100; // usa setter
c.saldo;       // 100 usa getter
```

### Campos privados (ES2022)

```js
class Banco {
  #secreto = "1234";

  validarClave(clave) {
    return clave === this.#secreto;
  }
}

const b = new Banco();
// b.#secreto; // ❌ SyntaxError
```

### Campos estáticos privados

```js
class App {
  static #instancia;
  static crear() {
    if (!this.#instancia) this.#instancia = new App();
    return this.#instancia;
  }
}
const app = App.crear();
```

### Herencia

```js
class Animal {
  constructor(nombre) {
    this.nombre = nombre;
  }
  hablar() {
    return `${this.nombre} hace un sonido`;
  }
}

class Perro extends Animal {
  hablar() {
    return `${this.nombre} ladra`;
  }
  traerZapatillas() {
    return `${this.nombre} trae las zapatillas`;
  }
}

class Cachorro extends Perro {
  hablar() {
    return `${this.nombre} ladra suave`;
  }
}

const fido = new Perro("Fido");
fido.hablar(); // "Fido ladra"
```

### `super`

```js
class Empleado extends Persona {
  constructor(nombre, edad, puesto) {
    super(nombre, edad); // llama al constructor padre
    this.puesto = puesto;
  }
  presentarse() {
    return `${super.saludar()}, trabajo como ${this.puesto}`;
  }
}
```

### instanceof

```js
new Perro("Fido") instanceof Animal; // true
```

### Métodos estáticos y de instancia

```js
class Matematica {
  static PI = 3.1416;
  static areaCirculo(r) { return this.PI * r * r; }
  sumar(a, b) { return a + b; }
}
Matematica.PI;
Matematica.areaCirculo(5);
new Matematica().sumar(2, 3);
```

### Actividad práctica

- Modela una jerarquía `Figura -> Circulo, Rectangulo, Triangulo` con un método `area()` y otro `perimetro()`.

---

## 4. Promesas

Una **Promise** representa un valor que estará disponible **ahora, en el futuro o nunca**.

### Estados

- **pending** (pendiente).
- **fulfilled** (resuelta).
- **rejected** (rechazada).

### Creación

```js
const miPromesa = new Promise((resolve, reject) => {
  setTimeout(() => {
    if (Math.random() > 0.5) resolve("Éxito");
    else reject(new Error("Falló"));
  }, 1000);
});
```

### Consumo

```js
miPromesa
  .then(resultado => console.log(resultado))
  .catch(error => console.error(error))
  .finally(() => console.log("Terminado"));
```

### Encadenamiento

```js
fetch("/api/user")
  .then(res => res.json())
  .then(user => fetch(`/api/posts/${user.id}`))
  .then(res => res.json())
  .then(posts => console.log(posts))
  .catch(err => console.error(err));
```

### Métodos estáticos

```js
Promise.resolve(42);
Promise.reject(new Error("Mal"));
Promise.all([p1, p2, p3]);         // espera todas (rechaza si una falla)
Promise.allSettled([p1, p2, p3]);  // espera todas, nunca rechaza
Promise.race([p1, p2, p3]);        // la primera en terminar gana
Promise.any([p1, p2, p3]);         // la primera fulfilled gana
```

### Ejemplo `Promise.allSettled`

```js
const resultados = await Promise.allSettled([
  fetch("/api/a"),
  fetch("/api/b"),
  fetch("/api/c"),
]);
resultados.forEach(r => {
  if (r.status === "fulfilled") console.log(r.value);
  else                          console.log(r.reason);
});
```

### Convertir callback a promesa

```js
const leerArchivo = (path) =>
  new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) reject(err);
      else     resolve(data);
    });
  });
```

### Actividad práctica

- Crea una función `cargarTodo()` que ejecute 3 promesas en paralelo y devuelva los resultados como un objeto.

---

## 5. Async / Await

Azúcar sintáctico sobre Promesas (ES2017). Hace el código asíncrono **legiblemente secuencial**.

### Función async siempre devuelve una promesa

```js
async function saludar(nombre) {
  return `Hola, ${nombre}`;
}
saludar("Ana").then(console.log); // "Hola, Ana"
```

### await suspende hasta que la promesa se resuelva

```js
async function obtenerUsuario(id) {
  const res  = await fetch(`/api/users/${id}`);
  const user = await res.json();
  return user;
}
```

### try / catch para errores

```js
async function cargar() {
  try {
    const res = await fetch("/no-existe");
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const data = await res.json();
    return data;
  } catch (err) {
    console.error("Error:", err.message);
    return null;
  } finally {
    console.log("Limpieza");
  }
}
```

### await en paralelo

```js
// ❌ Secuencial (más lento)
const a = await fetch("/api/a");
const b = await fetch("/api/b");

// ✅ Paralelo (más rápido)
const [a, b] = await Promise.all([
  fetch("/api/a"),
  fetch("/api/b"),
]);
const dataA = await a.json();
const dataB = await b.json();
```

### `for await...of` para iterar secuencialmente

```js
async function procesar(items) {
  for await (const item of items) {
    await guardar(item);
  }
}
```

### Patrón "Top-level await" en módulos

```js
const config = await fetch("/config").then(r => r.json());
```

### IIFE async

```js
(async () => {
  await init();
})();
```

### Actividad práctica

- Crea una función `obtenerPostsDeUsuario(id)` que haga dos peticiones (`/users/:id` y `/users/:id/posts`) y devuelva `{ user, posts }`.

---

## 6. Fetch API

API moderna para hacer peticiones HTTP desde el navegador. Devuelve promesas.

### GET

```js
const res = await fetch("/api/productos");
const data = await res.json();
console.log(data);
```

### POST

```js
const nuevo = { nombre: "Mouse", precio: 25 };

const res = await fetch("/api/productos", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer xxx",
  },
  body: JSON.stringify(nuevo),
});

const creado = await res.json();
```

### PUT / PATCH / DELETE

```js
await fetch(`/api/productos/${id}`, {
  method: "PUT",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(producto),
});

await fetch(`/api/productos/${id}`, {
  method: "PATCH",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ precio: 30 }),
});

await fetch(`/api/productos/${id}`, { method: "DELETE" });
```

### Manejo de respuestas

```js
const res = await fetch("/api/x");

res.status;       // 200
res.ok;           // true si status 200-299
res.statusText;   // "OK"
res.headers;      // Headers object
res.headers.get("Content-Type");

await res.json();     // parsea JSON
await res.text();     // texto plano
await res.blob();     // archivo binario
await res.formData(); // form data
await res.arrayBuffer();
```

### Errores de red

`fetch` **no rechaza** en errores HTTP (404, 500). Solo rechaza en fallos de red.

```js
async function getJSON(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

### Opciones avanzadas

```js
fetch(url, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(data),
  credentials: "include",     // enviar cookies
  mode: "cors",               // cors, no-cors, same-origin
  cache: "no-cache",          // default, no-store, reload, force-cache
  redirect: "follow",         // manual, error
  referrerPolicy: "no-referrer",
  signal: controller.signal,  // AbortController
});
```

### Cancelar con `AbortController`

```js
const controller = new AbortController();
document.querySelector("#cancelar").addEventListener("click", () => controller.abort());

try {
  const res = await fetch("/api/lenta", { signal: controller.signal });
} catch (err) {
  if (err.name === "AbortError") console.log("Cancelado");
}
```

### Timeout manual

```js
async function fetchConTimeout(url, ms) {
  const ctrl = new AbortController();
  const timeout = setTimeout(() => ctrl.abort(), ms);
  try {
    const res = await fetch(url, { signal: ctrl.signal });
    return await res.json();
  } finally {
    clearTimeout(timeout);
  }
}
```

### Subida de archivos

```js
const form = new FormData();
form.append("avatar", file);
form.append("userId", 1);

await fetch("/api/upload", {
  method: "POST",
  body: form,
});
```

### Actividad práctica

- Crea un mini-CRUD contra una API pública (jsonplaceholder.typicode.com o reqres.in).

---

## 7. JSON

**JSON (JavaScript Object Notation)** es el formato estándar de intercambio de datos.

### Sintaxis

```json
{
  "nombre": "Ana",
  "edad": 25,
  "hobbies": ["leer", "programar"],
  "direccion": { "ciudad": "Caracas" },
  "activo": true,
  "vacante": null
}
```

### `JSON.stringify(obj)` → texto

```js
JSON.stringify({ a: 1, b: 2 });         // '{"a":1,"b":2}'
JSON.stringify({ a: 1 }, null, 2);       // formateado con indentación
JSON.stringify({ fecha: new Date() });   // '{"fecha":"2025-01-01T00:00:00.000Z"}'
```

### `JSON.parse(texto)` → objeto

```js
JSON.parse('{"a":1}');          // {a:1}
JSON.parse('1');                // 1
JSON.parse('true');             // true
JSON.parse('null');             // null
JSON.parse('[1,2,3]');          // [1,2,3]
```

### Replacer y Reviver

```js
JSON.stringify({ a: 1, b: 2 }, (key, value) =>
  typeof value === "number" ? value * 2 : value
);
// '{"a":2,"b":4}'

JSON.parse('{"fecha":"2025-01-01T00:00:00.000Z"}', (key, value) =>
  key === "fecha" ? new Date(value) : value
);
```

### Limitaciones

- Solo soporta strings en claves.
- No comenta JSON.
- No soporta `undefined`, funciones, símbolos, `BigInt`.
- Fechas se serializan como strings ISO.

### Alternativas

- **YAML**: más legible, usado en configs.
- **XML**: aún presente en sistemas legacy.
- **Protocol Buffers / MessagePack**: binarios, más eficientes.

### Actividad práctica

- Crea una clase `User` con `toJSON()` y un método estático `fromJSON()` para serializar/recuperar.

---

## 8. LocalStorage

Almacenamiento persistente en el navegador (clave-valor) por **origen**. Capacidad típica: 5-10 MB.

### API

```js
localStorage.setItem("clave", "valor");
localStorage.getItem("clave");
localStorage.removeItem("clave");
localStorage.clear();                       // limpia todo
localStorage.length;                       // número de claves
localStorage.key(0);                       // clave en posición 0
```

### Solo almacena strings

```js
// ✅ Guardar objeto
const user = { nombre: "Ana", edad: 25 };
localStorage.setItem("user", JSON.stringify(user));

// ✅ Recuperar
const data = JSON.parse(localStorage.getItem("user"));

// ❌ Esto guarda "[object Object]"
localStorage.setItem("user", user);
```

### Wrappers útiles

```js
const storage = {
  set(key, value) { localStorage.setItem(key, JSON.stringify(value)); },
  get(key, defaultValue = null) {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : defaultValue;
  },
  remove(key) { localStorage.removeItem(key); },
  clear() { localStorage.clear(); },
};
```

### Eventos de storage (entre pestañas)

```js
window.addEventListener("storage", (e) => {
  if (e.key === "user") {
    console.log("cambió en otra pestaña", e.newValue);
  }
});
```

### Namespacing

```js
const NS = "miApp:";

localStorage.setItem(NS + "token", "abc123");
```

### Actividad práctica

- Implementa una función `temaActual()` que guarde y recupere el tema (claro/oscuro) en `localStorage`.

---

## 9. SessionStorage

Igual a `localStorage` pero **solo dura la sesión de la pestaña**.

```js
sessionStorage.setItem("token", "abc");
sessionStorage.getItem("token");
sessionStorage.removeItem("token");
sessionStorage.clear();
```

### Diferencias clave

| Característica   | localStorage | sessionStorage |
| ---------------- | ------------ | -------------- |
| Persistencia     | Persistente  | Solo pestaña   |
| Compartido tabs  | Sí           | No             |
| Tamaño típico    | 5-10 MB      | 5-10 MB        |
| Alcance          | Origen       | Origen         |

### Casos de uso

- `localStorage`: preferencias, tokens (con cuidado), carrito persistente.
- `sessionStorage`: datos sensibles de la sesión actual, formularios en pasos.

### Actividad práctica

- Guarda un borrador de formulario en `sessionStorage` y recíbelo en la siguiente pestaña (no debería estar).

---

## 10. Cookies

Pequeños datos que el navegador envía **en cada petición** al servidor.

### Con JS

```js
document.cookie = "usuario=Ana; max-age=3600; path=/";
document.cookie = "tema=oscuro; expires=Fri, 31 Dec 2025 23:59:59 GMT; path=/";

console.log(document.cookie); // "usuario=Ana; tema=oscuro"

// Helpers
const getCookie = (name) =>
  document.cookie
    .split("; ")
    .find(c => c.startsWith(`${name}=`))
    ?.split("=")[1];

const setCookie = (name, value, days = 7) => {
  const exp = new Date();
  exp.setTime(exp.getTime() + days * 24 * 60 * 60 * 1000);
  document.cookie = `${name}=${value}; expires=${exp.toUTCString()}; path=/; SameSite=Lax`;
};

const deleteCookie = (name) => {
  document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 GMT; path=/`;
};
```

### Flags de seguridad

- `HttpOnly`: no accesible desde JS. Solo servidor.
- `Secure`: solo HTTPS.
- `SameSite=Strict|Lax|None`: protección CSRF.
- `Path=/`, `Domain`.

### `cookieStore` API (experimental)

```js
const cookie = await cookieStore.set({
  name: "session",
  value: "abc",
  expires: Date.now() + 60_000,
  sameSite: "strict",
});
```

### Actividad práctica

- Implementa una función que guarde la última visita del usuario en una cookie y la muestre en pantalla.

---

## 11. Manejo de errores

### Tipos de errores

- `SyntaxError`: error de sintaxis.
- `ReferenceError`: variable no definida.
- `TypeError`: tipo incorrecto.
- `RangeError`: fuera de rango (ej. array con longitud negativa).
- `URIError`: problemas con codificación de URIs.
- `EvalError`: histórico.

### try / catch / finally

```js
try {
  const data = JSON.parse(texto);
  return data;
} catch (err) {
  console.error("Error al parsear:", err.message);
  return null;
} finally {
  console.log("siempre se ejecuta");
}
```

### Crear errores personalizados

```js
class ValidationError extends Error {
  constructor(campo, mensaje) {
    super(mensaje);
    this.name = "ValidationError";
    this.campo = campo;
  }
}

function validar(usuario) {
  if (!usuario.nombre) throw new ValidationError("nombre", "Requerido");
  if (usuario.edad < 0) throw new ValidationError("edad", "Inválida");
}

try {
  validar({ edad: -5 });
} catch (e) {
  if (e instanceof ValidationError) {
    console.log(`Campo ${e.campo}: ${e.message}`);
  }
}
```

### Capturar sin parar el flujo

```js
async function init() {
  const a = await safe(() => fetchA());
  const b = await safe(() => fetchB(), null);
  // incluso si fetchA falla, fetchB se ejecuta
}

const safe = async (fn, fallback) => {
  try { return await fn(); }
  catch (e) { console.error(e); return fallback; }
};
```

### `window.onerror` (captura global en navegador)

```js
window.addEventListener("error", (e) => {
  console.log("Error global:", e.message, e.filename, e.lineno);
});

window.addEventListener("unhandledrejection", (e) => {
  console.log("Promesa rechazada:", e.reason);
  e.preventDefault();
});
```

### `.catch` final

```js
process.on("unhandledRejection", (reason) => {
  console.error("Unhandled:", reason);
});
```

### Actividad práctica

- Crea una clase `AppError` y una jerarquía (`NetworkError`, `ValidationError`, `AuthError`) y manejadores específicos.

---

## 12. Eventos

Los eventos conectan la interacción del usuario con el código.

### `addEventListener`

```js
const btn = document.querySelector("#boton");

btn.addEventListener("click", (e) => {
  console.log("Click!", e.clientX, e.clientY);
});

// Remover
const handler = () => console.log("once");
btn.addEventListener("click", handler);
btn.removeEventListener("click", handler);
```

### Objeto `Event`

```js
btn.addEventListener("click", (e) => {
  e.preventDefault();   // evitar comportamiento por defecto
  e.stopPropagation();  // detiene la propagación
  e.target;             // elemento que disparó el evento
  e.currentTarget;      // elemento que tiene el listener
  e.type;               // "click"
});
```

### Eventos comunes

#### Mouse

```js
"click", "dblclick", "mousedown", "mouseup", "mousemove",
"mouseenter", "mouseleave", "mouseover", "mouseout", "contextmenu"
```

#### Teclado

```js
input.addEventListener("keydown", (e) => {
  if (e.key === "Enter") console.log("Enter");
  if (e.ctrlKey && e.key === "s") {
    e.preventDefault();
    guardar();
  }
});
```

#### Formulario

```js
"submit", "reset", "change", "input", "focus", "blur", "invalid"
```

Form completo:

```html
<form id="form">
  <input name="email" type="email" required>
  <button>Enviar</button>
</form>
```

```js
const form = document.querySelector("#form");
form.addEventListener("submit", async (e) => {
  e.preventDefault();
  const data = Object.fromEntries(new FormData(form));
  try {
    const res = await fetch("/api/save", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });
    if (!res.ok) throw new Error("HTTP " + res.status);
    alert("Guardado");
  } catch (err) {
    alert("Error: " + err.message);
  }
});
```

### Event delegation

Útil para listas dinámicas:

```html
<ul id="lista"></ul>
```

```js
const lista = document.querySelector("#lista");

lista.addEventListener("click", (e) => {
  if (e.target.matches("li")) {
    console.log("Click en:", e.target.textContent);
  }
});

// Render dinámico
items.forEach(item => {
  const li = document.createElement("li");
  li.textContent = item.nombre;
  lista.appendChild(li);
});
```

### `once` y opciones

```js
btn.addEventListener("click", handler, { once: true });
btn.addEventListener("click", handler, { passive: true }); // sin preventDefault
btn.addEventListener("click", handler, { capture: true });  // fase de captura
```

### CustomEvents

```js
btn.dispatchEvent(new CustomEvent("saludar", {
  detail: { usuario: "Ana" },
  bubbles: true,
}));

btn.addEventListener("saludar", (e) => {
  console.log("Hola", e.detail.usuario);
});
```

### Otros eventos

```js
window.addEventListener("load", () => console.log("Página cargada"));
window.addEventListener("DOMContentLoaded", () => console.log("DOM listo"));
window.addEventListener("resize", debounce(() => console.log("resize"), 200));
window.addEventListener("scroll", throttle(() => console.log("scroll"), 100));
window.addEventListener("beforeunload", (e) => {
  e.preventDefault();
  e.returnValue = "";
});
```

### Actividad práctica

- Implementa un `<input>` con búsqueda en tiempo real (debounce) y un botón que dispare un `CustomEvent`.

---

## 13. Event Loop

JavaScript es **single-threaded**. El Event Loop es el mecanismo que coordina la ejecución de código síncrono, microtasks y macrotasks.

### Componentes

- **Call Stack**: pila de ejecución de funciones.
- **Web APIs**: proveídas por el navegador (`setTimeout`, `fetch`, `DOM`, etc.).
- **Task Queue (Macrotask Queue)**: cola de tareas.
- **Microtask Queue**: cola de microtareas (más prioritaria).
- **Event Loop**: ciclo que mueve tareas a la pila cuando esta está vacía.

### Orden de ejecución

```
┌─────────────────────────────┐
│ 1. Código síncrono (call stack)
├─────────────────────────────┤
│ 2. Microtasks (Promises, queueMicrotask)
├─────────────────────────────┤
│ 3. Macrotasks (setTimeout, setInterval, I/O)
├─────────────────────────────┤
│ 4. Render del navegador (si aplica)
└─────────────────────────────┘
```

### Ejemplo clásico

```js
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

console.log("4");
```

Salida:
```
1
4
3
2
```

### ¿Por qué?

1. `console.log("1")` → call stack.
2. `setTimeout` → Web API → Macrotask queue.
3. `Promise.resolve().then(...)` → Microtask queue.
4. `console.log("4")` → call stack.
5. Call stack vacía → ejecuta microtasks → `3`.
6. Ejecuta macrotasks → `2`.

### Otro ejemplo más complejo

```js
async function test() {
  console.log("A");
  await Promise.resolve();
  console.log("B");
}

console.log("C");
test();
console.log("D");

setTimeout(() => console.log("E"), 0);
Promise.resolve().then(() => console.log("F"));
```

Salida:
```
C
A
D
F
B
E
```

### Por qué importa

- Permite entender por qué tu UI se bloquea.
- Por qué un `await` no bloquea.
- Por qué las promesas se ejecutan antes que los `setTimeout`.

### Actividad práctica

- Predice la salida de 5 snippets con combinaciones de promesas, `setTimeout` y `await`.

---

## 14. Microtasks

**Microtasks** son tareas de alta prioridad que se ejecutan **justo después** del código síncrono actual y **antes** del siguiente render o macrotask.

### Fuentes de microtasks

- **Promesas**: `.then`, `.catch`, `.finally`, `await`.
- **`queueMicrotask(fn)`**: API explícita (ES2018).
- **MutationObserver** callbacks.
- **process.nextTick** (Node.js, técnicamente fuera de spec).

### Ejemplo

```js
console.log("1");

queueMicrotask(() => console.log("microtask"));

Promise.resolve().then(() => console.log("promise"));

setTimeout(() => console.log("setTimeout"), 0);

console.log("2");
```

Salida:
```
1
2
promise     // microtask de la promesa
microtask   // microtask explícita (FIFO dentro de microtask queue)
setTimeout  // macrotask
```

### `queueMicrotask`

```js
queueMicrotask(() => {
  // código asíncrono sin crear una promesa
});
```

Útil cuando necesitas aplazar trabajo al final del tick actual.

### `await` y microtasks

Cada `await` "pausa" la función y reanuda en una **microtask** cuando la promesa se resuelve.

```js
async function ejemplo() {
  console.log("a");
  await null;
  console.log("b");
}
```

### Anidación profunda

```js
Promise.resolve()
  .then(() => {
    console.log("1");
    return Promise.resolve();
  })
  .then(() => console.log("2"));
// Cada .then crea una microtask adicional.
```

### Actividad práctica

- Experimenta variar el orden de promesas y `queueMicrotask` y observa los logs.

---

## 15. Macrotasks

**Macrotasks** (o "tasks") son tareas que se encolan en la **Task Queue** y se ejecutan una por una en cada vuelta del Event Loop.

### Fuentes

- `setTimeout`, `setInterval`.
- `setImmediate` (Node.js).
- I/O callbacks.
- Eventos del DOM (click, load, etc.) — algunos cuentan como task.

### Orden general del Event Loop

```
while (true) {
  ejecutar código síncrono actual;
  while (microtaskQueue.notEmpty) {
    ejecutar siguiente microtask;
  }
  renderizar (si es momento);
  tomar siguiente macrotask;
}
```

### Ejemplo

```js
setTimeout(() => console.log("timeout 1"), 0);
setTimeout(() => console.log("timeout 2"), 0);

Promise.resolve().then(() => console.log("microtask 1"));

setTimeout(() => {
  console.log("timeout 3");
  Promise.resolve().then(() => console.log("microtask 2"));
}, 0);
```

Salida:
```
microtask 1
timeout 1
microtask 2
timeout 2
timeout 3
```

### Trampas comunes

```js
// Bucle infinito de microtasks bloquea el render
function loop() {
  Promise.resolve().then(loop);
}
loop(); // 😱 navegador congelado
```

### Rendering

Tras todas las microtasks, si es momento, el navegador renderiza. Luego toma la siguiente macrotask.

### `requestAnimationFrame`

Se ejecuta **antes del próximo repaint** (ideal para animaciones):

```js
function animar() {
  element.style.transform = `translateX(${x++}px)`;
  requestAnimationFrame(animar);
}
requestAnimationFrame(animar);
```

`requestAnimationFrame` técnicamente es una "task" especial programada antes del render, no dentro de la Task Queue tradicional.

### `requestIdleCallback`

Programa trabajo en tiempos ociosos:

```js
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && tareas.length) {
    hacerTarea(tareas.shift());
  }
});
```

### Actividad práctica

- Crea una animación fluida con `requestAnimationFrame` y compara con un bucle de `setTimeout`.

---

## Ejemplo integrador

```js
// app.js - Módulo 5 - Ejemplo integrador

import { User, Product } from "./models/index.js";
import { storage } from "./utils/storage.js";
import { api } from "./utils/api.js";

const CART_KEY = "carrito";

const app = {
  async init() {
    this.cart = storage.get(CART_KEY, []);
    this.renderCart();

    try {
      const productos = await api.get("/api/productos");
      const items = productos.map(p => new Product(p));
      this.renderProducts(items);
    } catch (err) {
      console.error(err);
      this.showError("No se pudieron cargar los productos");
    } finally {
      document.querySelector("#loader").hidden = true;
    }

    document.addEventListener("product:added", (e) => {
      this.addToCart(e.detail);
    });

    document.querySelector("#vaciar").addEventListener("click", () => this.clearCart());

    window.addEventListener("storage", (e) => {
      if (e.key === CART_KEY) {
        this.cart = JSON.parse(e.newValue || "[]");
        this.renderCart();
      }
    });
  },

  addToCart(product) {
    const found = this.cart.find(p => p.id === product.id);
    if (found) found.cantidad++;
    else this.cart.push({ ...product, cantidad: 1 });
    storage.set(CART_KEY, this.cart);
    this.renderCart();
  },

  clearCart() {
    this.cart = [];
    storage.set(CART_KEY, this.cart);
    this.renderCart();
  },

  renderProducts(items) {
    const root = document.querySelector("#productos");
    root.innerHTML = "";
    items.forEach(p => {
      const card = document.createElement("div");
      card.className = "card";
      card.innerHTML = `
        <h3>${p.nombre}</h3>
        <p>$${p.precio}</p>
        <button data-id="${p.id}">Añadir</button>
      `;
      card.querySelector("button").addEventListener("click", () => {
        document.dispatchEvent(new CustomEvent("product:added", {
          detail: p,
          bubbles: true,
        }));
      });
      root.appendChild(card);
    });
  },

  renderCart() {
    document.querySelector("#contador").textContent =
      this.cart.reduce((acc, p) => acc + p.cantidad, 0);
  },

  showError(msg) {
    const div = document.createElement("div");
    div.className = "error";
    div.textContent = msg;
    document.body.appendChild(div);
    setTimeout(() => div.remove(), 3000);
  },
};

app.init();
```

---

## Recursos complementarios

### Lecturas

- [MDN - JavaScript Modules](https://developer.mozilla.org/es/docs/Web/JavaScript/Guide/Modules)
- [MDN - Fetch](https://developer.mozilla.org/es/docs/Web/API/Fetch_API)
- [MDN - Event Loop](https://developer.mozilla.org/es/docs/Web/JavaScript/Event_loop)
- [Jake Archibald - In the Loop](https://www.youtube.com/watch?v=cCOL7MC4Pl0) (JSConf Asia)
- [JavaScript.info - Promises, async/await](https://javascript.info/async)

### Práctica

- [JSONPlaceholder](https://jsonplaceholder.typicode.com/) (API fake)
- [Reqres](https://reqres.in/)
- [httpbin.org](https://httpbin.org/)

---

## Evaluación sugerida

1. **Cuestionario (20%)**: módulos, promesas, event loop.
2. **CRUD con Fetch (30%)**: aplicación que consuma una API pública.
3. **Persistencia (20%)**: carrito de compras en localStorage + evento storage.
4. **Refactor asíncrono (30%)**: código con callbacks convertido a `async/await` con manejo de errores.

---

> **Nota para el estudiante**: JavaScript moderno es muy distinto al de hace 10 años. Entender el **Event Loop** es lo que separa a un desarrollador junior de uno que realmente sabe lo que hace el navegador.
