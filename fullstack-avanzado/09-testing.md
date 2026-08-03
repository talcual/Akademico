# Testing

## Objetivo de aprendizaje

Probar software no es "ejecutar tests". Es **diseñar una estrategia** que equilibre velocidad, confianza y costo de mantenimiento. Aquí se estudian los niveles de la pirámide de testing, las herramientas modernas y los patrones que permiten que los tests **sean una ayuda y no una carga**.

La meta es que el estudiante entienda qué tipo de test usar para qué problema, y por qué **un test que nadie mantiene es peor que no tener test**.

---

## Mapa general

| Nivel         | Qué prueba                          | Velocidad | Confianza | Herramientas             |
| ------------- | ----------------------------------- | --------- | --------- | ------------------------ |
| Unit          | Una función/módulo aislado          | ms        | Baja      | Vitest, Pytest           |
| Integration   | Varios módulos juntos               | s         | Media     | Vitest, Pytest, Supertest |
| E2E           | Flujo completo del usuario          | s-min     | Alta      | Playwright, Cypress      |
| Contract      | API ↔ cliente                       | s         | Media     | Pact, Schemathesis       |
| Visual        | Apariencia de la UI                 | s         | Media     | Percy, Chromatic, Playwright |
| Performance   | Latencia, throughput                | min       | Específica| k6, Artillery, Playwright |
| Mutation       | Calidad de los tests mismos         | min       | Meta      | Stryker, mutmut          |

---

## 1. Tests Unitarios

### Idea central

Probar **la unidad más pequeña** de comportamiento (función, clase, método) **aislada** del resto. Sin DB, sin red, sin filesystem. Deterministas y rápidos.

### Características

- **Rápidos**: miles en segundos.
- **Deterministas**: mismo input → mismo output, siempre.
- **Aislados**: no comparten estado entre tests.
- **Una assertion por concepto**: un test, una cosa.

### Vitest (JS/TS)

```ts
import { describe, it, expect } from "vitest";
import { calcularTotal } from "./pedido";

describe("calcularTotal", () => {
  it("suma el precio * cantidad de cada item", () => {
    const items = [
      { precio: 100, cantidad: 2 },
      { precio: 50, cantidad: 1 },
    ];
    expect(calcularTotal(items)).toBe(250);
  });

  it("devuelve 0 si no hay items", () => {
    expect(calcularTotal([])).toBe(0);
  });

  it("ignora items con cantidad 0", () => {
    const items = [{ precio: 100, cantidad: 0 }];
    expect(calcularTotal(items)).toBe(0);
  });
});
```

### Pytest (Python)

```python
def test_suma_items():
    items = [{"precio": 100, "cantidad": 2}, {"precio": 50, "cantidad": 1}]
    assert calcular_total(items) == 250

def test_sin_items_devuelve_cero():
    assert calcular_total([]) == 0

@pytest.mark.parametrize("precio,cantidad,esperado", [
    (100, 2, 200),
    (50, 0, 0),
    (99.99, 3, 299.97),
])
def test_multiples_casos(precio, cantidad, esperado):
    assert calcular_total([{"precio": precio, "cantidad": cantidad}]) == esperado
```

### Qué **no** debe ser unit

- Cosas que necesitan DB (usa integration).
- Cosas que necesitan red (usa integration con mocks o e2e).
- Cosas que son básicamente configuración.

---

## 2. Tests de Integración

### Idea central

Probar **varios componentes juntos**: el caso de uso + el repositorio (con DB real o en memoria), el controller + el caso de uso, el parser + el validador.

### Lo que prueba

- Que las **interfaces** entre componentes encajan.
- Que las queries a la DB son correctas.
- Que la lógica de negocio completa funciona end-to-end **dentro del backend**.

### Test de integración con DB en memoria

