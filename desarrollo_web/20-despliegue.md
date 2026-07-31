# Módulo 20: Despliegue

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Desplegar aplicaciones en un VPS con Nginx y PM2.
- Configurar HTTPS con Let's Encrypt y Cloudflare.
- Conectar dominios y configurar DNS.
- Publicar en plataformas modernas (Render, Railway, Vercel, Netlify).
- Comparar opciones y elegir la mejor según el caso.

---

## 1. VPS (Servidor Privado Virtual)

Un **VPS** es una máquina virtual dedicada a tu proyecto, con root completo y recursos garantizados.

### Proveedores populares

- **DigitalOcean** (Droplets).
- **Linode** (Akamai).
- **Hetzner** (Falkenstein, Helsinki, Ashburn).
- **Vultr**.
- **OVH**.
- **Scaleway**.
- **AWS Lightsail** / **EC2**.
- **Google Compute Engine**.

### Tamaño típico inicial

- **Static frontend**: 1 vCPU, 1 GB RAM, 25 GB SSD.
- **Backend Node.js pequeño**: 1 vCPU, 2 GB RAM.
- **Stack completo**: 2 vCPU, 4 GB RAM.

### Primer acceso SSH

```bash
ssh root@TU_IP
```

### Configuración inicial

```bash
apt update && apt upgrade -y

# Crear usuario no root
adduser deploy
usermod -aG sudo deploy
```

### Configurar SSH seguro

```bash
nano /etc/ssh/sshd_config
```

```ini
PermitRootLogin no
PasswordAuthentication no
Port 2222   # cambiar default
```

```bash
# Autenticación con clave pública
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys  # pegar clave pública
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

systemctl restart ssh
```

### Firewall

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### Fail2ban (protección de fuerza bruta)

```bash
apt install fail2ban
systemctl enable fail2ban
```

### Actualizaciones automáticas

```bash
apt install unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

### Swap

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Actividad práctica

- Crea un droplet/VM Ubuntu, agrega tu SSH key y deshabilita root login.

---

## 2. Nginx (servidor web y reverse proxy)

Nginx es uno de los servidores web más usados. Sirve estáticos, hace de reverse proxy y maneja HTTPS.

### Instalación

```bash
apt install nginx
systemctl enable nginx
systemctl start nginx
```

### Estructura

```
/etc/nginx/
├── nginx.conf
├── sites-available/
│   └── app.conf
├── sites-enabled/
│   └── app.conf (symlink)
└── conf.d/
```

### Server block (SPA)

```nginx
server {
  listen 80;
  server_name app.example.com;
  root /var/www/app/dist;
  index index.html;

  gzip on;
  gzip_types text/css application/javascript application/json image/svg+xml;

  location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }

  location / {
    add_header Cache-Control "no-cache";
    try_files $uri $uri/ /index.html;
  }
}
```

### Reverse proxy a Node

```nginx
server {
  listen 80;
  server_name api.example.com;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_cache_bypass $http_upgrade;
    proxy_read_timeout 60s;
  }
}
```

### Activar sitio

```bash
ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### Logs

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### Múltiples apps en una sola IP

```nginx
server { listen 80; server_name a.com; ... }
server { listen 80; server_name b.com; ... }
```

O distinguir por path:

```nginx
location /api/ {
  proxy_pass http://localhost:4000;
}
location / {
  try_files $uri $uri/ /index.html;
}
```

### WebSockets

```nginx
location /socket.io/ {
  proxy_pass http://127.0.0.1:3000;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
```

### Rate limiting

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/auth/login {
  limit_req zone=api burst=5 nodelay;
  proxy_pass http://localhost:4000;
}
```

### Limit buffer para uploads grandes

```nginx
client_max_body_size 10M;
```

### Actividad práctica

- Configura nginx para servir tu frontend y hacer proxy al backend en el mismo servidor.

---

## 3. PM2 (process manager para Node)

PM2 es un gestor de procesos para Node.js en producción: reinicio automático, logs, monitorización.

### Instalación

```bash
npm install -g pm2
```

### Comandos básicos

```bash
pm2 start app.js --name "mi-api"
pm2 start npm --name "mi-app" -- start
pm2 list
pm2 monit
pm2 logs mi-api
pm2 logs mi-api --lines 200
pm2 restart mi-api
pm2 reload mi-api        # cluster: rolling reload
pm2 stop mi-api
pm2 delete mi-api
```

### Ecosystem file (`ecosystem.config.js`)

```js
module.exports = {
  apps: [{
    name: "mi-api",
    script: "src/index.js",
    instances: 2,          // o "max"
    exec_mode: "cluster",
    env: {
      NODE_ENV: "production",
      PORT: 4000,
    },
    max_memory_restart: "500M",
    wait_ready: true,
    listen_timeout: 10000,
    kill_timeout: 5000,
  }],
};
```

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup         # configura systemd para iniciar en boot
```

