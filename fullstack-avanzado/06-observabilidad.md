# Observabilidad

## Objetivo de aprendizaje

Una aplicación sin observabilidad es una **caja negra**: cuando algo falla, nadie sabe por qué. Aquí se estudian los **tres pilares** (logs, métricas, trazas) más OpenTelemetry como estándar unificado, y Grafana + Prometheus como stack de visualización y recolección.

La meta es que el estudiante entienda que **observar no es loguear todo**: es diseñar el sistema para que, cuando algo se rompe, las respuestas a "¿qué pasó?", "¿dónde?" y "¿por qué?" estén a un click.

---

## Mapa general

| Pilar        | Pregunta que responde              | Formato típico                | Herramientas                    |
| ------------ | ---------------------------------- | ----------------------------- | ------------------------------- |
| Logs         | ¿Qué pasó en este evento?          | Líneas estructuradas (JSON)   | Loki, ELK, Datadog              |
| Metrics      | ¿Cuánto / qué tan rápido?          | Números con etiquetas         | Prometheus, Datadog, CloudWatch |
| Traces       | ¿Por dónde pasó la request?        | Árbol de spans con timing     | Jaeger, Tempo, Honeycomb        |
| OpenTelemetry| Estándar para emitir los tres      | SDK + colectores              | OTel SDK + Collector            |
| Grafana      | Visualizar métricas, logs, traces  | Dashboards                    | Grafana                         |
| Prometheus   | Recolectar y almacenar métricas    | Series temporales             | Prometheus                      |

---

## 1. Logs

### Idea central

Un **log** es un registro inmutable de un evento que ocurrió en un momento dado. Es la forma más antigua y más ubicua de observabilidad.

### Logs estructurados

El log moderno es **JSON**, no texto libre. Esto permite buscar, filtrar y agregar sin parsear.

```json
{
  "ts": "2026-03-15T10:23:11.482Z",
  "level": "error",
  "service": "pedidos-api",
  "traceId": "8a3b1c2d...",
  "spanId": "f1e0d9c8...",
  "userId": "u-42",
  "requestId": "req-abc",
  "msg": "Pago rechazado",
  "pedidoId": "p-1001",
  "monto": 45990,
  "razon": "tarjeta_vencida"
}
```

### Niveles estándar

| Nivel    | Uso                                          |
| -------- | -------------------------------------------- |
| `debug`  | Información detallada, útil en desarrollo    |
| `info`   | Eventos normales del flujo                   |
| `warn`   | Algo inesperado pero no falló                |
| `error`  | Algo falló y afectó al usuario               |
| `fatal`  | El sistema no puede continuar                |

### Buenas prácticas

- **Incluir contexto**: `userId`, `requestId`, `traceId`. Sin contexto, el log es inútil.
- **No loguear secretos**: passwords, tokens, tarjetas, PII sin enmascarar.
- **No loguear en bucles**: millones de líneas por segundo ahogan el sistema.
- **Mensajes claros**: `"Pago rechazado"` no `"Error 500"`.
- **Timestamps en UTC y con milisegundos**.

### Ejemplo en Node

```ts
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: { service: "pedidos-api" },
});

logger.info({ pedidoId: "p-1", userId: "u-42" }, "pedido creado");
logger.error({ err, pedidoId: "p-1" }, "falló el cobro");
```

### Cuándo los logs no alcanzan

Cuando necesitas responder:
- "¿Cuántas requests por segundo estoy sirviendo?" → métricas.
- "¿Por qué esta request tardó 8 segundos?" → trazas.
- "¿Cuál es la distribución de latencia del endpoint X?" → métricas.

Los logs **complementan** a métricas y trazas, no las sustituyen.

---

## 2. Metrics (Métricas)

### Idea central

**Números agregados a lo largo del tiempo**. Son baratos de almacenar, rápidos de consultar y perfectos para alertas y dashboards.

### Tipos de métricas

