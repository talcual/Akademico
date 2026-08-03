# Comunicación entre Servicios

## Objetivo de aprendizaje

Una aplicación fullstack no vive sola. En la práctica combina **múltiples formas de comunicación** según el problema: HTTP síncrono para operaciones de usuario, eventos asíncronos para procesos de fondo, streaming para datos en vivo.

Aquí se estudian los **estilos de comunicación** más usados en sistemas modernos, sus trade-offs y cuándo combinarlos.

---

## Mapa general

| Estilo                  | Dirección        | Acoplamiento | Caso típico                          |
| ----------------------- | ---------------- | ------------ | ------------------------------------ |
| REST                    | Cliente ↔ Servidor | Medio        | API pública, CRUD, recursos          |
| GraphQL                 | Cliente ↔ Servidor | Medio        | Múltiples clientes con vistas distintas |
| gRPC                    | Servicio ↔ Servicio | Alto (schema) | Microservicios internos              |
| WebSockets              | Bidireccional    | Alto         | Chat, colaboración en tiempo real    |
| Server-Sent Events      | Servidor → Cliente | Bajo         | Notificaciones, feeds, dashboards    |
| Message Queue           | Asíncrono        | Muy bajo     | Procesos de fondo, integración       |

---

## 1. REST

### Idea central

Recursos identificados por URL, manipulados con verbos HTTP. Sin estado en el servidor (cada请求 lleva su contexto).

### Verbos y semántica

| Verbo  | Uso              | Idempotente |
| ------ | ---------------- | ----------- |
| GET    | Leer             | Sí          |
| POST   | Crear             | No          |
| PUT    | Reemplazar       | Sí          |
| PATCH  | Modificar parcial | No          |
| DELETE | Eliminar         | Sí          |

### Ejemplo

```http
GET /api/pedidos/123
Accept: application/json
Authorization: Bearer eyJhbGc...

200 OK
Content-Type: application/json

{
  "id": "123",
  "clienteId": "c-9",
  "total": 45990,
  "estado": "confirmado"
}
```

### Buenas prácticas

- Recursos con nombres, no verbos (`/pedidos`, no `/crearPedido`).
- Status codes correctos (`201` en creación, `404` si no existe, `409` en conflicto).
- Paginación (`?page=2&limit=20` o cursor-based).
- Versionado (`/api/v1/pedidos`).
- HATEOAS opcional, pero útil en APIs grandes.

### Cuándo usarlo

- API pública.
- CRUD clásico.
- Cuando los clientes son desconocidos o muy heterogéneos.

### Limitaciones

- **Over-fetching**: traes campos que no usas.
- **Under-fetching**: necesitas varios endpoints para armar una vista.
- **Contrato rígido**: cambiar la respuesta puede romper clientes.

---

## 2. GraphQL

### Idea central

El cliente declara **exactamente qué campos quiere** y el servidor se los devuelve en una sola petición. Un solo endpoint, un solo verbo (`POST`), muchas formas de consultar.

### Ejemplo: schema

```graphql
type Pedido {
  id: ID!
  cliente: Cliente!
  items: [Item!]!
  total: Float!
  estado: EstadoPedido!
}

type Query {
  pedido(id: ID!): Pedido
  pedidosDelCliente(clienteId: ID!): [Pedido!]!
}

type Mutation {
  crearPedido(input: CrearPedidoInput!): Pedido!
  cancelarPedido(id: ID!): Pedido!
}
```

### Ejemplo: query del cliente

```graphql
query {
  pedido(id: "123") {
    total
    estado
    cliente {
      nombre
    }
  }
}
```

El cliente pidió 3 campos, el servidor devolvió **solo esos 3**.

### Ventajas

- Una sola llamada para vistas complejas.
- El cliente decide qué necesita (menos over/under-fetching).
- Schema tipado, autodescubrible.
- Excelente para múltiples clientes (web, mobile, partners) con vistas distintas.

### Riesgos

- **N+1 oculto**: una query puede disparar muchos joins. Requiere `dataloaders` o `DataLoader`-style batching.
- **Caching complejo**: GET se vuelve difícil. Suele usarse caché a nivel de query.
- **Cost queries**: queries pesadas pueden tumbar el servidor. Hay que limitar profundidad y complejidad.
- **Menos maduro** en tooling y mental models que REST.

