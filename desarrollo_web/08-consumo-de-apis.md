# Módulo 8: Consumo de APIs

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Comprender la arquitectura REST y sus principios.
- Consumir APIs usando `fetch` y Axios.
- Realizar operaciones CRUD con distintos métodos HTTP.
- Manejar autenticación con JWT y Bearer Tokens.
- Gestionar errores en peticiones HTTP.
- Implementar paginación y filtros desde el cliente.
- Diseñar clientes HTTP reutilizables y robustos.

---

## 1. REST (Representational State Transfer)

REST es un **estilo arquitectónico** para sistemas distribuidos, propuesto por Roy Fielding en su tesis (2000).

### Principios

1. **Cliente-servidor**: separación de responsabilidades.
2. **Sin estado (stateless)**: cada petición es autocontenida.
3. **Cacheable**: las respuestas pueden declararse cacheables.
4. **Interfaz uniforme**: URIs, métodos HTTP, representaciones, hipermedios.
5. **Sistema de capas**: el cliente no sabe si habla con el servidor final o un proxy.
6. **Código bajo demanda** (opcional): enviar lógica ejecutable (ej. JavaScript).

### Recursos

En REST, todo es un **recurso** identificado por una URI:

```
/api/users
/api/users/42
/api/users/42/posts
/api/users/42/posts/7/comments
```

Las URI deben ser **sustantivos** (no verbos):

```
✅ /users/42
❌ /getUser/42
```

### Representaciones

JSON (estándar actual), XML (legacy), YAML, HTML.

### Buenas prácticas de URI

- Plural para colecciones: `/users`, no `/user`.
- Jerárquica: `/users/42/posts`.
- Kebab-case: `/blog-posts`, no `/blogPosts`.
- Filtrar con query string: `/users?role=admin&active=true`.

### HATEOAS

Las respuestas pueden incluir **hipervínculos** a recursos relacionados:

```json
{
  "id": 42,
  "name": "Ana",
  "links": {
    "self":   "/api/users/42",
    "posts":  "/api/users/42/posts",
    "avatar": "/api/users/42/avatar.png"
  }
}
```

### Actividad práctica

- Diseña la API REST de un blog: recursos `users`, `posts`, `comments`. Lista endpoints, verbos y respuestas esperadas.

---

## 2. JSON en APIs

JSON es el formato estándar de intercambio en APIs REST modernas.

### Sintaxis

```json
{
  "id": 1,
  "title": "Hola mundo",
  "tags": ["javascript", "rest"],
  "author": {
    "id": 42,
    "name": "Ana"
  },
  "publishedAt": "2025-01-15T12:00:00Z",
  "draft": false,
  "views": null
}
```

### Enviar y recibir

```js
const res = await fetch("/api/posts", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    title: "Nuevo post",
    content: "Contenido",
  }),
});

const data = await res.json();
```

### Convenciones de nombres

```js
// camelCase (común en JS)
{ firstName: "Ana" }

// snake_case (común en Rails, Python, SQL)
{ first_name: "Ana" }

// kebab-case (común en URLs)
{ "first-name": "Ana" }
```

Elige **una** convención y sé consistente.

### Fechas

ISO 8601: `"2025-01-15T12:00:00Z"`.

### Tipos básicos permitidos

- `string`, `number`, `boolean`, `null`.
- `array`, `object`.
- ❌ No `undefined`, `function`, `Symbol`, `BigInt`, fechas nativas.

### JSON vs JS Object

```js
JSON.parse('{"a": 1}');  // { a: 1 }
JSON.stringify({ a: 1 }); // '{"a":1}'
```

### Actividad práctica

- Convierte un array de objetos JS a JSON y luego parsea de vuelta. Observa el manejo de fechas y de tipos excluidos.

---

## 3. HTTP Methods (Verbos)

REST usa los verbos HTTP para indicar la **intención** sobre un recurso.

### Tabla principal

| Método  | Acción         | Idempotente | Seguro | Body |
| ------- | -------------- | ----------- | ------ | ---- |
| GET     | Leer           | Sí          | Sí     | No   |
| POST    | Crear          | No          | No     | Sí   |
| PUT     | Reemplazar     | Sí          | No     | Sí   |
| PATCH   | Modificar      | No          | No     | Sí   |
| DELETE  | Eliminar       | Sí          | No     | Opcional |
| HEAD    | Como GET sin body | Sí       | Sí     | No   |
| OPTIONS | Metadatos      | Sí          | Sí     | No   |

