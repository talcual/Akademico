# Comunicación en Tiempo Real

## Objetivo de aprendizaje

El "tiempo real" no es una sola cosa. Es un espectro de técnicas con distintas garantías de latencia, dirección y complejidad. Aquí se estudian los mecanismos que permiten que **dos o más clientes vean cambios del otro en milisegundos**, sin hacer polling.

---

## Mapa general

| Tecnología            | Dirección        | Transporte | Caso típico                                |
| --------------------- | ---------------- | ---------- | ------------------------------------------ |
| WebSockets (Socket.io) | Bidireccional  | TCP        | Chat, juegos, colaboración                 |
| Server-Sent Events    | Servidor → Cliente | HTTP     | Feeds, notificaciones, dashboards         |
| WebRTC                | Peer-to-peer     | UDP/SRTP   | Video, voz, transferencia P2P             |
| Presence               | Servicio       | Mixto      | "¿Quién está conectado ahora?"            |
| Collaborative Editing | Bidireccional   | Mixto      | Documentos compartidos tipo Google Docs    |
| CRDT                   | Modelo de datos | —         | Resolución de conflictos sin servidor autoritativo |

---

## 1. WebSockets y Socket.io

### WebSocket nativo

Protocolo bidireccional sobre TCP. Handshake HTTP → conexión persistente full-duplex.

```ts
// cliente
const ws = new WebSocket("wss://api.example.com/chat");
ws.onmessage = (e) => console.log(JSON.parse(e.data));
ws.send(JSON.stringify({ tipo: "hola" }));
```

```ts
// servidor (Node + ws)
import { WebSocketServer } from "ws";
const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (socket) => {
  socket.on("message", (raw) => {
    const msg = JSON.parse(raw.toString());
    // broadcast
    wss.clients.forEach((c) => c.send(JSON.stringify({ eco: msg })));
  });
});
```

### Socket.io

Capa de **abstracciones sobre WebSocket** con:

- Reconexión automática.
- Fallback a long-polling si WebSocket no está disponible.
- Salas (`rooms`) y namespaces.
- Acuse de recibo (`ack`).
- Broadcast eficiente.

```ts
// servidor
import { Server } from "socket.io";
const io = new Server(3000, { cors: { origin: "*" } });

io.on("connection", (socket) => {
  socket.join("sala:general");

  socket.on("mensaje", (texto, ack) => {
    io.to("sala:general").emit("mensaje", {
      de: socket.id,
      texto,
      ts: Date.now(),
    });
    ack?.({ ok: true });
  });
});
```

```ts
// cliente
import { io } from "socket.io-client";
const socket = io("https://api.example.com");

socket.emit("mensaje", "hola", (resp) => console.log("ack:", resp));
socket.on("mensaje", (m) => console.log(`${m.de}: ${m.texto}`));
```

### Cuándo usarlo

- Chat, mensajería.
- Juegos, interacción bidireccional intensa.
- Cualquier UI que requiera **empuje del servidor** y **envío del cliente** con baja latencia.

---

## 2. Server-Sent Events (EventSource)

### Idea central

Conexión HTTP unidireccional del servidor al cliente. El navegador reconecta solo si se cae.

```ts
// servidor
app.get("/stream/alertas", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const send = (data) => res.write(`data: ${JSON.stringify(data)}\n\n`);

  const interval = setInterval(() => send({ ts: Date.now() }), 1000);
  req.on("close", () => clearInterval(interval));
});
```

```ts
// cliente
const sse = new EventSource("/stream/alertas");
sse.addEventListener("alerta", (e) => {
  const data = JSON.parse((e as MessageEvent).data);
  console.log("Alerta:", data);
});
```

### Por qué importa en 2026

- Es **HTTP estándar**, atraviesa proxies, CDNs y firewalls sin fricción.
- No requiere librería. Una conexión `fetch` con `ReadableStream` también sirve.
- Su modelo unidireccional es perfecto para **dashboards, logs en vivo, feeds**.

### Cuándo usarlo

- Solo necesitas **empuje del servidor**.
- Quieres simplicidad operativa y compatibilidad máxima.
- El cliente no necesita mandar mensajes por el mismo canal (usa REST aparte).

---

## 3. WebRTC

### Idea central

Permite **comunicación peer-to-peer** entre navegadores (o apps) con audio, video y datos arbitrarios. El servidor solo se usa para la **señalización** (intercambiar candidatos ICE y ofertas SDP).

### Tres piezas clave

1. **Signaling**: cómo los peers se encuentran. NO está estandarizado. Tú eliges (WebSocket, Socket.io, REST).
2. **ICE / STUN / TURN**: cómo atraviesan NATs.
   - **STUN**: descubre la IP pública.
   - **TURN**: relay cuando no hay ruta directa.
