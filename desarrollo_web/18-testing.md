# Módulo 18: Testing

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Diferenciar los niveles de testing y su rol.
- Escribir tests unitarios, de integración y end-to-end.
- Configurar Jest y Vitest como test runners.
- Crear mocks y stubs efectivos.
- Automatizar tests end-to-end con Cypress y Playwright.
- Aplicar buenas prácticas de cobertura y aislamiento.

---

## 1. Testing manual vs automatizado

### Manual

- Exploración humana, decisiones subjetivas (UX).
- Útil para validar flujos nuevos aún no definidos.
- **No escala** y es propenso a omisiones.

### Automatizado

- Repetible, ejecutable en CI.
- Documenta el comportamiento esperado.
- **Escala** con el proyecto.

### Pirámide de testing

```
       ▲
      /E2E\            pocos, costosos, alta confianza
     /─────\
    / INT  \          medianos, componentes completos
   /────────\
  /  UNIT    \        muchos, rápidos, aislados
 /────────────\
```

| Nivel | Velocidad | Confianza | Costo | Cantidad |
| ----- | --------- | --------- | ----- | -------- |
| Unit  | ms        | media     | bajo  | Muchos   |
| Integr.| s        | alta      | medio | Algunos  |
| E2E   | lento     | muy alta  | alto  | Pocos    |

### Regla del 70/20/10

- ~70% unit tests.
- ~20% integration.
- ~10% e2e.

### Actividad práctica

- Identifica qué nivel aplicar a una feature nueva (ej. auth, drag & drop, formulario).

---

## 2. Unit Testing

Prueban **una unidad de código** (función, clase) **aislada**.

### F.A.I.R. / F.I.R.S.T.

- **Fast**, **Independent**, **Repeatable**, **Self-validating**, **Timely**.

### Estructura AAA (Arrange, Act, Assert)

```js
test("suma 2 + 3 = 5", () => {
  // Arrange
  const a = 2, b = 3;
  // Act
  const result = sum(a, b);
  // Assert
  expect(result).toBe(5);
});
```

### Características de un buen unit test

- **Una sola aserción conceptual**.
- **No depende de orden** (puede ejecutarse solo).
- **No toca red, disco, reloj del sistema**.
- **Mockear** lo externo.

### Ejemplo: carrito

```js
function calcTotal(items, discounts = {}) {
  return items.reduce((acc, i) => {
    const price = i.price * (discounts[i.id] ?? 1);
    return acc + price * i.qty;
  }, 0);
}

test("calcTotal con descuento", () => {
  const items = [
    { id: 1, price: 100, qty: 2 },
    { id: 2, price: 50,  qty: 1 },
  ];
  const total = calcTotal(items, { 1: 0.9 });
  expect(total).toBe(100 * 0.9 * 2 + 50 * 1);
});
```

### Tests parametrizados

```js
test.each([
  [0, 0],
  [1, 1],
  [2, 1],
  [3, 2],
])("fibonacci(%i) = %i", (input, expected) => {
  expect(fib(input)).toBe(expected);
});
```

### Cobertura de bordes

- Vacío, null, undefined.
- Arrays de 1 elemento.
- Strings largos, caracteres especiales.
- Muy grandes, muy negativos.
- Concurrencia (si aplica).

### Actividad práctica

- Escribe tests para un parser de CSV cubriendo casos normales, vacíos, errores y caracteres especiales.

---

## 3. Integration Testing

Validan **varias unidades juntas**: interacción entre servicios, repositorios, bases de datos, HTTP.

### Backend

```js
// tests/integration/products.api.test.js
import request from "supertest";
import app from "../../src/app.js";
import { sequelize } from "../../src/db.js";

beforeAll(async () => {
  await sequelize.sync({ force: true });
});

afterAll(async () => {
  await sequelize.close();
});

test("GET /api/products returns empty array initially", async () => {
  const res = await request(app).get("/api/products");
  expect(res.status).toBe(200);
  expect(res.body.data).toEqual([]);
});

test("POST /api/products creates a product", async () => {
  const res = await request(app)
    .post("/api/products")
    .send({ nombre: "Laptop", precio: 1200, stock: 5 })
    .set("Authorization", `Bearer ${token}`);

  expect(res.status).toBe(201);
  expect(res.body.nombre).toBe("Laptop");
});
```

### Bases de datos de prueba

```js
// jest.config.js
{
  testEnvironment: "node",
  setupFilesAfterEach: ["./tests/setup.js"],
}

beforeEach(async () => {
  await sequelize.truncate({ cascade: true });
});
```