```ts
import { describe, it, expect, beforeEach } from "vitest";
import { CrearPedido } from "./CrearPedido";
import { InMemoryPedidoRepository } from "./InMemoryPedidoRepository";

describe("CrearPedido (integration)", () => {
  let repo: InMemoryPedidoRepository;
  let sut: CrearPedido;

  beforeEach(() => {
    repo = new InMemoryPedidoRepository();
    sut = new CrearPedido(repo);
  });

  it("persiste el pedido y devuelve un id", async () => {
    const pedido = await sut.ejecutar({
      clienteId: "c-1",
      items: [{ productoId: "p-1", precio: 100, cantidad: 2 }],
    });

    expect(pedido.id).toBeDefined();
    expect(pedido.total).toBe(200);

    const stored = await repo.obtenerPorId(pedido.id);
    expect(stored).toEqual(pedido);
  });

  it("falla si no hay items", async () => {
    await expect(
      sut.ejecutar({ clienteId: "c-1", items: [] })
    ).rejects.toThrow("Pedido sin items");
  });
});
```

### Test de integración con API HTTP (Supertest)

```ts
import request from "supertest";
import { app } from "./app";

describe("POST /pedidos", () => {
  it("201 con el pedido creado", async () => {
    const response = await request(app)
      .post("/pedidos")
      .send({ clienteId: "c-1", items: [{ productoId: "p-1", cantidad: 1 }] });

    expect(response.status).toBe(201);
    expect(response.body).toMatchObject({
      clienteId: "c-1",
      total: expect.any(Number),
    });
  });
});
```

### Pytest con fixtures y DB

```python
import pytest
from app import create_app
from db import db

@pytest.fixture
def app():
    app = create_app(testing=True)
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()

def test_crear_pedido(client):
    response = client.post("/pedidos", json={
        "clienteId": "c-1",
        "items": [{"productoId": "p-1", "cantidad": 1, "precio": 100}],
    })
    assert response.status_code == 201
    assert response.json["total"] == 100
```

### Estrategia para tests rápidos

- **In-memory repos** en vez de DB real cuando es posible.
- **Testcontainers** (PostgreSQL en Docker por test) cuando necesitas DB real.
- **Mocks selectivos** solo del mundo exterior (HTTP, tiempo, aleatoriedad).

---

## 3. Tests E2E (End-to-End)

### Idea central

Probar el **flujo completo del usuario** en un entorno lo más parecido a producción: navegador real, red real, DB real (o equivalente). El **black-box** por excelencia.

### Lo que prueba

- Que el sistema entero funciona junto.
- Que los flujos críticos del negocio se completan.
- Que la integración con servicios externos funciona.

### Lo que **no** prueba bien

- Lógica de negocio detallada (lentos y frágiles).
- Edge cases exhaustivos.
- Performance.

### Cuántos tener

- **Pocos pero críticos**: login, signup, compra, flujo principal de cada feature.
- **Pirámide**: 70% unit, 20% integration, 10% e2e.
- **Un e2e vale por 100 unit** en confianza, pero tarda 1000x más.

---

## 4. Playwright

### Idea central

Framework de testing E2E **multi-navegador** (Chromium, Firefox, WebKit) con **auto-wait**, **trace viewer**, **fixtures** y **paralelización** nativa. Es el estándar moderno.

### Ejemplo básico

```ts
import { test, expect } from "@playwright/test";

test("usuario puede registrarse y ver su dashboard", async ({ page }) => {
  await page.goto("/signup");

  await page.getByLabel("Email").fill("ana@example.com");
  await page.getByLabel("Password").fill("supersecret123");
  await page.getByRole("button", { name: "Crear cuenta" }).click();

  await expect(page).toHaveURL("/dashboard");
  await expect(page.getByRole("heading", { name: "Bienvenida, Ana" })).toBeVisible();
});
```

### Locators resilientes

Playwright recomienda locators que **resisten cambios de UI**:

```ts
// ✅ bien
page.getByRole("button", { name: "Comprar" });
page.getByLabel("Email");
page.getByText("Pedido confirmado");

// ❌ frágil
page.locator("#btn-comprar");
page.locator("div > span > button");
```

### Auto-wait

Playwright **espera automáticamente** a que los elementos estén:

- Visible.
- Stable (no animando).
- Recibiendo eventos.
- Habilitados.

No más `waitForTimeout(1000)`. Si necesitas esperar algo concreto, usa `expect(locator).toHaveText(...)` que reintenta.

