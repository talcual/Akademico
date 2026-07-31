# Módulo 1: Fundamentos de la Web

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Explicar el funcionamiento de Internet y la arquitectura cliente-servidor.
- Diferenciar los protocolos HTTP y HTTPS, así como sus métodos y códigos de estado.
- Describir el papel del DNS en la resolución de nombres de dominio.
- Distinguir entre dominio y hosting, y comprender el proceso de publicación de un sitio web.
- Identificar los principales navegadores y motores de renderizado.
- Utilizar las DevTools del navegador para inspeccionar, depurar y perfilar aplicaciones web.
- Reconocer la estructura general de una aplicación web moderna.

---

## 1. ¿Cómo funciona Internet?

Internet es una **red global de redes** interconectadas que utilizan un conjunto de protocolos comunes (suite TCP/IP) para comunicarse entre sí. No es propiedad de nadie; es un sistema distribuido y descentralizado.

### Conceptos clave

- **Paquete**: unidad básica de información que viaja por la red. Contiene datos y metadatos (origen, destino, etc.).
- **IP (Internet Protocol)**: asigna una dirección única a cada dispositivo conectado (IPv4: `192.168.1.1`, IPv6: `2001:0db8::1`).
- **TCP (Transmission Control Protocol)**: garantiza que los paquetes lleguen en orden y sin errores.
- **UDP (User Datagram Protocol)**: envía paquetes sin garantía de entrega, usado en streaming y videojuegos.
- **ISP (Internet Service Provider)**: proveedor que da acceso a Internet (ej. Comcast, Movistar, Claro).
- **Routers y switches**: dispositivos que dirigen el tráfico entre redes.
- **Backbone**: infraestructura de fibra óptica de alta capacidad que conecta continentes y países.

### El viaje de un paquete

1. El usuario escribe una URL en el navegador.
2. El navegador consulta el DNS para traducir el dominio a una IP.
3. Se establece una conexión TCP (handshake de tres pasos: SYN, SYN-ACK, ACK) o una conexión TLS si es HTTPS.
4. Se envía una petición HTTP al servidor de destino.
5. El servidor procesa la petición y devuelve una respuesta.
6. El navegador interpreta la respuesta y renderiza la página.

### Modelo OSI (resumen)

| Capa | Función                              | Ejemplo              |
| ---- | ------------------------------------ | -------------------- |
| 7    | Aplicación                           | HTTP, DNS, FTP, SMTP |
| 6    | Presentación                         | TLS/SSL, cifrado     |
| 5    | Sesión                               | NetBIOS, RPC         |
| 4    | Transporte                           | TCP, UDP             |
| 3    | Red                                  | IP, ICMP             |
| 2    | Enlace de datos                      | Ethernet, Wi-Fi      |
| 1    | Física                               | Cables, señales      |

### Actividad práctica

- Abre una terminal y ejecuta `traceroute google.com` (macOS/Linux) o `tracert google.com` (Windows). Identifica cuántos saltos realiza un paquete hasta llegar a su destino.
- Usa `ping` para medir la latencia con distintos servidores.

---

## 2. Cliente vs Servidor

La web se basa en un modelo **cliente-servidor** donde dos roles intercambian información siguiendo un protocolo.

### Cliente

Es el programa que **solicita** recursos o servicios. En la web, el cliente más común es el navegador (Chrome, Firefox, Safari, Edge), pero también pueden serlo aplicaciones móviles, CLI, o incluso otros servidores.

Responsabilidades típicas:
- Renderizar la interfaz de usuario.
- Enviar peticiones al servidor.
- Manejar la interacción del usuario.
- Almacenar datos locales (cookies, localStorage, IndexedDB).

### Servidor

Es el programa que **escucha** peticiones en un puerto y devuelve respuestas. Puede ser físico (una máquina en un datacenter) o lógico (un proceso dentro de una máquina).

Responsabilidades típicas:
- Atender peticiones HTTP/HTTPS.
- Ejecutar lógica de negocio.
- Acceder a bases de datos.
- Servir archivos estáticos (HTML, CSS, JS, imágenes).
- Autenticar y autorizar usuarios.

### Tipos de servidores

- **Servidor web**: Nginx, Apache, Caddy. Sirven archivos estáticos y delegan al backend.
- **Servidor de aplicaciones**: Node.js, Tomcat, Gunicorn, Puma. Ejecutan la lógica.
- **Servidor de bases de datos**: PostgreSQL, MySQL, MongoDB.
- **Servidor proxy / balanceador**: HAProxy, AWS ALB.
- **Servidor de caché**: Redis, Varnish, Cloudflare.

