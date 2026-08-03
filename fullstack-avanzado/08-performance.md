# Performance

## Objetivo de aprendizaje

Performance no es "poner un CDN". Es un **sistema de decisiones** que atraviesa toda la arquitectura: cómo se envía el HTML, cuándo se ejecuta JavaScript, dónde corre el código, qué se cachea y por cuánto tiempo.

Aquí se estudian las técnicas que separan una app que se siente **instantánea** de una que se siente **lenta**, incluso con buen hardware.

---

## Mapa general

| Técnica              | Reduce                          | Capa             |
| -------------------- | ------------------------------- | ---------------- |
| Lazy Loading         | Bytes iniciales                 | Cliente / Build  |
| Streaming            | TTFB percibido                   | Transporte       |
| Suspense             | Tiempo de bloqueo               | Render           |
| Partial Hydration    | JS enviado al cliente           | Render           |
| Edge Rendering       | Latencia de red                  | Infraestructura  |
| CDN                  | Distancia al usuario             | Infraestructura  |
| Cache (HTTP)         | Requests repetidos              | Transporte       |
| Redis                | Carga a la DB                   | Aplicación       |

---

## 1. Lazy Loading

### Idea central

No cargar, ejecutar o renderizar **lo que el usuario aún no necesita**. Diferir hasta el último momento posible.

### Niveles

| Nivel             | Ejemplo                                       |
| ----------------- | --------------------------------------------- |
| **Ruta**          | `import()` dinámico de páginas                |
| **Componente**    | Modal, gráfico pesado cargado al abrir        |
| **Imagen**        | `loading="lazy"` en `<img>`                   |
| **Script**        | `<script type="module">` con `defer`          |
| **Dato**          | TanStack Query, SWR con `staleTime`           |
| **CSS**           | `@import` condicional, critical CSS           |

### Ejemplo: ruta lazy en React Router

```tsx
import { lazy, Suspense } from "react";
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));
const Reports = lazy(() => import("./pages/Reports"));

const router = createBrowserRouter([
  { path: "/", element: <Home /> },
  { path: "/dashboard", element: <Suspense fallback={<Spinner />}><Dashboard /></Suspense> },
  { path: "/settings", element: <Suspense fallback={<Spinner />}><Settings /></Suspense> },
  { path: "/reports", element: <Suspense fallback={<Spinner />}><Reports /></Suspense> },
]);
```

### Ejemplo: imagen lazy

```html
<img
  src="poster.jpg"
  data-src="full.jpg"
  loading="lazy"
  decoding="async"
  width="800"
  height="450"
  alt="..."
/>
```

### Ejemplo: bundle splitting en Vite/Webpack

```ts
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          charts: ["recharts", "d3"],
        },
      },
    },
  },
});
```

### Cuándo usarlo

- **Siempre** para rutas y componentes pesados.
- Imágenes debajo del fold.
- Bibliotecas grandes que se usan en una sola pantalla.

### Cuándo **no** abusar

- Micro-optimizaciones en assets < 10 KB: el costo de lazy > el beneficio.
- Above-the-fold: la primera vista debe estar lo más completa posible.

---

## 2. Streaming

### Idea central

En vez de esperar a tener **todo el HTML** listo para enviarlo, enviarlo **por partes** a medida que se genera. El navegador empieza a pintar y a cargar recursos mientras el servidor sigue trabajando.

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Versión     | Característica                                  |
| ----------- | ----------------------------------------------- |
| HTTP/1.1    | Una request por conexión (o pocas en paralelo)  |
| HTTP/2      | Multiplexación: muchas streams sobre 1 conexión |
| HTTP/3      | HTTP/2 sobre QUIC/UDP, mejor sobre redes con pérdida |

### Streaming con Node

```ts
import { createReadStream } from "node:fs";
import { renderToReadableStream } from "react-dom/server";

const stream = await renderToReadableStream(<App />);
return new Response(stream, {
  headers: { "Content-Type": "text/html" },
});
```

