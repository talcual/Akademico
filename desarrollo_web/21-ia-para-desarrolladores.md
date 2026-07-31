# Módulo 21: IA para Desarrolladores

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Aplicar técnicas de prompt engineering en el contexto de desarrollo.
- Usar IA de manera ética, segura y responsable.
- Sacar partido a GitHub Copilot, ChatGPT y similares.
- Comprender agentes de código y su valor en workflows.
- Conocer el Model Context Protocol (MCP) para herramientas.
- Generar documentación y automatizar tareas con IA.

---

## 1. Prompt Engineering

El **prompt engineering** es la disciplina de diseñar instrucciones para modelos de lenguaje (LLM) de manera que produzcan resultados útiles y precisos.

### Elementos básicos

- **Rol**: quién debe "ser" el modelo.
- **Tarea**: qué debe hacer.
- **Contexto**: datos relevantes.
- **Restricciones**: qué no debe hacer.
- **Formato**: cómo entregar el resultado.
- **Ejemplos**: input → output esperado.

### Ejemplo simple

```text
Actúa como un senior frontend developer.
Convierte el siguiente JSON en una interface TypeScript:

{
  "id": 1,
  "name": "Ana",
  "active": true,
  "tags": ["js", "css"]
}
```

Resultado esperado:

```ts
interface User {
  id: number;
  name: string;
  active: boolean;
  tags: string[];
}
```

### Técnicas avanzadas

#### Zero-shot
Sin ejemplos, solo instrucción.

#### Few-shot
Incluir varios pares input-output.

#### Chain-of-Thought (CoT)
"Piensa paso a paso antes de responder".

#### ReAct
Razonamiento + acciones (invocar herramientas).

#### Self-Consistency
Generar N respuestas y elegir la más frecuente.

### Estructura recomendada

```text
[SISTEMA]: Eres un experto en X. Responde en formato Y. Usa un tono conciso. 

[USUARIO]: Quiero Z. Contexto: W. Restricciones: A, B. 
Ejemplo de salida esperada: ...
```

### Patrones para código

```text
- "Genera código en JavaScript ES2022 con TypeScript."
- "Sin usar librerías externas."
- "Incluye manejo de errores."
- "Escribe 3 ejemplos, cada uno con un caso borde."
- "Devuelve solo el código en un bloque markdown."
```

### Iteración

```text
Prompt inicial → Evaluar → Pedir aclaraciones:
- "Más conciso"
- "Usar X en lugar de Y"
- "Añadir tests"
- "¿Cómo manejar caso Z?"
```

### Temperatura (en APIs)

```js
const completion = await openai.chat.completions.create({
  model: "gpt-4",
  temperature: 0,    // más determinista
  messages: [...],
});
```

- `0`: respuestas deterministas, mejor para código.
- `0.7-1`: creativo, útil para brainstorming.

### Tokens

- 1 token ≈ 4 caracteres en inglés, ~1-2 en español.
- Contexto máximo varía (4K, 8K, 32K, 128K, 200K).
- Más tokens = más caro y lento.

### Actividad práctica

- Toma una tarea real de tu trabajo y crea 3 versiones de prompt: zero-shot, few-shot y con CoT.

---

## 2. Uso responsable de IA

La IA es poderosa pero **no infalible**. Usarla responsablemente es clave.

### Riesgos

- **Alucinaciones**: inventa datos con seguridad.
- **Información desactualizada**.
- **Sesgos**.
- **Privacidad**: subir datos sensibles a servicios externos.
- **Derechos de autor**: código sugerido puede tener licencia dudosa.
- **Dependencia**: pérdida de habilidades propias.

### Buenas prácticas

#### Privacidad

- **No envíes secretos** (API keys, passwords, datos personales).
- PII (información personalmente identificable) **anonimizada** antes.
- Usa empresas privadas o self-hosted cuando aplique.

```text
❌ "Aquí está el archivo users.csv con 500k registros, dime qué anomalías hay."
✅ "Dame un patrón genérico para detectar anomalías en CSVs con campos X, Y, Z."
```