3. **SDP offer/answer**: descripción de los flujos de medios.

### Flujo simplificado

```
A                            SignalingServer                    B
│─── offer SDP ──────────────────────────────────────────────►│
│◄── answer SDP ─────────────────────────────────────────────│
│─── ICE candidates (bidireccional, trickle ICE) ──────────►│
│                                                              │
│◄══════════════ conexión P2P directa ══════════════════════►│
```

### Ejemplo: llamada 1-a-1 (extracto)

```ts
// A
const pc = new RTCPeerConnection({ iceServers: [{ urls: "stun:stun.l.google.com:19302" }] });
pc.onicecandidate = (e) => e.candidate && signaling.send({ type: "ice", candidate: e.candidate });

const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
signaling.send({ type: "offer", sdp: offer });

// al recibir answer
await pc.setRemoteDescription(new RTCSessionDescription(answer));
```

```ts
// B (simétrico pero inverso)
pc.ontrack = (e) => (videoElement.srcObject = e.streams[0]);
pc.ondatachannel = (e) => e.channel.onmessage = (m) => console.log("datos:", m.data);

pc.onicecandidate = (e) => e.candidate && signaling.send({ type: "ice", candidate: e.candidate });
signaling.on("offer", async (offer) => {
  await pc.setRemoteDescription(offer);
  const answer = await pc.createAnswer();
  await pc.setLocalDescription(answer);
  signaling.send({ type: "answer", sdp: answer });
});
```

### DataChannel

Además de audio/video, WebRTC permite **enviar datos arbitrarios** (mensajes, archivos, estado de juego) con baja latencia y sin pasar por tu servidor.

```ts
const channel = pc.createDataChannel("juego");
channel.onopen = () => channel.send("ready");
channel.onmessage = (e) => console.log("peer dice:", e.data);
```

### Cuándo usarlo

- Videollamadas, conferencias.
- Transferencia de archivos P2P.
- Juegos con estado sincronizado en tiempo real entre pocos clientes.
- Cualquier cosa donde la **latencia de servidor es inaceptable**.

### Limitaciones operativas

- TURN servers son necesarios para una fracción significativa de usuarios (NATs simétricos).
- La señalización es tu responsabilidad.
- No escala a "1000 participantes viendo un stream" sin SFU/MCU.

---

## 4. Presence (Presencia)

### Idea central

Responder en tiempo real a: **"¿quién está conectado y desde dónde?"**.

### Implementación ingenua

```ts
// al conectar
await redis.sadd("presence:usuarios", userId);
await redis.setex(`presence:${userId}`, 60, JSON.stringify({ ts: Date.now() }));

// heartbeat cada 30s
setInterval(async () => {
  await redis.setex(`presence:${userId}`, 60, JSON.stringify({ ts: Date.now() }));
}, 30_000);

// al desconectar
await redis.srem("presence:usuarios", userId);
```

### Implementación con Redis + Pub/Sub

```
WebSocket Gateway ──publish──► presence:online
                                     │
                                     ▼
                              Redis (Set + TTL)
                                     │
                                     ▼
                          Otros gateways se enteran
```

### Patrones avanzados

- **Online/Offline/Away**: combina TTL con estado explícito.
- **Typing indicators**: estado efímero con expiración corta (2-3s).
- **Multi-device**: un usuario puede tener varias sesiones; se cuenta por device.

### Cuándo usarlo

- Indicadores "en línea" en chat.
- Edición colaborativa ("María está editando esta celda").
- Listas de usuarios activos en un dashboard.
- Juegos (lista de amigos conectados).

---

## 5. Collaborative Editing (Edición Colaborativa)

### El problema

Dos personas editan **el mismo documento al mismo tiempo**. ¿Cómo se sincronizan sin perder cambios?

```
Estado inicial: "Hola"
A inserta " mundo" en posición 4  →  "Hola mundo"
B borla la posición 0  (también trabaja con su copia)
```

Si B aplica su borrado al estado final de A, podría borrar otra cosa. La sincronización ingenua rompe la consistencia.

### Solución: la teoría de la copia correcta

Existen dos familias principales:

1. **Operational Transformation (OT)**: usado por Google Docs en sus inicios. Las operaciones se transforman unas contra otras antes de aplicarse.
2. **CRDT**: estructuras de datos diseñadas matemáticamente para converger **sin servidor autoritativo**.

### Pipeline típico de un editor colaborativo

```
  Usuario A                        Servidor                      Usuario B
  Edita "abc" ──op──►  Cola OT/CRDT ──op──►  Broadcast ──►  Aplica "abc"
                            │
                            └── si hay conflicto, transforma
```

### Servicios y librerías

