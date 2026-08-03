# Arquitecturas de Software para Aplicaciones Fullstack

## Objetivo de aprendizaje

Comprender que **la arquitectura es la decisión que más cuesta cambiar** en un proyecto. Un framework se puede migrar; una arquitectura acoplada al framework es mucho más costosa de revertir.

Aquí se estudian seis estilos arquitectónicos frecuentes en aplicaciones fullstack modernas. **No son mutuamente excluyentes**: en la práctica se combinan.

---

## 1. Clean Architecture

### Idea central

Separar el software en **anillos concéntricos** donde las dependencias apuntan hacia adentro. Las reglas de negocio no saben nada del framework, la base de datos ni la web.

```
┌──────────────────────────────────────┐
│  Frameworks & Drivers (Web, DB, UI)  │   ← borde externo
├──────────────────────────────────────┤
│  Interface Adapters (Controllers,    │
│  Repositories, Presenters)           │
├──────────────────────────────────────┤
│  Use Cases (Application Services)    │
├──────────────────────────────────────┤
│  Entities (Reglas de negocio puras)  │   ← núcleo
└──────────────────────────────────────┘
        las dependencias apuntan hacia el centro
```

### Capas en detalle

- **Entities**: objetos de dominio puros. No importan nada externo. Son la verdad del negocio.
- **Use Cases**: orquestan entidades para cumplir un objetivo del usuario ("CrearPedido", "CancelarSuscripción").
- **Interface Adapters**: traducen entre el mundo externo (HTTP, SQL) y el mundo interno. Aquí viven los controllers, repositorios, presenters.
- **Frameworks & Drivers**: detalles. La web, la base de datos, el framework concreto.

### Regla de dependencia

> Una capa interna **nunca** debe importar nada de una capa externa. Si tu `Entity` importa `express` o `mongoose`, la arquitectura está rota.

### Ejemplo: Crear un pedido

```ts
// domain/entities/Pedido.ts  (núcleo)
export class Pedido {
  constructor(
    public readonly id: string,
    public readonly clienteId: string,
    public readonly items: Item[],
    public readonly total: number,
  ) {}

  static crear(clienteId: string, items: Item[]): Pedido {
    if (items.length === 0) throw new Error("Pedido sin items");
    const total = items.reduce((acc, i) => acc + i.precio * i.cantidad, 0);
    return new Pedido(crypto.randomUUID(), clienteId, items, total);
  }
}

// application/usecases/CrearPedido.ts
export class CrearPedido {
  constructor(private repo: PedidoRepository) {}
  async ejecutar(input: { clienteId: string; items: Item[] }) {
    const pedido = Pedido.crear(input.clienteId, input.items);
    await this.repo.guardar(pedido);
    return pedido;
  }
}

// infrastructure/web/pedidoController.ts  (borde)
app.post("/pedidos", async (req, res) => {
  const usecase = container.resolve(CrearPedido);
  const pedido = await usecase.ejecutar(req.body);
  res.json(pedido);
});
```

`Pedido` y `CrearPedido` **no saben** que existe Express ni una base de datos concreta.

### Cuándo usarla

- Aplicaciones de negocio con reglas complejas.
- Equipos grandes que necesitan que las decisiones de infraestructura sean intercambiables.
- Proyectos con vida útil larga (años).

### Cuándo **no** usarla

- CRUDs simples: la sobreingeniería mata la velocidad.
- Prototipos de pocos días.

---

## 2. Arquitectura Hexagonal (Ports & Adapters)

### Idea central

Aislar el **núcleo de la aplicación** del mundo exterior mediante **puertos** (interfaces) y **adaptadores** (implementaciones).

```
        ┌──────────────────────┐
        │                      │
  Adapter│   Application Core   │Adapter
   (Web) │                      │(DB)
        │   Puerto  →  Puerto  │
        └──────────────────────┘
              Adapter
            (Mensajería)
```

- **Puerto**: interfaz que define cómo el mundo exterior habla con la aplicación.
- **Adaptador**: implementación concreta de un puerto. Puede ser un controller HTTP, un consumer de Kafka, un repositorio SQL, un mock para tests.

### Ejemplo: el mismo caso de pedidos

```ts
// application/ports/PedidoRepository.ts
export interface PedidoRepository {
  guardar(pedido: Pedido): Promise<void>;
  obtenerPorId(id: string): Promise<Pedido | null>;
}

// infrastructure/adapters/PostgresPedidoRepository.ts
export class PostgresPedidoRepository implements PedidoRepository {
  async guardar(pedido: Pedido) { /* INSERT ... */ }
  async obtenerPorId(id: string) { /* SELECT ... */ }
}

// infrastructure/adapters/InMemoryPedidoRepository.ts  (tests)
export class InMemoryPedidoRepository implements PedidoRepository {
  private store = new Map<string, Pedido>();
  async guardar(p: Pedido) { this.store.set(p.id, p); }
  async obtenerPorId(id: string) { return this.store.get(id) ?? null; }
}
```

