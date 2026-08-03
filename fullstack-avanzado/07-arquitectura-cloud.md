# Arquitectura Cloud

## Objetivo de aprendizaje

Llevar una aplicación del "funciona en mi máquina" a **producción real** requiere decisiones sobre contenedorización, balanceo, almacenamiento, caché, base de datos y escalabilidad. Aquí se estudian los **bloques básicos** que componen cualquier arquitectura cloud moderna.

No se trata de aprender un proveedor (AWS, GCP, Azure) sino de entender los **patrones** que son transferibles entre ellos.

---

## Mapa general

| Componente         | Función                                            |
| ------------------ | -------------------------------------------------- |
| Docker             | Empaquetar la app con sus dependencias             |
| Docker Compose     | Orquestar varios contenedores localmente           |
| Reverse Proxy      | Recibe requests y las distribuye (Nginx, Traefik)  |
| CDN                | Cachea contenido estático cerca del usuario        |
| Object Storage     | Archivos grandes: S3, GCS, MinIO                   |
| Redis              | Caché, sesiones, colas efímeras, locks             |
| PostgreSQL         | Base de datos relacional principal                 |
| Horizontal Scaling | Múltiples instancias detrás de un balanceador      |

---

## 1. Docker

### Idea central

Empaquetar la app y **todas sus dependencias** (sistema, librerías, configuración) en una unidad portable: la **imagen**. Un contenedor es una instancia corriendo de una imagen.

### Dockerfile de una app Node

```dockerfile
# build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# runtime stage
FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Conceptos clave

- **Multi-stage builds**: separar compilación de ejecución → imágenes finales más pequeñas.
- **Capas**: cada instrucción crea una capa. Cachear lo que no cambia.
- **Usuario no root**: `USER node` reduce superficie de ataque.
- **`.dockerignore`**: excluye `node_modules`, `.git`, etc.
- **Tags inmutables**: usa versiones específicas, no `latest` en producción.

### Buenas prácticas

- Imagen base mínima (`alpine`, `distroless`).
- Variables de entorno para configuración.
- Un proceso por contenedor.
- `HEALTHCHECK` para que el orquestador sepa si el contenedor está sano.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

---

## 2. Docker Compose

### Idea central

Definir **múltiples contenedores** que se ejecutan juntos en un archivo declarativo. Ideal para desarrollo local y stacks pequeños.

```yaml
# docker-compose.yml
services:
  api:
    build: ./apps/api
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/app
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  web:
    build: ./apps/web
    ports:
      - "5173:5173"
    environment:
      VITE_API_URL: http://localhost:3000

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app"]
      interval: 5s

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### Comandos esenciales

```bash
docker compose up -d             # arrancar
docker compose logs -f api       # ver logs
docker compose exec api sh       # entrar al contenedor
docker compose down -v           # parar y borrar volúmenes
```

### Redes

Por defecto Compose crea una red donde los servicios se resuelven **por nombre** (`db`, `redis`). En producción esto se reemplaza por una red gestionada (VPC, k8s service discovery).

### Cuándo usar Compose

- **Desarrollo local**: toda la stack en tu máquina.
- **Staging pequeño**: un solo host con varios servicios.
- **CI/CD**: correr tests contra dependencias reales.

### Cuándo **no** usarlo

- Producción con alta disponibilidad → Kubernetes, ECS, Nomad.
- Más de ~10-20 servicios → se vuelve difícil de mantener.

---

## 3. Reverse Proxy

### Idea central

Un servidor que **recibe** las requests de los clientes y las **reenvía** a los servicios internos. Oculta la topología, agrega TLS, hace balanceo.

```
Cliente → Reverse Proxy → [api-1, api-2, api-3]
                │
                ├── TLS termination
                ├── Compresión
                ├── Rate limiting
                ├── Caching
                └── Logging
```

### Nginx (clásico)