#### Verificación

**Nunca** subas código a producción sin leerlo y entenderlo.

- *"Trust but verify"*: la IA es un colega junior, no un senior.
- Tests manuales.
- Code review humano.

#### Propiedad intelectual

- Revisa políticas de tu empresa.
- Open source vs código de empresa.
- Licencias.

#### Transparencia

- En código generado, documenta qué partes vienen de IA.
- En productos, avisa a usuarios cuando un chatbot es IA.

#### Atribución en trabajos académicos

- Si tu universidad prohíbe IA, respétalo.

#### Medidas de protección

- Filtros de PII antes de subir.
- Auditorías internas.
- Logs de uso.

### Actividad práctica

- Identifica qué datos de tu proyecto NO deberías compartir nunca con una IA.

---

## 3. GitHub Copilot

Copilot es un asistente de IA integrado en el editor (VS Code, JetBrains, etc.).

### Funciones

- **Autocompletado inline** (sugerencias fantasma).
- **Chat** lateral (similar a ChatGPT).
- **Edit** (refactor con IA).
- **Workspace** (multi-archivo).
- **Agents** (terminal, navegación, análisis).
- **PR reviews** automáticas.

### Activación

- GitHub → Settings → Copilot → activar.
- VS Code → Install extension → login.

### Atajos clave (VS Code)

- `Tab` → aceptar sugerencia.
- `Esc` → rechazar.
- `Ctrl + Enter` → abrir panel de alternativas.
- `Ctrl + I` → Inline Chat.
- `Ctrl + L` → abrir chat lateral.
- `Ctrl + Shift + I` → Copilot en terminal.

### Inline suggestions

```js
function fibonacci(n) {
  // sugerir recursión con memo
}
```

Copilot autocompleta lo que escribes.

### Chat

```text
"¿Cómo implementar este endpoint con validación?"
"Genera unit tests para esta función."
"Explícame qué hace este código."
```

### Inline chat

```js
// selecciona el código
// Ctrl + I → "Refactor this to async/await"
```

### Copilot Workspace

Trabaja sobre **planes multi-paso** que editan varios archivos. Útil para features medianas.

### Buenas prácticas

#### Aceptar / rechazar

Acepta lo que **entiendes**. Si es complejo, modifica.

#### Contexto

Añade comentarios descriptivos:

```js
// Calcula el total con descuento, impuestos y envío según la región del usuario.
// Devuelve un objeto {subtotal, discount, tax, shipping, total}.
```

#### Documentar

- Genera docstrings y JSDoc.
- Commit messages (Conventional Commits).

#### Tests

```js
// Test the fibonacci function:
test("...")
```

### Configuración

```json
// settings.json
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "text": "Always use TypeScript with strict mode." }
  ]
}
```

### Copilot para empresas

- Exclusión de archivos sensibles.
- Lineage de código.
- Sin entrenamiento con tu código.

### Actividad práctica

- En una función, escribe 3 variaciones con ayuda de Copilot. Compara y selecciona la mejor.

---

## 4. ChatGPT / LLMs en el flujo de desarrollo

### Casos de uso comunes

#### Diseño

- Brainstorming de features.
- Análisis de trade-offs.
- Diagramas (Mermaid, draw.io).

#### Refactoring

```text
"Refactoriza esta clase a funciones puras separadas."
```

#### Debugging

```text
"¿Por qué este código produce X en lugar de Y?"
```

#### Testing

- Generar tests.
- Identificar casos no cubiertos.

```text
"Casos de prueba borde para esta función de validación."
```

#### Documentación

```text
"Genera un JSDoc completo para esta función."
"Crea un README.md con badges, quick-start y ejemplos."
```

#### Code review

```text
"Encuentra issues de seguridad en este código."
"¿Hay memory leaks en este componente React?"
```

#### Explicación

```text
"Explica este regex: (?<=^|\\s)[a-zA-Z]+"
```

#### Conversión entre lenguajes

```text
"Convierte este script Bash a Node.js con ESM."
```

#### Generar código boilerplate