### Containers efímeros (Docker)

```js
import { GenericContainer } from "testcontainers";

const container = await new GenericContainer("postgres:16")
  .withExposedPorts(5432)
  .withEnvironment({ POSTGRES_PASSWORD: "test" })
  .start();

const url = `postgres://postgres:test@${container.getHost()}:${container.getMappedPort(5432)}/test`;
```

### Test doubles

- **Stub**: respuestas predefinidas.
- **Mock**: verifica interacciones.
- **Fake**: implementación simplificada (in-memory DB).
- **Spy**: registra llamadas sin alterar.

```js
const mailerSpy = {
  sent: [],
  sendWelcome(email) { this.sent.push(email); },
};

await userService.register({ email: "a@x.com", password: "secret123" });
expect(mailerSpy.sent).toContain("a@x.com");
```

### Actividad práctica

- Implementa un test de integración del endpoint `/api/products` con `supertest` y DB en memoria SQLite.

---

## 4. End to End (E2E)

Prueban **flujos completos** desde la perspectiva del usuario.

### Características

- Lentos, frágiles, costosos.
- Alta confianza (cubren stack completo).
- Críticos para flujos críticos (pagos, signup).

### Patrón "Page Object Model"

```js
// pages/LoginPage.js
export class LoginPage {
  constructor(page) { this.page = page; }
  async goto()     { await this.page.goto("/login"); }
  async login(user, pwd) {
    await this.page.fill('[name="email"]', user);
    await this.page.fill('[name="password"]', pwd);
    await this.page.click('button[type="submit"]');
  }
}
```

### Datos de prueba

- Crear usuarios cada test o usar fixture.
- Sembrar DB antes con un seeder (`seed:test`).

### Actividad práctica

- Escribe un E2E que complete un flujo de login + agregar producto al carrito + checkout.

---

## 5. Jest

El test runner más popular en JavaScript/Node.

### Instalación

```bash
npm install -D jest @types/jest
```

### `package.json`

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

### Config

```js
// jest.config.js
export default {
  testEnvironment: "node", // "jsdom" para frontend
  setupFilesAfterEach: ["<rootDir>/tests/setup.js"],
  testMatch: ["**/?(*.)+(test|spec).js"],
  coverageThreshold: {
    global: { statements: 80, branches: 70, functions: 80, lines: 80 },
  },
};
```

### Matchers

```js
expect(x).toBe(5);
expect(x).toEqual({ a: 1 });
expect(x).toBeNull();
expect(x).toBeUndefined();
expect(x).toBeTruthy();
expect(x).toBeFalsy();
expect(arr).toContain("x");
expect(arr).toHaveLength(3);
expect(fn).toThrow();
expect(m).toMatchObject({ id: 1 });
expect(m).toMatchSnapshot();
```

### Mocking

```js
import { sendMail } from "../mailer.js";

jest.mock("../mailer.js");

test("sends email on register", async () => {
  sendMail.mockResolvedValue({ id: "abc" });
  // ...
  expect(sendMail).toHaveBeenCalledWith("a@x.com", expect.any(String));
});
```

### Spies

```js
const obj = { greet: () => "hi" };
const spy = jest.spyOn(obj, "greet");
obj.greet("Ana");
expect(spy).toHaveBeenCalledWith("Ana");
```

### Timers

```js
jest.useFakeTimers();
jest.advanceTimersByTime(1000);
jest.runAllTimers();
```

### Hooks

```js
beforeAll(() => {});      // una vez por archivo
afterAll(() => {});
beforeEach(() => {});     // antes de cada test
afterEach(() => {});
```

### Setup global

```js
// tests/setup.js
beforeEach(() => {
  jest.clearAllMocks();
});
```

### TS

```bash
npm install -D ts-jest @types/jest
```

```json
{
  "preset": "ts-jest",
  "testEnvironment": "node"
}
```

### Async patterns

```js
test("resolves", async () => {
  await expect(api()).resolves.toEqual({ ok: true });
});

test("rejects", async () => {
  await expect(api()).rejects.toThrow("Network");
});
```

### Cobertura

```bash
npm test -- --coverage
```

Salida en consola + `coverage/index.html`.

### Actividad práctica

- Configura Jest en un proyecto con `setupFiles`, mocks y un test de integración con supertest.

---

## 6. Vitest

Alternativa moderna, compatible con sintaxis de Jest, construida sobre Vite.

### Instalación

```bash
npm install -D vitest
```

### `package.json`

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage"
  }
}
```