### Idempotencia

Un método es idempotente si **múltiples llamadas idénticas** tienen el **mismo efecto** que una sola.

- `GET` siempre idempotente.
- `PUT` (reemplazo completo) idempotente.
- `DELETE` idempotente (borrar lo ya borrado = ya borrado).
- `POST` **no** idempotente (crea duplicados).
- `PATCH` depende del patch.

### Ejemplo con fetch

```js
const HEADERS = { "Content-Type": "application/json" };

await fetch("/api/users",     { method: "GET"  });
await fetch("/api/users",     { method: "POST",   headers: HEADERS, body: JSON.stringify(payload) });
await fetch("/api/users/42",  { method: "PUT",    headers: HEADERS, body: JSON.stringify(payload) });
await fetch("/api/users/42",  { method: "PATCH",  headers: HEADERS, body: JSON.stringify({name:"Ana"}) });
await fetch("/api/users/42",  { method: "DELETE" });
await fetch("/api/users",     { method: "HEAD" });
await fetch("/api/users",     { method: "OPTIONS" });
```

### Códigos de estado comunes

- **2xx éxito**: 200 OK, 201 Created, 202 Accepted, 204 No Content.
- **3xx redirección**: 301 Moved, 304 Not Modified.
- **4xx cliente**: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable, 429 Too Many Requests.
- **5xx servidor**: 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout.

### Actividad práctica

- Construye una tabla de endpoints CRUD para un recurso `productos`. Especifica método, ruta, body y respuesta esperada.

---

## 4. CRUD (Create, Read, Update, Delete)

Operaciones básicas sobre recursos.

### Ejemplo completo con Fetch

```js
const API = "https://api.example.com";

const http = async (path, options = {}) => {
  const res = await fetch(`${API}${path}`, {
    headers: { "Content-Type": "application/json", ...(options.headers || {}) },
    ...options,
  });
  if (!res.ok) {
    const err = new Error(`HTTP ${res.status}`);
    err.status = res.status;
    err.body = await res.text();
    throw err;
  }
  if (res.status === 204) return null;
  return res.json();
};

const ProductoAPI = {
  listar:  ()                => http("/productos"),
  obtener: (id)              => http(`/productos/${id}`),
  crear:   (data)            => http("/productos",      { method: "POST",   body: JSON.stringify(data) }),
  reemplazar: (id, data)     => http(`/productos/${id}`, { method: "PUT",    body: JSON.stringify(data) }),
  actualizar: (id, cambios)  => http(`/productos/${id}`, { method: "PATCH",  body: JSON.stringify(cambios) }),
  eliminar: (id)             => http(`/productos/${id}`, { method: "DELETE" }),
};
```

### Uso

```js
try {
  const nuevo = await ProductoAPI.crear({ nombre: "Laptop", precio: 1200 });
  console.log("Creado:", nuevo);

  const actualizado = await ProductoAPI.actualizar(nuevo.id, { precio: 1100 });

  await ProductoAPI.eliminar(nuevo.id);
} catch (err) {
  console.error(err.message, err.body);
}
```

### CRUD con AsyncData pattern

```js
const cargarProducto = async (id) => {
  const { data, error } = await safeFetch(`/api/productos/${id}`);
  if (error) mostrarError(error);
  return data;
};

const safeFetch = async (url, options) => {
  try {
    const res = await fetch(url, options);
    if (!res.ok) return { error: { status: res.status, message: res.statusText } };
    return { data: await res.json() };
  } catch (e) {
    return { error: { network: true, message: e.message } };
  }
};
```

### Bulk operations

Algunas APIs soportan operaciones por lotes:

```js
await fetch("/api/productos/bulk", {
  method: "POST",
  body: JSON.stringify([
    { id: 1, precio: 100 },
    { id: 2, precio: 200 },
  ]),
});
```

### Actividad práctica