El `CrearPedido` recibe un `PedidoRepository` por el constructor. No le importa si es Postgres, Mongo o un Map en memoria. **Eso es la hexagonalidad.**

### Diferencia con Clean

Clean propone capas concéntricas; Hexagonal propone un núcleo rodeado de adaptadores. En la práctica se parecen mucho, pero:

- Hexagonal hace explícitas las **interfaces/puertos**.
- Clean se centra más en la **dirección de las dependencias**.

### Cuándo usarla

- Cuando necesitas **probar la lógica sin levantar la base de datos**.
- Cuando un mismo caso de uso puede ser disparado por HTTP, por una cola, por un cron, por CLI.
- Equipos que alternan infraestructura con frecuencia.

---

## 3. DDD Básico (Domain-Driven Design)

### Idea central

El software debe modelar el **lenguaje del negocio**. Los términos del experto de dominio aparecen en el código, y las decisiones técnicas se subordinan al modelo.

### Bloques tácticos

| Concepto             | Significado                                                            |
| -------------------- | ---------------------------------------------------------------------- |
| **Entidad**          | Objeto con identidad (id) y ciclo de vida.                             |
| **Value Object**     | Objeto sin identidad, definido por sus valores (Email, Dinero).        |
| **Aggregate**        | Cluster de entidades y VOs con una raíz y una invariante.              |
| **Repositorio**      | Colección de aggregates en memoria (interfaz, no implementación).      |
| **Servicio de Dominio** | Lógica que no cabe en una entidad ni en un VO.                      |
| **Bounded Context**  | Frontera donde un modelo es consistente. "Cliente" significa cosas distintas en Ventas y en Soporte. |

### Lenguaje ubicuo

> Si el cliente dice "un pedido tiene items y un estado", el código debe tener `Pedido`, `Item` y `EstadoPedido`. Si el equipo de negocio usa una palabra y el código usa otra, hay un problema.

### Ejemplo: aggregate Pedido con invariantes

```ts
type EstadoPedido = "borrador" | "confirmado" | "cancelado";

class Pedido {
  private items: Item[] = [];
  private estado: EstadoPedido = "borrador";

  agregarItem(productoId: string, precio: number, cantidad: number) {
    if (this.estado !== "borrador") {
      throw new Error("Solo se pueden agregar items en borrador");
    }
    if (cantidad <= 0) throw new Error("Cantidad inválida");
    this.items.push(new Item(productoId, precio, cantidad));
  }

  confirmar() {
    if (this.items.length === 0) throw new Error("Pedido vacío");
    this.estado = "confirmado";
  }

  total(): number {
    return this.items.reduce((acc, i) => acc + i.subtotal(), 0);
  }
}
```

Las reglas ("no se confirma vacío", "solo se modifica en borrador") **viven con el modelo**, no en un controller.

### Cuándo usarlo

- Dominios con reglas de negocio no triviales.
- Cuando hay un experto de dominio accesible y dispuesto a modelar contigo.
- Proyectos donde el lenguaje del negocio se repite en muchas pantallas y servicios.

### Cuándo **no** usarlo

- CRUDs simples: un ORM y un controller bastan.
- Equipos sin acceso a un experto de dominio.

---

## 4. CQRS (Command Query Responsibility Segregation)

### Idea central

**Separar las operaciones que escriben (commands) de las que leen (queries)**, modelando cada lado según sus necesidades.

```
        Commands                Queries
Cliente ───────►  Write Model ────► Read Model ───────► Cliente
                      │                 ▲
                      └───── eventos ───┘
```

- **Command**: intención de cambiar el estado ("CrearPedido", "CancelarPedido"). Devuelve confirmación o error, no datos.
- **Query**: petición de lectura ("ObtenerPedidosDelCliente"). Nunca modifica el estado.

### ¿Por qué separar?

- Los modelos de lectura suelen necesitar joins, agregaciones y desnormalización que ensucian el modelo de escritura.
- Escalar lectura y escritura de forma independiente.
- Permite usar **proyecciones** distintas para cada caso de uso (una vista para la UI, otra para un reporte, otra para exportar).

### Ejemplo

```ts
// command side
class CrearPedidoHandler {
  async handle(cmd: CrearPedidoCommand) {
    const pedido = Pedido.crear(cmd.clienteId, cmd.items);
    await this.repo.guardar(pedido);
    await this.eventBus.publish(new PedidoCreado(pedido));
  }
}

// query side
class ObtenerPedidosDelClienteHandler {
  async handle(q: ObtenerPedidosDelClienteQuery) {
    return this.readDb.query(
      "SELECT id, total, estado FROM pedidos WHERE cliente_id = $1",
      [q.clienteId],
    );
  }
}
```