| Tipo      | Ejemplo                          | Uso                              |
| --------- | -------------------------------- | -------------------------------- |
| Counter   | `http_requests_total`            | Solo sube. Conteos.              |
| Gauge     | `conexiones_activas`             | Sube y baja. Estado actual.      |
| Histogram | `request_duration_seconds`       | Distribución. Percentiles.       |
| Summary   | Similar a histogram, pre-agregado | Percentiles en el cliente       |

### Etiquetas (labels)

Las métricas se **segmentan** con etiquetas. Cuidado: alta cardinalidad (ej. `userId` como label) **destruye** el sistema de métricas.

```promql
# bien
http_requests_total{service="pedidos-api", method="POST", status="200"} 1240

# mal (cardinalidad explosiva)
http_requests_total{userId="u-1234567"} 1
```

### Métricas RED (lo mínimo por servicio)

Para cada servicio, expone:

- **R**ate: requests por segundo.
- **E**rrors: requests fallidas por segundo.
- **D**uration: latencia (percentiles p50, p95, p99).

Métricas **USE** (para recursos):

- **U**tilization: % de uso de CPU, memoria, disco.
- **S**aturation: cola/longitud de espera.
- **E**rrors: errores de hardware o del sistema.

### Ejemplo en Node (Prometheus)

```ts
import client from "prom-client";

const register = new client.Registry();
client.collectDefaultMetrics({ register });

const httpRequests = new client.Counter({
  name: "http_requests_total",
  help: "Total de requests HTTP",
  labelNames: ["method", "route", "status"],
  registers: [register],
});

const httpDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "Duración de requests HTTP",
  labelNames: ["method", "route", "status"],
  buckets: [0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register],
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on("finish", () => {
    httpRequests.inc({ method: req.method, route: req.route?.path, status: res.statusCode });
    end({ method: req.method, route: req.route?.path, status: res.statusCode });
  });
  next();
});

app.get("/metrics", async (_req, res) => {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
});
```

### Cuándo usarlas

- Dashboards operacionales.
- Alertas (si `error_rate > 1%` durante 5 min → avisar).
- Capacity planning.
- Comparar versiones en producción (canary / A/B).

---

## 3. Tracing (Trazas distribuidas)

### Idea central

Una **traza** registra el camino completo de una request a través de **todos los servicios** que toca. Cada paso es un **span** con su duración y metadatos.

### Terminología

- **Trace**: una request de extremo a extremo. Tiene un `traceId`.
- **Span**: una unidad de trabajo dentro de la trace. Tiene un `spanId` y un `parentSpanId` (excepto el root span).
- **Context**: `traceId` + `spanId` + baggage, propagado entre servicios.

```
Trace: 8a3b1c2d
├── Span: gateway (12ms)
│   ├── Span: auth (3ms)
│   └── Span: pedidos-api (9ms)
│       ├── Span: db.query SELECT (4ms)
│       ├── Span: kafka.publish (2ms)
│       └── Span: inventario.check (3ms)
```

### Propagación de contexto

Entre servicios, el contexto viaja en **headers** estándar:

```
traceparent: 00-8a3b1c2d...-f1e0d9c8...-01
tracestate: vendor=value
```

### Ejemplo en Node (OpenTelemetry)

```ts
import { trace, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("pedidos-api");

async function crearPedido(input: CrearPedidoInput) {
  return tracer.startActiveSpan("crearPedido", async (span) => {
    span.setAttribute("cliente.id", input.clienteId);
    span.setAttribute("items.count", input.items.length);

    try {
      const pedido = await this.repo.guardar(input);
      span.setAttribute("pedido.id", pedido.id);
      span.setStatus({ code: SpanStatusCode.OK });
      return pedido;
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

### Cuándo usarlas

- "Esta request tardó 8s, ¿dónde se fue el tiempo?"
- "¿Qué servicios se ven afectados cuando X se cae?"
- Depuración de sistemas distribuidos.
- Análisis de cuellos de botella.

### Cuándo **no** abusar

- No traces para **todo**: instrumentar manualmente las operaciones de negocio, no cada función.
- Sampling: en alto volumen, captura solo el 1-10% de las traces. 100% es insostenible.

---

## 4. OpenTelemetry (OTel)

### Idea central

**Estándar vendor-neutral** para emitir logs, métricas y trazas. Una vez instrumentado con OTel, **cambias de backend** (Jaeger, Tempo, Datadog, Honeycomb) sin tocar el código.

### Componentes

```
┌─────────────────────┐
│  Aplicación         │
│  (SDK OTel)         │  → emite OTLP
└─────────────────────┘
            │
            ▼