### Streaming con React Server Components

```tsx
// app/feed/page.tsx
import { Suspense } from "react";

export default function Feed() {
  return (
    <>
      <Header />
      <Suspense fallback={<FeedSkeleton />}>
        <PostList />   {/* async server component */}
      </Suspense>
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />    {/* async server component */}
      </Suspense>
    </>
  );
}
```

El shell (Header) se envía **de inmediato**. Los `<Suspense>` marcan lugares donde se está esperando data; el navegador puede pintar placeholders y el server va emitiendo HTML a medida que cada componente resuelve.

### Beneficio medible

En una página con header + lista + sidebar:

- **Sin streaming**: TTFB = 800ms (se espera la lista lenta).
- **Con streaming**: TTFB = 100ms (envía header), FCP = 400ms, LCP = 1000ms.

El usuario ve la página antes, aunque el contenido tarde lo mismo.

---

## 3. Suspense

### Idea central

API para **declarar boundaries de carga**. El componente superior sigue mostrándose mientras el inferior carga. No más `if (loading) return <Spinner />` esparcido por el código.

### Ejemplo clásico

```tsx
<Suspense fallback={<Skeleton />}>
  <Profile />
  <Feed />
</Suspense>
```

### Suspense para datos (React 18+, frameworks data-aware)

En frameworks como Next.js, Remix, TanStack Start, **Suspense también espera datos**:

```tsx
function Profile() {
  // Esta función "suspende" mientras los datos llegan
  const user = use(fetchUser());
  return <h1>{user.name}</h1>;
}

<Suspense fallback={<ProfileSkeleton />}>
  <Profile />
</Suspense>
```

### Patrones avanzados

- **Nested Suspense**: cada parte tiene su propio fallback.
- **Skeletons vs spinners**: skeletons son mejores percibidos.
- **No envolver todo**: el shell de la app no debe tener Suspense.

### Buenas prácticas

- Suspense **al nivel de la ruta**, no en cada componente.
- Skeletons que imitan la forma del contenido (evita layout shift).
- Combinar con prefetch: cargar datos antes de que el usuario llegue.

---

## 4. Partial Hydration

### Idea central

En SSR tradicional, **todo** el JavaScript de la página se hidrata. En partial hydration, **solo se hidratan los componentes que necesitan interactividad**. El resto queda como HTML estático.

### ¿Por qué importa?

Una página típica tiene:
- Header estático.
- 90% de contenido de solo lectura.
- Un formulario, un dropdown, un botón.

Hidratar todo = enviar y ejecutar JS de toda la página. Con partial hydration, solo el botón se vuelve interactivo.

### Island Architecture (Astro, Qwik)

```
┌──────────────────────────────────────┐
│  Header (HTML estático)              │  ← no se hidrata
├──────────────────────────────────────┤
│  Artículo largo (HTML estático)      │  ← no se hidrata
├──────────────────────────────────────┤
│  ┌──────────────┐                    │
│  │ Comentarios  │ ← ISLA, se hidrata │
│  └──────────────┘                    │
├──────────────────────────────────────┤
│  Footer (HTML estático)              │  ← no se hidrata
└──────────────────────────────────────┘
```

### Ejemplo en Astro

```astro
---
// pages/post/[id].astro
import Layout from "../../layouts/Layout.astro";
import CommentSection from "../../components/CommentSection.svelte";
const post = await getPost(Astro.params.id);
---
<Layout>
  <article>
    <h1>{post.title}</h1>
    <p>{post.content}</p>
  </article>

  <CommentSection client:visible postId={post.id} />
</Layout>
```

`client:visible` le dice al framework: "hidrata este componente solo cuando entre en viewport". El resto de la página es HTML estático.

### Estrategias de hidratación en Astro