### Logs con rotación

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
```

### Cluster mode

```js
// apps: [{ exec_mode: "cluster", instances: "max" }]
```

PM2 usa `cluster` para forkear procesos; con load balancing interno.

### Watch & reload

```bash
pm2 start app.js --watch   # reinicia ante cambios
```

> Solo en dev, no en producción.

### Deploy con pm2

```bash
# ecosystem.config.js
deploy: {
  production: {
    user: "deploy",
    host: "1.2.3.4",
    ref: "origin/main",
    repo: "git@github.com:org/repo.git",
    path: "/var/www/app",
    "post-deploy": "npm ci && npm run build && pm2 reload ecosystem.config.js",
    env: { NODE_ENV: "production" },
  },
}
```

```bash
pm2 deploy production
```

(Requiere SSH key configurado.)

### Actividad práctica

- Despliega una app Node con PM2 en cluster mode y configúralo para iniciar en boot.

---

## 4. SSL/HTTPS con Let's Encrypt

HTTPS es **obligatorio** en producción (seguridad, SEO, HTTP/2, Service Workers, geolocalización).

### Instalación de Certbot

```bash
apt install certbot python3-certbot-nginx
```

### Generar certificado

```bash
certbot --nginx -d app.example.com -d www.app.example.com
```

Certbot modifica `nginx.conf` automáticamente para redirigir HTTP a HTTPS.

### Renovación

Certbot instala un cron que renueva automáticamente cada 60 días.

```bash
certbot renew --dry-run
systemctl list-timers | grep certbot
```

### Tipos de certificados

- **DV (Domain Validated)**: Let's Encrypt, gratuito.
- **OV (Organization Validated)**: de pago, valida empresa.
- **EV (Extended Validation)**: máxima confianza.

### Wildcard

```bash
certbot certonly --nginx -d "*.example.com" -d example.com --dns-cloudflare
```

Requiere DNS-01 challenge (proveedor con API).

### TLS moderno

Solo TLS 1.2 y 1.3:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:...;
ssl_prefer_server_ciphers on;
```

### HSTS

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### OCSP Stapling

```nginx
ssl_stapling on;
ssl_stapling_verify on;
```

### SSL en Node directo (sin nginx)

```bash
sudo certbot certonly --standalone -d api.example.com
```

```js
const fs = require("node:fs");
const https = require("node:https");

const server = https.createServer({
  cert: fs.readFileSync("/etc/letsencrypt/live/api.example.com/fullchain.pem"),
  key:  fs.readFileSync("/etc/letsencrypt/live/api.example.com/privkey.pem"),
}, app);
```

### Actividad práctica

- Configura Let's Encrypt para tu dominio y verifica HSTS.

---

## 5. Dominio y DNS

### Registrar dominio

- **Namecheap**, **GoDaddy**, **Google Domains**, **Porkbun**, **Cloudflare Registrar**.

### Tipos de registros DNS

| Registro | Uso                                | Ejemplo                          |
| -------- | ---------------------------------- | -------------------------------- |
| A        | Dominio → IPv4                     | `app.com → 1.2.3.4`             |
| AAAA     | Dominio → IPv6                     | `app.com → ::1`                 |
| CNAME    | Alias                              | `www → app.com`                 |
| MX       | Correo                             | `mail → mail.app.com`           |
| TXT      | SPF, DKIM, verification           | `v=spf1 include:_spf.google.com` |
| NS       | Servidores de nombres autoritativos | `ns1.cloudflare.com`            |

### TTL

- **Bajo (300s)**: cambio rápido pero más consultas a root.
- **Alto (3600+ o 86400)**: estable pero tarda en propagar.

### CDN y Cloudflare

Al usar Cloudflare:

- Cambiar NS a los de Cloudflare.
- La IP origen es "oculta".
- Tráfico pasa por edge.

### HTTPS con CDN

```
Visitante → Cloudflare (TLS) → Origin (TLS o HTTP con Full/Full Strict)
```

Modos:

- **Full (Strict)**: Cloudflare exige HTTPS válido en origin.
- **Full**: HTTPS pero acepta cert inválido.
- **Flexible**: HTTPS al cliente pero HTTP al origin (inseguro).

### Para Cloudflare + origen HTTP

```nginx
server {
  listen 8080;
  location / { proxy_pass http://localhost:3000; }
}
```

### Actividad práctica

- Configura un dominio, apunta a tu VPS y verifica DNS con `dig` + `whatsmydns.net`.

---

## 6. Cloudflare

Cloudflare ofrece CDN, seguridad, DNS y opcionalmente Functions, KV, R2.

### Características

- **CDN global** en +200 ciudades.
- **DDoS protection** automático.
- **SSL universal**.
- **DNS** ultrarrápido.
- **Workers** (Edge functions en JS, Rust, Python).
- **Pages** (static hosting).
- **R2** (object storage sin egress).
- **D1** (SQLite distribuido).
- **KV** (key-value).
- **Turnstile** (captcha invisible).

### Configuración básica

1. Crear cuenta.
2. Añadir sitio.
3. Cambiar NS.
4. Activar proxy (nube naranja) por DNS.

### Page Rules / Configuración de cache

```
ejemplo.com/assets/* → Cache Level: Cache Everything, Edge TTL: 1 month
```

### Transform Rules

Modificar request/response (URL rewrite, headers).