┌─────────────────────┐
│  OpenTelemetry      │  → enruta, filtra, batch
│  Collector          │  → exporta a backends
└─────────────────────┘
            │
   ┌────────┼────────┐
   ▼        ▼        ▼
 Jaeger   Tempo   Datadog
```

- **API**: interfaces (`tracer`, `meter`, `logger`).
- **SDK**: implementación (cómo se muestrean, exportan, procesan los datos).
- **Collector**: proceso separado que recibe OTLP y exporta a múltiples backends.
- **Auto-instrumentation**: instrumenta librerías comunes (HTTP, DB, frameworks) sin tocar código.

### Ejemplo: SDK configurado

```ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { Resource } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: "pedidos-api",
    [SemanticResourceAttributes.SERVICE_VERSION]: "1.4.2",
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: "production",
  }),
  traceExporter: new OTLPTraceExporter({ url: "http://otel-collector:4318/v1/traces" }),
});

sdk.start();
```

### Auto-instrumentation

```ts
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";

const sdk = new NodeSDK({
  instrumentations: [getNodeAutoInstrumentations()],
  // ...
});
```

Esto instrumenta automáticamente `http`, `express`, `pg`, `redis`, `kafkajs`, `axios`, etc. **Cero código de aplicación**.

### Convensiones semánticas

OTel define nombres estándar para atributos: `http.method`, `http.status_code`, `db.system`, `messaging.destination.name`. Usarlos hace que los datos sean comparables entre servicios y lenguajes.

### Por qué importa

- **No vendor lock-in**: cambias de proveedor de traces sin re-instrumentar.
- **Correlación**: traceId y spanId viajan en logs y métricas, uniendo los tres pilares.
- **Ecosistema**: prácticamente todas las herramientas modernas hablan OTLP.

---

## 5. Grafana

### Idea central

**Capa de visualización** que se conecta a múltiples fuentes de datos (Prometheus, Loki, Tempo, Elasticsearch, BigQuery, etc.) y permite construir dashboards y alertas unificadas.

### Dashboards por tipo de audiencia

| Audiencia       | Necesita ver                                |
| --------------- | ------------------------------------------- |
| Desarrollador   | Traces de sus servicios, logs de errores    |
| SRE / Operador  | Dashboards RED/USE, alertas                 |
| Producto        | Métricas de uso, conversión, retención      |
| Negocio         | KPIs, disponibilidad, costos                |

### Ejemplo de dashboard RED

```promql
# Rate
sum(rate(http_requests_total[5m]))

# Errors
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# Duration p95
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
```

### Correlación Logs ↔ Traces ↔ Metrics

Lo que hace poderoso a Grafana es **saltar entre los tres pilares** con el mismo `traceId`. Click en un pico de errores en una métrica → ver los logs de ese momento → abrir la traza concreta.

### Alertas en Grafana

```yaml
# alerta: error rate alto
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m])) > 0.01
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error rate > 1% durante 5 minutos"
    runbook: "https://wiki.example.com/runbooks/high-error-rate"
```

---

## 6. Prometheus

### Idea central

**Sistema de monitoreo** basado en **pull**: tu aplicación expone `/metrics`, Prometheus las **scrapea** periódicamente y las guarda como series temporales.

### Modelo de datos

```
metric_name{label1="value1", label2="value2"} = 42.0 @ 1700000000
```

### Componentes

- **Prometheus server**: scrapea, almacena, evalúa reglas.
- **Exporters**: agentes que exponen métricas de sistemas que no las exponen nativamente (Node exporter, MySQL exporter, etc.).
- **Alertmanager**: recibe alertas y las enruta (email, Slack, PagerDuty).
- **Pushgateway**: para jobs batch de corta duración que no pueden ser scrapeados.

### Configuración básica

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "pedidos-api"
    static_configs:
      - targets: ["pedidos-api:3000"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

### PromQL básico

```promql
# requests por segundo últimos 5 min
rate(http_requests_total[5m])

