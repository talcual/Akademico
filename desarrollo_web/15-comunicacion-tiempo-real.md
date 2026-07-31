# Módulo 15: Comunicación en Tiempo Real

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Diferenciar las tecnologías de comunicación en tiempo real.
- Implementar polling, long polling y Server-Sent Events.
- Crear aplicaciones con WebSockets y Socket.IO.
- Construir un chat en tiempo real con broadcast.
- Implementar notificaciones push desde el navegador.

---

## 1. Polling

El **polling** es la técnica más simple: el cliente pregunta periódicamente al servidor.

```js
// Cliente
async function poll() {
  try {
    const res = await fetch("/api/messages?since=" + lastId);
    const data = await res.json();
    if (data.messages.length) render(data.messages);
    lastId = data.lastId;
  } catch (e) { /* ... */ }
  setTimeout(poll, 5000);
}
poll();
```

### Ventajas

- Simple.
- Funciona con cualquier backend.
- Compatible con todas las infraestructuras.

### Desventajas

- Desperdicia peticiones (cuando no hay cambios).
- Latencia proporcional al intervalo.
- Carga del servidor crece con el número de clientes.

### Short polling vs Long polling

- **Short polling**: cliente pregunta cada X segundos.
- **Long polling**: el servidor espera antes de responder.

### Cuándo usarlo

- Datos que cambian con frecuencia pero toleran latencia (ej. precio de acciones cada minuto).
- Compatible con HTTP/1.1 sin infraestructura extra.

### Actividad práctica

- Implementa un endpoint que devuelve mensajes nuevos desde un timestamp y un cliente que los pide cada 5 segundos.

---

## 2. Long Polling

El cliente hace una petición y el servidor **la mantiene abierta** hasta tener datos nuevos.

### Cliente

```js
async function longPoll() {
  try {
    const res = await fetch("/api/messages/wait?since=" + lastId);
    const data = await res.json();
    render(data.messages);
    lastId = data.lastId;
  } catch (e) {
    await new Promise(r => setTimeout(r, 3000));
  }
  longPoll(); // reconectar inmediatamente
}
longPoll();
```

### Servidor (Node.js)

```js
import express from "express";
const app = express();

const subscribers = new Map(); // userId -> [(since, res), ...]

app.get("/api/messages/wait", (req, res) => {
  const since = Number(req.query.since) || 0;
  const userId = req.user.id;

  const subs = subscribers.get(userId) ?? [];
  subs.push({ since, res });
  subscribers.set(userId, subs);

  const timeout = setTimeout(() => {
    res.json({ messages: [], lastId: since });
  }, 30000);

  req.on("close", () => clearTimeout(timeout));
});

const broadcast = (userId, message) => {
  const subs = subscribers.get(userId) ?? [];
  for (const sub of subs) {
    if (message.id > sub.since) {
      clearTimeout(sub.timeout);
      sub.res.json({ messages: [message], lastId: message.id });
    }
  }
  subscribers.set(userId, subs.filter(s => !s.ended));
};
```

### Ventajas

- Latencia baja (push-like).
- Compatible con HTTP/1.1.
- Funciona con proxies.

### Desventajas

- Conexiones abiertas consumen recursos.
- Requiere manejo cuidadoso de timeouts y timeouts del proxy.
- No escala muy bien (mantener miles de conexiones).

### Buenas prácticas

- Timeout razonable (30-60 segundos).
- Reconexión inmediata si se cierra.
- Backoff exponencial ante errores.

### Actividad práctica

- Implementa long polling con timeout de 30s y reconexión cliente en bucle.

---

## 3. Server-Sent Events (SSE)

**SSE** permite al servidor enviar actualizaciones al cliente sobre una conexión HTTP unidireccional.

### Server