| Directiva        | Cuándo hidrata                            |
| ---------------- | ----------------------------------------- |
| `client:load`    | Inmediatamente al cargar                 |
| `client:idle`    | Cuando el navegador esté idle             |
| `client:visible` | Cuando entra en viewport                 |
| `client:media`   | Cuando coincide un media query           |
| `client:only`    | No SSR, solo cliente                     |

### Trade-offs

- **Menos JS inicial** = mejor TTI, mejor en móviles lentos.
- **Más complejidad**: el "qué es interactivo y qué no" debe pensarse de antemano.
- **SEO**: el HTML sigue siendo completo, así que no hay penalty.

---

## 5. Edge Rendering

### Idea central

Ejecutar el renderizado **lo más cerca posible del usuario**, en servidores distribuidos globalmente. Reduce la latencia de red que es a menudo el mayor cuello de botella.

### Dónde corre

| Capa              | Latencia al usuario | Casos                    |
| ----------------- | ------------------- | ------------------------ |
| Origin (Virginia) | 200-400 ms          | Render pesado, DB        |
| Edge (50+ POPs)   | 5-50 ms             | Render simple, A/B, auth |
| Browser           | 0 ms                | Cache local              |

### Plataformas

- **Cloudflare Workers + Pages**: V8 isolates, latencia mínima.
- **Vercel Edge Functions**: Deno-based.
- **Netlify Edge Functions**: Deno-based.
- **Fastly Compute@Edge**: WebAssembly.
- **AWS Lambda@Edge**: Node.

### Limitaciones del edge

- **Runtime limitado**: no todo Node API está disponible.
- **Sin estado local**: cualquier cache vive en KV / Durable Objects.
- **Cold starts casi cero**: porque son isolates, no containers.
- **Tamaño del bundle**: hay límites (~1 MB).

### Cuándo usarlo

- **A/B testing y routing**: decidir versión por cookie/geo.
- **Auth y redirecciones**: validar token, redirigir.
- **HTML con datos cacheables**:首页, listados.
- **Personalización ligera**: locale, moneda, tema.

### Cuándo **no** usarlo

- Queries pesadas a la DB (latencia DB domina).
- Lógica que necesita SDK pesado.
- Procesamiento CPU-bound largo.

### Ejemplo: edge middleware

```ts
// middleware.ts (Next.js)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export const config = {
  matcher: ["/((?!_next|api|favicon).*)"],
};

export function middleware(req: NextRequest) {
  const country = req.geo?.country ?? "US";
  const url = req.nextUrl.clone();

  if (country === "DE" && !req.cookies.get("consent")) {
    url.pathname = "/consent";
    return NextResponse.redirect(url);
  }

  const res = NextResponse.next();
  res.headers.set("x-geo", country);
  return res;
}
```

---

## 6. CDN (en el contexto de performance)

### Para estáticos

Ya cubierto en el tema de Cloud. Repite el principio: **bytes más cerca del usuario = menos latencia**.

### Para HTML dinámico (edge caching)

```
Usuario en Madrid → Edge Madrid → cache HIT → respuesta en 10ms
                                          ↓ MISS
                                   Origin → 200ms
```

### Cache keys

Qué determina si un request es "el mismo":

- URL completa.
- Cookies (si afecta la respuesta).
- Headers (`Accept-Language`, `Authorization` para variantes).
- Query params.

### Patrones de invalidación

| Estrategia            | Uso                                       |
| --------------------- | ----------------------------------------- |
| TTL fijo              | Lo más simple. Acepta staleness.          |
| Surrogate-Key + purge | Invalidación quirúrgica por grupo.        |
| Stale-while-revalidate | Sirve viejo, refresca en background.     |
| Cache tag              | Similar a surrogate-key, nombre varía.   |

### Ejemplo de headers

```http
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=600
CDN-Cache-Control: max-age=86400
Surrogate-Key: page-home user-42
```

---

## 7. Cache HTTP

### Idea central