### Workers

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname.startsWith("/api/")) return fetch("https://api.origin.com" + url.pathname, request);
    return env.ASSETS.fetch(request);
  },
};
```

### Tunnels (Cloudflare Tunnel)

Expón un servidor local sin abrir puertos:

```bash
cloudflared tunnel --url http://localhost:3000
```

Útil para demos sin configurar DNS.

### Actividad práctica

- Configura Cloudflare delante de tu VPS y mide latencia antes/después con `curl -w "%{time_total}\n"`.

---

## 7. Render

Render es un PaaS simple (Heroku-style) con servicios web, estáticos, PostgreSQL y más.

### Tipos de servicios

- **Web Service**: Node, Python, Go, etc.
- **Static Site**: html/css/js.
- **Cron Job**: tareas programadas.
- **Worker**: procesos background.
- **Postgres**: DB gestionada.
- **Redis**: cache gestionada.
- **Private Service**: red privada.

### Desplegar

1. Conectar repo GitHub.
2. Elegir branch (`main`).
3. Build command: `npm install && npm run build`.
4. Start command: `npm start` o `node dist/index.js`.
5. Plan (free, starter, etc).

### Variables de entorno

```
Environment → Environment Variables
```

### `render.yaml`

```yaml
services:
  - type: web
    name: mi-api
    runtime: node
    buildCommand: npm install && npm run build
    startCommand: node dist/index.js
    envVars:
      - key: DATABASE_URL
        fromService: { type: postgres, name: mi-db, property: connectionString }
      - key: JWT_SECRET
        generateValue: true
  - type: static
    name: mi-app
    buildCommand: npm run build
    staticPublishPath: dist
    routes:
      - type: rewrite
        source: /*
        destination: /index.html
databases:
  - name: mi-db
    plan: starter
```

### CLI

```bash
npm install -g @render/cli
render services list
render deploy
```

### Ventajas

- Setup rápido.
- HTTPS y dominio gratis.
- Deploy de previews.
- Buildpacks varios.

### Limitaciones

- Free tier duerme tras inactividad (cold start).
- Menos flexible que Railway/Fly.

### Actividad práctica

- Desplegar una app Node.js + Postgres en Render usando `render.yaml`.

---

## 8. Railway

Railway es un PaaS centrado en simplicidad, integraciones y DX.

### Setup

1. Conectar GitHub.
2. Elegir repo.
3. Railway detecta el stack (Node, etc.).
4. Configurar build/start commands.
5. Variables de entorno.

### CLI

```bash
npm install -g @railway/cli
railway login
railway init
railway up
railway open
```

### Servicios típicos

- App Node/Go/Python.
- Postgres, Redis, MongoDB.
- Workers (cron).
- Networking entre servicios.

### Variables entre servicios

```yaml
# railway.json o dashboard
DATABASE_URL=${{ Postgres.DATABASE_URL }}
REDIS_URL=${{ Redis.REDIS_URL }}
```

### Pricing

- Por uso (CPU, RAM).
- Trial inicial ($5).

### Ventajas

- DX excelente.
- Provisiona DB en clicks.
- Networking privada entre servicios.

### Actividad práctica

- Desplegar una API Node + Postgres en Railway y conectar las variables.

---

## 9. Vercel (frontend / Next.js)

Vercel es el creador de Next.js. Optimizado para frontend, serverless y edge.

### Características

- **Static + SSR + ISR**.
- Edge network global (40+ regiones).
- Automatic HTTPS.
- Preview deployments por PR.
- `vercel.json` config.
- Analytics integrado.
- Soporta frameworks: Next.js, SvelteKit, Nuxt, Remix, Astro, Vite + React.

### Deploy

```bash
npm i -g vercel
vercel                  # preview deploy
vercel --prod           # producción
```

### `vercel.json`

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "framework": "vite",
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "SAMEORIGIN" }
      ]
    },
    {
      "source": "/assets/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

### Variables de entorno

- En dashboard.
- Por ambiente (Production, Preview, Development).

### Edge Functions / Middleware

```ts
// middleware.ts
export const config = { matcher: "/api/:path*" };

export default async function middleware(req) {
  const token = req.cookies.get("token");
  if (!token) return NextResponse.redirect("/login");
}
```

### Cron Jobs

```json
{
  "crons": [{ "path": "/api/cleanup", "schedule": "0 2 * * *" }]
}
```

### Limitaciones

- Funciones con timeout corto (10s en hobby, 60s en pro).
- Cold starts.

### Actividad práctica

- Desplegar una SPA con Vercel y configurar custom domain.

---

## 10. Netlify

Pionero en JAMstack, similar a Vercel con su propio enfoque.

### Características

- **Static + SSR + Functions**.
- Edge Functions (Deno-based).
- Forms serverless.
- Identity (auth) integrado.
- Deploy previews.
- Plugin ecosystem.

### Deploy

```bash
npm i -g netlify-cli
netlify init
netlify deploy --prod
```

### `netlify.toml`

```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to   = "/index.html"
  status = 200

[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "SAMEORIGIN"
    X-Content-Type-Options = "nosniff"
```

### Functions

```js
// netlify/functions/hello.js
exports.handler = async () => ({
  statusCode: 200,
  body: JSON.stringify({ hello: "world" }),
});
```

### Edge Functions (Deno)

```ts
// netlify/edge-functions/redirect.ts
export default async (req: Request) => {
  if (new URL(req.url).pathname === "/old") {
    return Response.redirect(new URL("/new", req.url), 301);
  }
};
```

```toml
[[edge_functions]]
  function = "redirect"
  path = "/*"
```

### Identity (auth built-in)

```js
netlify identity.on("login", (user) => console.log(user));
netlify identity.gotUser((user) => ...);
```

Útil para CMS y prototipos.

### Pricing

- Free: 100 GB bandwidth.
- Pro: $19/mes.

### Actividad práctica

- Desplegar SPA + una serverless function en Netlify.

---

## Comparativa de plataformas

| Característica | VPS + Nginx | Render | Railway | Vercel | Netlify |
| -------------- | ----------- | ------ | ------- | ------ | ------- |
| Costo inicial | Bajo-medio  | Free tier | Trial | Free tier | Free tier |
| Control root  | **Sí**      | No     | No      | No     | No      |
| HTTPS         | Let's Encrypt | Auto | Auto    | Auto   | Auto    |
| DB incluida   | Manual      | Sí     | Sí      | Sí (Neon/Supabase) | No |
| Static        | Sí          | Sí     | Sí      | **Excelente** | **Excelente** |
| SSR/Edge      | Manual      | Sí     | Limitado | **Excelente** | Sí |
| Cron jobs     | Manual      | Sí     | Sí      | Sí (Pro) | Sí (Functions scheduled) |
| Cold starts   | No          | Sí     | No      | Sí     | Sí |
| Custom domain | Manual      | Auto   | Auto    | Auto   | Auto |
| Logs/Monitor  | Manual      | Sí     | Sí      | Sí     | Sí |

### Cuándo usar cada uno

- **VPS + Nginx + PM2**: control total, apps pesadas, costos fijos.
- **Render**: simple, multi-lenguaje, DB fácil.
- **Railway**: DX simple, integración con servicios.
- **Vercel**: Next.js, edge runtime, JAMstack.
- **Netlify**: forms, identity, CDN rápido.

---

## Despliegue completo (paso a paso)

### Frontend estático

1. `npm run build` genera `dist/`.
2. Subir a:
   - S3 + CloudFront.
   - Vercel/Netlify/Cloudflare Pages.
   - VPS con Nginx.

### Backend Node

1. Build: `npm run build` (transpila TS) o `npm install --production`.
2. Variables de entorno en plataforma.
3. DB: provisionada y `DATABASE_URL` configurada.
4. Migraciones: ejecutar tras deploy.
5. HTTPS + dominio.

### Fullstack (Next.js / SvelteKit / Nuxt)

1. Detección automática por la plataforma.
2. Variable de entorno pública para backend (si separado).
3. Deploy.

### Healthcheck

```js
app.get("/healthz", (req, res) => res.json({ status: "ok" }));
```

Activar en plataforma para monitorización.

### Post-deploy checklist

- [ ] HTTPS válido (A+ en SSL Labs).
- [ ] Cabeceras de seguridad activas (CSP, HSTS).
- [ ] Logs centralizados.
- [ ] Backups de DB.
- [ ] Alertas (uptime, errores 5xx, latencia).
- [ ] Rollback probado.

---

## Buenas prácticas

### Seguridad VPS

- SSH por clave, no password.
- 2FA en plataformas.
- Fail2ban.
- Actualizaciones automáticas.
- Backups offsite.

### Resiliencia

- **Multi-región** para producción seria.
- **Database replicas** y backups.
- **Graceful shutdown** en Node (`SIGTERM` → cerrar conexiones).

### Observabilidad

- Logs estructurados.
- APM (Application Performance Monitoring).
- Uptime monitoring (UptimeRobot, BetterStack).
- Alertas a Slack/Discord.

### Rollback

En PaaS, cada deploy es una "release". Rollback con un click.

### DB Migrations

- **Expand → contract**: no elimines columnas en producción en el mismo deploy.
- Versionar migraciones.
- Probar rollback en staging.

### Backups

```bash
# Postgres
pg_dump dbname > backup.sql
aws s3 cp backup.sql s3://backups/db/$(date +%F).sql
```

```bash
# Cron
0 3 * * * /usr/local/bin/backup.sh
```

### Actividad práctica

- Crea un runbook de despliegue con pasos para VPS y otro para PaaS.

---

## Recursos complementarios

### Lecturas

- [nginx documentation](https://nginx.org/en/docs/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Cloudflare Docs](https://developers.cloudflare.com/)
- [Render Docs](https://docs.render.com/)
- [Railway Docs](https://docs.railway.app/)
- [Vercel Docs](https://vercel.com/docs)
- [Netlify Docs](https://docs.netlify.com/)

### Herramientas

- **MobaXterm**, **Termius**: SSH clients.
- **Hetzner**, **DigitalOcean**: VPS económicos.
- **StatusCake**, **UptimeRobot**: monitoring.
- **Sentry**, **Logtail**, **Better Stack**: errores y logs.
- **Tailscale**, **Cloudflare Tunnel**: acceso seguro.

### Cursos

- [Nginx fundamentals](https://www.youtube.com/results?search_query=nginx+basics)
- [DevOps with Docker](https://www.youtube.com/results?search_query=docker+for+developers)

---

## Evaluación sugerida

1. **VPS (25%)**: configurar servidor, instalar nginx, PM2, certificados.
2. **DNS + HTTPS (25%)**: dominio personalizado + Let's Encrypt + Cloudflare.
3. **PaaS (25%)**: desplegar en Vercel + Render y comparar tiempos.
4. **Buenas prácticas (25%)**: monitoring, backups, graceful shutdown, rollback.

---

> **Nota para el estudiante**: elegir plataforma es un **trade-off entre control y conveniencia**. Empieza con PaaS (Vercel, Render) para validar ideas. Cuando necesites control, escala a VPS. Y siempre: **HTTPS desde el día 1**.