```js
app.get("/api/stream", (req, res) => {
  res.set({
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache, no-transform",
    "Connection": "keep-alive",
    "X-Accel-Buffering": "no", // desactivar buffering en nginx
  });
  res.flushHeaders();

  const send = (event, data) => {
    res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  send("hello", { message: "Conectado", time: Date.now() });

  const interval = setInterval(() => {
    send("tick", { time: Date.now() });
  }, 1000);

  req.on("close", () => clearInterval(interval));
});
```

### Cliente

```js
const es = new EventSource("/api/stream");

es.addEventListener("hello", (e) => console.log("Hola:", e.data));
es.addEventListener("tick",  (e) => console.log("Tick:", e.data));

es.onerror = (e) => console.log("Error SSE:", e);
es.onopen  = () => console.log("SSE conectado");
```

### Formato del protocolo

```
event: message
data: {"id": 1, "text": "hola"}

event: tick
data: 1234567890

: comentario ignorado

retry: 5000
```

### Reconexión automática

El navegador **reconecta automáticamente** por defecto. Puedes ajustar el retry:

```
retry: 5000

```

### Ventajas

- HTTP simple, pasa por proxies.
- Auto-reconexión del navegador.
- Texto plano (fácil debug con `curl`).

### Desventajas

- **Unidireccional** (servidor → cliente).
- Limitado a 6 conexiones por origen en HTTP/1.1 (HTTP/2 lo mitiga).
- Buffering de proxies puede romperlo (header `X-Accel-Buffering: no`).
- Solo texto (UTF-8).

### Patrones

```js
// Heartbeat para mantener viva la conexión
setInterval(() => res.write(":heartbeat\n\n"), 15000);
```

### Actividad práctica

- Crea un endpoint SSE que transmita el precio de un activo y actualice la UI cada vez que cambia.

---

## 4. WebSockets

**WebSocket** es un protocolo **full-duplex** sobre TCP, ideal para aplicaciones que requieren comunicación bidireccional en tiempo real.

### Características

- Conexión persistente sobre HTTP (upgrade).
- Bidireccional.
- Bajo overhead.
- Mensajes binarios o texto.

### Handshake

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9bYG...
```

### Cliente nativo

```js
const ws = new WebSocket("ws://localhost:3000/chat");

ws.onopen    = () => ws.send("Hola server");
ws.onmessage = (e) => console.log("Mensaje:", e.data);
ws.onerror   = (e) => console.error("WS error", e);
ws.onclose   = (e) => console.log("Cerrado:", e.code, e.reason);

ws.send(JSON.stringify({ type: "msg", text: "hola" }));
ws.close(1000, "Adiós"); // 1000 = normal closure
```

### Ready states

```js
ws.readyState:
CONNECTING = 0
OPEN       = 1
CLOSING    = 2
CLOSED     = 3
```

### Servidor nativo (`ws`)

```bash
npm install ws
```

```js
import { WebSocketServer } from "ws";
import http from "node:http";

const server = http.createServer();
const wss = new WebSocketServer({ server, path: "/chat" });

wss.on("connection", (socket, req) => {
  console.log("Cliente conectado");

  socket.on("message", (raw, isBinary) => {
    const msg = isBinary ? raw : raw.toString();
    console.log("Mensaje:", msg);

    // Echo
    socket.send(`Eco: ${msg}`);

    // Broadcast a todos
    wss.clients.forEach(client => {
      if (client !== socket && client.readyState === 1) {
        client.send(msg);
      }
    });
  });

  socket.on("close", (code, reason) => {
    console.log(`Cerrado (${code}): ${reason}`);
  });
});

server.listen(3000);
```

### Frame format (resumen)

- `FIN` (1 bit): último fragmento.
- `opcode` (4 bits): texto (1), binario (2), close (8), ping (9), pong (10).
- `MASK` (1 bit): siempre 1 en cliente → servidor.
- `payload length`: 7/16/64 bits.
- `masking key` (cliente → server): 4 bytes.
- `payload data`: contenido.

### Heartbeat / ping-pong

```js
// Servidor
setInterval(() => {
  wss.clients.forEach((socket) => {
    if (socket.isAlive === false) return socket.terminate();
    socket.isAlive = false;
    socket.ping();
  });
}, 30000);