La base de lectura puede ser SQL, una vista materializada, un Elasticsearch, una proyección en memoria. **El command no lo sabe.**

### Cuándo usarla

- Reportes, dashboards, vistas con muchos joins.
- Sistemas con alta carga de lectura vs. escritura.
- Combinada con **Event Sourcing**.

### Cuándo **no** usarla

- CRUDs pequeños. La duplicación de modelos no se justifica.
- Cuando el equipo no puede pagar la complejidad de mantener dos modelos sincronizados.

---

## 5. Event-Driven Architecture

### Idea central

Los componentes se comunican **emitiendo y escuchando eventos**, sin llamarse directamente. El emisor no sabe quién consume.

```
ServicioA --emit--> EventBus --fanout--> ServicioB
                                     └─> ServicioC
```

### Conceptos

- **Evento**: hecho del pasado ("PedidoCreado", "PagoRecibido"). Inmutable.
- **Producer**: quien emite.
- **Consumer**: quien reacciona.
- **Broker**: el bus (Kafka, RabbitMQ, NATS, Redis Streams).

### Ejemplo

```ts
// producer
await eventBus.publish("PedidoCreado", {
  pedidoId: pedido.id,
  clienteId: pedido.clienteId,
  total: pedido.total(),
});

// consumer: servicio de inventario
eventBus.subscribe("PedidoCreado", async (event) => {
  await inventario.reservar(event.pedidoId, event.items);
});

// consumer: servicio de notificaciones
eventBus.subscribe("PedidoCreado", async (event) => {
  await mail.enviar(event.clienteId, "Tu pedido fue recibido");
});
```

### Beneficios

- **Desacoplamiento**: añadir un nuevo consumidor no requiere tocar al productor.
- **Resiliencia**: si un consumidor cae, el evento queda en el broker y se procesa cuando vuelva.
- **Escalabilidad**: cada consumer escala de forma independiente.

### Riesgos

- **Eventual consistency**: los datos no son inmediatamente consistentes entre servicios.
- **Debugging distribuido**: seguir un flujo entre 5 servicios requiere trazabilidad.
- **Idempotencia**: los consumers deben poder recibir el mismo evento dos veces sin corromper datos.

### Cuándo usarla

- Microservicios.
- Procesos largos que no necesitan respuesta inmediata (emails, reportes, auditoría).
- Integración entre equipos que avanzan a velocidades distintas.

---

## 6. Vertical Slice Architecture

### Idea central

En lugar de organizar el código **por capas técnicas** (`/controllers`, `/services`, `/repositories`), se organiza **por feature o caso de uso**. Cada slice es vertical: atraviesa todas las capas que necesita.

```
src/
├── features/
│   ├── pedidos/
│   │   ├── crearPedido.ts          ← command + handler + validación + repo
│   │   ├── obtenerPedido.ts        ← query + handler
│   │   ├── cancelarPedido.ts
│   │   └── pedido.test.ts
│   ├── clientes/
│   │   ├── registrarCliente.ts
│   │   └── ...
│   └── inventario/
└── shared/                          ← solo lo verdaderamente transversal
    ├── db.ts
    └── eventBus.ts
```

### Características

- Cada feature **se contiene a sí misma**: su controller, su validación, su lógica, su acceso a datos.
- Lo "compartido" se reduce al mínimo.
- Es la antítesis de `controllers/services/repositories` global.

### Ventajas

- Cambios pequeños: tocar `crearPedido` no requiere navegar por 7 carpetas.
- Equipos paralelos: cada feature es propiedad de un equipo o persona.
- Onboarding: un nuevo dev entiende una feature leyendo **un solo directorio**.

### Cuándo usarla

- La opción por defecto en aplicaciones fullstack modernas.
- Equipos que entregan features completas (vertical) y no capas (horizontal).
- Combina muy bien con CQRS (cada command/query es un slice).

### Cuándo **no** usarla

- Bases de código muy pequeñas donde la organización por capas es trivial.
- Cuando hay muchísimo código verdaderamente transversal (poco común).

---

## Cómo se combinan en la práctica

| Combinación                              | Cuándo encaja                                         |
| ---------------------------------------- | ----------------------------------------------------- |
| Clean + Hexagonal                        | Núcleo puro + puertos explícitos.                     |
| DDD + Hexagonal                          | Modelar el dominio y aislarlo del exterior.           |
| DDD + CQRS + Event-Driven                | Sistemas con muchos eventos de negocio.               |
| Vertical Slice + CQRS                    | El default moderno: cada feature es un slice.         |
| Vertical Slice + DDD ligero              | Cuando el dominio no es enorme pero quieres orden.    |