### Cuándo usarlo

- Múltiples clientes consumiendo el mismo backend con vistas distintas.
- Modelos de datos muy conectados (redes sociales, CMS).
- Equipos frontend que quieren autonomía para pedir nuevos campos sin esperar al backend.

### Cuándo **no** usarlo

- Una sola app, un solo cliente.
- APIs públicas donde el contrato debe ser estable y cacheable fácilmente.
- Equipos sin experiencia: el costo cognitivo inicial es alto.

---

## 3. gRPC

### Idea central

Framework de **llamadas a procedimiento remoto** sobre HTTP/2, con contratos definidos en **Protocol Buffers** (binario, tipado, estricto).

### Contrato `.proto`

```proto
syntax = "proto3";

package pedidos;

service PedidoService {
  rpc ObtenerPedido(PedidoRequest) returns (PedidoResponse);
  rpc ListarPedidos(ListarRequest) returns (stream PedidoResponse);
  rpc CrearPedido(CrearRequest) returns (PedidoResponse);
}

message PedidoRequest { string id = 1; }
message PedidoResponse {
  string id = 1;
  string cliente_id = 2;
  double total = 3;
  string estado = 4;
}
```

### Características clave

- **HTTP/2**: multiplexación, headers comprimidos, server push descontinuado.
- **Binario**: más compacto y rápido que JSON.
- **Tipado fuerte**: el contrato es ley, no se puede "improvisar".
- **Streaming nativo**: unary, server-streaming, client-streaming, bidireccional.
- **Generación de código**: stub en 10+ lenguajes.

### Cuándo usarlo

- **Comunicación interna entre microservicios**: bajo acoplamiento contractual, alta performance.
- Redes con restricciones de ancho de banda.
- Sistemas políglotas (Java ↔ Go ↔ Rust ↔ TS).

### Cuándo **no** usarlo

- APIs públicas consumidas por navegadores: gRPC-Web existe pero es engorroso.
- Equipos pequeños sin experiencia con Protobuf.

### REST vs gRPC

| Criterio              | REST            | gRPC                |
| --------------------- | --------------- | ------------------- |
| Formato               | JSON (texto)    | Protobuf (binario)  |
| Contrato              | OpenAPI (opcional) | `.proto` (obligatorio) |
| Performance           | Moderada        | Alta                |
| Streaming             | No nativo       | Sí                  |
| Navegador             | Trivial         | Requiere gRPC-Web   |
| Debugging             | Fácil (curl)    | Requiere herramientas |

---

## 4. WebSockets

### Idea central

Conexión **bidireccional persistente** sobre TCP (upgrade desde HTTP). Ambas partes pueden enviar mensajes en cualquier momento.

### Handshake

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

### Ejemplo: cliente

```ts
const ws = new WebSocket("wss://api.example.com/chat");

ws.onopen = () => ws.send(JSON.stringify({ tipo: "join", sala: "general" }));
ws.onmessage = (event) => console.log("Recibido:", JSON.parse(event.data));
ws.onclose = () => console.log("Desconectado");
```

### Ejemplo: servidor (Node)