- **Yjs** (CRDT): librería JS madura.
- **Automerge** (CRDT):，同样 tipo de modelo, más portable.
- **Liveblocks / PartyKit / Y-Sweet**: backend托管 para collaborative apps.
- **ShareDB**: OT clásico.

---

## 6. CRDT (Conflict-free Replicated Data Types)

### Idea central

Tipos de datos **diseñados para que múltiples réplicas converjan al mismo estado** sin coordinación central, aplicando operaciones en cualquier orden.

### Ejemplo: G-Counter (Grow-only Counter)

Cada réplica tiene su propio contador. El total es la suma de todos.

```ts
class GCounter {
  private counts: Map<string, number> = new Map();

  increment(replicaId: string, n = 1) {
    this.counts.set(replicaId, (this.counts.get(replicaId) ?? 0) + n);
  }

  value(): number {
    return [...this.counts.values()].reduce((a, b) => a + b, 0);
  }

  merge(other: GCounter) {
    for (const [id, count] of other.counts) {
      this.counts.set(id, Math.max(this.counts.get(id) ?? 0, count));
    }
  }
}

const a = new GCounter();
a.increment("A", 3);

const b = new GCounter();
b.increment("B", 5);

a.merge(b);
b.merge(a);

console.log(a.value()); // 8
console.log(b.value()); // 8
```

### Ejemplo: LWW Register (Last-Writer-Wins)

Cada escritura lleva un timestamp. Al hacer merge, gana la más reciente.

```ts
class LWWRegister<T> {
  constructor(private value: T, private ts: number) {}

  set(value: T) {
    if (Date.now() > this.ts) {
      this.value = value;
      this.ts = Date.now();
    }
  }

  merge(other: LWWRegister<T>) {
    if (other.ts > this.ts) {
      this.value = other.value;
      this.ts = other.ts;
    }
  }
}
```

### Ejemplo: Yjs (Y.Text en la práctica)

```ts
import * as Y from "yjs";
import { WebsocketProvider } from "y-websocket";

const ydoc = new Y.Doc();
const provider = new WebsocketProvider("wss://...", "sala-doc-1", ydoc);
const ytext = ydoc.getText("contenido");

ytext.insert(0, "Hola");

// dos clientes ven lo mismo eventualmente, sin lock central
```

### Propiedades garantizadas

- **Convergence**: todas las réplicas convergen al mismo estado.
- **Commutativity**: el orden de aplicación no importa.
- **Idempotencia**: aplicar la misma operación dos veces no rompe nada.

### Trade-offs

- **Metadatos**: cada CRDT lleva info de réplica, timestamps, IDs lógicos. Más memoria.
- **Complejidad**: diseñar CRDTs nuevos es difícil; usar librerías como Yjs es lo sensato.
- **Semántica**: "qué gana" no siempre es intuitivo. Yjs usa caracteres únicos, Automerge usa JSON merges.

### Cuándo usarlos

- Edición colaborativa de texto, JSON, listas.
- Apps offline-first que sincronizan al reconectar.
- Juegos con estado compartido entre clientes.

---

## Cómo se combinan en la práctica

```
                         ┌─────────────────┐
                         │  Signaling      │  (WebSocket/Socket.io)
                         │  Server         │
                         └─────────────────┘
                              │         ▲
                 ICE/SDP      │         │ ICE/SDP
                              ▼         │
   ┌──────────────────┐                  ┌──────────────────┐
   │  Browser A       │◄═════P2P════════►│  Browser B       │
   │  (WebRTC)        │   audio/video    │  (WebRTC)        │
   └──────────────────┘   DataChannel    └──────────────────┘
            ▲                                    ▲
            │ texto/CRDT (Yjs/Automerge)        │
            │                                    │
            └──────────────┬─────────────────────┘
                           ▼
                   ┌────────────────┐
                   │  Yjs backend   │  (Y-Sweet, PartyKit)
                   │  + Presence    │
                   └────────────────┘
                           ▲
                           │  REST/GraphQL (metadata, auth)
                           │
                   ┌────────────────┐
                   │  App Server   │
                   └────────────────┘
```

En una app real:

- **REST/GraphQL** para datos duraderos y autenticación.
- **WebSocket/Socket.io** para notificaciones efímeras y presence.
- **SSE** cuando solo necesitas empuje del servidor.
- **WebRTC** para audio/video/P2P de baja latencia.
- **CRDT (Yjs/Automerge)** para el estado colaborativo.
- **Redis** como bus interno para sincronizar gateways.

---

## Ejercicios

### Ejercicio 1 — Elige la tecnología

Para cada caso, justifica la elección:

1. Chat 1-a-1 con indicadores "escribiendo...".
2. Notificación "tu pedido cambió de estado" en un dashboard.
3. Videollamada grupal de 4 personas.
4. Documento de texto editable por 10 personas a la vez.
5. Transferencia de un archivo de 2 GB entre dos usuarios.
6. "Cuántos usuarios hay en esta página ahora mismo".

### Ejercicio 2 — Socket.io con salas

Implementa un chat con:
- Salas por tema (`/sala/nodejs`, `/sala/react`).
- Mensaje persiste los últimos 50 mensajes por sala.
- Indicador "X está escribiendo" con expiración de 3s.
- Mensajes privados entre dos usuarios.

### Ejercicio 3 — SSE con autenticación

Implementa un endpoint SSE protegido con JWT. Considera:
- ¿Cómo pasa el token el cliente?
- ¿Qué pasa si el token expira mientras la conexión está abierta?
- ¿Cómo manejas la reconexión?

### Ejercicio 4 — WebRTC 1-a-1 con signaling por Socket.io

Implementa una llamada de video entre dos pestañas del mismo navegador. Dibuja el flujo completo de mensajes de signaling.

### Ejercicio 5 — Presence con Redis

Implementa presence con:
- Heartbeat cada 10s.
- TTL de 30s.
- Broadcast de cambios de estado a todos los gateways.
- Endpoint REST para "¿está este usuario online?".

### Ejercicio 6 — Editor colaborativo con Yjs

Construye un textarea compartido entre dos pestañas usando Yjs + y-websocket. Edita en una y observa en la otra. Mata el servidor, sigue editando, reconecta. ¿Qué pasa con el estado?

### Ejercicio 7 — Compara OT vs CRDT

Investiga brevemente:
1. ¿Por qué Google migró de OT a CRDT en su stack?
2. ¿Qué ventajas tiene CRDT para apps offline-first?
3. ¿Cuándo OT sigue siendo preferible?

---

## Problema propuesto

### Editor de documentos colaborativo tipo Notion

Construye un **editor colaborativo** donde múltiples usuarios pueden editar un documento al mismo tiempo, ver cursores ajenos, chatear y recibir notificaciones.

### Requisitos funcionales

1. **Edición concurrente**: dos usuarios editan el mismo párrafo simultáneamente. Ambos ven los cambios del otro en menos de 200ms.
2. **Cursores remotos**: cada usuario ve dónde están los cursores de los demás, con su nombre y color.
3. **Selección visible**: si un usuario selecciona un rango, los demás lo ven resaltado.
4. **Presence**: lista de usuarios activos en el documento con avatares.
5. **Chat lateral**: los usuarios pueden chatear dentro del documento sin salir de la página.
6. **Notificaciones**: si llevo más de 5 minutos inactivo, los demás ven un indicador "afk".
7. **Resiliencia**: si pierdo conexión, sigo editando. Al reconectar, los cambios se sincronizan sin perder nada.
8. **Persistencia**: el documento se guarda en el servidor y se puede reabrir en una sesión nueva.

### Requisitos técnicos

Tu solución debe:

1. Usar **WebSocket (o Socket.io)** para signaling, chat y presence.
2. Usar un **CRDT (Yjs o Automerge)** para el estado del documento y la posición de cursores.
3. Implementar **presence con expiración** (sin actualizar hace 30s = ausente).
4. Tener un **backend que persista** el documento (puede ser en memoria, archivo, SQLite o PostgreSQL).
5. Manejar **reconexión**: si el cliente se cae, al volver debe traer el estado actual y seguir editando.
6. Servir el documento por **REST/GraphQL** para la carga inicial (metadata, permisos).

### Entregables

- Diagrama de componentes y de protocolos.
- Estructura de carpetas (frontend / backend / signaling / CRDT).
- Implementación funcional de:
  - Editor con CRDT sincronizado entre 2+ pestañas/usuarios.
  - Presence con avatares y expiración.
  - Chat efímero.
  - Persistencia del documento.
- Documento de 1 página justificando:
  - Por qué elegiste Yjs o Automerge.
  - Cómo resolviste la persistencia (snapshot + updates).
  - Cómo manejarías la autenticación y permisos por documento.
  - Qué trade-offs aceptarías si el documento tuviera 1000 usuarios simultáneos.

### Criterios de evaluación

| Criterio                                       | Peso |
| ---------------------------------------------- | ---- |
| CRDT correctamente aplicado (sin "último gana")| 25%  |
| Presence funcional con expiración              | 15%  |
| Cursores remotos visibles                      | 15%  |
| Reconexión sin pérdida de cambios              | 15%  |
| Chat efímero en tiempo real                    | 10%  |
| Persistencia funcional                         | 10%  |
| Justificación arquitectónica                   | 10%  |