---

## Ejercicios

### Ejercicio 1 — Identificar la arquitectura

Para cada proyecto, identifica qué estilo o combinación usarías y por qué:

1. Un e-commerce con catálogo, carrito, pagos y envíos.
2. Una API pública de clima consumida por apps móviles.
3. Un sistema de hospital con historiales clínicos y reglas regulatorias estrictas.
4. Un dashboard interno que muestra métricas en tiempo real desde 5 fuentes distintas.
5. Un MVP de una startup para validar idea en 3 semanas.

### Ejercicio 2 — Mapea tu último proyecto

Tomando el último proyecto que hayas hecho:

1. ¿Estaba organizado por capas técnicas o por features?
2. ¿Las reglas de negocio vivían en entities o dispersas en controllers?
3. ¿Podrías cambiar la base de datos tocando un solo archivo?
4. ¿Podrías añadir una nueva fuente de eventos (HTTP → cola) sin reescribir la lógica?

Donde la respuesta sea "no", identifica qué principio arquitectónico te habría ayudado.

### Ejercicio 3 — Dibuja los anillos

Dibuja los anillos de Clean Architecture para una aplicación de **gestión de citas médicas** con las siguientes capas y ubícalas:

- `MedicoController`
- `Medico` (entidad)
- `AgendarCita` (use case)
- `MedicoRepository` (interfaz)
- `PostgresMedicoRepository` (implementación)
- `Appointment` (entidad)

Justifica por qué cada elemento va donde va.

### Ejercicio 4 — Hexagonal con tests rápidos

Implementa un caso `CrearProducto` con:
- Un puerto `ProductoRepository`.
- Un adaptador en memoria.
- Un test del caso de uso **sin** base de datos real.

Luego implementa un segundo adaptador con `sqlite` o `pg` y comprueba que el caso de uso **no cambia**.

### Ejercicio 5 — Diseña los eventos

Modela los eventos del dominio para un sistema de **préstamos**:

- ¿Qué hechos del pasado son relevantes?
- ¿Qué servicio publica cada uno?
- ¿Qué servicios consumen cada uno?
- ¿Dónde puede fallar la consistencia eventual?

### Ejercicio 6 — Refactor a Vertical Slice

Toma un proyecto CRUD organizado en `controllers/services/models`. Reorganízalo a vertical slice. Una feature a la vez. Mide cuánto tardas y qué partes del código se rompieron.

---

## Problema propuesto

### Sistema de reservas de un restaurante

Debes construir el backend de un sistema de reservas para una cadena de restaurantes. El sistema debe:

1. Permitir a un cliente **crear, modificar y cancelar** una reserva.
2. Asignar **mesas** según capacidad, horario y disponibilidad.
3. Permitir al personal del restaurante **confirmar o rechazar** reservas manualmente.
4. Generar un **reporte diario** de ocupación por sucursal.
5. Mandar una **notificación** al cliente 2 horas antes de su reserva.
6. Registrar **eventos de dominio** (ReservaCreada, ReservaConfirmada, ReservaCancelada, ClienteNoAsistio) para auditoría.

### Requisitos arquitectónicos

Tu solución debe:

1. Estar organizada en **Vertical Slices** (una carpeta por feature).
2. Separar **Commands de Queries** (CQRS).
3. Tener un **modelo de dominio** con al menos un aggregate (`Reserva`) y reglas de negocio que **vivan en la entidad**, no en el controller.
4. Publicar **eventos de dominio** que disparen al menos dos efectos secundarios (notificación y reporte).
5. Definir **interfaces (puertos)** para el repositorio de reservas y para el bus de eventos. Tener al menos una implementación en memoria y otra con persistencia real.
6. Tener **tests del modelo de dominio** que no necesiten base de datos, ni HTTP, ni framework.

### Entregables

- Diagrama de capas/anillos de tu solución.
- Listado de bounded contexts identificados.
- Listado de eventos de dominio.
- Estructura de carpetas (vertical slice).
- Implementación funcional del slice `CrearReserva` con tests.
- Un documento breve (1 página) explicando qué decisiones tomaste y qué trade-offs aceptaste.

### Criterios de evaluación

| Criterio                                              | Peso |
| ----------------------------------------------------- | ---- |
| Reglas de negocio en el dominio (no en controllers)   | 25%  |
| Separación Commands / Queries                         | 15%  |
| Vertical Slice bien aplicado                          | 15%  |
| Puertos + adaptadores (test sin infra real)           | 20%  |
| Eventos de dominio desacoplados                       | 15%  |
| Claridad de las decisiones en el documento            | 10%  |