```ts
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

### Cuándo usarlo

- Chat, mensajería.
- Colaboración en tiempo real (documentos, pizarras, editores).
- Juegos multijugador.
- Dashboards con interacción bidireccional.

### Limitaciones

- **Estado**: el servidor debe recordar conexiones. Escala horizontal requiere sticky sessions o un broker.
- **Reconexión**: el cliente debe manejar caídas y reintentos.
- **Proxies y firewalls**: algunos siguen dando problemas con `Upgrade`.

---

## 5. Server-Sent Events (SSE)

### Idea central

Conexión **unidireccional** del servidor al cliente sobre HTTP. El servidor "empuja" eventos al cliente cuando ocurren.

### Ejemplo: servidor

```ts
app.get("/stream/pedidos", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const interval = setInterval(() => {
    res.write(`data: ${JSON.stringify({ ts: Date.now() })}\n\n`);
  }, 1000);

  req.on("close", () => clearInterval(interval));
});
```

### Ejemplo: cliente

```ts
const sse = new EventSource("/stream/pedidos");
sse.onmessage = (event) => console.log("Nuevo:", event.data);
sse.onerror = () => console.log("Conexión perdida, reintentando...");
```

### Comparación con WebSockets

| Criterio              | SSE                       | WebSockets              |
| --------------------- | ------------------------- | ----------------------- |
| Dirección             | Servidor → Cliente        | Bidireccional           |
| Protocolo             | HTTP                      | WebSocket (sobre HTTP)  |
| Reconexión            | Automática del navegador | Manual                  |
| Texto vs binario      | Solo texto                | Texto y binario         |
| Complejidad servidor  | Baja                      | Media                   |
| Browsers              | Todos los modernos        | Todos los modernos      |

### Cuándo usar SSE

- Notificaciones, feeds, actualizaciones unidireccionales.
- Progreso de tareas largas.
- Logs en vivo.
- Cuando **no** necesitas mensajes del cliente al servidor.

---

## 6. Message Queues

### Idea central

Servicios se comunican **publicando y consumiendo mensajes** a través de un broker. **Desacoplamiento temporal**: el emisor no espera al receptor.

### Patrones

- **Point-to-point**: una cola, un consumidor procesa cada mensaje.
- **Pub/Sub**: un tema, varios consumidores reciben cada mensaje.
- **Streaming/Log**: el mensaje persiste y se puede releer.

### Comparación de brokers

| Broker   | Modelo        | Fortalezas                              | Casos típicos                        |
| -------- | ------------- | --------------------------------------- | ------------------------------------ |
| RabbitMQ | Cola + Pub/Sub | Maduro, enrutamiento rico,acknowledgements | Tareas asíncronas, workers           |
| NATS     | Pub/Sub ligero | Muy rápido, simple, ideal para cloud    | Microservicios, IoT, baja latencia   |
| Kafka    | Log distribuido | Alto throughput, retención, replay     | Eventos de negocio, analytics, auditoría |

---

### 6.1 RabbitMQ

#### Idea central

Broker AMQP con **colas, intercambiadores (exchanges) y bindings**. Garantiza entrega y soporte rico para enrutamiento.

#### Conceptos

- **Producer**: publica mensajes a un exchange.
- **Exchange**: rutea a colas según tipo (`direct`, `topic`, `fanout`, `headers`).
- **Queue**: almacena mensajes hasta que un consumer los procese.
- **Consumer**: se suscribe a una cola.

#### Ejemplo: publicar

```ts
import amqp from "amqplib";

const conn = await amqp.connect("amqp://localhost");
const ch = await conn.createChannel();
await ch.assertExchange("pedidos", "topic", { durable: true });

ch.publish(
  "pedidos",
  "pedido.creado",
  Buffer.from(JSON.stringify({ id: "p-1", total: 1000 })),
  { persistent: true },
);
```

#### Ejemplo: consumir

```ts
await ch.assertQueue("notificaciones", { durable: true });
await ch.bindQueue("notificaciones", "pedidos", "pedido.*");

ch.consume("notificaciones", (msg) => {
  if (!msg) return;
  console.log("Notificación:", JSON.parse(msg.content.toString()));
  ch.ack(msg);
});
```

#### Cuándo usarlo

- Tareas de fondo (enviar email, generar PDF, procesar imágenes).
- Workers que procesan jobs uno a uno.
- Enrutamiento complejo (topic exchanges).

---

### 6.2 NATS

#### Idea central

Sistema de mensajería **ultraligero y de baja latencia** basado en pub/sub y, opcionalmente, persistencia con JetStream.

#### Ejemplo: publicar

```ts
import { connect, StringCodec } from "nats";
const nc = await connect({ servers: "nats://localhost:4222" });
const sc = StringCodec();
nc.publish("pedidos.creado", sc.encode(JSON.stringify({ id: "p-1" })));
await nc.drain();
```

#### Ejemplo: suscribirse

```ts
const sub = nc.subscribe("pedidos.creado");
for await (const msg of sub) {
  console.log("Recibido:", sc.decode(msg.data));
}
```

#### Características

- **At-most-once** por defecto (fire and forget).
- Con **JetStream**: persistencia, exactly-once, replay, ack.
- Subjects jerárquicos: `pedidos.creado`, `pedidos.cancelado`.
- Descubrimiento de servicios integrado.

#### Cuándo usarlo

- Microservicios con muchos mensajes pequeños.
- IoT y edge computing.
- Cuando la **simplicidad operativa** es prioridad.
- Service mesh ligero.

---

### 6.3 Kafka (introductorio)

#### Idea central

**Log distribuido, persistente y replicado**. Los productores escriben a un topic particionado; los consumidores leen desde un offset y pueden releer.

#### Conceptos

- **Topic**: flujo de eventos con nombre.
- **Partición**: unidad de paralelismo. Orden garantizado **dentro** de una partición.
- **Offset**: posición de lectura de un consumer.
- **Consumer Group**: varios consumers cooperando para procesar un topic en paralelo.
- **Retention**: los mensajes no se borran al ser consumidos; se retienen por tiempo o tamaño.

#### Ejemplo: productor

```ts
import { Kafka } from "kafkajs";
const kafka = new Kafka({ brokers: ["localhost:9092"] });
const producer = kafka.producer();
await producer.connect();