### Modelo petición-respuesta

```
Cliente                          Servidor
  | ------- GET /index.html ----> |
  |                                |
  | <-------- 200 OK + HTML ------ |
```

### Evolución del modelo

- **Monolito**: frontend y backend en una misma aplicación.
- **Cliente delgado (thin client)**: el servidor renderiza todo (PHP, JSP, Rails con ERB).
- **Cliente grueso (fat client)**: el navegador ejecuta gran parte de la lógica (SPA con React, Vue, Angular).
- **Arquitecturas modernas**: JAMstack, microservicios, serverless, BFF (Backend for Frontend).

### Actividad práctica

- Inspecciona las cabeceras HTTP de una petición con las DevTools (pestaña Network).
- Identifica qué partes vienen del cliente y cuáles del servidor.

---

## 3. HTTP y HTTPS

**HTTP (HyperText Transfer Protocol)** es el protocolo de aplicación que define cómo se intercambian recursos en la web. Es **sin estado** (cada petición es independiente) y trabaja sobre TCP, normalmente en el puerto **80**.

**HTTPS** es la versión segura de HTTP: añade una capa **TLS/SSL** que cifra la comunicación. Opera en el puerto **443**.

### Estructura de una petición HTTP

```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: text/html
Accept-Language: es-ES
Cookie: session=abc123
```

Partes:
- **Línea de inicio**: método + path + versión.
- **Cabeceras (headers)**: metadatos de la petición.
- **Cuerpo (body)**: opcional, presente en POST, PUT, PATCH.

### Métodos HTTP principales

| Método  | Uso                                  | Idempotente | Seguro |
| ------- | ------------------------------------ | ----------- | ------ |
| GET     | Obtener un recurso                   | Sí          | Sí     |
| POST    | Crear un recurso o enviar datos      | No          | No     |
| PUT     | Reemplazar un recurso completo       | Sí          | No     |
| PATCH   | Modificar parcialmente un recurso    | No          | No     |
| DELETE  | Eliminar un recurso                  | Sí          | No     |
| HEAD    | Igual que GET pero sin cuerpo        | Sí          | Sí     |
| OPTIONS | Consultar métodos permitidos (CORS)  | Sí          | Sí     |

### Códigos de estado

- **1xx**: Informativos.
- **2xx**: Éxito (200 OK, 201 Created, 204 No Content).
- **3xx**: Redirección (301 Moved Permanently, 302 Found, 304 Not Modified).
- **4xx**: Error del cliente (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests).
- **5xx**: Error del servidor (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout).

### HTTPS y TLS

- **TLS (Transport Layer Security)** cifra la comunicación y autentica al servidor.
- Se basa en **certificados digitales** emitidos por **CA (Certificate Authorities)**.
- Proceso resumido (handshake TLS 1.3):
  1. Cliente envía "ClientHello".
  2. Servidor responde con su certificado y la suite criptográfica elegida.
  3. Se intercambian claves para derivar una clave de sesión.
  4. Toda la comunicación posterior va cifrada.

### Headers importantes para developers

- `Content-Type`: tipo MIME (`application/json`, `text/html`).
- `Authorization`: token o credenciales (`Bearer xxx`).
- `Cache-Control`: directivas de caché.
- `Set-Cookie`: cookies que define el servidor.
- `Access-Control-Allow-Origin`: configuración CORS.

### HTTP/2 y HTTP/3

- **HTTP/1.1**: una petición por conexión (o varias con keep-alive).
- **HTTP/2**: multiplexación, compresión de cabeceras (HPACK), server push (deprecado).
- **HTTP/3**: usa QUIC sobre UDP, reduce la latencia del handshake.

### Actividad práctica

- Con `curl -v https://example.com` observa las cabeceras de petición y respuesta.
- Crea un pequeño servidor con `python -m http.server` y haz peticiones desde otra terminal con `curl`.

---

## 4. DNS (Domain Name System)

El DNS es el **"sistema de nombres"** de Internet. Traduce nombres legibles por humanos (`www.example.com`) a direcciones IP (`93.184.216.34`).

### ¿Por qué existe?

Recordar `93.184.216.34` es difícil. El DNS permite usar nombres como `example.com`.

### Estructura jerárquica