### Page Object Model

```ts
// pages/LoginPage.ts
export class LoginPage {
  constructor(private page: Page) {}

  async login(email: string, password: string) {
    await this.page.goto("/login");
    await this.page.getByLabel("Email").fill(email);
    await this.page.getByLabel("Password").fill(password);
    await this.page.getByRole("button", { name: "Iniciar sesión" }).click();
  }
}

// tests/login.spec.ts
test("login exitoso", async ({ page }) => {
  const login = new LoginPage(page);
  await login.login("ana@example.com", "secret");
  await expect(page).toHaveURL("/dashboard");
});
```

### Fixtures para setup

```ts
// fixtures.ts
import { test as base } from "@playwright/test";
import { LoginPage } from "./pages/LoginPage";

type Fixtures = { loginPage: LoginPage; authenticatedPage: Page };

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  authenticatedPage: async ({ page }, use) => {
    await new LoginPage(page).login("ana@example.com", "secret");
    await use(page);
  },
});
```

### Trace Viewer

Cuando un test falla en CI, Playwright guarda un **trace** con DOM snapshots, network, console y acciones. Abre localmente y ves exactamente qué pasó.

```ts
// playwright.config.ts
use: {
  trace: "on-first-retry",
  screenshot: "only-on-failure",
  video: "retain-on-failure",
}
```

### Network mocking

```ts
test("muestra fallback si la API falla", async ({ page }) => {
  await page.route("**/api/pedidos", (route) =>
    route.fulfill({ status: 500, body: "Internal Server Error" })
  );

  await page.goto("/pedidos");
  await expect(page.getByText("No pudimos cargar tus pedidos")).toBeVisible();
});
```

### MCP de Playwright

El servidor MCP de Playwright permite que **agentes de IA** ejecuten y depuren tests en navegadores reales. Útil para debugging asistido.

---

## 5. Vitest

### Idea central

Test runner **moderno y rápido** para JS/TS, compatible con la API de Jest pero más rápido, con mejor soporte de ESM, TypeScript nativo y modo watch inteligente.

### Por qué Vitest

- **Vite-powered**: arranque casi instantáneo.
- **ESM first**: sin config adicional.
- **TypeScript sin config**.
- **Compatible con Jest API**: migración trivial.
- **UI mode**: navegador para debug.

### Configuración

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "node", // o "jsdom", "happy-dom"
    setupFiles: ["./tests/setup.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "html"],
      exclude: ["**/node_modules/**", "**/*.test.ts"],
    },
  },
});
```

### Mocking

```ts
import { vi, describe, it, expect } from "vitest";
import { enviarNotificacion } from "./notificacion";
import { Pedido } from "./pedido";

vi.mock("./email");

describe("Pedido", () => {
  it("envía email al confirmar", async () => {
    const email = await import("./email");
    const pedido = new Pedido();
    await pedido.confirmar();
    expect(email.enviar).toHaveBeenCalledWith("ana@example.com", expect.any(String));
  });
});
```

### Test de funciones async con tiempo

```ts
import { vi } from "vitest";

vi.useFakeTimers();
vi.setSystemTime(new Date("2026-01-01T12:00:00Z"));

// código bajo test usa new Date() — está bajo control
```

### Snapshot testing (con cuidado)

```ts
expect(generarFactura(pedido)).toMatchSnapshot();
```

Útil para JSON, **peligroso** para HTML completo (cambios de estilos rompen tests sin razón).

---

## 6. Pytest

### Idea central

Framework de testing **más usado en Python**. Simple para empezar, potente cuando le añades plugins. La elección por defecto en Python.

### Por qué Pytest

- Sintaxis clara (`assert` nativo).
- **Fixtures** poderosas para setup reusable.
- **Parametrize** para data-driven tests.
- **Plugins** para todo (Django, FastAPI, asyncio, mock, coverage).
- Output legible con `-v`, fallos con `--tb=short`.

### Fixtures con scope

```python
import pytest

@pytest.fixture
def usuario():
    return Usuario(nombre="Ana", email="ana@example.com")