### Config

```js
// vitest.config.js
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: "./tests/setup.js",
    coverage: {
      provider: "v8",
      reporter: ["text", "html"],
    },
  },
});
```

### Sintaxis idéntica a Jest

```js
import { describe, it, expect, vi } from "vitest";
import { sum } from "./sum";

describe("sum", () => {
  it("adds two numbers", () => {
    expect(sum(2, 3)).toBe(5);
  });
});
```

### Mocking

```js
import { vi } from "vitest";

const sendMail = vi.fn();
vi.mock("./mailer.js", () => ({ sendMail }));

sendMail.mockResolvedValue({ id: "x" });

expect(sendMail).toHaveBeenCalledWith("a@x.com");
```

### Watch UI

```bash
vitest --ui
```

### Compatibilidad

Vitest es compatible con la mayoría de la API de Jest. Migración suele ser cambiar imports y configuración.

### Vitest vs Jest

| Característica  | Jest                | Vitest              |
| --------------- | ------------------- | ------------------- |
| Velocidad       | Moderada            | **Más rápido**       |
| ESM nativo      | Requiere Babel/TS   | **Sí** (Vite-like)   |
| Watch mode      | Sí                  | **Más rápido**       |
| UI              | externa             | **`--ui`**           |
| Matchers        | Compatibles         | Compatibles         |
| Mocking         | `jest.mock`         | **`vi.mock`**       |
| Cobertura       | istanbul/v8         | v8/istanbul         |

### Actividad práctica

- Migra un test de Jest a Vitest en un proyecto y compara tiempos.

---

## 7. Cypress

Framework E2E para navegadores web.

### Instalación

```bash
npm install -D cypress
npx cypress open
```

### Estructura

```
cypress/
├── e2e/
│   ├── login.cy.js
│   └── products.cy.js
├── fixtures/
├── support/
│   ├── commands.js
│   └── e2e.js
└── videos/screenshots/ (auto)
```

### `cypress.config.js`

```js
import { defineConfig } from "cypress";

export default defineConfig({
  e2e: {
    baseUrl: "http://localhost:5173",
    supportFile: "cypress/support/e2e.js",
    video: false,
    screenshotOnRunFailure: true,
  },
});
```

### Primer test

```js
// cypress/e2e/login.cy.js
describe("login", () => {
  it("allows valid user to login", () => {
    cy.visit("/login");
    cy.get('[name="email"]').type("ana@x.com");
    cy.get('[name="password"]').type("secret123");
    cy.get('button[type="submit"]').click();
    cy.url().should("include", "/dashboard");
    cy.contains("Bienvenida, Ana");
  });
});
```

### API principal

```js
cy.visit("/");
cy.get("selector");       // query
cy.contains("texto");
cy.click();
cy.type("texto");
cy.check();
cy.select("value");
cy.wait(1000);
cy.url().should("include", "/x");
cy.get(".alert").should("be.visible");
cy.get(".btn").should("not.be.disabled");

cy.intercept("GET", "/api/*", { fixture: "users.json" });
cy.intercept("POST", "/api/login", { statusCode: 200, body: { token: "abc" } });

cy.request("POST", "/api/login", { email, password });
cy.getCookie("token");

cy.clock();
cy.tick(1000);
```

### Custom commands

```js
// support/commands.js
Cypress.Commands.add("login", (email, password) => {
  cy.request("POST", "/api/auth/login", { email, password })
    .its("body.accessToken")
    .then((token) => {
      window.localStorage.setItem("accessToken", token);
    });
});

Cypress.Commands.add("dataCy", (value) => cy.get(`[data-cy="${value}"]`));
```

```js
// uso
cy.login("a@x.com", "secret123");
cy.dataCy("submit").click();
```

### Plugins

```js
// cypress.config.js
setupNodeEvents(on, config) {
  require("cypress-plugin-retries/lib/plugin")(on, config);
}
```

### Reintentos

```js
{
  retries: { runMode: 2, openMode: 0 }
}
```

### Actividad práctica

- Crea un test E2E de signup + login + acciones en dashboard.

---

## 8. Playwright

Framework E2E multi-navegador moderno, de Microsoft.

### Instalación

```bash
npm install -D @playwright/test
npx playwright install
```