```text
"Genera un Dockerfile multi-stage con Node 20 Alpine."
```

#### Conversación sostenida

Los LLMs mantienen contexto: puedes pedir correcciones iterativas.

#### Prompt caching

Reutiliza contexto largo con `cache_control` (Anthropic) o prompts estructurados.

### Límites a respetar

- **Crítico**: lógica de negocio, decisiones de arquitectura.
- **Mejor acompañado**: prototipos, ideas iniciales.

### Herramientas similares

- **Claude** (Anthropic): razonamiento extenso, mejor para código.
- **Gemini** (Google): multimodal, contexto enorme.
- **Perplexity**: respuestas con citas.
- **Mistral**: open weights.
- **Ollama**, **LM Studio**: LLMs locales.
- **Claude Code**, **Cursor**, **Windsurf**: editores con IA integradas.

### Actividad práctica

- Toma un bug difícil de tu proyecto y consulta 2 herramientas diferentes. Compara respuestas.

---

## 5. Agentes de código

**Agentes de código** son LLM que ejecutan **acciones** en tu proyecto: editar archivos, correr tests, abrir PRs.

### Características

- Loop: planificar → actuar → observar → iterar.
- Acceso a herramientas: bash, filesystem, browser, GitHub.
- Logs de cada acción.
- Aprobación humana antes de cambios.

### Herramientas principales

#### OpenAI Codex / GPT-5 (CLI)

```bash
npm install -g @openai/codex
codex "add docstrings to all functions in src/"
```

#### Claude Code

```bash
npm install -g @anthropic-ai/claude-code
claude "refactor the auth module to use jose library"
```

#### Cursor (Editor con agentes)

- Edit multi-archivo.
- Codebase-wide context.
- YOLO mode (sin pedir permiso).

#### Aider

```bash
pip install aider-chat
aider --model claude-3-5-sonnet file.js
```

#### Cline / Continue (extensiones VS Code)

- Chat que puede ejecutar comandos.

### Casos de uso

- Migraciones grandes (TS → JS, framework X → Y).
- Tests a partir de código existente.
- Documentación derivada del código.
- Búsqueda de bugs con múltiples archivos.

### Riesgos

- **Costo**: tokens se disparan con codebase grande.
- **Calidad inconsistente**: puede romper features.
- **Seguridad**: agentes pueden leer secretos.

### Buenas prácticas

- Sandbox en rama de feature, no en main.
- Solo lectura en prod data.
- Limitar comandos permitidos (allowlist).
- **Revisión humana** de todo cambio.

### Agentes en CI

- **OpenAI Codex**, **GitHub Copilot Coding Agent** (experimental).
- Autos como TDD en PR.

### Actividad práctica

- Configura un agente (Codex o similar) en un proyecto pequeño. Pídele una tarea concreta y revisa el diff.

---

## 6. MCP (Model Context Protocol)

**MCP** es un protocolo abierto para conectar LLMs a **herramientas externas** de forma estandarizada.

### Concepto

```
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│   Cliente    │        │ Servidor MCP │        │ Herramienta  │
│   (LLM)      │ ─────▶ │ (tool)       │ ─────▶ │ Real         │
└──────────────┘        └──────────────┘        └──────────────┘
       ▲                       │                       │
       └───────────────────── Response ────────────────┘
```

### Beneficios

- Cualquier LLM con soporte MCP puede usar cualquier herramienta.
- Estandarizado: similar a USB-C para IA.
- Composabilidad.

### Servidores MCP populares

- **Filesystem**: lectura/escritura de archivos.
- **GitHub**: issues, PRs, código.
- **Postgres**: queries SQL.
- **Browser**: navegación automatizada.
- **Sentry**: errores.
- **Stripe**: gestión de pagos.
- **Notion**: gestión de docs.
- **Puppeteer/Playwright**: browser automation.

### Configurar en VS Code (Copilot)

```json
// .vscode/settings.json
{
  "github.copilot.chat.mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./src"]
    },
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  }
}
```

### Implementar un servidor MCP propio

```bash
npm install @modelcontextprotocol/sdk
```