@pytest.fixture(scope="module")
def db():
    db = setup_test_db()
    yield db
    db.teardown()

@pytest.fixture(scope="session")
def docker():
    # levanta un contenedor una vez por sesión
    container = start_postgres_container()
    yield container
    container.stop()
```

### Parametrize (data-driven)

```python
@pytest.mark.parametrize("email,esperado", [
    ("ana@example.com", True),
    ("no-es-email", False),
    ("", False),
    ("a@b.c", True),
])
def test_validar_email(email, esperado):
    assert validar_email(email) is esperado
```

### Mocking con `unittest.mock`

```python
from unittest.mock import Mock, patch
import pytest

def test_enviar_email_al_crear_usuario():
    with patch("app.email_service.enviar") as mock_enviar:
        crear_usuario("ana@example.com")
        mock_enviar.assert_called_once()
```

### Pytest con FastAPI

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_crear_pedido():
    response = client.post("/pedidos", json={"clienteId": "c-1", "items": []})
    assert response.status_code == 422
```

### Pytest con asyncio

```python
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_crear_pedido_async():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/pedidos", json={...})
    assert response.status_code == 201
```

### Plugins útiles

| Plugin            | Para qué                          |
| ----------------- | --------------------------------- |
| `pytest-cov`      | Coverage                          |
| `pytest-mock`     | Mocking más limpio                |
| `pytest-asyncio`  | Tests async                       |
| `pytest-xdist`    | Paralelización                    |
| `pytest-django`   | Integración con Django            |
| `pytest-postgresql` | PostgreSQL en tests            |
| `hypothesis`      | Property-based testing            |

---

## Patrones de testing

### AAA (Arrange, Act, Assert)

```ts
it("calcula el total con descuento", () => {
  // Arrange
  const items = [{ precio: 100, cantidad: 2 }];
  const cupon = { tipo: "porcentaje", valor: 10 };

  // Act
  const total = calcularTotalConDescuento(items, cupon);

  // Assert
  expect(total).toBe(180);
});
```

### Given-When-Then (BDD)

```ts
test("Given un usuario nuevo, When se registra, Then recibe email de bienvenida", async () => {
  // Given
  const nuevoUsuario = { email: "ana@example.com", password: "secret123" };

  // When
  await registrarUsuario(nuevoUsuario);

  // Then
  expect(emailService.enviar).toHaveBeenCalledWith(
    "ana@example.com",
    expect.stringContaining("Bienvenida"),
  );
});
```

### Test Doubles (tipos)

| Tipo          | Cuándo                                            |
| ------------- | ------------------------------------------------- |
| **Dummy**     | Valor pasado por obligación, nunca se usa.        |
| **Stub**      | Devuelve respuestas predefinidas.                 |
| **Spy**       | Stub que además registra llamadas.                |
| **Mock**      | Esperanzas preprogramadas sobre llamadas.         |
| **Fake**      | Implementación funcional pero simplificada (in-memory DB). |

### Property-Based Testing (Hypothesis)

En lugar de probar **casos**, pruebas **propiedades** que deben cumplirse para cualquier input:

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_suma_es_conmutativa(items):
    assert sum(items) == sum(reversed(items))

@given(st.lists(st.integers(min_value=1, max_value=100), min_size=1))
def test_total_siempre_positivo(items):
    assert calcular_total([{"precio": i, "cantidad": 1} for i in items]) > 0
```

Hypothesis genera cientos de inputs aleatorios buscando contraejemplos.

### Contract Testing (Pact)

Cuando frontend y backend son equipos distintos, los **contratos** se rompen en producción. Pact verifica que el consumidor y el proveedor **están de acuerdo** en el schema.

```
Frontend (consumer) genera un contrato:
  "GET /users/42 devuelve { id: number, name: string }"

Backend (provider) verifica que cumple el contrato
```

### Visual Regression

Herramientas como Percy, Chromatic o el propio Playwright comparan **screenshots** entre versiones. Cambios de pixel no intencionados rompen el test.

---

## Pirámide de testing

```
        /\
       /  \        E2E (pocos, lentos, alta confianza)
      /────\
     /      \      Integration (moderados, mediums)
    /────────\
   /          \    Unit (muchos, rápidos, baja confianza individual)
  /────────────\