```nginx
upstream api_backend {
  least_conn;
  server api-1:3000;
  server api-2:3000;
  server api-3:3000;
}

server {
  listen 80;
  server_name api.example.com;

  gzip on;
  client_max_body_size 10m;

  location / {
    proxy_pass http://api_backend;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
  }

  location /health {
    return 200 "ok\n";
  }
}
```

### Traefik (cloud-native)

Se configura **por labels** en Docker y detecta servicios automáticamente. Excelente para entornos dinámicos.

```yaml
# docker-compose.yml
services:
  api:
    image: mi-api:1.0
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"
      - "traefik.http.services.api.loadbalancer.server.port=3000"

  traefik:
    image: traefik:v3
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml
```

### Funciones típicas

- **TLS termination**: descifra HTTPS y habla HTTP con el backend.
- **Balanceo**: round-robin, least connections, ip-hash.
- **Rate limiting**: prevenir abuso.
- **Compresión**: gzip/brotli para reducir ancho de banda.
- **Caching**: servir respuestas cacheables sin pasar al backend.
- **Header propagation**: `X-Forwarded-For`, `X-Real-IP`.

---

## 4. CDN (Content Delivery Network)

### Idea central

Una red de servidores **distribuidos geográficamente** que cachea contenido cerca del usuario. Reduce latencia y descarga al origen.

### Qué se sirve por CDN

| Contenido               | ¿CDN? | Por qué                                  |
| ----------------------- | ----- | ---------------------------------------- |
| JS, CSS, fuentes        | ✅    | Estático, cacheable, muchos usuarios     |
| Imágenes, video         | ✅    | Alto peso, distribuidos globalmente      |
| HTML dinámico           | ❌    | No cacheable (o cacheable con cuidado)   |
| APIs JSON               | ❌/parcial | Cacheable por query, pero invalidación compleja |
| Assets privados         | ❌    | Firmados o con cookies                   |

### Configuración típica (CloudFront / Cloudflare / Fastly)

```
origen: api.example.com
comportamiento:
  /static/* → cache 1 año, immutable
  /api/*    → no cache (o cache corto con stale-while-revalidate)
  /*        → cache 5 min
```

### Headers importantes

- `Cache-Control: public, max-age=31536000, immutable` para assets con hash.
- `Cache-Control: no-store` para contenido sensible.
- `Vary: Accept-Encoding` para gzip/brotli.
- `ETag` y `Last-Modified` para revalidación.

### Invalidación

Cuando cambias un asset, el CDN debe **invalidar** la versión vieja. Con **fingerprints en el nombre** (`app.a3b1c2.js`) no hace falta: el nombre cambia y el CDN pide la nueva versión.

---

## 5. Object Storage (Almacenamiento de Objetos)

### Idea central

Almacenar **archivos como objetos** en buckets planos. Sin jerarquía de carpetas real, con metadatos y URLs. **Prácticamente infinito** y muy barato.

### Proveedores

- **AWS S3** (el estándar de facto).
- **Google Cloud Storage**.
- **Azure Blob Storage**.
- **MinIO** (self-hosted, compatible con API S3).

### Casos de uso

- Archivos subidos por usuarios (avatares, documentos).
- Assets estáticos servidos por CDN.
- Backups.
- Archivos grandes (videos, datasets).
- Logs de larga retención.

### Patrones de uso

```
[Cliente] → POST /upload (API genera URL prefirmada) → API devuelve URL
[Cliente] → PUT archivo a URL prefirmada → S3
[Cliente] → GET archivo vía CDN con URL prefirmada
```

**URLs prefirmadas**: el cliente sube **directamente** a S3 sin pasar por tu API. Tu API solo genera la URL firmada (típicamente 5-15 min). Reduce carga y带宽 del backend.

### Ejemplo con S3 SDK

```ts
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: "us-east-1" });

const uploadUrl = await getSignedUrl(
  s3,
  new PutObjectCommand({ Bucket: "uploads", Key: `avatar-${userId}` }),
  { expiresIn: 600 },
);

const downloadUrl = await getSignedUrl(
  s3,
  new GetObjectCommand({ Bucket: "uploads", Key: `avatar-${userId}` }),
  { expiresIn: 3600 },
);
```