# p99 de latencia
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# errores por servicio
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
```

### Limitaciones

- **Pull model**: si tu servicio está detrás de un firewall, necesitas service discovery o pushgateway.
- **Alta cardinalidad**: muchas etiquetas con valores únicos destruyen el rendimiento.
- **Retención**: el almacenamiento local no es infinito. Para largo plazo, Thanos o Cortex.
- **Una sola dimensión temporal**: para multi-dimensional, considera otras opciones.

---

## Cómo se conectan los tres pilares

```
HTTP Request
    │
    ├──► Span (trace): "GET /pedidos/123" dur=82ms
    │       atributos: userId, pedidoId, status
    │
    ├──► Counter++: http_requests_total{route="/pedidos/:id", status="200"}
    │       Histogram observe: http_request_duration_seconds{route="/pedidos/:id"}
    │
    └──► Log: { level: info, msg: "pedido obtenido", traceId, userId, pedidoId }
                ↑ todos comparten traceId
```

Cuando algo falla:

1. La alerta (Prometheus) dice "error rate > 1%".
2. Abres Grafana y ves el pico en el dashboard.
3. Filtras los logs (Loki) en ese intervalo y encuentras el error.
4. Abres la traza (Tempo/Jaeger) con el `traceId` del log.
5. Ves exactamente qué span se demoró o falló.

---

## SLIs, SLOs y SLAs

### SLI (Service Level Indicator)

La **métrica** que importa. Ejemplos:

- Disponibilidad = requests exitosas / total.
- Latencia = fracción de requests bajo X ms.

### SLO (Service Level Objective)

El **objetivo** que te propones. Ejemplo:

- 99.9% de requests exitosas en 30 días.
- p99 de latencia < 300ms.

### SLA (Service Level Agreement)

El **contrato con el cliente**, generalmente con consecuencias económicas.

```
SLI  → qué mido
SLO  → qué me propongo
SLA  → qué le prometo al cliente (más estricto que SLO)
```

### Error budget

Si tu SLO es 99.9%, tu error budget es 0.1%. Cuando se gasta, el equipo prioriza confiabilidad sobre features.

---

## Buenas prácticas

- **Empieza por los tres pilares, no por la herramienta**. Saber qué medir antes de elegir Prometheus vs Datadog.
- **Convención semántica**. Usa los nombres de OTel (`http.request.method` en vez de `httpMethod`).
- **Correlación por traceId**. Logs, métricas y trazas deben compartir identificadores.
- **No inventar métricas**. Primero necesitas, luego instrumentas.
- **Sampling en traces**. 100% es insostenible en alto volumen.
- **Alertas accionables**. Si no hay acción clara, no es una alerta, es ruido.
- **Dashboards por audiencia**. Un dev no necesita el mismo dashboard que un CFO.
- **No loguear PII ni secretos**. Cumplimiento legal y de seguridad.
- **Retención realista**. Logs por 30 días, métricas por 1 año, traces por 7-14 días.
- **Runbooks**. Cada alerta debería tener un link a "qué hacer cuando esto pasa".

---

## Ejercicios

### Ejercicio 1 — Elige el pilar

Para cada pregunta, identifica si la responde mejor un log, una métrica o una traza:

1. ¿Cuántas requests por segundo recibe `/pedidos`?
2. ¿Por qué esta request tardó 6 segundos?
3. ¿Cuántos errores 500 hubo ayer?
4. ¿Qué servicio downstream está lento?
5. ¿Cuánta memoria está usando este pod?
6. ¿Qué usuario experimentó este error?
7. ¿Cuál es la disponibilidad del sistema en los últimos 30 días?

### Ejercicio 2 — Instrumenta un servicio

Toma un servicio HTTP pequeño y:

1. Añade **logging estructurado** (pino, winston, zap).
2. Expón **métricas RED** con Prometheus.
3. Instrumenta con **OpenTelemetry** (auto + manual).
4. Configura un `traceId` en los logs que coincida con el span.

### Ejercicio 3 — Diseña un dashboard RED

Para un servicio `pedidos-api`, define:

- 4 panels para Rate.
- 4 panels para Errors.
- 4 panels para Duration.
- 2 panels de uso de recursos (CPU, memoria).

Incluye las queries PromQL.

### Ejercicio 4 — Alertas accionables

Define 5 alertas para `pedidos-api`. Para cada una:

- Condición (PromQL).
- Severidad.
- Runbook (qué hacer).
- Umbral de `for` (cuánto tiempo sostenido).

### Ejercicio 5 — Propaga contexto

En un sistema con 3 servicios (gateway → API → DB):

1. Genera un `traceId` en el gateway.
2. Propágalo al API (header `traceparent`).
3. Propágalo a la query a la DB.
4. Comprueba que la traza muestra los 3 spans jerárquicos.

### Ejercicio 6 — SLO realista

Para `pedidos-api`:

- Define 2 SLIs (disponibilidad, latencia).
- Define 2 SLOs mensuales.
- Calcula el error budget.
- Diseña una alerta que avise cuando se gaste 50%, 80% y 100% del budget.

---

## Problema propuesto

### Plataforma de observabilidad para un e-commerce

Vas a construir la **capa de observabilidad** de un e-commerce con esta arquitectura:

```
Web/Mobile → API Gateway → [pedidos-api, pagos-api, inventario-api, auth-api]
                                          ↓
                              PostgreSQL, Redis, Kafka