```

### Reglas prácticas

| Cantidad        | Velocidad      | Confianza | Cuándo correr       |
| --------------- | -------------- | --------- | ------------------- |
| Muchos unit     | < 1s cada uno  | Baja      | Cada commit         |
| Moderados integ | < 1s cada uno  | Media     | Cada commit         |
| Pocos e2e       | 5-30s cada uno | Alta      | Pre-deploy / nightly|

### Invertir la pirámide es un anti-patrón

```
   ┌──────┐
   │ E2E  │   ← pocos
   │  ██  │
   │██████│
   │██████│   ← muchos unit
   └──────┘
```

Si tienes **muchos e2e y pocos unit**, los tests son lentos, frágiles y caros de mantener.

### Trophy de testing (alternativa moderna)

```
   /\
  / E\      End-to-end
 /────\
/ Int  \   Integration (más énfasis)
/────────\
/  Unit   \ Unit
/────────────\
/  Static    \ Type check, lint, format
```

Esta variante pondera más integration porque es donde la mayoría del valor real vive.

---

## Buenas prácticas

- **Tests deterministas**: nada de `Date.now()`, `Math.random()`, red real sin mock.
- **Tests independientes**: no depender del orden de ejecución.
- **Un test, un concepto**: si falla, sabes exactamente qué se rompió.
- **Nombres descriptivos**: `it("falla con email inválido")` no `it("test 1")`.
- **No testees implementación**: testea comportamiento. Si renombras una variable privada, el test no debe romperse.
- **Cubre el camino feliz, los edge cases y los errores**, no persigas 100% de coverage.
- **Coverage es una métrica, no un objetivo**. 80% con tests buenos > 100% con tests que solo llaman funciones.
- **Tests deben ser rápidos**: si un unit tarda > 100ms, algo está mal.
- **Refactoriza los tests**: código duplicado en tests es código que nadie quiere tocar.
- **No testees frameworks**: no tests para `useState`, sí para tu lógica que usa `useState`.
- **Snapshot con cuidado**: solo para serializaciones pequeñas y estables.

---

## Cuándo NO testear

- **Código de configuración pura**: si un archivo es solo constantes, no aporta valor testearlo.
- **Wrappers triviales**: `function doble(x) { return x * 2 }` se puede testear, pero el ROI es bajo.
- **Código de terceros**: no testees React, testea tu uso de React.
- **UI puramente estética**: usa visual regression, no unit tests.

---

## CI/CD y tests

### Pipeline típico

```
PR abierto
  ├── lint
  ├── type check
  ├── unit + integration tests
  └── build

Merge a main
  ├── todo lo anterior
  ├── e2e tests
  └── deploy a staging

