# Guía de instalación: Express.js con TypeScript (ejecutando con TSX)

Esta guía te llevará paso a paso para crear un proyecto de **Express.js** con **TypeScript**, ejecutándose con **tsx** (en lugar de `ts-node`), que es más rápido y no requiere configuración adicional de `tsconfig-paths` ni `@types/node` extra para el modo watch.

---

## Requisitos previos

- Node.js 18 o superior (recomendado: LTS actual). Verificar con:
  ```bash
  node -v
  ```
- npm (viene incluido con Node.js):
  ```bash
  npm -v
  ```

---

## 1. Crear el proyecto e inicializar npm

```bash
mkdir mi-app-express
cd mi-app-express
npm init -y
```

## 2. Instalar dependencias

### Dependencias de producción

```bash
npm install express
```

### Dependencias de desarrollo

```bash
npm install -D typescript tsx @types/node @types/express
```

¿Qué hace cada paquete?

| Paquete | Descripción |
|---|---|
| `typescript` | Compilador de TypeScript (`tsc`) |
| `tsx` | Ejecuta archivos `.ts` directamente, sin compilar antes (usa esbuild por debajo, muy rápido) |
| `@types/node` | Tipos de las APIs nativas de Node.js |
| `@types/express` | Tipos de Express |

## 3. Inicializar TypeScript

```bash
npx tsc --init
```

Esto crea el archivo `tsconfig.json`. Edítalo con una configuración recomendada para un proyecto Express:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

> **Nota:** `tsx` no usa strictamente el `tsconfig.json` para transpilar (deja eso al runtime), pero la configuración sí la usa tu editor y el comando `tsc` para verificar tipos.

## 4. Crear la estructura del proyecto

```bash
mkdir src
```

Dentro de `src` crea el archivo `index.ts`:

```bash
touch src/index.ts
```

## 5. Escribir el servidor básico

`src/index.ts`:

```typescript
import express, { Request, Response } from "express";

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get("/", (req: Request, res: Response) => {
  res.json({ mensaje: "¡Hola mundo con Express + TypeScript!" });
});

app.listen(PORT, () => {
  console.log(`Servidor corriendo en http://localhost:${PORT}`);
});
```

## 6. Configurar los scripts en `package.json`

Abre `package.json` y agrega/modifica la sección `scripts`:

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "start": "tsx src/index.ts",
    "build": "tsc",
    "serve": "node dist/index.js"
  }
}
```

¿Qué hace cada script?

- `dev`: ejecuta el servidor con **modo watch** — reinicia automáticamente al guardar cambios (equivalente a nodemon, pero integrado en tsx).
- `start`: ejecuta el proyecto TypeScript una sola vez.
- `build`: compila a JavaScript en `dist/`.
- `serve`: ejecuta el build compilado (útil para producción).

## 7. Ejecutar el proyecto

### Modo desarrollo (con auto-reload)

```bash
npm run dev
```

Deberías ver:

```
Servidor corriendo en http://localhost:3000
```

Prueba con el navegador o con curl:

```bash
curl http://localhost:3000
# {"mensaje":"¡Hola mundo con Express + TypeScript!"}
```

### Para producción

```bash
npm run build
npm run serve
```

---

## Solución de problemas comunes

### `Cannot find module 'express' o sus tipos`
Instala los tipos: `npm install -D @types/express`

### El puerto ya está en uso
Cambia el puerto o mata el proceso:
```bash
lsof -ti:3000 | xargs kill -9
```

### Cambios no se recargan
Asegúrate de usar `tsx watch` (script `dev`) y no solo `tsx`.

---

## Resumen de comandos

```bash
mkdir mi-app-express && cd mi-app-express
npm init -y
npm install express
npm install -D typescript tsx @types/node @types/express
npx tsc --init
mkdir src && touch src/index.ts
# ... editar tsconfig.json, package.json y src/index.ts
npm run dev
```

¡Listo! Ahora tienes un servidor Express con TypeScript que se ejecuta con tsx y recarga automáticamente en desarrollo.
