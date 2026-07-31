# Módulo 19: DevOps para Frontend

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Gestionar variables de entorno de forma segura.
- Configurar Vite para builds de producción.
- Aplicar optimizaciones de bundles, imágenes y cacheo.
- Empaquetar aplicaciones frontend con Docker.
- Definir pipelines CI/CD con GitHub Actions.
- Desplegar en diferentes plataformas (Vercel, Netlify, S3+CloudFront, etc.).

---

## 1. Variables de entorno

Las **variables de entorno** permiten separar la configuración del código (URLs de API, claves, flags por entorno).

### `.env`

```env
VITE_API_URL=https://api.example.com
VITE_SENTRY_DSN=https://...
VITE_ANALYTICS_ID=UA-12345
```

### `import.meta.env`

Vite expone **solo** las variables prefijadas con `VITE_` por seguridad.

```js
const apiUrl = import.meta.env.VITE_API_URL;
const mode   = import.meta.env.MODE;        // "development" | "production"
const isDev  = import.meta.env.DEV;
const isProd = import.meta.env.PROD;
const base   = import.meta.env.BASE_URL;
```

### `.env` por entorno

```
.env                # general (también se usa si no hay específico)
.env.local          # local, NO se commitea
.env.development    # solo en dev
.env.production     # solo en prod
```

### Buenas prácticas

- `.env.local` para secretos: añadir a `.gitignore`.
- `.env.example` documentado en el repo.
- Prefijo único del proyecto: `VITE_`, `NEXT_PUBLIC_`, `REACT_APP_`.
- **Nunca** subir `.env` con secretos reales al repo.

### `.gitignore`

```
.env
.env.local
.env.*.local
```

### `.env.example`

```env
VITE_API_URL=http://localhost:4000
VITE_SENTRY_DSN=
VITE_ANALYTICS_ID=
```

### TypeScript y tipado de env

```ts
// src/env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_SENTRY_DSN?: string;
  readonly VITE_ANALYTICS_ID?: string;
  readonly MODE: "development" | "production";
  readonly DEV: boolean;
  readonly PROD: boolean;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### Runtime configuration

Para apps con configuración que cambia **sin redeploy** (feature flags, claves que rotan):

- Endpoint `/api/config` al iniciar la app.
- Caché local con TTL.
- Mezcla: build-time defaults + runtime overrides.

### Actividad práctica

- Configura `.env.development` y `.env.production` con una variable `VITE_API_URL` distinta en cada uno.

---

## 2. Vite

**Vite** es la herramienta de build moderna basada en ESBuild y Rollup. Reemplaza a Webpack en la mayoría de proyectos.

### Scripts

```json
{
  "scripts": {
    "dev":      "vite",
    "build":    "tsc -b && vite build",
    "preview":  "vite preview --port 4173"
  }
}
```

### Estructura

```
project/
├── public/        # estáticos sin procesar
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   └── ...
├── index.html
├── vite.config.ts
└── package.json
```

### `vite.config.ts`

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { "@": "/src" },
  },
  server: {
    port: 5173,
    proxy: {
      "/api": "http://localhost:4000",
    },
  },
  preview: {
    port: 4173,
  },
  build: {
    outDir: "dist",
    sourcemap: true,
    target: "es2022",
    cssCodeSplit: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          router: ["react-router-dom"],
        },
      },
    },
  },
});
```

### Plugins comunes

- `@vitejs/plugin-react` / `react-swc`.
- `@vitejs/plugin-vue`.
- `vite-plugin-pwa` (service worker).
- `vite-plugin-checker` (TS check).
- `vite-plugin-svgr` (SVG como componentes).
- `vite-plugin-compression` (gzip/brotli).
- `unplugin-icons` / `unplugin-vue-components`.

### `index.html`

```html
<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mi App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### HMR

Vite incluye HMR (Hot Module Replacement) muy rápido vía ESBuild.

### Variables de build en Vite

```ts
define: {
  __API_URL__: JSON.stringify(process.env.VITE_API_URL),
}
```

### Asset handling

```jsx
import imgUrl from "./logo.png?url";
import Logo from "./logo.svg?react";