```
.                  (root)
 ├── com           (TLD genérico)
 │    └── example  (dominio)
 │         └── www (subdominio)
```

- **Root servers**: 13 lógicas (en realidad cientosAnycast) que conocen los servidores TLD.
- **TLD (Top Level Domain)**: `.com`, `.org`, `.net`, `.es`, `.io`.
- **Dominio**: nombre único registrado bajo un TLD.
- **Subdominio**: `www`, `mail`, `api`, `blog`.

### Tipos de registros DNS

| Registro | Función                                                | Ejemplo                          |
| -------- | ------------------------------------------------------ | -------------------------------- |
| A        | Mapea dominio a IPv4                                   | `example.com → 93.184.216.34`    |
| AAAA     | Mapea dominio a IPv6                                   | `example.com → 2606:2800:220:1:248:1893:25c8:1946` |
| CNAME    | Alias hacia otro dominio                               | `www → example.com`              |
| MX       | Servidores de correo                                  | `mail.example.com`               |
| TXT      | Texto libre (SPF, DKIM, verificación)                  | `v=spf1 include:_spf.google.com ~all` |
| NS       | Servidores de nombres autoritativos                    | `ns1.example.com`                |
| SOA      | Información de la zona                                 | datos administrativos             |
| SRV      | Servicios específicos (ej. XMPP, SIP)                  | `_sip._tcp.example.com`          |
| PTR      | Resolución inversa (IP → nombre)                       | usado en rDNS                    |

### Proceso de resolución

1. El navegador pregunta al **resolver** del sistema operativo (o al del ISP).
2. Si no tiene la respuesta, consulta un **servidor recursivo** (8.8.8.8, 1.1.1.1).
3. El recursivo pregunta al **root server**.
4. El root le indica el servidor del TLD (`.com`).
5. El servidor TLD le indica el servidor autoritativo del dominio.
6. El servidor autoritativo devuelve la IP.
7. El resultado se **cachea** durante el TTL.

### TTL (Time To Live)

Tiempo en segundos que un resolver puede cachear un registro. TTL altos reducen consultas pero dificultan cambios rápidos.

### DNS over HTTPS (DoH) y DNS over TLS (DoT)

Protocolos que cifran las consultas DNS para evitar espionaje y manipulación. Servicios: Cloudflare `1.1.1.1`, Google `8.8.8.8`.

### Actividad práctica

- `nslookup example.com` o `dig example.com` para ver los registros.
- `dig +trace example.com` para ver todo el proceso de resolución.
- Compara los tiempos de respuesta cambiando de DNS en tu sistema operativo.

---

## 5. Dominios y Hosting

### Dominio

Es un **nombre único** que identifica un sitio en Internet. Se contrata mediante un **registrar** (GoDaddy, Namecheap, Google Domains, Cloudflare) y tiene un costo anual.

Componentes:
- **SLD (Second Level Domain)**: la parte "example" en `example.com`.
- **TLD (Top Level Domain)**: `.com`, `.org`, `.io`, `.dev`.
- **Subdominio**: `www.example.com`, `api.example.com`.

Reglas:
- No puede haber dos dominios iguales bajo el mismo TLD.
- Los TLD tienen reglas específicas (`.edu`, `.gov`).
- Algunos son premium (`.ai`, `.io`) y cuestan mucho más.

### Ciclo de vida

1. **Disponible** → se puede registrar.
2. **Registrado** → pertenece a un dueño por un periodo (1 a 10 años).
3. **Expirado** → vuelve a estar disponible tras un periodo de gracia.
4. **Redención** → se puede recuperar pagando una penalización.
5. **Pending Delete** → a punto de liberarse.

### Hosting

Es el **espacio en un servidor** donde se alojan los archivos, bases de datos y servicios de un sitio web. Lo ofrece un **hosting provider** o **cloud provider**.

### Tipos de hosting

- **Compartido (shared)**: muchos sitios en un mismo servidor. Barato, poco rendimiento.
- **VPS (Virtual Private Server)**: máquina virtual con recursos dedicados. Ej. DigitalOcean, Linode.
- **Servidor dedicado (dedicated)**: máquina física completa para un solo cliente.
- **Cloud hosting**: recursos elásticos en la nube (AWS EC2, GCP Compute, Azure VMs).
- **Managed hosting**: proveedor administra el stack (Heroku, Vercel, Render, Railway).
- **Static hosting**: ideal para sitios estáticos y SPAs (GitHub Pages, Netlify, Cloudflare Pages).
- **Serverless / FaaS**: funciones bajo demanda (AWS Lambda, Cloudflare Workers).