wss.on("connection", (socket) => {
  socket.isAlive = true;
  socket.on("pong", () => (socket.isAlive = true));
});
```

### Subprotocolos

```js
// Cliente
new WebSocket("wss://server/chat", ["graphql-ws", "wamp"]);

// Servidor
new WebSocketServer({ server, handleProtocols: (p, req) => p[0] });
```

### TLS (`wss://`)

Imprescindible en producción.

### Actividad práctica

- Construye un servidor WebSocket con `ws`, broadcast de mensajes a todos los clientes y heartbeat.

---

## 5. Socket.IO

Biblioteca que envuelve WebSocket con **fallbacks, rooms, reconexión** y API simplificada.

### Instalación

```bash
npm install socket.io
npm install socket.io-client     # cliente
```

### Servidor

```js
import { createServer } from "node:http";
import { Server } from "socket.io";

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: { origin: "http://localhost:5173" },
});

io.on("connection", (socket) => {
  console.log("Conectado:", socket.id);

  socket.on("chat:message", (msg) => {
    io.emit("chat:message", { id: socket.id, text: msg, at: Date.now() });
  });

  socket.on("chat:typing", () => {
    socket.broadcast.emit("chat:typing", { id: socket.id });
  });

  socket.on("disconnect", (reason) => {
    console.log("Desconectado:", reason);
  });
});

httpServer.listen(3000);
```

### Cliente

```js
import { io } from "socket.io-client";

const socket = io("http://localhost:3000", { withCredentials: true });

socket.on("connect",    () => console.log("Conectado:", socket.id));
socket.on("disconnect", (reason) => console.log("Desconectado:", reason));
socket.on("connect_error", (err) => console.error("Conexión:", err));

socket.on("chat:message", (msg) => {
  console.log("Mensaje:", msg);
});

socket.emit("chat:message", "Hola");

document.querySelector("#input").addEventListener("keyup", () => {
  socket.emit("chat:typing");
});
```

### Namespaces

```js
// Servidor
const admin = io.of("/admin");
admin.use(adminMiddleware);
admin.on("connection", (socket) => { /* ... */ });

// Cliente
const admin = io("http://localhost:3000/admin");
```

### Rooms

```js
socket.join("room1");

socket.on("chat:message", (msg) => {
  io.to("room1").emit("chat:message", msg);
});

socket.to("room1").emit("user:join", { id: socket.id });
io.to(socket.rooms).emit("hi");
```

### Middleware (autenticación)

```js
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const user = verifyAccess(token);
    socket.user = user;
    next();
  } catch {
    next(new Error("invalid token"));
  }
});

io.on("connection", (socket) => {
  console.log(`User ${socket.user.sub} connected`);
});
```

Cliente:

```js
const socket = io("http://localhost:3000", {
  auth: { token: localStorage.getItem("accessToken") },
});
```

### ACK (confirmación de recepción)

```js
// Servidor
socket.emit("chat:message", msg, (ack) => {
  if (ack.error) console.error("Cliente no recibió");
});

// Cliente
socket.emit("chat:message", msg, (response) => {
  console.log("Recibido:", response);
});
```

### Volatile (eventos que no se retransmiten)

```js
socket.volatile.emit("cursor:move", { x: 100, y: 200 });
```

### Reconexión del cliente

Socket.IO reconecta automáticamente con backoff exponencial.

### Actividad práctica

- Crea una app con Socket.IO que tenga un chat global, rooms por canal y autenticación en handshake.

---

## 6. Broadcast

Enviar el mismo mensaje a **múltiples clientes**.

### Tipos de broadcast

#### A todos

```js
io.emit("anuncio", { text: "Bienvenidos" });
socket.broadcast.emit("alguien_unido", { id: socket.id });
```

