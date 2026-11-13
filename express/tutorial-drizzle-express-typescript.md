# Cómo instalar y usar Drizzle ORM en Express con TypeScript

Drizzle es un ORM que se siente como escribir SQL con tipos. Defines tus tablas en TypeScript y obtienes autocompletado y errores en tiempo de compilación, sin generadores pesados ni capas mágicas.

En esta guía montas un proyecto Express desde cero, lo conectas a PostgreSQL con Drizzle, creas una tabla, corres migraciones y expones un CRUD básico. Al final tendrás una base sólida para cualquier API.

**Requisitos:** Node.js 20 o superior, una base de datos PostgreSQL (local, Docker o un servicio como Neon o Supabase) y conocimientos básicos de TypeScript.

## 1. Crea el proyecto

```bash
mkdir api-drizzle && cd api-drizzle
npm init -y
```

Instala las dependencias de ejecución:

```bash
npm install express drizzle-orm pg dotenv
```

Y las de desarrollo:

```bash
npm install -D typescript tsx drizzle-kit @types/express @types/node @types/pg
```

Qué hace cada una:

- `drizzle-orm`: el ORM en sí.
- `pg`: el driver de PostgreSQL.
- `drizzle-kit`: la herramienta de línea de comandos para migraciones.
- `tsx`: ejecuta TypeScript directamente, sin compilar en desarrollo.

## 2. Configura TypeScript

Genera el archivo base:

```bash
npx tsc --init
```

Déjalo con esta configuración mínima en `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

En `package.json` agrega `"type": "module"` y estos scripts:

```json
{
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:studio": "drizzle-kit studio"
  }
}
```

> Con `NodeNext`, los imports relativos llevan extensión `.js` (por ejemplo `./db/index.js`), aunque el archivo sea `.ts`. Es una regla de Node, no de Drizzle.

## 3. Define las variables de entorno

Crea un archivo `.env` en la raíz:

```env
DATABASE_URL=postgres://usuario:contraseña@localhost:5432/mi_base
PORT=3000
```

Agrega `.env` a tu `.gitignore`. Nunca subas credenciales al repositorio.

## 4. Estructura de carpetas

```
api-drizzle/
├── src/
│   ├── db/
│   │   ├── index.ts      # conexión
│   │   └── schema.ts     # tablas
│   └── index.ts          # servidor Express
├── drizzle/              # migraciones (se genera solo)
├── drizzle.config.ts
├── .env
└── tsconfig.json
```

## 5. Escribe el esquema

En `src/db/schema.ts` defines las tablas como código:

```ts
import { pgTable, serial, text, boolean, timestamp } from "drizzle-orm/pg-core";

export const tareas = pgTable("tareas", {
  id: serial("id").primaryKey(),
  titulo: text("titulo").notNull(),
  completada: boolean("completada").notNull().default(false),
  creadaEn: timestamp("creada_en").notNull().defaultNow(),
});

export type Tarea = typeof tareas.$inferSelect;
export type NuevaTarea = typeof tareas.$inferInsert;
```

Las dos últimas líneas son la gracia de Drizzle: los tipos salen del esquema. Si cambias una columna, TypeScript te avisa dónde se rompe algo.

## 6. Configura la conexión

En `src/db/index.ts`:

```ts
import "dotenv/config";
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "./schema.js";

export const db = drizzle(process.env.DATABASE_URL!, { schema });
```

Drizzle crea el pool de `pg` por ti a partir de la URL. Pasar `schema` habilita las consultas relacionales más adelante.

## 7. Configura Drizzle Kit

En la raíz, crea `drizzle.config.ts`:

```ts
import "dotenv/config";
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  dialect: "postgresql",
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

Este archivo le dice a `drizzle-kit` dónde está tu esquema, dónde guardar las migraciones y cómo conectarse.

## 8. Genera y aplica la migración

```bash
npm run db:generate
npm run db:migrate
```

El primer comando compara tu esquema con el estado anterior y crea un archivo SQL en `./drizzle`. El segundo lo ejecuta contra la base de datos. Revisa el SQL generado antes de migrar: es un archivo normal y legible.

**Atajo para prototipos:** `npx drizzle-kit push` sincroniza el esquema directamente, sin archivos de migración. Úsalo solo en desarrollo; en producción conviene el flujo `generate` + `migrate` para tener historial.

## 9. Crea el servidor con un CRUD

En `src/index.ts`:

```ts
import "dotenv/config";
import express from "express";
import { eq } from "drizzle-orm";
import { db } from "./db/index.js";
import { tareas } from "./db/schema.js";

const app = express();
app.use(express.json());

// Listar
app.get("/tareas", async (_req, res) => {
  const resultado = await db.select().from(tareas);
  res.json(resultado);
});

// Crear
app.post("/tareas", async (req, res) => {
  const { titulo } = req.body;
  if (!titulo) {
    return res.status(400).json({ error: "El título es obligatorio" });
  }
  const [nueva] = await db.insert(tareas).values({ titulo }).returning();
  res.status(201).json(nueva);
});

// Actualizar
app.patch("/tareas/:id", async (req, res) => {
  const id = Number(req.params.id);
  const [actualizada] = await db
    .update(tareas)
    .set({ completada: req.body.completada })
    .where(eq(tareas.id, id))
    .returning();

  if (!actualizada) return res.status(404).json({ error: "No encontrada" });
  res.json(actualizada);
});

// Eliminar
app.delete("/tareas/:id", async (req, res) => {
  const id = Number(req.params.id);
  const [eliminada] = await db.delete(tareas).where(eq(tareas.id, id)).returning();

  if (!eliminada) return res.status(404).json({ error: "No encontrada" });
  res.status(204).send();
});

const port = process.env.PORT ?? 3000;
app.listen(port, () => console.log(`Servidor en http://localhost:${port}`));
```

Arráncalo:

```bash
npm run dev
```

## 10. Pruébalo

```bash
curl -X POST http://localhost:3000/tareas \
  -H "Content-Type: application/json" \
  -d '{"titulo": "Aprender Drizzle"}'

curl http://localhost:3000/tareas
```

Si ves la tarea en la respuesta, todo funciona.

## Errores comunes

- **`Cannot find module './db/index'`:** falta la extensión `.js` en el import (ver sección 2).
- **`DATABASE_URL` indefinida en `drizzle-kit`:** asegúrate de importar `dotenv/config` en `drizzle.config.ts`.
- **Error de conexión o SSL en servicios en la nube:** agrega `?sslmode=require` al final de la URL.
- **Cambiaste el esquema y nada pasa:** corre `db:generate` otra vez; las migraciones no se crean solas.

## Bonus: explora tus datos

```bash
npm run db:studio
```

Abre Drizzle Studio, una interfaz web para ver y editar tus tablas sin salir del proyecto.

## Siguientes pasos

Ya tienes lo esencial: esquema tipado, migraciones y un CRUD funcional. Desde aquí puedes:

- Agregar relaciones entre tablas con `relations()` y consultas con `db.query`.
- Validar el `req.body` con Zod (o `drizzle-zod`, que genera los esquemas desde tus tablas).
- Separar rutas y lógica en carpetas (`routes/`, `services/`).
- Usar otro motor: MySQL o SQLite solo cambian el `dialect` y el driver.

Si te sirvió, pruébalo con tu propio modelo de datos. Es la forma más rápida de que se te quede.