- Implementa un CRUD completo contra JSONPlaceholder (https://jsonplaceholder.typicode.com) para `posts`.

---

## 5. Fetch (avanzado)

### Opciones

```js
fetch(url, {
  method: "POST",                // GET, POST, PUT...
  headers: { "X-Custom": "..." },
  body: JSON.stringify(data),    // string, FormData, Blob...
  credentials: "include",        // omit, same-origin, include
  mode: "cors",                  // cors, no-cors, same-origin, navigate
  cache: "no-cache",             // default, no-store, reload, force-cache, only-if-cached
  redirect: "follow",            // follow, error, manual
  referrerPolicy: "no-referrer", // no-referrer, origin, ...
  signal: controller.signal,     // AbortController
  integrity: "sha256-...",        // SRI
  keepalive: true,               // mantener conexión viva
});
```

### Headers comunes en respuesta

- `Content-Type`.
- `Cache-Control`.
- `ETag` (validación).
- `Link` (relaciones, paginación).
- `RateLimit-*` (rate limiting).
- `WWW-Authenticate` (401).

### Streaming de respuestas

```js
const res = await fetch("/api/logs");
const reader = res.body.getReader();
const decoder = new TextDecoder();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log(decoder.decode(value));
}
```

### `Response.clone()`

Permite leer el body más de una vez:

```js
const res = await fetch(url);
const data = await res.clone().json();
```

### `Request` y `Response` (objetos)

```js
const req = new Request("/api/x", { method: "POST", body: JSON.stringify({ a: 1 }) });
const res = await fetch(req);

const resB = new Response(JSON.stringify({ ok: true }), {
  status: 200,
  headers: { "Content-Type": "application/json" },
});
```

### `cache: "force-cache"` y offline-first

```js
fetch("/api/data", { cache: "force-cache" });
```

### Actividad práctica

- Implementa un cliente HTTP con timeout, reintento y abort.

```js
async function fetchWithRetry(url, options = {}, retries = 3, backoff = 300) {
  const ctrl = new AbortController();
  const timeout = setTimeout(() => ctrl.abort(), options.timeout || 10000);
  try {
    const res = await fetch(url, { ...options, signal: ctrl.signal });
    clearTimeout(timeout);
    if (!res.ok && retries > 0) {
      await new Promise(r => setTimeout(r, backoff));
      return fetchWithRetry(url, options, retries - 1, backoff * 2);
    }
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  } catch (err) {
    if (retries > 0 && err.name !== "AbortError") {
      await new Promise(r => setTimeout(r, backoff));
      return fetchWithRetry(url, options, retries - 1, backoff * 2);
    }
    throw err;
  }
}
```

---

## 6. Axios

**Axios** es un cliente HTTP basado en promesas con una API más rica.

### Instalación

```bash
npm install axios
```

```html
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
```

### Configuración global

```js
const api = axios.create({
  baseURL: "https://api.example.com",
  timeout: 10000,
  headers: { "Content-Type": "application/json", Accept: "application/json" },
  withCredentials: false,
});
```

### CRUD

```js
// GET
const { data } = await api.get("/productos");

// GET con params
const { data } = await api.get("/productos", { params: { page: 2, limit: 20 } });

// POST
const { data: nuevo } = await api.post("/productos", { nombre: "Laptop", precio: 1200 });

// PUT
const { data: actualizado } = await api.put(`/productos/${id}`, producto);

// PATCH
const { data } = await api.patch(`/productos/${id}`, { precio: 1100 });

// DELETE
await api.delete(`/productos/${id}`);
```

### Interceptores (¡gran fortaleza!)

```js
// Request: añade token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem("token");
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
  },
  (error) => Promise.reject(error)
);

// Response: maneja errores globales
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem("token");
      location.href = "/login";
    }
    return Promise.reject(error);
  }
);
```

### Cancelación

```js
const controller = new AbortController();
try {
  const { data } = await api.get("/productos", { signal: controller.signal });
} catch (err) {
  if (axios.isCancel(err)) console.log("Cancelado");
}

// Cancelación nativa con CancelToken (legacy)
const source = axios.CancelToken.source();
api.get("/x", { cancelToken: source.token }).catch(err => {
  if (axios.isCancel(err)) console.log("Cancelado");
});
source.cancel("Operación cancelada");
```

### Configs por petición

```js
await api.get("/x", {
  headers: { "X-Custom": "value" },
  params: { foo: "bar" },
  timeout: 5000,
  responseType: "json", // json, text, blob, arraybuffer, stream
  validateStatus: (s) => s < 500, // no rechaza 4xx
});
```

### Serialización de Query Params

```js
import qs from "qs";

api.get("/x", { params: { filter: { name: "Ana", age: ">20" } }, paramsSerializer: p => qs.stringify(p) });
```

### `FormData` con Axios

```js
const fd = new FormData();
fd.append("avatar", file);
await api.post("/upload", fd, { headers: { "Content-Type": "multipart/form-data" } });
```

Axios lo detecta automáticamente.

### Cliente Axios completo

```js
class HttpClient {
  constructor(baseURL, options = {}) {
    this.client = axios.create({ baseURL, timeout: 15000, ...options });
    this.setupInterceptors();
  }

  setupInterceptors() {
    this.client.interceptors.request.use((config) => {
      const token = localStorage.getItem("token");
      if (token) config.headers.Authorization = `Bearer ${token}`;
      return config;
    });

    this.client.interceptors.response.use(
      (res) => res,
      (err) => {
        const e = {
          status: err.response?.status,
          message: err.response?.data?.message || err.message,
          data: err.response?.data,
          original: err,
        };
        return Promise.reject(e);
      }
    );
  }

  get(url, config)         { return this.client.get(url, config).then(r => r.data); }
  post(url, data, config)  { return this.client.post(url, data, config).then(r => r.data); }
  put(url, data, config)   { return this.client.put(url, data, config).then(r => r.data); }
  patch(url, data, config) { return this.client.patch(url, data, config).then(r => r.data); }
  delete(url, config)      { return this.client.delete(url, config).then(r => r.data); }
}

const api = new HttpClient("https://api.example.com");
```

### Actividad práctica

- Reescribe el CRUD del ejercicio anterior usando Axios e interceptores para token y errores.

---

## 7. Autenticación

La autenticación verifica **quién eres**; la autorización, **qué puedes hacer**.

### Métodos comunes

| Método                  | Cómo viaja                  | Riesgos                       |
| ----------------------- | --------------------------- | ----------------------------- |
| Basic Auth              | `Authorization: Basic base64(user:pass)` | Sin cifrado (usar HTTPS)      |
| API Key                 | `X-API-Key: ...` o query    | Filtración en logs            |
| Bearer Token (JWT/OAuth)| `Authorization: Bearer ...` | Robo, expiración              |
| Cookies de sesión       | Automático                  | CSRF                          |
| OAuth 2.0 / OIDC        | Flujo completo              | Complejidad                   |
| mTLS                    | Certificados cliente        | Infraestructura PKI           |

### Flujos típicos

#### Stateless (JWT)

```
1. Login → Server retorna JWT
2. Cliente guarda JWT (memoria/localStorage)
3. Cliente envía JWT en cada request
4. Server valida firma
```

#### Stateful (Sesión)

```
1. Login → Server crea sesión, retorna cookie
2. Cookie se envía automáticamente
3. Server valida sesión en cada petición
```

### Enviar credenciales por fetch

```js
fetch("/api/login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ email, password }),
  credentials: "include", // enviar cookies
});
```

### Buenas prácticas

- HTTPS siempre.
- Tokens de **corta duración** + **refresh tokens**.
- Guardar tokens en **memoria** o cookies HttpOnly.
- **Nunca** en `localStorage` si son sensibles (XSS).
- Implementar logout que invalide el token en el servidor.

### Actividad práctica

- Implementa un `authService` con `login`, `logout`, `register`, `refreshToken`.

---

## 8. JWT (JSON Web Token)

**JWT** es un estándar abierto (RFC 7519) para transmitir información firmada entre partes. Es un string con tres partes separadas por puntos: `header.payload.signature`.

### Estructura

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NSIsIm5hbWUiOiJBbmEifQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Partes

#### Header

```json
{ "alg": "HS256", "typ": "JWT" }
```

#### Payload (claims)

```json
{
  "sub": "1234567890",          // subject (usuario)
  "name": "Ana Pérez",
  "role": "admin",
  "iat": 1717000000,             // issued at
  "exp": 1717003600,             // expires at (1h después)
  "iss": "mi-app",                // issuer
  "aud": "users",                 // audience
  "jti": "abc123"                 // id único del token
}
```

Tipos de claims:
- **Registered**: `iss`, `sub`, `aud`, `exp`, `iat`, `jti`.
- **Public**: `name`, `email`, `role`.
- **Private**: claims personalizados.

#### Signature

```
HMACSHA256(
  base64Url(header) + "." + base64Url(payload),
  secret
)
```

### Verificación en el servidor

1. Decodifica header y payload.
2. Verifica firma con el `secret`.
3. Valida `exp`, `iat`, `nbf`, `iss`, `aud`.
4. Acepta o rechaza.

### Flujo típico

```js
// 1. Login → token
const { token } = await api.post("/auth/login", { email, password });
localStorage.setItem("token", token);

// 2. Peticiones autenticadas
const { data } = await api.get("/perfil", {
  headers: { Authorization: `Bearer ${token}` },
});

// 3. Refresh
const { token: nuevo } = await api.post("/auth/refresh", {
  refreshToken: localStorage.getItem("refresh"),
});
```

### Decodificar en cliente (sin verificar firma)

```js
const parseJwt = (token) => {
  const base64Url = token.split(".")[1];
  const base64 = base64Url.replace(/-/g, "+").replace(/_/g, "/");
  return JSON.parse(atob(base64));
};

console.log(parseJwt(token));
// { sub: "123", name: "Ana", exp: 1717003600, ... }
```

### Verificar expiración

```js
const isExpired = (token) => {
  const { exp } = parseJwt(token);
  return Date.now() >= exp * 1000;
};
```

### Vulnerabilidades

- **`alg: none`**: nunca permitir si el servidor lo acepta.
- **Firma débil**: usar claves de 256+ bits.
- **Tokens en localStorage**: vulnerables a XSS. Preferir cookies HttpOnly.
- **No revocables**: usar listas de revocación o refresh tokens.

### Actividad práctica

- Implementa un helper `auth` con login, almacenamiento, refresh automático y logout.

---

## 9. Bearer Token

Es un esquema de autenticación donde el cliente envía un token (generalmente JWT) en cada petición.

### Header HTTP

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### En Fetch

```js
const token = localStorage.getItem("token");

fetch("/api/perfil", {
  headers: {
    "Authorization": `Bearer ${token}`,
    "Accept": "application/json",
  },
});
```

### En Axios

```js
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

### 401 → refresh / reauth

```js
api.interceptors.response.use(
  (res) => res,
  async (err) => {
    if (err.response?.status === 401 && !err.config._retry) {
      err.config._retry = true;
      try {
        const { data } = await axios.post("/auth/refresh", {
          refreshToken: localStorage.getItem("refresh"),
        });
        localStorage.setItem("token", data.token);
        err.config.headers.Authorization = `Bearer ${data.token}`;
        return api.request(err.config);
      } catch (e) {
        localStorage.removeItem("token");
        location.href = "/login";
      }
    }
    return Promise.reject(err);
  }
);
```

### Otros esquemas de Authorization

- `Basic base64(user:pass)`.
- `Digest ...` (legacy).
- `OAuth ...` (OAuth 1.0a, legacy).
- `Bearer ...` (más común).
- `HOBA`, `Mutual`, `AWS4-HMAC-SHA256` (específicos).

### Actividad práctica

- Configura Axios para enviar Bearer automáticamente y renovar tokens expirados.

---

## 10. Manejo de errores

### Tipos de error en HTTP

1. **Errores de red**: `fetch` rechaza (sin internet, DNS, CORS).
2. **Errores HTTP**: status 4xx/5xx (`fetch` resuelve, `ok=false`).
3. **Errores de parseo**: JSON malformado.
4. **Errores de timeout**: `AbortController`.

### Wrapper seguro

```js
async function http(url, options = {}) {
  const ctrl = new AbortController();
  const timeoutId = setTimeout(() => ctrl.abort(), options.timeout || 10000);

  try {
    const res = await fetch(url, { ...options, signal: ctrl.signal });
    clearTimeout(timeoutId);

    if (!res.ok) {
      const err = new Error(`Error ${res.status}: ${res.statusText}`);
      err.status = res.status;
      try { err.body = await res.json(); } catch { err.body = await res.text(); }
      throw err;
    }

    if (res.status === 204) return null;
    return await res.json();
  } catch (err) {
    if (err.name === "AbortError") {
      const e = new Error("Tiempo de espera agotado");
      e.code = "TIMEOUT";
      throw e;
    }
    if (err.message === "Failed to fetch") {
      const e = new Error("Sin conexión con el servidor");
      e.code = "NETWORK";
      throw e;
    }
    throw err;
  }
}
```

### Errores personalizados por status

```js
const manejarError = (err) => {
  switch (err.status) {
    case 400: console.log("Petición inválida"); break;
    case 401: console.log("No autenticado");      location.href = "/login"; break;
    case 403: console.log("Sin permisos");       break;
    case 404: console.log("Recurso no encontrado"); break;
    case 409: console.log("Conflicto");           break;
    case 422: console.log("Validación fallida", err.body); break;
    case 429: console.log("Demasiadas peticiones, espera"); break;
    case 500: console.log("Error del servidor");  break;
    case 503: console.log("Servicio no disponible"); break;
    default:  console.error(err.message);
  }
};
```

### Reintentos exponenciales

```js
async function fetchWithRetry(url, options = {}, max = 3) {
  let attempt = 0;
  while (true) {
    try {
      return await http(url, options);
    } catch (err) {
      attempt++;
      if (attempt >= max || (err.status && err.status < 500)) throw err;
      const wait = 2 ** attempt * 100 + Math.random() * 100;
      await new Promise((r) => setTimeout(r, wait));
    }
  }
}
```

### Logging y monitorización

```js
api.interceptors.response.use(
  (res) => {
    console.log("[API]", res.config.method, res.config.url, res.status);
    return res;
  },
  (err) => {
    if (err.response) {
      console.error("[API ERROR]", err.response.status, err.response.config.url, err.response.data);
    }
    return Promise.reject(err);
  }
);
```

Sentry: `Sentry.captureException(err)`.

### UI feedback

```js
const { data, error } = await safe(() => api.get("/x"));

if (error) {
  if (error.code === "NETWORK") {
    mostrarToast("Sin conexión");
  } else if (error.status === 401) {
    location.href = "/login";
  } else {
    mostrarToast("Error: " + error.message);
  }
}
```

### Actividad práctica

- Crea un cliente HTTP con timeout, retries y notificación al usuario según el tipo de error.

---

## 11. Paginación

APIs que devuelven grandes conjuntos de datos casi siempre **pagian** los resultados.

### Estrategias

#### Offset / Limit (la más común)

```http
GET /api/productos?page=2&limit=20
```

```json
{
  "data": [...],
  "meta": { "page": 2, "limit": 20, "total": 134, "totalPages": 7 }
}
```

#### Page / Size

```http
GET /api/posts?page=3&size=10
```

#### Cursor / After

```http
GET /api/feed?after=eyJpZCI6MTIzfQ&limit=20
```

Más eficiente para feeds, evita saltos al insertar nuevos elementos.

```json
{
  "data": [...],
  "next": "eyJpZCI6MTQzfQ",
  "hasMore": true
}
```

#### Link Header (HATEOAS)

```http
Link: <https://api.example.com/products?page=2>; rel="next",
      <https://api.example.com/products?page=10>; rel="last"
```

### Implementación cliente

```js
async function paginar(url, page = 1) {
  const res = await fetch(`${url}?page=${page}&limit=20`);
  const json = await res.json();
  return {
    items: json.data,
    hasMore: page < json.meta.totalPages,
    nextPage: page + 1,
  };
}

let page = 1;
let stop = false;
async function cargarTodo() {
  while (!stop) {
    const { items, hasMore, nextPage } = await paginar("/api/posts", page);
    items.forEach(renderPost);
    if (!hasMore) stop = true;
    page = nextPage;
  }
}
```

### Infinite scroll con `IntersectionObserver`

```js
const observer = new IntersectionObserver(async (entries) => {
  if (entries[0].isIntersecting && !cargando) {
    cargando = true;
    const { items, hasMore } = await paginar("/api/posts", pagina++);
    items.forEach(renderPost);
    cargando = !hasMore;
  }
});
observer.observe(sentinel);
```

### Actividad práctica

- Implementa un listado paginado con botones "Anterior/Siguiente" y un infinite scroll alternativo.

---

## 12. Filtros y búsqueda

### Tipos de filtros

#### Query params

```http
GET /api/productos?categoria=tecnologia&precio_min=100&precio_max=500&ordenar=precio&dir=asc
```

#### Search (búsqueda full-text)

```http
GET /api/productos?q=iphone
```

#### Filtros anidados (LHS like)

```http
GET /api/productos?filtros[precio][gt]=100&filtros[precio][lt]=500
```

#### GraphQL / lenguaje propio (RQL, Mongo-like)

```json
POST /api/productos/query
{
  "where": { "category": "tech", "price": { "$lt": 500 } },
  "sort": "-createdAt",
  "limit": 10
}
```

### Implementación cliente

```js
class ProductoQuery {
  constructor(base = "https://api.example.com/productos") {
    this.base = base;
    this.params = new URLSearchParams();
  }

  categoria(value) { this.params.set("categoria", value); return this; }
  precio(min, max) {
    if (min != null) this.params.set("precio_min", min);
    if (max != null) this.params.set("precio_max", max);
    return this;
  }
  search(q) { this.params.set("q", q); return this; }
  ordenar(campo, dir = "asc") {
    this.params.set("ordenar", campo);
    this.params.set("dir", dir);
    return this;
  }
  paginar(page, limit = 20) {
    this.params.set("page", page);
    this.params.set("limit", limit);
    return this;
  }

  build() {
    return `${this.base}?${this.params.toString()}`;
  }

  async execute() {
    const res = await fetch(this.build());
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  }
}

// Uso
const data = await new ProductoQuery()
  .categoria("tecnologia")
  .precio(100, 500)
  .ordenar("precio", "desc")
  .paginar(1, 20)
  .execute();
```

### Server-side vs client-side

- **Server-side**: adecuado para grandes datasets, índices DB.
- **Client-side**: filtro en memoria con pocos miles de registros.

```js
const filtrados = productos.filter(p =>
  (!cat   || p.categoria === cat) &&
  (!q     || p.nombre.toLowerCase().includes(q.toLowerCase())) &&
  (!min   || p.precio >= min) &&
  (!max   || p.precio <= max)
);
```

### Debounce en búsquedas

```js
const debounce = (fn, ms = 300) => {
  let id;
  return (...args) => {
    clearTimeout(id);
    id = setTimeout(() => fn(...args), ms);
  };
};

const buscar = debounce(async (q) => {
  const data = await api.get(`/productos?q=${q}`);
  render(data);
}, 400);

input.addEventListener("input", (e) => buscar(e.target.value));
```

### Actividad práctica

- Construye un filtro de productos combinando query params + debounce + paginación.

---

## Cliente HTTP completo (ejemplo integrador)

```js
// http.js

class HttpError extends Error {
  constructor(message, { status, body, code } = {}) {
    super(message);
    this.name = "HttpError";
    this.status = status;
    this.body = body;
    this.code = code;
  }
}

export class HttpClient {
  constructor(baseURL, options = {}) {
    this.baseURL = baseURL;
    this.defaultOptions = {
      timeout: 15000,
      headers: { Accept: "application/json" },
      ...options,
    };
    this.interceptors = { request: [], response: [] };
  }

  useRequest(fn)   { this.interceptors.request.push(fn);  return this; }
  useResponse(fn)  { this.interceptors.response.push(fn); return this; }

  async request(method, path, { body, params, headers, signal, ...rest } = {}) {
    let url = new URL(path, this.baseURL);
    if (params) Object.entries(params).forEach(([k, v]) => {
      if (v != null) url.searchParams.append(k, v);
    });

    let config = {
      method,
      headers: { ...this.defaultOptions.headers, ...headers },
      signal,
      ...rest,
    };
    if (body !== undefined) {
      if (body instanceof FormData) {
        config.body = body;
      } else {
        config.body = JSON.stringify(body);
        config.headers["Content-Type"] = "application/json";
      }
    }

    for (const fn of this.interceptors.request) config = await fn(config) || config;

    const ctrl = new AbortController();
    const timer = setTimeout(() => ctrl.abort(), config.timeout);

    try {
      const res = await fetch(url, config);
      clearTimeout(timer);
      let data = null;
      const ct = res.headers.get("Content-Type") || "";
      if (ct.includes("application/json")) data = await res.json().catch(() => null);
      else if (ct.includes("text"))        data = await res.text();

      let result = { status: res.status, ok: res.ok, data, headers: res.headers, config };
      for (const fn of this.interceptors.response) result = await fn(result) || result;

      if (!result.ok) {
        throw new HttpError(`HTTP ${result.status}`, {
          status: result.status, body: result.data,
        });
      }
      return result.data;
    } catch (err) {
      if (err.name === "AbortError") throw new HttpError("Tiempo agotado", { code: "TIMEOUT" });
      if (err instanceof HttpError) throw err;
      throw new HttpError(err.message || "Error de red", { code: "NETWORK" });
    }
  }

  get(path, opts)    { return this.request("GET", path, opts); }
  post(path, body, opts)   { return this.request("POST",   path, { body, ...opts }); }
  put(path, body, opts)    { return this.request("PUT",    path, { body, ...opts }); }
  patch(path, body, opts)  { return this.request("PATCH",  path, { body, ...opts }); }
  delete(path, opts) { return this.request("DELETE", path, opts); }
}

export const api = new HttpClient("https://api.example.com")
  .useRequest((config) => {
    const token = localStorage.getItem("token");
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
  })
  .useResponse(async ({ ok, status, data, config }) => {
    if (status === 401 && !config._retried) {
      try {
        const refresh = localStorage.getItem("refresh");
        const { data: { token } } = await new HttpClient("https://api.example.com").post("/auth/refresh", { refresh });
        localStorage.setItem("token", token);
        config.headers.Authorization = `Bearer ${token}`;
        config._retried = true;
      } catch (e) {
        localStorage.removeItem("token");
        location.href = "/login";
      }
    }
    return { ok, status, data, config, headers: undefined };
  });

export { HttpError };
```

### Uso

```js
import { api, HttpError } from "./http.js";

try {
  const productos = await api.get("/productos", {
    params: { page: 1, limit: 20, q: "laptop" },
  });
} catch (err) {
  if (err instanceof HttpError && err.code === "NETWORK") {
    console.log("Sin conexión");
  }
}
```

---

## Recursos complementarios

### Lecturas

- [MDN - Fetch API](https://developer.mozilla.org/es/docs/Web/API/Fetch_API/Using_Fetch)
- [REST API Tutorial](https://restfulapi.net/)
- [JWT.io - Intro](https://jwt.io/introduction)
- [HTTP Caching (MDN)](https://developer.mozilla.org/es/docs/Web/HTTP/Caching)
- [Status Code Definitions](https://developer.mozilla.org/es/docs/Web/HTTP/Status)

### APIs públicas para practicar

- [JSONPlaceholder](https://jsonplaceholder.typicode.com/)
- [Reqres](https://reqres.in/)
- [PokéAPI](https://pokeapi.co/)
- [Rick and Morty API](https://rickandmortyapi.com/)
- [Restcountries](https://restcountries.com/)
- [OpenWeatherMap](https://openweathermap.org/api)
- [CoinGecko](https://www.coingecko.com/en/api)
- [The Movie DB (TMDB)](https://www.themoviedb.org/documentation/api)

### Herramientas

- [Postman](https://www.postman.com/) o [Insomnia](https://insomnia.rest/)
- [Hoppscotch](https://hoppscotch.io/)
- [Thunder Client (VS Code)](https://marketplace.visualstudio.com/items?itemName=rangav.vscode-thunder-client)

---

## Evaluación sugerida

1. **Cliente HTTP (25%)**: implementa una clase con retries, timeout y Bearer automático.
2. **CRUD (30%)**: aplicación que consuma una API real (JSONPlaceholder o similar).
3. **Autenticación (20%)**: login, almacenamiento seguro, refresh automático, manejo de 401.
4. **Paginación y filtros (25%)**: listado con paginación + búsqueda con debounce + filtros múltiples.

---

> **Nota para el estudiante**: consumir APIs es el pan de cada día en el desarrollo web. Aprende a leer documentación de OpenAPI/Swagger, a manejar errores elegantemente y a diseñar un cliente HTTP reusable: tu yo del futuro te lo agradecerá.