#### A una sala (room)

```js
io.to("sala1").emit("msg", msg);
io.to(["sala1", "sala2"]).emit("anuncio", { ... });

socket.to("sala1").emit("msg", msg); // excepto el emisor
```

#### A un socket específico

```js
io.to(socketId).emit("private", { msg: "..." });
```

#### Excluir socket

```js
io.except(socketId).emit("event", payload);
```

### Patrón Pub/Sub con Redis adapter

Para múltiples servidores.

```bash
npm install @socket.io/redis-adapter redis
```

```js
import { createAdapter } from "@socket.io/redis-adapter";
import { createClient } from "redis";

const pubClient = createClient({ url: "redis://localhost:6379" });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));
```

### Scales horizontalmente

```
┌────────┐    ┌────────┐    ┌────────┐
│  Node1 │    │  Node2 │    │  Node3 │
└───┬────┘    └───┬────┘    └───┬────┘
    └─────────────┼──────────────┘
                  │
            ┌─────┴─────┐
            │   Redis   │  ← pub/sub
            └───────────┘
```

### Actividad práctica

- Configura un servidor con dos procesos y Redis para que un mensaje publicado en uno llegue a clientes conectados en el otro.

---

## 7. Chat en tiempo real

Implementación completa con **Socket.IO** + frontend.

### Backend

```js
// server.js
import { createServer } from "node:http";
import { Server } from "socket.io";
import jwt from "jsonwebtoken";

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: { origin: "http://localhost:5173", credentials: true },
});

const SECRET = process.env.JWT_SECRET;
const messages = []; // en producción, DB

io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    socket.user = jwt.verify(token, SECRET);
    next();
  } catch {
    next(new Error("auth"));
  }
});

io.on("connection", (socket) => {
  socket.join("global");
  socket.emit("history", messages.slice(-50));

  socket.on("msg:send", (text) => {
    const msg = { id: Date.now(), user: socket.user.name, text, at: Date.now() };
    messages.push(msg);
    io.to("global").emit("msg:new", msg);
  });

  socket.on("msg:typing", () => {
    socket.to("global").emit("msg:typing", { user: socket.user.name });
  });

  socket.on("disconnect", () => {
    io.to("global").emit("user:left", { id: socket.user.name });
  });
});

httpServer.listen(3000);
```

### Frontend (React)

```jsx
// ChatRoom.jsx
import { useEffect, useState, useRef } from "react";
import { io } from "socket.io-client";

const token = sessionStorage.getItem("accessToken");
const socket = io("http://localhost:3000", { auth: { token } });

export default function ChatRoom() {
  const [messages, setMessages] = useState([]);
  const [text, setText] = useState("");
  const [typing, setTyping] = useState(null);
  const ref = useRef(null);

  useEffect(() => {
    socket.on("history",  (m) => setMessages(m));
    socket.on("msg:new",  (m) => setMessages((prev) => [...prev, m]));
    socket.on("msg:typing", (u) => { setTyping(u.user); setTimeout(() => setTyping(null), 1500); });
    socket.on("connect_error", (e) => console.error("Conexión:", e.message));
    return () => {
      socket.off("history");
      socket.off("msg:new");
      socket.off("msg:typing");
    };
  }, []);

  useEffect(() => {
    ref.current?.scrollTo({ top: ref.current.scrollHeight, behavior: "smooth" });
  }, [messages]);

  const send = (e) => {
    e.preventDefault();
    if (!text.trim()) return;
    socket.emit("msg:send", text);
    setText("");
  };

  const onType = () => socket.emit("msg:typing");

  return (
    <section className="chat">
      <ul ref={ref}>
        {messages.map((m) => (
          <li key={m.id}>
            <strong>{m.user}:</strong> {m.text}
          </li>
        ))}
      </ul>
      {typing && <p><em>{typing} está escribiendo...</em></p>}
      <form onSubmit={send}>
        <input value={text} onChange={(e) => { setText(e.target.value); onType(); }} />
        <button>Enviar</button>
      </form>
    </section>
  );
}
```