### `playwright.config.ts`

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,
  reporter: process.env.CI ? "github" : "list",
  use: {
    baseURL: "http://localhost:5173",
    trace: "retain-on-failure",
    screenshot: "only-on-failure",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox",  use: { ...devices["Desktop Firefox"] } },
    { name: "webkit",   use: { ...devices["Desktop Safari"] } },
  ],
  webServer: {
    command: "npm run dev",
    url: "http://localhost:5173",
    reuseExistingServer: !process.env.CI,
  },
});
```

### Primer test

```ts
// tests/e2e/login.spec.ts
import { test, expect } from "@playwright/test";

test("user can log in", async ({ page }) => {
  await page.goto("/login");
  await page.fill('input[name="email"]', "ana@x.com");
  await page.fill('input[name="password"]', "secret123");
  await page.click('button[type="submit"]');

  await expect(page).toHaveURL(/\/dashboard/);
  await expect(page.getByText("Bienvenida, Ana")).toBeVisible();
});
```

### Locators

```ts
page.getByRole("button", { name: "Enviar" });
page.getByText("Hola");
page.getByLabel("Email");
page.getByPlaceholder("Buscar...");
page.getByTestId("submit-button");

page.locator(".card");
page.locator("xpath=//a[contains(text(), 'Login')]");
```

### Esperas

```ts
await page.waitForURL(/\/dashboard/);
await expect(page.getByText("Lista")).toBeVisible();

await page.waitForResponse((r) => r.url().includes("/api/products") && r.status() === 200);

await page.waitForLoadState("networkidle");
```

### Mocking con `route`

```ts
await page.route("**/api/products", (route) =>
  route.fulfill({ status: 200, body: JSON.stringify([{ id: 1, name: "Laptop" }]) })
);
```

### Custom fixtures

```ts
// fixtures/auth.ts
import { test as base } from "@playwright/test";

export const test = base.extend({
  loginAs: async ({ page }, use) => {
    await use(async (email, pwd) => {
      await page.goto("/login");
      await page.fill('[name="email"]', email);
      await page.fill('[name="password"]', pwd);
      await page.click('button[type="submit"]');
    });
  },
});
```

```ts
// uso
test("...", async ({ page, loginAs }) => {
  await loginAs("ana@x.com", "secret123");
});
```

### Visual regression

```ts
await expect(page).toHaveScreenshot();
```

### Traces

```bash
npx playwright test --trace=on
npx playwright show-report
```

### Ventajas vs Cypress

| Característica      | Cypress           | Playwright             |
| ------------------- | ----------------- | ---------------------- |
| Navegadores         | Chromium-family   | **Chromium, Firefox, WebKit** |
| Lenguaje            | JS (Cypress DSL)  | **JS, TS, Java, Python** |
| Multi-tab           | Limitado          | **Nativo**             |
| Shadow DOM          | Limitado          | **Sí**                 |
| Velocidad           | Moderada          | **Más rápido**         |
| Headless config     | Requiere plugin   | **Built-in**           |
| Auto-wait           | Sí                | **Mejor implementado** |

### Actividad práctica

- Configura Playwright con 3 navegadores y crea un test del flujo de login + dashboard.

---

## Estrategia y patrones de testing

### Cobertura de código

- **Statements**: líneas ejecutadas.
- **Branches**: caminos condicionales.
- **Functions**: funciones llamadas.
- **Lines**: líneas reales (suele coincidir con statements).

```bash
npm test -- --coverage
```

Threshold saludable: 70-80% combinado. 100% no garantiza calidad.

### Qué probar

- **Casos de éxito**.
- **Casos de error esperados**.
- **Validaciones**.
- **Bordes**.

### Qué NO probar

- Librerías externas (jest matchers, bcrypt, etc.).
- Trivial getters/setters.
- Boilerplate que cambia constantemente.

### TDD (Test-Driven Development)

```
1. 🔴 Test fallido
2. 🟢 Mínima implementación que pasa
3. 🔵 Refactor
```

### BDD (Behavior-Driven Development)

```js
describe("Como usuario", () => {
  it("debería poder registrarme con email válido", () => {});
});
```

### Pirámide inversa (testing trophy moderna)

```
       E2E
     ┌─────┐
     │     │   algunos
     ├─────┤
     │ INT │   muchos (proporcional al valor)
     ├─────┤
     │ UNIT│   mayoría
     └─────┘