Caché en el **camino del request**: navegador, CDN, proxies intermedios. Es la **forma más barata** de mejorar performance porque evita generar la respuesta.

### Tipos de caché

| Tipo             | Dónde vive                  | Controlado por          |
| ---------------- | --------------------------- | ----------------------- |
| Browser          | Disco del usuario           | `Cache-Control`         |
| CDN              | POPs regionales             | Headers + reglas        |
| Service Worker   | En el cliente (control fino) | Tu código              |
| Server-side      | Memoria del proceso o Redis | Tu código               |

### Directivas clave de `Cache-Control`

| Directiva                  | Significado                                       |
| -------------------------- | ------------------------------------------------- |
| `public`                   | Puede ser cacheado por cualquier caché            |
| `private`                  | Solo caché del navegador, no CDNs                |
| `no-store`                 | No cachear nunca                                  |
| `no-cache`                 | Cachear pero revalidar siempre                    |
| `max-age=N`                | Segundos de vida en caché                         |
| `s-maxage=N`               | Igual pero para CDNs/proxies                      |
| `stale-while-revalidate=N` | Sirve viejo mientras refrescas en background     |
| `immutable`                | No revalidar durante la vida del cache            |

### Patrón por tipo de recurso

```http
# HTML dinámico
Cache-Control: no-cache

# Assets con hash
Cache-Control: public, max-age=31536000, immutable

# API JSON
Cache-Control: private, max-age=0, must-revalidate
ETag: "abc123"

# Imágenes sin hash
Cache-Control: public, max-age=86400
```

### ETag y Last-Modified

Para revalidación condicional:

```http
GET /api/user/42
If-None-Match: "abc123"

→ 304 Not Modified (no envía body)
```

### Service Worker para control fino

```ts
// sw.js
self.addEventListener("fetch", (event) => {
  event.respondWith((async () => {
    const cached = await caches.match(event.request);
    if (cached) {
      // revalidate en background
      event.waitUntil(fetch(event.request).then((r) => caches.put(event.request, r)));
      return cached;
    }
    const response = await fetch(event.request);
    return response;
  })());
});
```

---

## 8. Redis como capa de performance

### Caché de aplicación

