# Comparativa de Frameworks Fullstack

## Objetivo de aprendizaje

Entender que **todos los frameworks fullstack resuelven el mismo problema fundamental** —construir aplicaciones web donde el servidor y el cliente colaboran para producir interfaces dinámicas— pero lo hacen desde **filosofías, lenguajes y trade-offs distintos**.

La elección de un framework no es una decisión técnica aislada: refleja una forma de pensar sobre cómo se estructura el software, dónde vive la lógica y cuánta "opinión" se le impone al desarrollador.

---

## Tabla comparativa

| Framework             | Lenguaje          | Filosofía                          |
| --------------------- | ----------------- | ---------------------------------- |
| Next.js               | React + Node      | SSR / ISR / RSC                    |
| Nuxt                  | Vue               | Muy opinionado                     |
| SvelteKit             | Svelte            | Muy rápido y simple                |
| Remix                 | React             | Web Standards                      |
| Django + HTMX         | Python            | Backend First                      |
| FastAPI + React       | Python            | API First                          |
| Laravel               | PHP               | MVC clásico                        |
| AdonisJS              | Node              | Laravel para Node                  |
| NestJS                | Node              | Arquitectura empresarial           |
| Leptos                | Rust              | SSR muy rápido                     |
| Axum + Leptos         | Rust              | Performance extrema                |

---

## Lectura por dimensiones

### 1. Lenguaje base

Los frameworks se agrupan naturalmente por el ecosistema del lenguaje que los sostiene:

- **JavaScript / TypeScript**: Next.js, Nuxt, SvelteKit, Remix, AdonisJS, NestJS. Comparten Node.js como runtime.
- **Python**: Django, FastAPI. Ecosistema fuerte en datos, ciencia y backend tradicional.
- **PHP**: Laravel. Maduro, económico de desplegar, presente en gran parte de la web.
- **Rust**: Leptos, Axum. Priorizan rendimiento y seguridad de memoria.

### 2. Filosofía de renderizado y ubicación de la lógica

| Enfoque                 | Frameworks representativos          | Implicación                                                  |
| ----------------------- | ----------------------------------- | ------------------------------------------------------------ |
| SSR / ISR / RSC         | Next.js                             | Híbrido: server, estático y componentes de servidor.         |
| Muy opinionado          | Nuxt, Laravel, AdonisJS, NestJS     | Convención sobre configuración; menos decisiones del dev.    |
| Ligero y compilado      | SvelteKit, Leptos                   | Menos JS al cliente; reactividad afinada.                    |
| Web Standards first     | Remix                               | Se apoya en `fetch`, `FormData`, `Request`, `Response`.     |
| Backend primero         | Django + HTMX                       | El servidor devuelve HTML; el cliente solo potencia UX.     |
| Separación API/Cliente  | FastAPI + React                     | Frontend y backend desacoplados, hablándose por HTTP/JSON.   |
| Arquitectura empresarial| NestJS                              | Inyección de dependencias, módulos, decoradores.             |
| Performance extrema     | Axum + Leptos                       | Rust de extremo a extremo; latencia y throughput máximos.   |

### 3. Opinión vs. flexibilidad

- **Muy opinionados** (Nuxt, Laravel, AdonisJS, NestJS): dictan estructura de carpetas, ORM, sistema de plantillas y convenciones. Aceleran el arranque y homogenizan equipos.
- **Flexibles** (Next.js, FastAPI + React): permiten elegir ORM, autenticación, despliegue. Más libertad, más decisiones.
- **Mínimos** (SvelteKit, Leptos): menos capas, menos magia; el código es lo que se ejecuta.

### 4. Modelo de ejecución

- **Basados en Node**: Next.js, Nuxt, SvelteKit, Remix, AdonisJS, NestJS.
- **Basados en Python**: Django, FastAPI (Uvicorn/Gunicorn).
- **Basados en PHP**: Laravel (FPM, Octane).
- **Basados en Rust**: Leptos compila a WASM; Axum corre sobre Tokio.

### 5. Casos de uso típicos

| Framework        | Caso ideal                                                    |
| ---------------- | ------------------------------------------------------------- |
| Next.js          | Sitios con contenido + app, SEO crítico, equipos React.      |
| Nuxt             | Equipos Vue que quieren convenciones claras.                 |
| SvelteKit        | Apps pequeñas/medias con foco en bundle pequeño.              |
| Remix            | Apps centradas en formularios y datos, web fundamentals.      |
| Django + HTMX    | Equipos Python que evitan SPA, prefieren HTML progresivo.    |
| FastAPI + React  | APIs públicas consumidas por múltiples clientes.             |
| Laravel          | Apps tradicionales, equipos PHP, MVP rápido.                 |
| AdonisJS         | Equipos Node que extrañan Laravel.                           |
| NestJS           | Microservicios, equipos grandes, arquitectura tipada.         |
| Leptos           | Equipos Rust o apps con UI reactiva de bajo costo.           |
| Axum + Leptos    | Cuando la latencia y el uso de CPU importan más que el DX.    |

---

## Ideas clave para el estudiante

1. **No hay framework "mejor", hay contextos apropiados.** Un mismo producto puede reescribirse en tres frameworks y los tres serán correctos.
2. **La filosofía importa más que el lenguaje.** Aprender un framework nuevo es aprender una nueva forma de pensar, no solo una nueva sintaxis.
3. **SSR no es lo opuesto a SPA.** Next.js, Nuxt, SvelteKit y Remix conviven con hidratación parcial y rutas cliente.
4. **El "API First" no significa "sin servidor".** FastAPI + React sigue teniendo un servidor; la diferencia es que su contrato es JSON, no HTML.
5. **El rendimiento extremo (Rust) tiene un costo de productividad.** Úsalo cuando el problema lo justifique, no por moda.

---

## Preguntas para discusión en clase

- ¿Qué problema compartidos resuelven todos los frameworks de la tabla?
- Si tuvieras que construir un portal de noticias con mucho contenido y SEO, ¿cuál elegirías y por qué?
- ¿En qué escenario tendría sentido elegir Django + HTMX sobre Next.js, aunque el equipo ya conozca React?
- ¿Por qué frameworks como Laravel y AdonisJS se sienten similares aunque vivan en ecosistemas distintos?
- ¿Cuándo deja de tener sentido elegir JavaScript y se justifica pasar a Rust?

---

## Ejercicio sugerido

En grupos, elijan **dos frameworks** de la tabla que no conozcan. Investiguen:
1. Cómo se crea un proyecto nuevo.
2. Cómo se define una ruta.
3. Cómo se renderiza una página con datos dinámicos.
4. Cómo se maneja el estado compartido entre servidor y cliente.

Presenten una comparación de 5 minutos destacando similitudes con frameworks que sí conocen.