### Buenas prácticas

- **No servir desde tu API**: usa el bucket directo detrás de un CDN.
- **Versionado activado**: para rollback y protección contra borrados accidentales.
- **Lifecycle policies**: mover a Glacier (frío) después de N días.
- **Permisos mínimos**: IAM por bucket, no llaves globales.
- **URLs prefirmadas** para uploads directos.

---

## 6. Redis

### Idea central

Almacén **clave-valor en memoria** con estructuras ricas (strings, hashes, lists, sets, sorted sets, streams). Usado como **caché, sesión, cola, lock y mucho más**.

### Casos de uso

| Caso             | Ejemplo                                                   |
| ---------------- | --------------------------------------------------------- |
| Caché            | `GET user:42` en vez de ir a PostgreSQL                   |
| Sesiones         | Cookie de sesión → lookup en Redis                        |
| Rate limiting    | `INCR rate:user:42` con TTL                               |
| Colas            | `LPUSH jobs` / `BRPOP jobs`                               |
| Locks distribuidos | `SET lock:recurso 1 NX EX 30`                          |
| Pub/Sub          | Chat, notificaciones, presence                            |
| Contadores       | Views, likes, métricas en tiempo real                     |
| Leaderboards     | `ZSET` con score = puntos                                 |

### Ejemplo: caché de usuario

```ts
async function getUser(id: string) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.user.findUnique({ where: { id } });
  if (!user) return null;

  await redis.set(`user:${id}`, JSON.stringify(user), "EX", 300);
  return user;
}
```

### Patrones de invalidación

- **TTL**: lo más simple. Acepta datos "viejos" por N segundos.
- **Write-through**: al escribir, actualiza DB y caché.
- **Write-behind**: escribe en caché, la caché persiste a DB async.
- **Invalidación explícita**: `DEL` al actualizar.
- **Pub/Sub para invalidación**: muchos nodos se enteran al mismo tiempo.

### Persistencia

Redis puede persistir (RDB snapshots, AOF), pero **no es su fuerte**. Si necesitas durabilidad, usa una base de datos. Redis es **caché y coordinación**, no fuente de verdad.

### Alta disponibilidad

- **Replica set**: un primario + N réplicas de lectura.
- **Sentinel**: failover automático.
- **Cluster**: sharding horizontal nativo.

### Cuándo usarlo

- Cualquier cosa que necesite **microsegundos de latencia**.
- Sesiones, rate limit, locks, colas pequeñas.
- Contadores, leaderboards, presence.

### Cuándo **no** usarlo

- Como base de datos principal con datos críticos.
- Volúmenes que no caben en RAM.
- Queries complejas relacionales → PostgreSQL.

---

## 7. PostgreSQL

### Idea central

Base de datos relacional **open source** más avanzada del mundo. Soporta JSON, índices GIN/GIST, full-text search, CTEs recursivos, window functions, y extensiones (PostGIS, TimescaleDB, pgvector).

### Patrones de uso

| Patrón            | Cuándo                                              |
| ----------------- | --------------------------------------------------- |
| Primary-replica   | Reads en réplicas, writes en el primario            |
| Connection pool   | PgBouncer para manejar muchas conexiones            |
| Migrations        | Herramientas versionadas (Prisma Migrate, Flyway, Liquibase, Drizzle) |
| Backups           | `pg_dump` + WAL archiving para PITR                 |
| Particionamiento  | Tablas enormes por fecha o tenant                   |
| Sharding manual   | Solo cuando es inevitable (Citus, lógica propia)    |

### Connection pooling

PostgreSQL no maneja bien miles de conexiones directas. **PgBouncer** (o Pgpool) se sienta entre la app y la DB.

```
[App 1] ─┐
[App 2] ─┼─→ PgBouncer ─→ PostgreSQL
[App 3] ─┘
```

Modos:

- **Session pooling**: una conexión por sesión cliente.
- **Transaction pooling**: una conexión por transacción (recomendado para serverless).
- **Statement pooling**: una conexión por statement (no compatible con todo).

### Ejemplo: schema con relaciones

```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  email CITEXT NOT NULL,
  UNIQUE (tenant_id, email)
);

CREATE INDEX users_tenant_idx ON users(tenant_id);
```

### Backups y PITR

- **Logical**: `pg_dump` / `pg_restore`. Fáciles, lentos a escala.
- **Physical**: copia de archivos. Rápido, requiere downtime o snapshots.
- **WAL archiving**: continuo, permite Point-In-Time Recovery.

### Observabilidad de PostgreSQL

- `pg_stat_statements`: queries más lentas y frecuentes.
- `pg_stat_activity`: conexiones activas.
- Métricas del exporter de Prometheus para Postgres.

---

## 8. Horizontal Scaling

### Idea central

Añadir **más instancias** del mismo servicio para manejar más carga, en vez de hacer la instancia más grande (vertical).

### Stateless vs Stateful

| Tipo        | Ejemplos                   | Escala horizontal                    |
| ----------- | -------------------------- | ------------------------------------ |
| **Stateless** | API REST, workers, web    | Trivial: poner N réplicas tras un LB |
| **Stateful**  | DB, Redis, Kafka          | Requiere sharding, replicas, clusters |

**Regla de oro**: tu API **debe ser stateless**. Sesión, uploads, caché → a Redis, S3, etc. Si guardas estado en el proceso, no puedes escalar.

### Load balancing

```
        ┌──► api-1
Cliente─┼──► api-2
        └──► api-3
```

Algoritmos:

- **Round-robin**: turnarse.
- **Least connections**: al menos ocupado.
- **IP hash**: mismo cliente siempre al mismo backend (útil para WebSocket).
- **Weighted**: dar más carga a máquinas más potentes.

### Health checks

El balanceador debe **saber** qué instancias están sanas. Configura:

```nginx
upstream api_backend {
  server api-1:3000 max_fails=3 fail_timeout=10s;
  server api-2:3000 max_fails=3 fail_timeout=10s;
  server api-3:3000 max_fails=3 fail_timeout=10s;
}
```

### Auto-scaling

Reglas típicas:

- CPU > 70% durante 5 min → +1 instancia.
- CPU < 30% durante 10 min → -1 instancia.
- Cola de mensajes > N → +1 worker.
- Latencia p99 > X → +1 instancia.

En Kubernetes: `HorizontalPodAutoscaler` (HPA). En AWS: ECS Service Auto Scaling, EC2 Auto Scaling Groups.

### Stateless, en serio

Patrones que rompen la escalabilidad horizontal:

- Sesión en memoria del proceso.
- Archivos subidos al disco local.
- Cache local que se vuelve inconsistente entre instancias.
- WebSocket stateful sin sticky sessions o broker.

**Regla**: si la instancia se puede matar en cualquier momento y otra asumir su trabajo, es stateless.

### Sharding

Cuando una sola DB no aguanta, **partes los datos** entre múltiples DBs.

Estrategias:

- **Por tenant**: `tenant_0..9` cada uno con su DB.
- **Por rango**: usuarios A-M en DB1, N-Z en DB2.
- **Por hash**: `hash(userId) % N` decide la DB.

Trade-offs: cross-shard queries son caras, transacciones distribuidas son un dolor.

### Caches, réplicas, CDNs: cómo escalan

| Capa          | Escala horizontal                          |
| ------------- | ------------------------------------------ |
| CDN           | Automática (es el punto)                   |
| Reverse proxy | Múltiples con anycast / DNS round-robin    |
| API stateless | Trivial, suma N réplicas                   |
| Redis         | Cluster sharding + replicas                |
| PostgreSQL    | Replicas de lectura; sharding si hace falta|
| Object storage| Nativa (es la base del modelo)             |
| Workers       | Suma réplicas; la cola reparte trabajo     |