await producer.send({
  topic: "pedidos",
  messages: [
    { key: "p-1", value: JSON.stringify({ id: "p-1", total: 1000 }) },
  ],
});
```

#### Ejemplo: consumidor

```ts
const consumer = kafka.consumer({ groupId: "notificaciones" });
await consumer.connect();
await consumer.subscribe({ topic: "pedidos", fromBeginning: false });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    console.log({
      topic,
      partition,
      value: message.value?.toString(),
    });
  },
});
```

#### Cuándo usarlo (introductorio)

- **Event Sourcing**: el log es la fuente de verdad.
- **Alto throughput**: millones de eventos por segundo.
- **Replay**: necesitas reprocesar eventos pasados (regenerar proyecciones, cargar un data warehouse).
- **Analítica en tiempo real**: el log alimenta a varios consumers (notificaciones, métricas, ML, auditoría) sin que se entorpezcan.

#### Cuándo **no** usarlo (todavía)

- Mensajería ocasional con pocas decenas de mensajes por segundo: RabbitMQ o NATS bastan.
- Sistemas pequeños: la operativa de Kafka (ZooKeeper/KRaft, brokers, particiones) es pesada.

#### Diferencia clave con colas tradicionales

| Aspecto                | Cola tradicional (RabbitMQ) | Log (Kafka)               |
| ---------------------- | --------------------------- | ------------------------- |
| Mensaje tras consumir  | Se elimina                  | Permanece, se mueve offset |
| Replay                 | No                          | Sí, desde cualquier offset |
| Throughput típico      | Miles/seg                   | Millones/seg              |
| Orden                  | Por cola                    | Por partición             |
| Retención              | Hasta que se consuma        | Por tiempo/tamaño         |

---

## Cómo se combinan en la práctica

Una aplicación real **rara vez usa un solo estilo**:

```
[Browser]
   │ REST/GraphQL (operaciones del usuario)
   │ WebSocket / SSE (tiempo real)
   ▼
[API Gateway] ─── REST/gRPC ───► [Servicios]
                                       │
                                       │ publica eventos
                                       ▼
                                  [Kafka / NATS / RabbitMQ]
                                       │
                       ┌───────────────┼───────────────┐
                       ▼               ▼               ▼
                  [Notificaciones] [Reportes]    [Auditoría]