```

### Test isolation

- **Reset DB** entre tests (truncate).
- Evitar orden implícito.
- Usar IDs aleatorios para datos únicos.

### Flaky tests

Causas:
- Timeouts demasiado cortos.
- Esperas por `setTimeout` en vez de eventos.
- Datos compartidos.

Solución:
- `await` estados de Playwright/Cypress.
- `waitFor` específicos.
- Reintentos solo en CI.

### Buenas prácticas

- Nombrar tests descriptivamente:
  - `it("returns 404 when product doesn't exist")`.
- **Una cosa por test**.
- AAA o Given-When-Then.
- Mocks solo lo necesario.
- Test data factories.

```js
function makeUser(overrides = {}) {
  return {
    email: "a@x.com",
    name: "Ana",
    role: "user",
    ...overrides,
  };
}
```

### Snapshot testing

```js
import { render } from "@testing-library/react";
import Button from "./Button";

test("renders correctly", () => {
  const { container } = render(<Button>Click</Button>);
  expect(container).toMatchSnapshot();
});
```

> Cuidado: snapshots actualizados sin revisar pueden enmascarar bugs.

### Mocking del tiempo

```js
vi.useFakeTimers();
vi.advanceTimersByTime(1000);
vi.runOnlyPendingTimers();
```

### Mutation testing (avanzado)

Stryker muta tu código y verifica que los tests fallen cuando se introduce un bug artificial.

```bash
npx stryker run
```

### Actividad práctica

- Identifica en tu proyecto qué unidades son candidatas a test unit, integration y e2e.

---

## Setup completo de un proyecto

### Backend (Jest + Supertest)

```json
// package.json
{
  "scripts": {
    "test": "node --experimental-vm-modules node_modules/jest/bin/jest.js",
    "test:watch": "node --experimental-vm-modules node_modules/jest/bin/jest.js --watch",
    "test:coverage": "node --experimental-vm-modules node_modules/jest/bin/jest.js --coverage",
    "test:e2e": "node --experimental-vm-modules node_modules/jest/bin/jest.js tests/e2e"
  }
}
```

```js
// jest.config.js
export default {
  testEnvironment: "node",
  transform: {},
  testMatch: ["**/*.test.js"],
  setupFilesAfterEach: ["<rootDir>/tests/setupAfterEach.js"],
};
```

```js
// tests/setupAfterEach.js
import { sequelize } from "../src/db.js";
afterEach(async () => {
  await sequelize.truncate({ cascade: true });
});
```

### Frontend (Vitest + RTL)

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: "./tests/setup.ts",
  },
});
```

```ts
// tests/setup.ts
import "@testing-library/jest-dom";
```

```tsx
// Counter.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, it, expect } from "vitest";
import { Counter } from "./Counter";

describe("Counter", () => {
  it("increments", async () => {
    render(<Counter />);
    await userEvent.click(screen.getByRole("button", { name: "+" }));
    expect(screen.getByText("1")).toBeInTheDocument();
  });
});
```

### E2E (Playwright)

```bash
npx playwright install --with-deps
```

```bash
npm run test:e2e
```

### CI (GitHub Actions)

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: "npm" }
      - run: npm ci
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run build
      - run: npx playwright install --with-deps
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report
```

---

## Recursos complementarios

### Lecturas

- [Jest Docs](https://jestjs.io/)
- [Vitest Docs](https://vitest.dev/)
- [Cypress Docs](https://docs.cypress.io/)
- [Playwright Docs](https://playwright.dev/)
- [Testing JavaScript - Kent C. Dodds](https://testingjavascript.com/) (de pago)
- [JavaScript Testing Best Practices](https://github.com/goldbergyoni/javascript-testing-best-practices)

### Cursos y tutoriales

- [Testing With Vitest (Video)](https://www.youtube.com/results?search_query=vitest+tutorial)
- [Playwright with TypeScript (Video)](https://www.youtube.com/results?search_query=playwright+typescript)

### Herramientas

- **MSW (Mock Service Worker)**: mocks de red realistas para frontend.
- **nock**: mocks HTTP para Node.
- **Sinon**: stubs/spies standalone.
- **nyc**: cobertura para Node.
- **Stryker**: mutation testing.

---

## Evaluación sugerida

1. **Unit tests (25%)**: lógica de negocio pura (servicios, helpers).
2. **Integration tests (25%)**: endpoints HTTP con supertest o MSW.
3. **E2E (30%)**: flujos críticos con Cypress o Playwright (multi-navegador).
4. **Configuración (10%)**: setup correcto con cobertura ≥ 70%.
5. **CI (10%)**: pipeline que ejecute unit + e2e en cada PR.

---

> **Nota para el estudiante**: tests son una **inversión**, no un gasto. Empizan a pagar dividendos cuando refactorizas con confianza. Empieza con pocos tests importantes e itera. La mejor suite es la que **se ejecuta siempre**, no la que tiene 100% de cobertura.