---

## Patrón de arquitectura de referencia

```
                          ┌──────────┐
                          │   CDN    │  (assets estáticos)
                          └────┬─────┘
                               │
                          ┌────▼─────┐
                          │  DNS /   │
                          │ Anycast  │
                          └────┬─────┘
                               │
                          ┌────▼─────┐
                          │  Edge    │  (TLS, WAF, rate limit)
                          │  LB      │
                          └────┬─────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
          ┌───────┐        ┌───────┐        ┌───────┐
          │ api-1 │        │ api-2 │        │ api-3 │   ← stateless, auto-scale
          └───┬───┘        └───┬───┘        └───┬───┘
              │                │                │
              └────────────────┼────────────────┘
                               │
              ┌────────────────┼─────────────────┐
              ▼                ▼                 ▼
         ┌────────┐        ┌────────┐        ┌────────┐
         │ Redis  │        │ Postgres│       │ Object │
         │ cluster│        │ primary │       │ Storage│
         │        │        │  + repl.│       │  (S3)  │
         └────────┘        └────────┘        └────────┘
```

---

## Buenas prácticas transversales

- **12-factor app**: configuración en variables de entorno, logs a stdout, sin estado en el proceso.
- **Imágenes inmutables**: tag con `git-sha`, no con `latest`.
- **Health checks** claros: `/health/live` (sigues vivo) y `/health/ready` (puedes recibir tráfico).
- **Graceful shutdown**: terminar requests en curso antes de parar (SIGTERM).
- **Backups probados**: un backup no testeado no es un backup.
- **TLS everywhere**: incluso tráfico interno entre servicios.
- **Secrets en vaults**: nunca en el código, ni en `.env` committeado.
- **Logs estructurados a stdout**: el colector se encarga de llevarlos a donde corresponda.
- **Observabilidad primero**: sin métricas y trazas, escalar es volar a ciegas.

---

## Ejercicios

### Ejercicio 1 — Dockerfile multi-stage

Toma una app en el lenguaje que prefieras y:

1. Escribe un Dockerfile multi-stage.
2. Reduce la imagen final a < 200 MB.
3. Añade un `HEALTHCHECK`.
4. Ejecuta el contenedor y comprueba `/health`.

### Ejercicio 2 — Docker Compose completo

Construye un `docker-compose.yml` con:

- App API (Node/Python/Go).
- PostgreSQL con volumen.
- Redis con volumen.
- Nginx como reverse proxy.
- Un servicio que dependa de los otros.

Comprueba que todo arranca y se comunican por nombre.

### Ejercicio 3 — Reverse proxy con TLS

Configura Nginx (o Traefik) para:

- Terminar TLS (cert auto-firmado o Let's Encrypt local).
- Hacer proxy a un backend con `X-Forwarded-*` headers.
- Aplicar rate limit (10 req/s por IP).
- Servir un endpoint `/health` que no pase al backend.

### Ejercicio 4 — CDN y caché de assets

1. Sube 3 imágenes a S3 (o MinIO).
2. Configúralas con `Cache-Control: public, max-age=31536000, immutable`.
3. Pon un CDN delante (CloudFront, Cloudflare, o local con Varnish).
4. Mide latencia con y sin CDN desde "otra región" (puedes simular con throttling).

### Ejercicio 5 — Redis en 4 roles distintos

Implementa los 4 casos en una sola app:

1. **Caché** de un endpoint que lee de PostgreSQL.
2. **Rate limit** con `INCR` + TTL.
3. **Lock distribuido** con `SET NX EX`.
4. **Cola** simple con `LPUSH` / `BRPOP`.

Mide latencia de cada caso y observa qué pasa cuando matas Redis.

### Ejercicio 6 — PostgreSQL con réplicas

Levanta PostgreSQL con:

- 1 primary.
- 2 réplicas de lectura.

Configura tu app para:

- Writes → primary.
- Reads → réplicas (round-robin).

Comprueba el lag de replicación y qué pasa si matas la primary.

### Ejercicio 7 — Stateless por diseño

Refactoriza una app que tenía:

- Sesiones en memoria → a Redis.
- Uploads locales → a S3.
- WebSocket stateful → a Socket.io con Redis adapter.

Demuestra que ahora puedes matar cualquier instancia sin perder estado.

### Ejercicio 8 — Decide la estrategia de escalado

Para cada caso, define:

- ¿Stateless o stateful?
- ¿Vertical u horizontal?
- ¿Cuántas réplicas?
- ¿Cómo se balancea?
- ¿Cuándo auto-scale?

Casos:

1. API REST de un marketplace.
2. Servicio que procesa imágenes.
3. WebSocket de chat 1-a-1.
4. Worker de generación de PDFs.
5. PostgreSQL de un SaaS multi-tenant.

---

## Problema propuesto

### Despliegue de un SaaS multi-tenant en cloud

Vas a llevar una aplicación fullstack (frontend + API + DB + storage) a un entorno cloud production-ready. La aplicación tiene ~50,000 usuarios activos, picos de 10x en eventos especiales.

### Requisitos funcionales

1. **Frontend** estático servido por CDN.
2. **API** stateless, escalable horizontalmente, detrás de un reverse proxy.
3. **PostgreSQL** con primary + réplicas de lectura, connection pooling.
4. **Redis** cluster para caché y sesiones.
5. **Object storage** para uploads de usuarios, accedido vía URLs prefirmadas.
6. **Workers** para procesos de fondo (envío de emails, generación de reportes).
7. **Observabilidad**: métricas RED, logs centralizados, tracing distribuido.
8. **CI/CD** que buildea imágenes, las pushea, y despliega sin downtime.

### Requisitos técnicos

Tu solución debe:

1. Tener un **Dockerfile multi-stage** por servicio con imagen < 300 MB.
2. Un **docker-compose.yml** que levante todo el stack localmente.
3. Un **reverse proxy** (Nginx o Traefik) con TLS, rate limit, gzip, headers de forwarding.
4. **PostgreSQL** configurado con al menos 1 réplica y PgBouncer.
5. **Redis** con persistencia y alta disponibilidad (puede ser Sentinel).
6. **Object storage** con URLs prefirmadas para upload/download.
7. **CDN** sirviendo assets del frontend.
8. **Horizontal scaling** demostrado: matar una instancia de la API y comprobar que el servicio sigue respondiendo.
9. **Health checks** que diferencien liveness y readiness.
10. **Graceful shutdown** que termina requests en curso.

### Entregables

- Diagrama de la arquitectura final con todos los componentes.
- Dockerfiles y `docker-compose.yml` completos.
- Configs de Nginx/Traefik, PostgreSQL, PgBouncer, Redis.
- Scripts o pipeline de CI/CD (puede ser GitHub Actions o similar).
- Capturas o logs de:
  - API sirviendo tráfico con 3 réplicas.
  - Kill de una réplica sin pérdida de requests.
  - Upload de un archivo vía URL prefirmada.
  - Latencia antes y después del CDN.
- Documento de 1 página justificando:
  - Por qué cada componente.
  - Cómo escala cada pieza a 10x.
  - Qué single point of failure queda y cómo lo resolverías.
  - Qué harías con Kubernetes en vez de Docker Compose.

### Criterios de evaluación

| Criterio                                       | Peso |
| ---------------------------------------------- | ---- |
| Dockerfiles optimizados y funcionales          | 15%  |
| Docker Compose completo y bien estructurado    | 15%  |
| Reverse proxy con TLS, rate limit, forwarding  | 10%  |
| PostgreSQL con réplicas y pooling              | 10%  |
| Redis correctamente usado (múltiples roles)    | 10%  |
| Object storage con URLs prefirmadas            | 10%  |
| Horizontal scaling demostrado                  | 15%  |
| Observabilidad básica presente                 | 10%  |
| Justificación y trade-offs                     |  5%  |