### Características avanzadas

- Indicador de escribiendo (mostrado arriba).
- Confirmación de lectura.
- Mensajes privados 1:1 (rooms).
- Subir archivos vía pre-signed URLs.
- Persistencia con offset (`LIMIT` + cursor).

### Salas privadas

```js
socket.on("dm:join", (otherId) => {
  const room = [socket.user.id, otherId].sort().join(":");
  socket.join(room);
});
```

```js
io.to(room).emit("msg:new", msg);
```

### Markdown y menciones

Cliente transforma con `marked` o similar, y sanitiza con `DOMPurify`.

```js
import { marked } from "marked";
import DOMPurify from "dompurify";

const html = DOMPurify.sanitize(marked.parse(msg.text));
```

### Actividad práctica

- Construye un chat completo con historial, salas privadas, indicador de escribiendo y scroll automático.

---

## 8. Notificaciones Push (general)

Disparar mensajes al cliente incluso cuando **no tiene la app abierta en primer plano**.

### Opciones

| Tecnología                  | Bidireccional | Soporte navegador | Persistente offline |
| --------------------------- | ------------- | ------------------ | ------------------- |
| WebSockets / Socket.IO      | Sí            | Sí                 | No (se pierde)      |
| Long polling                | No            | Sí                 | No                  |
| Server-Sent Events          | No            | Sí                 | No (se pierde)      |
| Push API (Service Worker)   | No            | Sí (HTTPS)         | Sí                  |
| Web Push con Service Worker| No            | Sí (HTTPS)         | Sí (Service Worker) |

### Requisitos para Web Push

- HTTPS (servicio válido).
- Permiso del usuario.
- Service Worker instalado.
- Endpoint público del proveedor (VAPID keys).

### Actividad práctica

- Compara cuándo usar WebSockets, SSE y Push API según el caso.

---

## 9. Web Push API

Permite al servidor enviar notificaciones al navegador **incluso si la app está cerrada**.

### Componentes

1. **VAPID keys**: par de claves para autenticar al servidor.
2. **Service Worker**: receptor en el navegador.
3. **PushManager**: suscripción del usuario.
4. **Backend**: envía notificaciones usando la suscripción.

### Generar VAPID keys

```bash
npm install web-push
npx web-push generate-vapid-keys
```

```js
// server.js
import webpush from "web-push";

webpush.setVapidDetails(
  "mailto:admin@example.com",
  process.env.VAPID_PUBLIC,
  process.env.VAPID_PRIVATE
);
```

### Estructura

```
┌──────────────┐                                 ┌──────────────┐
│  Navegador   │ ─ 1. Suscribirse a Push ─▶      │   Tu App     │
│              │ ◀── 2. Subscription ────         │   (server)   │
│              │                                 │              │
│              │ ─── 3. Enviar push ───────────▶  │              │
│              │                                 └──────┬───────┘
│  Service     │                                        │
│  Worker      │ ◀── 4. Push service (FCM/Mozilla) ────┘
│ (push event) │ → muestra notificación
└──────────────┘
```

### Cliente: Service Worker

```js
// public/sw.js
self.addEventListener("push", (e) => {
  const data = e.data?.json() ?? { title: "Notificación", body: "" };
  e.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: "/icon.png",
      badge: "/badge.png",
      data: data.url,
    })
  );
});

self.addEventListener("notificationclick", (e) => {
  e.notification.close();
  e.waitUntil(clients.openWindow(e.notification.data));
});
```

### Cliente: suscripción

```js
// src/notifications.js
export async function subscribeToPush() {
  if (!("serviceWorker" in navigator)) return;
  const reg = await navigator.serviceWorker.register("/sw.js");
  const permission = await Notification.requestPermission();
  if (permission !== "granted") return null;

  const sub = await reg.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: process.env.NEXT_PUBLIC_VAPID_KEY,
  });

  // Enviar al backend
  await fetch("/api/notifications/subscribe", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(sub),
  });

  return sub;
}
```