### Proveedores de hosting populares

| Tipo           | Proveedores                                       |
| -------------- | ------------------------------------------------- |
| Compartido     | Bluehost, Hostinger, SiteGround                   |
| VPS            | DigitalOcean, Linode, Vultr, Hetzner              |
| Cloud          | AWS, Google Cloud, Azure, Oracle Cloud            |
| Managed PaaS   | Heroku, Render, Railway, Fly.io                   |
| Estático       | Vercel, Netlify, Cloudflare Pages, GitHub Pages   |
| Especializado  | Firebase, Supabase, PlanetScale, MongoDB Atlas    |

### Pasos para publicar un sitio

1. Registrar un dominio.
2. Contratar un hosting.
3. Configurar los **DNS records** del dominio para apuntar al hosting (A, CNAME, MX).
4. Subir los archivos por **FTP/SFTP** o hacer **deploy** desde Git.
5. Configurar **HTTPS** con Let's Encrypt o el panel del hosting.

### Actividad práctica

- Registra un dominio gratuito en `freenom.com` (o usa un subdominio gratuito de `js.org`, `netlify.app`).
- Despliega una página estática en Netlify o GitHub Pages.
- Configura un CNAME personalizado.

---

## 6. Navegadores y motores de renderizado

### ¿Qué es un navegador?

Es una aplicación que permite al usuario **acceder, recuperar y visualizar** recursos de la web. Además, ejecuta código JavaScript, gestiona almacenamiento local, plugins, etc.

### Navegadores más usados (2024-2025)

- Google Chrome (motor **Blink**)
- Safari (motor **WebKit**)
- Microsoft Edge (motor **Blink**, antes EdgeHTML)
- Mozilla Firefox (motor **Gecko**)
- Brave, Opera, Arc, Vivaldi (basados en Chromium/Blink)

### Motores de renderizado

El motor de renderizado transforma HTML, CSS y JS en píxeles en pantalla.

Etapas principales:
1. **Parsing HTML** → construye el **DOM** (Document Object Model).
2. **Parsing CSS** → construye el **CSSOM**.
3. **Layout** (reflow): calcula geometría y posición de cada nodo.
4. **Paint**: dibuja píxeles en capas.
5. **Composite**: combina las capas y las muestra en pantalla.

### Motores más relevantes

| Motor   | Usado por                       | Notas                                  |
| ------- | ------------------------------- | -------------------------------------- |
| Blink   | Chrome, Edge, Opera, Brave      | Fork de WebKit (2013)                  |
| WebKit  | Safari, navegadores iOS         | Usado también en apps nativas iOS      |
| Gecko   | Firefox                         | Desarrollado por Mozilla               |
| Goanna  | Pale Moon                       | Fork de Gecko                          |
| Servo   | Investigaciòn de Mozilla        | Escrito en Rust                        |

### Motores de JavaScript

- **V8** (Chrome, Node.js, Edge): compilador JIT, optimizaciones especulativas.
- **SpiderMonkey** (Firefox).
- **JavaScriptCore / Nitro** (Safari).
- **Chakra** (Edge legacy, deprecado).

### Compatibilidad

- **Can I use** (caniuse.com): tabla de compatibilidad de features.
- **MDN Web Docs**: documentación oficial de referencia.
- **Baseline**: iniciativa para marcar features "listos para producción" en todos los navegadores modernos.

### Actividad práctica

- Visita caniuse.com y consulta la compatibilidad de una feature como `fetch`, `import maps` o `view-transitions`.
- Compara el comportamiento de una misma página en Chrome, Firefox y Safari.

---

## 7. DevTools del navegador

Las **DevTools** son un conjunto de herramientas integradas en los navegadores para inspeccionar, depurar y perfilar aplicaciones web.

### Cómo abrirlas

- **Atajo**: F12 o `Ctrl/Cmd + Shift + I`.
- Click derecho → "Inspeccionar".
- Menú del navegador → "Más herramientas" → "Herramientas para desarrolladores".

### Paneles principales (Chrome/Edge)

#### 1. Elements / Inspector

- Muestra el **DOM** y el **CSS** aplicado.
- Permite editar en vivo HTML y estilos.
- Ver box model, pseudoclases (`:hover`, `:focus`), y herencias.
- Useful para entender selectores y resolver problemas de maquetación.