```

Debes entregar dashboards, alertas, traces, logs y un SLO público.

### Requisitos funcionales

1. **Logging estructurado** en los 4 servicios con `traceId` consistente.
2. **Métricas RED** expuestas vía `/metrics` y scrapeadas por Prometheus.
3. **Tracing distribuido** con OpenTelemetry que muestre el camino de una request desde el gateway hasta la DB.
4. **Logs centralizados** consultables por `traceId`, `userId`, `servicio`, `nivel`.
5. **Dashboards en Grafana**:
   - Uno por servicio (RED + recursos).
   - Uno de arquitectura global (errores, latencia, throughput por servicio).
6. **Alertas en Alertmanager** con severidades y runbooks.
7. **SLO documentado** para el servicio `pedidos-api` con error budget tracking.

### Requisitos técnicos

Tu solución debe:

1. Usar **OpenTelemetry SDK** en al menos 2 servicios (uno Node, uno Python o Go si puedes).
2. Configurar un **OpenTelemetry Collector** que reciba OTLP y exporte a Prometheus (métricas), Loki (logs) y Jaeger/Tempo (traces).
3. Implementar **propagación de contexto** entre servicios vía headers.
4. Generar **logs JSON** con `traceId` y `spanId`.
5. Exponer **métricas RED** y al menos 3 métricas de negocio (ej. `pedidos_creados_total`).
6. Definir **5 alertas accionables** con runbook.
7. Calcular el **error budget mensual** del SLO de `pedidos-api`.

### Entregables

- Diagrama de la arquitectura de observabilidad.
- Configs: `prometheus.yml`, `loki-config.yaml`, `otel-collector-config.yaml`, dashboards JSON.
- Código de instrumentación de los servicios.
- Capturas o exports de:
  - Dashboard RED de un servicio.
  - Una traza distribuida con al menos 4 spans.
  - Logs correlacionados por `traceId`.
  - Una alerta disparada en Alertmanager.
- Documento de 1 página con:
  - SLOs definidos.
  - Error budget policy.
  - Lista de alertas y sus runbooks.
  - Decisiones sobre retención, sampling y cardinalidad.
  - Qué harías distinto a escala 10x o 100x.

### Criterios de evaluación

| Criterio                                            | Peso |
| --------------------------------------------------- | ---- |
| Los tres pilares correctamente implementados        | 25%  |
| OpenTelemetry con propagación de contexto           | 20%  |
| Dashboards útiles y bien diseñados                  | 15%  |
| Alertas accionables con runbooks                    | 15%  |
| SLO y error budget bien definidos                   | 10%  |
| Decisiones justificadas (retención, sampling)       | 15%  |