Manual approval → deploy a producción
```

### Estrategias para CI rápido

- **Cache** de `node_modules`/`pip`.
- **Paralelización** con `pytest-xdist` o sharding de Playwright.
- **Test selection**: solo correr tests afectados por el cambio (Nx, Turborepo).
- **Fail fast**: si lint falla, no seguir.

---

## Ejercicios

### Ejercicio 1 — Unit tests de una kata

Toma una kata clásica (FizzBuzz, Roman Numerals, Bowling Score) y:

1. Escribe unit tests **antes** de la implementación (TDD).
2. Cubre el camino feliz, los edge cases y los errores.
3. Mide coverage. ¿Llegaste al 100%? ¿Tiene sentido?

### Ejercicio 2 — Integration con DB

Toma un repositorio con PostgreSQL:

1. Configura testcontainers.
2. Escribe tests que persisten y leen datos reales.
3. Implementa un `InMemoryRepository` alternativo y comprueba que los tests pasan con ambos.

### Ejercicio 3 — E2E con Playwright

Implementa un e2e que recorra el flujo principal de una app:

1. Signup → login → crear recurso → logout.
2. Usa Page Object Model.
3. Configura trace y video.
4. Hazlo correr en paralelo con otro test.

### Ejercicio 4 — Mocking vs Fakes

Tienes un servicio que envía emails. Implementa el mismo set de tests con:

1. Mock del servicio de email.
2. Fake in-memory que registra los envíos.

Discute cuándo prefieres cada uno.

### Ejercicio 5 — Property-based testing

Toma una función de cálculo (descuentos, totales, conversiones) y:

1. Escribe properties con Hypothesis.
2. Deja que Hypothesis encuentre un caso que rompe tu implementación.
3. Corrige el bug y verifica que el property vuelve a pasar.

### Ejercicio 6 — Cobertura significativa

Toma un proyecto con 40% de coverage. Súbelo a 80%:

1. Identifica qué **no** está cubierto y por qué.
2. Decide qué merece tests y qué no.
3. Añade tests solo donde aportan valor.

### Ejercicio 7 — Contract testing

Simula frontend y backend en el mismo repo:

1. Frontend genera un contrato Pact.
2. Backend verifica que cumple el contrato.
3. Rompe el backend a propósito y observa el fallo.

### Ejercicio 8 — Visual regression con Playwright

Implementa tests que:

1. Capturan screenshots de páginas clave.
2. Comparan contra baseline.
3. Fallan cuando hay un cambio visual no intencionado.

---

## Problema propuesto

### Sistema de testing completo para un SaaS

Debes diseñar e implementar la **estrategia de testing** de un SaaS multi-tenant con:

- API REST/GraphQL en Node (o Python).
- Frontend React.
- Worker de procesamiento de eventos.
- DB PostgreSQL.
- Redis.
- Kafka (o NATS).

### Requisitos funcionales

1. **Unit tests** con al menos 80% de coverage en el dominio (entities, value objects, servicios de dominio).
2. **Integration tests** que cubran los casos de uso principales con DB real (testcontainers).
3. **E2E tests** con Playwright que cubran los 3 flujos críticos del producto.
4. **Contract tests** entre API y frontend (o entre dos servicios).
5. **Tests de mutación** (Stryker o mutmut) en al menos el módulo de dominio.
6. **CI pipeline** que corra todo y falle rápido.

### Requisitos técnicos

Tu solución debe:

1. Estructura de tests:
   ```
   tests/
     unit/
     integration/
     e2e/
     contract/
     setup/
   ```

2. **Unit**:
   - Domain entities con 100% coverage.
   - Use cases con mocks solo del mundo exterior.
   - Sin DB, sin red.

3. **Integration**:
   - Repository tests con testcontainers (PostgreSQL real).
   - API tests con Supertest o TestClient.
   - Redis tests con contenedor o mock según convenga.

4. **E2E (Playwright)**:
   - 3 flujos críticos completos.
   - Page Object Model.
   - Trace y video en fallos.
   - Correr en 2 navegadores (Chromium + WebKit).

5. **Contract**:
   - Frontend y backend verificados contra el mismo schema (Pact o Schemathesis).

6. **CI**:
   - GitHub Actions o similar.
   - Cache de dependencias.
   - Paralelización.
   - Reporte de coverage.
   - Falla el PR si coverage baja más de un umbral.

### Entregables

- Estructura completa de tests del proyecto.
- Configuración de cada framework.
- Scripts del CI.
- Reporte de coverage (HTML + badge).
- Documento de 1 página justificando:
  - Por qué esa distribución de tests (pirámide).
  - Qué NO testeas y por qué.
  - Cómo decides qué test es unit vs integration vs e2e.
  - Qué harías distinto si el equipo fuera 1 persona vs 10.
  - Cuánto tarda el pipeline y cómo lo harías 3x más rápido.

### Criterios de evaluación

| Criterio                                          | Peso |
| ------------------------------------------------- | ---- |
| Unit tests de calidad (comportamiento, no impl.)  | 20%  |
| Integration con DB real                          | 15%  |
| E2E flujos críticos con Playwright               | 20%  |
| Contract testing                                 | 10%  |
| CI pipeline funcional y rápido                   | 15%  |
| Documentación de la estrategia                   | 10%  |
| Cobertura significativa (>80% dominio)           | 10%  |