#### 2. Console

- Ejecuta JavaScript en el contexto de la página.
- Muestra `console.log`, `warn`, `error`, `info`.
- Permite inspeccionar objetos, autocompletar, y usar `$0`, `$$()`, `dir()`.

#### 3. Sources

- Depurador de JavaScript con breakpoints.
- Step over, step into, step out, watch expressions.
- "Blackboxing" de librerías externas.

#### 4. Network

- Registra **todas** las peticiones (HTML, CSS, JS, imágenes, XHR, fetch, websockets).
- Muestra método, status, tamaño, tiempo, cabeceras, cuerpo.
- Permite simular **throttling** (3G, Slow 4G) y ver waterfall.
- Replay de peticiones con cURL.

#### 5. Performance

- Graba la carga de la página o interacciones.
- Mide FPS, Long Tasks, layout shifts, scripting, rendering.
- Analiza el hilo principal y composición.

#### 6. Memory

- Snapshots del heap, perfiles de asignación.
- Detección de **memory leaks**.
- Distingue entre memoria retenida y liberada.

#### 7. Application

- **Local Storage**, **Session Storage**, **IndexedDB**, **Cookies**.
- Service Workers, Cache Storage.
- Manifest, Background Sync, Push.

#### 8. Lighthouse

- Auditorías automáticas de performance, accesibilidad, SEO, buenas prácticas.
- Genera un reporte con puntuaciones y recomendaciones.

#### 9. Security

- Certificado TLS, recursos inseguros (HTTP, mixed content).

#### 10. Sensors

- Simula geolocalización, orientación del dispositivo.

### Tips de productividad

- **Preserve log**: mantener logs entre recargas.
- **Disable cache** desde Network.
- **Device toolbar**: emular móviles y tablets.
- **Throttling**: simular conexiones lentas.
- **Coverage**: ver qué CSS/JS no se está usando.

### Firefox: características únicas

- Inspector de Grid y Flexbox muy completos.
- Vista de accesibilidad.
- Debugger de WebSocket y eventos.

### Actividad práctica

- Abre cualquier sitio, ve a la pestaña Network, recarga y filtra por `JS`. Identifica el archivo JS más pesado.
- Usa la pestaña Performance para grabar una recarga y observa el orden de eventos.
- Ejecuta `document.querySelectorAll('img').length` en la consola y compara con el HTML.

---

## 8. Estructura de una aplicación web

Una aplicación web moderna se compone de varias capas y servicios que colaboran. Comprender su estructura global es clave antes de profundizar en frontend o backend.

### Vista general

```
┌────────────────────────────────────────────────────┐
│                  Cliente (Browser)                 │
│  HTML + CSS + JS  ·  SPA / MPA  ·  Service Worker  │
└────────────────────────┬───────────────────────────┘
                         │  HTTPS / JSON / GraphQL
┌────────────────────────┴───────────────────────────┐
│                    Edge / CDN                       │
│  Cloudflare, Fastly, Akamai, Vercel Edge Network   │
└────────────────────────┬───────────────────────────┘
                         │
┌────────────────────────┴───────────────────────────┐
│                  Backend / API                      │
│  Node.js, Python, Go, Java, PHP, Ruby, .NET         │
│  REST  ·  GraphQL  ·  gRPC  ·  WebSockets          │
└────────────────────────┬───────────────────────────┘
                         │
┌────────────────────────┴───────────────────────────┐
│                  Datos                              │
│  PostgreSQL, MySQL, MongoDB, Redis, S3, BigQuery    │
└────────────────────────────────────────────────────┘
```

### Frontend

La parte que se ejecuta en el navegador del usuario.

Tecnologías:
- **HTML**: estructura semántica.
- **CSS**: estilos, layout (Flexbox, Grid), animaciones.
- **JavaScript**: lógica de interacción.
- **Frameworks**: React, Vue, Angular, Svelte, Solid.
- **Build tools**: Vite, Webpack, esbuild, Rollup.
- **Estilos**: Tailwind, Sass, CSS Modules, Styled Components.

Tipos:
- **MPA (Multi Page Application)**: cada ruta es un HTML distinto. SEO amigable.
- **SPA (Single Page Application)**: una sola página, navegación vía JS.
- **SSR (Server Side Rendering)**: el servidor renderiza HTML en cada request.
- **SSG (Static Site Generation)**: HTML pre-generado en build time.
- **Hydration**: combinar SSR con interactividad de SPA.
- **Islands architecture**: Astro, Fresh, Qwik.