<img src={imgUrl} alt="" />
<Logo />
```

### HTTPS local

```ts
server: { https: true, host: true } // genera cert autofirmado
```

### Actividad práctica

- Configura Vite con proxy, manual chunks y alias `@/*`.

---

## 3. Build de producción

### `vite build`

Genera `dist/` con HTML, JS, CSS y assets optimizados.

```
dist/
├── assets/
│   ├── index-abc123.js
│   ├── index-def456.css
│   └── logo.svg
├── favicon.svg
└── index.html
```

### Opciones de build

```ts
build: {
  outDir: "dist",
  assetsDir: "assets",
  sourcemap: "hidden", // no expuestas por defecto
  cssCodeSplit: true,
  chunkSizeWarningLimit: 1000,
  minify: "esbuild", // "terser" más lento, mejor ratio
  target: "es2020",
  rollupOptions: { ... },
}
```

### Minificación

- **Terser**: tradicional.
- **esbuild**: mucho más rápido (default en Vite).
- **SWC**: muy rápido.

### Tree shaking

Vite + Rollup eliminan exports no usados automáticamente.

```js
// utils.js
export function used() { return "x"; }
export function unused() { return "y"; }

// Solo `used` queda en el bundle.
```

### Hash de archivos

Cada asset tiene un hash en el nombre (`index-abc123.js`). Permite **cache infinito** porque al cambiar el contenido cambia el hash.

### Tipos de target

```ts
build: {
  target: {
    chrome: "> 100",
    firefox: "> 100",
    safari: "> 14",
  },
}
```

### Análisis de bundle

```bash
npm install rollup-plugin-visualizer
```

```ts
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    visualizer({ open: true, gzipSize: true }),
  ],
});
```

```bash
npm run build
```

Genera `dist/stats.html` con un sunburst.

### Bundlephobia (online)

Para evaluar dependencias antes de añadir.

### Actividad práctica

- Analiza el bundle de tu proyecto con `rollup-plugin-visualizer` y aplica optimizations.

---

## 4. Optimización

### Performance web (Core Web Vitals)

- **LCP** (Largest Contentful Paint) < 2.5s.
- **FID/INP** (Interaction to Next Paint) < 200ms.
- **CLS** (Cumulative Layout Shift) < 0.1.

### Reducción de tamaño

- Code splitting.
- Tree shaking.
- Compresión (gzip/brotli).
- Imágenes WebP/AVIF.
- Tree de fuentes subset.

### Compresión

```ts
import compress from "vite-plugin-compression";

plugins: [
  compress({ algorithm: "gzip", ext: ".gz" }),
  compress({ algorithm: "brotliCompress", ext: ".br" }),
]
```

Activar en servidor/edge para servir `.br` o `.gz` con `Content-Encoding: br`.

### Code splitting

```jsx
const ProductList = lazy(() => import("./ProductList"));
const Profile = lazy(() => import("./Profile"));

<Suspense fallback={<Spinner />}>
  <Routes>
    <Route path="/products" element={<ProductList />} />
    <Route path="/profile" element={<Profile />} />
  </Routes>
</Suspense>
```

### Preload / Prefetch

```html
<link rel="preload"  href="critical.js"  as="script" crossorigin>
<link rel="prefetch" href="next-page.js" as="script">
```

En SPA:

```tsx
import { Link } from "react-router-dom";

const PrefetchLink = ({ to, children }) => (
  <Link to={to} onMouseEnter={() => import(to)}>
    {children}
  </Link>
);
```

### Dynamic imports por feature

```js
// Solo cargar Chart cuando se necesita
const Chart = lazy(() => import("./Chart"));
```

### Eliminación de runtime checks en build

```ts
build: {
  minify: "esbuild",
  define: {
    "process.env.NODE_ENV": JSON.stringify("production"),
  },
}
```

### Imágenes y assets

```tsx
// Next.js style (también funciona en Vite con <picture>)
<picture>
  <source srcSet="img.avif" type="image/avif" />
  <source srcSet="img.webp" type="image/webp" />
  <img  src="img.jpg" alt="..." loading="lazy" decoding="async" width="..." height="..." />
</picture>
```

### Prerendering (SSG)

```bash
npm install vite-plugin-prerender
```

```ts
import { prerender } from "vite-plugin-prerender";

plugins: [prerender({ routes: ["/", "/about", "/blog"] })];
```

### Service Worker (PWA)

```bash
npm install vite-plugin-pwa
```

```ts
import { VitePWA } from "vite-plugin-pwa";

plugins: [
  VitePWA({
    registerType: "autoUpdate",
    includeAssets: ["favicon.svg"],
    manifest: {
      name: "Mi App",
      short_name: "App",
      theme_color: "#0a7d54",
      icons: [
        { src: "icon-192.png", sizes: "192x192", type: "image/png" },
        { src: "icon-512.png", sizes: "512x512", type: "image/png" },
      ],
    },
  }),
];
```

### Caché HTTP

`Cache-Control: public, max-age=31536000, immutable` para archivos con hash.
`Cache-Control: no-cache` para `index.html` (revalidate en origen).

### Critical CSS inline

Insertar CSS necesario en `<head>` y diferir el resto.

### Lighthouse

```bash
npm install -g lighthouse
lighthouse http://localhost:4173 --view
```

### Web Vitals (medición en producción)

```bash
npm install web-vitals
```

```ts
import { onCLS, onINP, onLCP } from "web-vitals";
onLCP(console.log);
onCLS(console.log);
onINP(console.log);
```

### Actividad práctica

- Aplica al menos 3 optimizaciones (compression, code splitting, PWA) y mide Lighthouse antes/después.

---

## 5. Docker para Frontend

Empaquetar la app frontend en una imagen para correr en cualquier entorno.

### `Dockerfile` multi-stage

```dockerfile
# Etapa 1: build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Etapa 2: servir con nginx
FROM nginx:1.27-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### `.dockerignore`

```
node_modules
dist
.git
.gitignore
.env*
Dockerfile
docker-compose.yml
.vscode
.idea
coverage
```

### `nginx.conf`

```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  # Compression
  gzip on;
  gzip_types text/css application/javascript application/json image/svg+xml;
  gzip_min_length 1024;

  # Cache for hashed assets
  location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }

  # SPA fallback
  location / {
    add_header Cache-Control "no-cache";
    try_files $uri $uri/ /index.html;
  }

  # Security headers
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header Strict-Transport-Security "max-age=63072000" always;
}
```

### Variables de entorno en runtime

Vite **sustituye** las variables en **build time**. Para runtime:

#### Build-time (default)

```env
VITE_API_URL=https://api.prod.com
npm run build  # queda "hardcodeado" en el bundle
```

#### Runtime con script

```html
<script>
  window.__CONFIG__ = {
    API_URL: "https://api.prod.com",
  };
</script>
```

```ts
const config = (window as any).__CONFIG__;
const apiUrl = config?.API_URL ?? import.meta.env.VITE_API_URL;
```

#### Con container env (Docker)

```html
<script>
  window.__CONFIG__ = {
    API_URL: "${API_URL}",
  };
</script>
```

Sustituir en runtime con `envsubst` o entrypoint script:

```dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

```sh
#!/bin/sh
set -e
envsubst < /usr/share/nginx/html/config.template.js \
  > /usr/share/nginx/html/config.js
exec nginx -g "daemon off;"
```

### Optimización de imagen

```dockerfile
FROM node:20-alpine
# Multi-stage minimiza tamaño final
```

- `node:20-alpine` (~180 MB) vs nginx final (~50 MB).
- BuildKit con cache.

### Healthcheck

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1
```

### Actividad práctica

- Crea un `Dockerfile` multi-stage para tu app Vite y un `nginx.conf` con cache.

---

## 6. Docker Compose

Para correr frontend + backend + BD localmente.

### `docker-compose.yml`

```yaml
version: "3.9"

services:
  app:
    build: .
    ports:
      - "5173:80"
    environment:
      - API_URL=http://localhost:4000
    depends_on:
      - api

  api:
    build: ./backend
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/app
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  db_data:
```

### Comandos

```bash
docker compose up -d
docker compose logs -f app
docker compose down -v
docker compose exec api npm run db:migrate
```

### Diferentes compose por entorno

```
docker-compose.yml         # base
docker-compose.dev.yml     # dev con hot reload
docker-compose.prod.yml    # prod
```

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

### `.env` para compose

```env
POSTGRES_USER=user
POSTGRES_PASSWORD=pass
```

### Volúmenes para desarrollo

```yaml
volumes:
  - ./:/app
  - /app/node_modules
```

Permite editar el código sin rebuild.

### Actividad práctica

- Levanta una app frontend + backend + postgres + redis con docker compose.

---

## 7. CI/CD (General)

Pipeline que automatiza integración, testing y despliegue.

### Fases

```
Lint → Test → Build → E2E → Deploy
```

### GitHub Actions

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run typecheck

      - name: Tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: "npm" }

      - run: npm ci
      - run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist
```

### Branch protection

Settings → Branches → Branch protection rules → `main`:

- Requerir PR review.
- Requerir checks: `test`, `build`.
- No permitir force push.

### Environments y secrets

```
Settings → Secrets and variables → Actions
```

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://app.example.com
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

### Matrix testing

```yaml
strategy:
  matrix:
    node: [18, 20, 22]
```

### Deploy condicional

```yaml
on:
  push:
    branches: [main]
```

Solo `main` deploya. PRs solo ejecutan tests.

### Cache efectivo

```yaml
- uses: actions/setup-node@v4
  with:
    cache: "npm"
```

Cachea `~/.npm`. Para Vite/Rollup también:

```yaml
- uses: actions/cache@v4
  with:
    path: |
      node_modules
      .vite
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### Actividad práctica

- Crea un workflow que ejecute lint + test + build y falle si alguno falla.

---

## 8. Deploy (despliegue)

### Vercel

- Push a main → redeploy.
- HTTPS automático.
- Preview deployments por PR.
- Edge network global.

```bash
npm install -g vercel
vercel
```

### Netlify

```bash
npm install -g netlify-cli
netlify init
netlify deploy --prod
```

`netlify.toml`:

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
```

### GitHub Pages

```yaml
# .github/workflows/pages.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: "npm" }
      - run: npm ci
      - run: npm run build
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with: { path: ./dist }
      - uses: actions/deploy-pages@v3
```

### Cloudflare Pages

- Conecta repositorio.
- Build command: `npm run build`.
- Output dir: `dist`.
- Variables de entorno configurables.

### S3 + CloudFront (manual)

```bash
aws s3 sync dist/ s3://my-bucket --delete
aws cloudfront create-invalidation --distribution-id xxx --paths "/*"
```

### Firebase Hosting

```bash
npm install -g firebase-tools
firebase init hosting
firebase deploy
```

### Surge

```bash
npm install -g surge
surge dist/ myapp.surge.sh
```

### Static app vs SSR

Este módulo cubre **static apps** (SPA). Para SSR (Next.js, Remix, etc.) el deploy es similar pero el contenedor necesita Node.

### Deploy Hooks

Algunos servicios proveen webhooks para disparar builds manualmente.

### Preview Deployments

- Vercel: cada PR tiene una URL única.
- Netlify: igual.

### Canary/Blue-green

Estrategias de release: nuevo servidor en paralelo, redirigir tráfico gradualmente.

### Post-deploy verifications

- Smoke tests contra la URL real.
- Sentry/Datadog reportando errores.
- Lighthouse CI.
- Synthetic monitoring.

### Actividad práctica

- Despliega una app en Vercel/Netlify con un deploy preview en una PR.

---

## Pipeline completo para frontend

```
PR → Lint + Test + Build → Preview deploy
Merge to main → Deploy production
```

### Workflow completo

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: "npm" }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: "npm" }
      - run: npm ci
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v4

  e2e:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run build
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: playwright-report, path: playwright-report }

  docker:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myorg/frontend:${{ github.sha }} .
      - run: docker push myorg/frontend:${{ github.sha }}

  deploy:
    if: github.ref == 'refs/heads/main'
    needs: [lint, test, e2e, docker]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET }}
          aws-region: us-east-1
      - run: |
          aws s3 sync dist/ s3://my-bucket --delete
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CF_ID }} --paths "/*"
```

---

## Buenas prácticas de DevOps para frontend

### Performance

- Subresource Integrity (SRI) en CDN scripts.
- `preconnect` a orígenes externos.
- Critical CSS inline.
- Imágenes responsivas.

### Seguridad

- CSP estricta.
- HTTPS forzado (HSTS).
- Subir el bundle a CDN privado + URLs firmadas.
- NO subir `.env` ni secretos al repo.
- `npm audit` automatizado en CI.

### Observabilidad

- **Sentry** (errores).
- **LogRocket** (session replay).
- **Datadog RUM** (real user monitoring).
- **Google Analytics / Plausible**.

### Estrategia de versiones

- `package.json`: `version`.
- Tags git (`v1.0.0`).
- Branch por env: `main` → prod, `develop` → staging.

### Releases

- **Semantic versioning**.
- Changelog (release-it, standard-version).
- Conventional Commits + release-please (auto).

### Caché multi-capa

| Capa       | TTL         | Cómo                          |
| ---------- | ----------- | ----------------------------- |
| HTML       | No-cache    | Always revalidate             |
| JS/CSS hash| 1 year      | `immutable`                   |
| Imágenes   | 30 days     | `max-age=2592000`             |
| API (frontend) | 5 min   | Cache-Control en headers      |

### Subresource Integrity

```html
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-..."
        crossorigin="anonymous"></script>
```

### Versionado de assets

```
app.abc123.js   (hash)
```

Permite cache infinito. Al deploy, cambia el nombre y todos reciben el nuevo.

### Healthchecks SPA

```js
// /healthz
app.get("/healthz", (req, res) => res.json({ ok: true }));
```

### Feature flags

- LaunchDarkly, Unleash, GrowthBook.
- En frontend:

```js
import { FEATURE_NEW_CART } from "./flags";
if (FEATURE_NEW_CART) { /* ... */ }
```

### Actividad práctica

- Configura Lighthouse CI o Sentry y crea alertas de performance/regresión.

---

## Recursos complementarios

### Lecturas

- [Vite Docs](https://vitejs.dev/)
- [Docker Docs](https://docs.docker.com/)
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [web.dev](https://web.dev/)
- [MDN - Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Cloudflare Pages](https://pages.cloudflare.com/)
- [Vercel Docs](https://vercel.com/docs)

### Herramientas

- **Cloudflare**, **Vercel**, **Netlify**, **Render**, **Railway**.
- **Sentry**, **Datadog**, **Plausible**.
- **Lighthouse CI**, **webhint**.
- **Docker**, **Lovable Dev**, **Podman**.
- **Dependabot**, **Snyk**, **Renovate**.
- **release-please** (Google), **standard-version**.

### Cursos

- **Frontend Masters - DevOps for Developers** (no específicos frontend).
- **Docker Mastery (Udemy)**.

---

## Evaluación sugerida

1. **Build optimizado (25%)**: configuración de Vite con manual chunks, compresión y sourcemaps.
2. **Docker (25%)**: Dockerfile multi-stage + nginx con cache y security headers.
3. **CI (25%)**: workflow con lint + test + build + e2e + upload de artefactos.
4. **Deploy (25%)**: aplicación desplegada con URL pública, preview por PR y rollback documentado.

---

> **Nota para el estudiante**: DevOps no se aprende "en teoría". Cada decisión (caché, compresión, container, secrets) impacta directamente en la UX de tu app. Empieza con un deploy simple, mide, itera. La mejor infra es la que puedes mantener.