```

- REST/GraphQL para el **tráfico de usuario**.
- gRPC para **servicio a servicio interno** (contrato estricto, performance).
- WebSocket/SSE para **tiempo real hacia el cliente**.
- Colas/logs para **trabajo asíncrono y desacoplado**.

---

## Ejercicios

### Ejercicio 1 — Elige el estilo

Para cada caso, justifica el estilo de comunicación y por qué descartas los demás:

1. App móvil que muestra el menú de un restaurante.
2. Chat de soporte entre cliente y agente.
3. 50 microservicios que se llaman entre sí dentro de un cluster privado.
4. Dashboard de métricas que se actualiza cada 2 segundos.
5. Procesar 1 millón de eventos de telemetría por hora.
6. Generar un PDF de factura cuando se confirma un pedido.
7. Notificación push cuando un pedido cambia de estado.

### Ejercicio 2 — Compara REST y GraphQL

Modela el siguiente recurso en ambos estilos:

> Una página de detalle de pedido necesita: nombre del cliente, total, estado, últimos 3 productos, dirección de envío, método de pago.

1. ¿Cuántas llamadas harías en REST?
2. ¿Cuántas en GraphQL?
3. ¿Qué pasa si la pantalla añade un nuevo campo (historial de cambios)?

### Ejercicio 3 — Diseña un contrato gRPC

Define el `.proto` para un servicio de **inventario** con estas operaciones:

- `ConsultarStock(productoId) → stock`
- `ReservarStock(productoId, cantidad) → ok/error`
- `StreamMovimientos(productoId) → stream<Movimiento>` (server streaming)

### Ejercicio 4 — WebSocket vs SSE

Implementa el mismo caso (un feed de "últimos pedidos") con WebSocket y con SSE. Compara:

1. Líneas de código en cliente y servidor.
2. Manejo de reconexión.
3. Capacidad de enviar mensajes del cliente al servidor.
4. Soporte en proxies corporativos.

### Ejercicio 5 — Elige el broker

Tres equipos discuten qué broker usar. Tu trabajo es decidir:

- Equipo A procesa 10 mensajes/segundo de emails transaccionales.
- Equipo B procesa 500k eventos/segundo de logs de aplicación y necesita reprocesarlos.
- Equipo C tiene 30 microservicios pequeños que se pasan mensajes de estado.

Asigna broker a cada equipo y justifica.

### Ejercicio 6 — Dibuja la arquitectura de comunicación

Modela un sistema de **e-commerce** con:

- Frontend web.
- API pública.
- Servicio de pedidos.
- Servicio de inventario.
- Servicio de notificaciones.
- Reportes diarios.

Dibuja qué estilo de comunicación usarías entre cada par de componentes y por qué.

---

## Problema propuesto

### Plataforma de monitoreo de dispositivos IoT

Una empresa tiene **10,000 dispositivos IoT** que reportan telemetría (temperatura, batería, estado) cada 5 segundos. Debes construir la plataforma que recibe, procesa y distribuye estos datos.

### Requisitos funcionales

1. **Ingesta**: recibir mensajes de los dispositivos (MQTT o HTTP).
2. **Procesamiento en tiempo real**: detectar anomalías (temperatura > 80°C, batería < 10%).
3. **Persistencia**: guardar el historial para reportes.
4. **API de consulta**: una app web interna debe poder:
   - Ver el estado actual de un dispositivo.
   - Ver el histórico de las últimas 24h.
   - Listar dispositivos con alertas activas.
5. **Notificaciones**: cuando se detecta una anomalía, notificar al operador (email + WebSocket al dashboard).
6. **Alto throughput**: el sistema debe tolerar picos de 10,000 mensajes/segundo.

### Requisitos arquitectónicos

Tu solución debe:

1. Usar **gRPC** entre los microservicios internos.
2. Usar **REST o GraphQL** para la API que consume la app web.
3. Usar **WebSocket o SSE** para alertas en tiempo real al dashboard.
4. Usar un **message broker** (RabbitMQ, NATS o Kafka) para desacoplar la ingesta del procesamiento. Justifica cuál elegiste.
5. Tener al menos **3 servicios** con responsabilidades claras:
   - Ingesta.
   - Procesamiento de anomalías.
   - API de consulta + dashboard.
6. Garantizar que un dispositivo que reporta 2 veces la misma anomalía **no** genere 2 notificaciones (idempotencia o deduplicación).

### Entregables

- Diagrama de componentes y de comunicación.
- Contratos: `.proto`, schema GraphQL/REST, formato de eventos.
- Listado de topics/queues/exchanges y qué publica/consume cada servicio.
- Implementación funcional de al menos:
  - 1 servicio con gRPC.
  - 1 endpoint REST/GraphQL.
  - 1 flujo WebSocket o SSE.
  - 1 publicador y 1 consumidor del broker.
- Documento de 1 página justificando:
  - Por qué elegiste ese broker.
  - Por qué elegiste REST o GraphQL.
  - Cómo manejarías la idempotencia de notificaciones.
  - Qué harías distinto si el volumen fuera 100x mayor.

### Criterios de evaluación

| Criterio                                            | Peso |
| --------------------------------------------------- | ---- |
| Estilos de comunicación correctamente elegidos      | 20%  |
| Contratos claros (proto, schema, eventos)           | 20%  |
| Desacoplamiento vía broker                          | 15%  |
| Tiempo real funcional (WS o SSE)                    | 15%  |
| Idempotencia y manejo de fallos                     | 15%  |
| Justificación arquitectónica en el documento        | 15%  |