```js
// server-mcp/index.js
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "example",
  version: "1.0.0",
}, { capabilities: { tools: {} } });

server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "sumar",
    description: "Suma dos números",
    inputSchema: {
      type: "object",
      properties: {
        a: { type: "number" },
        b: { type: "number" },
      },
      required: ["a", "b"],
    },
  }],
}));

server.setRequestHandler("tools/call", async (req) => {
  const { name, arguments: args } = req.params;
  if (name === "sumar") {
    return { content: [{ type: "text", text: String(args.a + args.b) }] };
  }
  throw new Error(`Tool not found: ${name}`);
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Cliente MCP

```js
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const transport = new StdioClientTransport({
  command: "node",
  args: ["./server-mcp/index.js"],
});
const client = new Client({ name: "claude", version: "0.1.0" }, { capabilities: {} });
await client.connect(transport);

const tools = await client.listTools();
const result = await client.callTool({ name: "sumar", arguments: { a: 2, b: 3 } });
```

### Buenas prácticas

- **Permisos mínimos** (read-only cuando posible).
- **Logs** de cada acción.
- **Validar inputs** como cualquier endpoint.
- **No exponer secretos** en respuestas.

### Actividad práctica

- Crea un servidor MCP simple (por ejemplo: "consultar clima" o "leer archivo") y configúralo en VS Code o Claude Desktop.

---

## 7. Generación de documentación

La IA puede generar y mantener documentación con humanos en el loop.

### README

```text
Genera un README.md para este proyecto Node.js con:
- Badges (build, coverage)
- Descripción clara
- Requisitos
- Instalación y uso
- Scripts
- Variables de entorno
- Estructura del proyecto
- Contribución
- Licencia
```

### JSDoc

```ts
/**
 * Calcula el precio final aplicando descuentos y tasas.
 *
 * @param {number} basePrice - Precio base
 * @param {number} discountPct - Porcentaje de descuento (0-1)
 * @param {number} taxRate - Tasa de impuestos (0-1)
 * @returns {Object} {subtotal, tax, total}
 * @throws {Error} Si los argumentos son negativos
 */
function calcFinalPrice(basePrice, discountPct, taxRate) { ... }
```

### Docstrings / Comentarios

```text
Añade comentarios significativos al siguiente código, sin comentar lo obvio.
```

### CHANGELOG

```text
A partir de los commits entre v1.0 y v2.0, genera CHANGELOG.md con formato Keep a Changelog.
```

### Doc API

```text
Genera una documentación OpenAPI 3.1 para estos endpoints REST.
```

```yaml
openapi: 3.1.0
info:
  title: Mi API
  version: 1.0.0
paths:
  /api/products:
    get:
      summary: Listar productos
      ...
```

### Tutoriales

```text
Crea un tutorial paso a paso para usar este feature.
```

### Mantener docs en sync

- Generar docs al hacer PR.
- Hook en CI que valide cobertura de doc.

### Herramientas

- **TypeDoc** (TS).
- **JSDoc** (JS).
- **Storybook** (componentes).
- **Docusaurus**, **VitePress** (sitios de docs).
- **Mintlify**, **GitBook**.

### Generación con IA + revisión humana

1. Genera documentación inicial con IA.
2. Persona técnica la revisa.
3. Ajustes.
4. Commits.

### Actividad práctica

- Genera el README de un proyecto tuyo usando un prompt estructurado. Revisa y mejora.

---

## 8. Automatización de tareas con IA

La IA brilla en tareas **repetitivas y bien definidas**.

### Tareas típicas

- **Code review** automatizado.
- **Detección de bugs** heurística.
- **Generación de tests**.
- **Migraciones** (sintaxis, framework).
- **Resumen de PRs** y commits.
- **Generación de changelogs**.
- **Llamadas asíncronas** desde PRs.

### GitHub Actions con Copilot Coding Agent

```yaml
on:
  pull_request:
    types: [opened]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Copilot review
        uses: github/copilot-coding-agent@v1
        with:
          ...