```ts
async function getDashboardData(userId: string) {
  const cacheKey = `dashboard:${userId}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const data = await db.dashboard.aggregate({ where: { userId } });
  await redis.set(cacheKey, JSON.stringify(data), "EX", 60);
  return data;
}
```

### Patrones de invalidación

| Patrón            | Cuándo                                            |
| ----------------- | ------------------------------------------------- |
| **TTL**           | Acepta datos viejos por N segundos. Simple.       |
| **Write-through** | Al escribir, actualiza DB y caché. Consistente.   |
| **Write-behind**  | Escribe en caché, async a DB. Rápido, arriesgado. |
| **Cache-aside**   | App gestiona el caché. Explícito.                 |
| **Tag-based**     | Invalida por grupo lógico. Complejo.              |

### Cache stampede

Problema: 1000 requests llegan a la vez para un dato expirado. Todos van a la DB.

Solución: **lock distribuido** + single-flight.

```ts
async function getOrCompute(key, compute) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const got = await redis.set(lockKey, "1", "NX", "EX", 10);
  if (got) {
    try {
      const value = await compute();
      await redis.set(key, JSON.stringify(value), "EX", 300);
      return value;
    } finally {
      await redis.del(lockKey);
    }
  }

  // otro está computando, espera breve y reintenta
  await sleep(100);
  return getOrCompute(key, compute);
}
```

### Pre-warming

Recalentar caches antes de que el usuario llegue. Típico en:

- Cron nocturno que recalcula首页s.
- Background worker que rellena caches de dashboards.
- Al deploy: trigger de un job que pre-carga lo crítico.

### Métricas clave de caché

- **Hit rate**: `hits / (hits + misses)`. Objetivo: > 90% para热点 data.
- **Latency p99 del cache hit** vs miss.
- **Memoria usada**: Redis no escala infinito, monitorea.

---

## Cómo se combinan

Imagina una request a una página de producto:

```
1. Usuario en Brasil hace click en un producto
2. DNS → CDN POP São Paulo
3. Edge cache: HIT (HTML cacheado 60s) → responde en 15ms
4. Si MISS: Edge → Origin (Virginia) en 120ms
5. Origin: Redis HIT (datos del producto) → responde
6. Origin: Redis MISS → query a PostgreSQL → cachea → responde
7. Streaming: envía header + skeleton + producto
8. Browser pinta, suspende comentarios y reseñas
9. Lazy loads: imagen del producto (intersection observer)
10. Partial hydration: solo el botón "Agregar al carrito" se hidrata
11. JS interactivo hace fetch a API (que también está en edge)
```

Cada técnica suma. La suma de todas es lo que produce una app que se siente instantánea.

---

## Métricas que importan (Web Vitals)

| Métrica  | Qué mide                          | Objetivo |
| -------- | --------------------------------- | -------- |
| **LCP**  | Cuándo aparece el contenido principal | < 2.5s  |
| **INP**  | Latencia de interacción           | < 200ms  |
| **CLS**  | Saltos visuales de layout         | < 0.1    |
| **TTFB** | Tiempo hasta el primer byte       | < 600ms  |
| **FCP**  | Primer contenido visible          | < 1.8s   |
| **TBT**  | Tiempo total de bloqueo           | < 200ms  |

### Cómo se mueven con estas técnicas

| Técnica           | Mejora                              |
| ----------------- | ----------------------------------- |
| Lazy loading      | TBT, JS inicial                     |
| Streaming         | TTFB, FCP, LCP                      |
| Suspense          | TBT, percepción de velocidad        |
| Partial hydration | TBT, TTI, JS inicial                |
| Edge rendering    | TTFB, LCP                           |
| CDN               | TTFB, FCP, LCP                      |
| Cache HTTP        | TTFB, FCP, LCP, INP                 |
| Redis             | TTFB (indirecto), LCP, INP          |

---

## Buenas prácticas

- **Mide antes de optimizar**. No asumas; prueba.
- **Cuidado con over-caching**: datos stale pueden ser peor que un poco más de latencia.
- **Stale-while-revalidate** es tu amigo para perceived performance.
- **Invalida por clave/tag, no por TTL global**, cuando puedas.
- **Bundle analysis定期**: bundlephobia, source-map-explorer, `vite-bundle-visualizer`.
- **Prefetch** del siguiente paso probable del usuario.
- **Critical CSS inline** en el HTML, el resto en archivos externos.
- **Imágenes modernas**: WebP/AVIF, `srcset`, `sizes`, formato responsivo.
- **Fonts**: `font-display: swap` y preload de la principal.
- **Edge functions** para auth/routing, no para renderizar todo.

---

## Ejercicios

### Ejercicio 1 — Lazy loading progresivo

Toma una página con 5 rutas. Haz que cada ruta sea un chunk separado. Mide:

1. Tamaño del bundle inicial sin lazy.
2. Tamaño del bundle inicial con lazy.
3. Tiempo hasta interactivo en la primera ruta.

### Ejercicio 2 — Streaming real

Implementa una página con dos componentes async: uno rápido (50ms) y uno lento (2s). Mide TTFB y LCP con y sin streaming. ¿Qué cambia?

### Ejercicio 3 — Suspense anidado

Crea una página con:

- Shell que se ve de inmediato.
- Sidebar que carga en 1s con skeleton.
- Main que carga en 2s con skeleton.

Verifica que el shell aparece antes de que el resto termine.

### Ejercicio 4 — Astro con islas

Migra una página React existente a Astro:

1. Header y footer: estáticos.
2. 1 componente interactivo: isla con `client:visible`.
3. Compara bundle de JS y TTI antes/después.

### Ejercicio 5 — Edge function

Implementa un middleware edge que:

1. Lee el país del usuario.
2. Redirige a la versión localizada si está en un país soportado.
3. Añade un header con el país.
4. Mide TTFB en 3 regiones (puedes simular con throttle).

### Ejercicio 6 — Estrategia de caché por tipo de recurso

Define headers de `Cache-Control` para estos recursos:

1. `index.html`
2. `app.a3b1c2.js` (con hash)
3. `app.css` (sin hash)
4. `/api/user/me`
5. `/api/products?category=x`
6. Imágenes de productos
7. Un JSON público que cambia 1 vez al día

Justifica cada uno.

### Ejercicio 7 — Caché Redis con stampede protection

Implementa `getOrCompute` con:

- TTL de 5 minutos.
- Lock distribuido con timeout de 10s.
- Métrica de hit/miss.
- Invalidación explícita.

Demuestra que con 1000 requests concurrentes a una key fría, solo 1 va a la DB.

### Ejercicio 8 — Auditoría de performance

Toma una web real (la tuya) y mide:

- Web Vitals con Lighthouse.
- Tamaño de bundles.
- Requests concurrentes.
- Tiempo de LCP vs TTFB.

Identifica las 3 mejoras con mayor impacto y aplica 1.

---

## Problema propuesto

### E-commerce de alta performance

Debes construir una **tienda online** optimizada para Core Web Vitals en el percentil 75 de usuarios (3G, móvil gama media). El sitio tiene:

- Homepage con listado de productos.
- Página de producto con reseñas y recomendaciones.
- Carrito persistente.
- Búsqueda con filtros.

### Requisitos de performance

1. **LCP < 2.0s** en p75 móvil.
2. **INP < 200ms**.
3. **CLS < 0.05**.
4. **TTFB < 200ms** desde edge.
5. **JS inicial < 100 KB** gzipped (sin contar el framework).
6. **Imágenes servidas en formato moderno** (WebP/AVIF) con srcset.
7. **Caché hit rate > 90%** en el CDN para assets.

### Requisitos técnicos

Tu solución debe:

1. Usar **streaming SSR** con React Server Components, Solid Start o similar.
2. **Partial hydration** o islands: solo el carrito y la búsqueda son interactivos en la homepage.
3. **Lazy load** de todas las rutas y de imágenes below-the-fold.
4. **Edge rendering** para el HTML cacheable (homepage, listados).
5. **CDN** sirviendo assets con cache headers apropiados.
6. **Redis** como caché de consultas de producto y sesión de carrito.
7. **Cache stampede protection** en la capa de aplicación.
8. **Suspense** con skeletons realistas (mismo tamaño que el contenido).
9. **Prefetch** de la siguiente ruta probable al hacer hover.

### Entregables

- Diagrama de la arquitectura de performance.
- Código del sitio con todas las técnicas aplicadas.
- Headers de `Cache-Control` documentados por tipo de recurso.
- Configuración del edge runtime.
- Reportes Lighthouse antes y después (mínimo -50% en LCP y TBT).
- Documento de 1 página justificando:
  - Qué técnica aporta qué métrica.
  - Qué NO optimizaste y por qué.
  - Cómo se ve a 10x el tráfico (¿qué se rompe?).
  - Trade-offs de UX vs performance que tomaste.

### Criterios de evaluación

| Criterio                                        | Peso |
| ----------------------------------------------- | ---- |
| Web Vitals en verde (p75 móvil)                 | 25%  |
| JS inicial < 100 KB gzipped                     | 10%  |
| Streaming + Suspense aplicado correctamente     | 15%  |
| Partial hydration / islands                     | 10%  |
| Edge rendering con cache                        | 15%  |
| Redis con stampede protection                   | 10%  |
| Headers de caché por tipo de recurso            | 10%  |
| Justificación y métricas antes/después          |  5%  |