### Backend: guardar suscripción

```js
import { Subscription } from "./models/index.js";

router.post("/api/notifications/subscribe", authMiddleware, async (req, res) => {
  const sub = Subscription.build({
    userId: req.user.sub,
    endpoint: req.body.endpoint,
    p256dh: req.body.keys.p256dh,
    auth:   req.body.keys.auth,
  });
  await sub.save();
  res.status(201).json({ ok: true });
});
```

### Backend: enviar notificación

```js
import webpush from "web-push";

async function sendToUser(userId, payload) {
  const subs = await Subscription.findAll({ where: { userId } });
  for (const sub of subs) {
    try {
      await webpush.sendNotification(
        {
          endpoint: sub.endpoint,
          keys: { p256dh: sub.p256dh, auth: sub.auth },
        },
        JSON.stringify(payload)
      );
    } catch (err) {
      if (err.statusCode === 410) await sub.destroy(); // suscripción muerta
      else throw err;
    }
  }
}

// Uso
sendToUser(42, {
  title: "Nuevo mensaje",
  body:  "Tienes un comentario nuevo",
  url:   "/posts/42",
});
```

### Acciones en la notificación

```js
self.registration.showNotification("Nuevo mensaje", {
  body: "Tienes un comentario",
  actions: [
    { action: "view", title: "Ver" },
    { action: "dismiss", title: "Cerrar" },
  ],
});

self.addEventListener("notificationclick", (e) => {
  if (e.action === "view") clients.openWindow("/posts/42");
  e.notification.close();
});
```

### Modo silencioso (sin UI)

```js
webpush.sendNotification(sub, JSON.stringify({ silent: true }));
```

En el Service Worker:

```js
self.addEventListener("push", (e) => {
  const data = e.data?.json();
  if (data.silent) {
    fetch("/api/sync");
    return;
  }
  e.waitUntil(self.registration.showNotification(data.title, { ... }));
});
```

### Etiquetas y agrupación

```js
self.registration.showNotification("Mensaje", {
  tag: "messages",
  renotify: true,
});
```

Reemplaza notificaciones con el mismo `tag` en lugar de apilar.

### Buenas prácticas

- **HTTPS obligatorio**.
- VAPID keys privadas en servidor, públicas en cliente.
- **TTL corto**: las notificaciones dead se eliminan en 24-48h.
- Manejar **endpoint expirado** (410 Gone).
- Limita frecuencia para no molestar al usuario.

### Actividad práctica

- Implementa el ciclo completo: VAPID keys, Service Worker, suscripción, envío y click handler.

---

## Proyecto integrador del módulo

App de chat en tiempo real con:
- Autenticación JWT en handshake de Socket.IO.
- Salas públicas y privadas (DMs).
- Persistencia con PostgreSQL.
- Indicador de escribiendo.
- Historial paginado.
- Notificaciones push cuando el usuario está offline.
- Despliegue (opcional): Railway/Render + Vercel/Netlify.

### Estructura backend

```
backend/
├── package.json
├── .env
└── src/
    ├── index.js
    ├── app.js
    ├── socket.js
    ├── controllers/
    ├── services/
    ├── models/
    └── routes/
```

### `socket.js`