```

### CodeRabbit / PR-Agent

```yaml
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  codereview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: coderabbitai/coderabbit-action@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Hooks locales

```bash
# .husky/pre-commit
npx lint-staged
npx ai-commit-msg  # genera mensaje basado en cambios
```

### Generación de tests

```text
Crea tests unitarios para src/utils/calculate.ts cubriendo todos los caminos.
```

### Generación de fixtures

```text
Genera 30 productos de prueba realistas para usar como fixtures.
```

### Documentación interactiva

```text
Crea un chatbot que responda preguntas sobre este proyecto basándose en el README y código.
```

### Bots de triaje

En Slack/Discord, bots que resumen issues, asignan prioridades, sugieren PRs.

### Riesgos

- Costo de tokens.
- Latencia.
- Falsos positivos en reviews automatizados.
- Sobredependencia.

### Actividad práctica

- Configura una action automatizada (CodeRabbit o similar) en un repo y observa 2-3 PRs.

---

## Buenas prácticas y patrones de adopción

### Empezar por valor

- Asistir, no reemplazar.
- Tareas tediosas primero (migración, tests, docs).
- Medir impacto (tiempo ahorrado, calidad).

### Equipo

- Capacitación.
- Acuerdos sobre uso (qué se permite).
- Code review sigue siendo humano.

### Toolkit personal

| Tarea                       | Herramienta        |
| --------------------------- | ------------------ |
| Autocompletar               | Copilot            |
| Diseño / brainstorming      | ChatGPT, Claude    |
| Generación de tests        | Copilot Chat       |
| Búsqueda en codebase        | Cursor, Cody       |
| Agentes en terminal         | Claude Code, Codex |
| Documentación automática    | Mintlify, Docusaurus + IA |
| Code review                 | CodeRabbit         |

### Aprender las bases sigue siendo clave

> "La IA no convierte a alguien sin conocimientos en desarrollador senior. Convierte a un senior en un super-senior."

Mantén fundamentos: arquitectura, debugging a mano, lectura sin IA.

### Cultura de equipo

- Compartir prompts que funcionan.
- AI office hours / workshops.
- Lista de qué NO hacer (no compartir secretos, no automatizar decisiones legales).

### Actividad práctica

- Define un "AI policy" para tu equipo o estudio con 5 reglas claras.

---

## Recursos complementarios

### Lecturas

- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [OpenAI Cookbook](https://cookbook.openai.com/)
- [MCP Specification](https://modelcontextprotocol.io/)
- [GitHub Copilot Docs](https://docs.github.com/en/copilot)
- [AI for Web Dev - Houssein](https://www.youtube.com/results?search_query=ai+for+developers)

### Cursos y tutoriales

- [DeepLearning.AI Prompt Engineering](https://www.deeplearning.ai/short-courses/)
- [ChatGPT Prompt Engineering for Developers](https://www.deeplearning.ai/short-courses/chatgpt-prompt-engineering-for-developers/)

### Herramientas

- **Cursor**, **Windsurf**, **Zed** (editores con IA).
- **ChatGPT**, **Claude.ai**, **Gemini**.
- **Claude Code**, **OpenAI Codex CLI**.
- **CodeRabbit**, **PR-Agent**.
- **Continue**, **Cline** (VS Code).
- **v0.dev**, **Bolt.new**, **Lovable** (generadores UI).

---

## Evaluación sugerida

1. **Prompts (20%)**: tres prompts con técnicas distintas (zero-shot, few-shot, CoT).
2. **Copilot / flujo (25%)**: demostrar uso real con código antes/después.
3. **Agente (25%)**: ejecutar una tarea con un agente de código y auditarla.
4. **MCP (15%)**: crear un servidor MCP simple y conectarlo.
5. **Documentación (15%)**: README + JSDoc generados y revisados.

---

> **Nota para el estudiante**: la IA es una herramienta poderosa pero **no sustituye el pensamiento**. Úsala para amplificar tu capacidad, no para saltarte el aprendizaje. Los fundamentos siguen siendo tu mejor ventaja competitiva.