### Backend

La parte que se ejecuta en el servidor y expone una API.

Responsabilidades:
- Lógica de negocio.
- Autenticación y autorización.
- Validación de datos.
- Persistencia en bases de datos.
- Integración con servicios externos.
- Manejo de errores y logs.

Patrones:
- **MVC (Model-View-Controller)**.
- **REST**: recursos + verbos HTTP.
- **GraphQL**: un solo endpoint, queries definidas por el cliente.
- **Microservicios**: pequeños servicios independientes.
- **Serverless / FaaS**: funciones efímeras por evento.
- **Event-driven**: pub/sub, Kafka, RabbitMQ.

### Base de datos

Almacenamiento persistente.

Tipos:
- **Relacionales (SQL)**: PostgreSQL, MySQL, MariaDB, SQLite.
- **NoSQL documentales**: MongoDB, CouchDB, Firestore.
- **Key-value**: Redis, DynamoDB.
- **Columnares**: Cassandra, Bigtable.
- **De búsqueda**: Elasticsearch, Meilisearch.
- **Grafos**: Neo4j.

### Capas adicionales

- **CDN (Content Delivery Network)**: cachea contenido cerca del usuario.
- **Load Balancer**: distribuye peticiones entre varios servidores.
- **Cache**: Redis, Memcached, Varnish.
- **Message broker**: Kafka, RabbitMQ, SQS, Pub/Sub.
- **Object storage**: S3, GCS, Azure Blob.
- **CI/CD**: GitHub Actions, GitLab CI, CircleCI, Jenkins.
- **Monitoring**: Datadog, New Relic, Sentry, Grafana, Prometheus.

### DevOps y despliegue

- **Contenedores**: Docker.
- **Orquestación**: Kubernetes, ECS, Nomad.
- **IaC (Infrastructure as Code)**: Terraform, Pulumi, CloudFormation.
- **Serverless platforms**: Vercel, Netlify, Cloudflare Pages, AWS Amplify.

### Ejemplo: arquitectura de un e-commerce simple

1. El usuario abre `tienda.com`.
2. Cloudflare sirve el HTML estático cacheado y un bundle JS.
3. El frontend (Next.js) llama a `/api/productos`.
4. La API (Node.js + Express) consulta PostgreSQL.
5. Redis cachea los productos más consultados.
6. La respuesta vuelve al cliente, que renderiza la lista.
7. El usuario añade al carrito → POST a `/api/carrito`.
8. El pedido se procesa mediante un job en RabbitMQ.
9. Un worker envía confirmación por email vía SendGrid.

### Actividad práctica

- Dibuja el diagrama de la arquitectura de una aplicación que uses habitualmente (Instagram, Spotify, Notion, etc.).
- Identifica frontend, backend, base de datos y servicios externos.

---

## Recursos complementarios

### Lecturas

- [MDN Web Docs - Cómo funciona la Web](https://developer.mozilla.org/es/docs/Learn/Common_questions/Web_mechanics/How_does_the_Internet_work)
- [HTTP - MDN](https://developer.mozilla.org/es/docs/Web/HTTP)
- [High Performance Browser Networking - Ilya Grigorik](https://hpbn.co/)

### Herramientas

- [Can I use](https://caniuse.com/)
- [WebPageTest](https://www.webpagetest.org/)
- [SSL Labs](https://www.ssllabs.com/ssltest/)
- [DNS Checker](https://dnschecker.org/)

### Videos recomendados

- "¿Cómo funciona Internet?" - Computerphile.
- "HTTP/3 and QUIC" - Hussein Nasser.
- Serie "HTTP" de Hussein Nasser en YouTube.

---

## Evaluación sugerida

1. **Cuestionario teórico (20%)**: preguntas sobre DNS, HTTP, modelo cliente-servidor.
2. **Laboratorio DevTools (25%)**: entregar capturas y análisis de una página analizada.
3. **Despliegue de sitio estático (25%)**: registrar dominio/subdominio, desplegar página y configurar HTTPS.
4. **Diagrama de arquitectura (30%)**: explicar la estructura de una aplicación real.

---

> **Nota para el estudiante**: este módulo es la base conceptual sobre la que se apoyan todos los demás. Antes de escribir tu primera línea de HTML, asegúrate de entender cómo viaja realmente la información de un punto a otro.