```js
import { Server } from "socket.io";
import jwt from "jsonwebtoken";
import { Message, Room, DirectRoom } from "./models/index.js";

export function createSocket(httpServer) {
  const io = new Server(httpServer, {
    cors: { origin: process.env.FRONTEND_URL, credentials: true },
  });

  io.use((socket, next) => {
    try {
      socket.user = jwt.verify(socket.handshake.auth.token, process.env.JWT_SECRET);
      next();
    } catch { next(new Error("auth")); }
  });

  io.on("connection", (socket) => {
    socket.on("join", async (room) => {
      socket.join(room);
      const history = await Message.findAll({
        where: { room },
        order: [["createdAt", "DESC"]],
        limit: 50,
      });
      socket.emit("history", history.reverse());
    });

    socket.on("msg", async ({ room, text }) => {
      const msg = await Message.create({
        room,
        userId: socket.user.sub,
        text,
      });
      io.to(room).emit("msg", msg);
    });

    socket.on("typing", ({ room }) => {
      socket.to(room).emit("typing", { user: socket.user.name });
    });
  });

  return io;
}
```

### Estructura frontend

```
frontend/
├── package.json
├── public/
│   ├── sw.js
│   └── icon.png
└── src/
    ├── App.jsx
    ├── hooks/
    │   ├── useSocket.js
    │   └── usePush.js
    ├── pages/
    │   ├── Login.jsx
    │   ├── Chat.jsx
    └── components/
```

### `useSocket.js`

```js
import { useEffect, useState } from "react";
import { io } from "socket.io-client";

export function useSocket(token) {
  const [socket, setSocket] = useState(null);

  useEffect(() => {
    if (!token) return;
    const s = io(import.meta.env.VITE_API_URL, { auth: { token } });
    setSocket(s);
    return () => s.disconnect();
  }, [token]);

  return socket;
}
```

### `usePush.js`

```js
import { useEffect } from "react";

export function usePush(token) {
  useEffect(() => {
    if (!token) return;
    if (!("serviceWorker" in navigator)) return;
    (async () => {
      const reg = await navigator.serviceWorker.register("/sw.js");
      const permission = await Notification.requestPermission();
      if (permission !== "granted") return;
      const sub = await reg.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: import.meta.env.VITE_VAPID_KEY,
      });
      await fetch("/api/notifications/subscribe", {
        method: "POST",
        headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
        body: JSON.stringify(sub),
      });
    })();
  }, [token]);
}
```

---

## Recursos complementarios

### Lecturas

- [MDN - WebSockets](https://developer.mozilla.org/es/docs/Web/API/WebSockets_API)
- [MDN - Server-Sent Events](https://developer.mozilla.org/es/docs/Web/API/Server-sent_events)
- [MDN - Push API](https://developer.mozilla.org/es/docs/Web/API/Push_API)
- [web.dev - Notifications](https://web.dev/notifications/)
- [Socket.IO Docs](https://socket.io/docs/v4/)

### Herramientas

- [Beeceptor](https://beeceptor.com/) (mock de API/Webhook).
- [web-push npm](https://www.npmjs.com/package/web-push).
- [Socket.IO Admin UI](https://socket.io/docs/v4/admin-ui/).
- [Try it: `wscat`](https://github.com/websockets/wscat).

### Patrones y buenas prácticas

- Limita tamaño de mensajes y rate (anti-spam).
- Heartbeat para detectar desconexiones (ping/pong).
- Reconexión con **backoff exponencial + jitter**.
- Idempotencia con `messageId`.
- Persistencia: DB transaction + emisión posterior.
- **No confíes** en timestamps del cliente.

---

## Evaluación sugerida

1. **SSE + Long Polling (20%)**: dos endpoints con casos de uso distintos y un cliente para cada uno.
2. **WebSocket nativo (20%)**: chat usando `ws` con heartbeat.
3. **Socket.IO con rooms (30%)**: chat multi-sala con auth en handshake, typing indicator e historial.
4. **Notificaciones Push (30%)**: ciclo completo VAPID + Service Worker + suscripción + envío + click.

---

> **Nota para el estudiante**: la elección de tecnología depende del caso. **Chat 1:1 → WebSockets. Stream de cotizaciones → SSE. Noticias offline → Push API. Tareas de baja prioridad → Long Polling/Polling.** El "tiempo real" no es una tecnología, es un requisito que puede resolverse de varias formas.
