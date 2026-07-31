# Módulo 12: Backend con Node.js + Express

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Comprender Node.js y el gestor npm.
- Construir servidores HTTP con Express.
- Organizar un backend profesional con capas (rutas, controladores, servicios).
- Implementar middleware y validaciones.
- Modelar datos con un ORM (Sequelize o Prisma).
- Crear APIs CRUD seguras.
- Implementar autenticación con JWT y protección de rutas.

---

## 1. Node.js

**Node.js** es un **runtime de JavaScript** construido sobre el motor V8 de Chrome, que permite ejecutar JS fuera del navegador (servidor, CLI, scripts, IoT).

### Características

- **Asíncrono y no bloqueante**: basado en el **Event Loop**.
- **Single-threaded**: escalable con `cluster` y `worker_threads`.
- **NPM**: enorme ecosistema de paquetes.
- **Multi-plataforma**: Windows, macOS, Linux.

### Instalación

```bash
# Verificar versión
node -v
npm -v

# nvm (recomendado para múltiples versiones)
nvm install 20
nvm use 20
```

### Hola mundo

```bash
node -e "console.log('Hola Node')"
```

```js
// app.js
console.log("Hola Node");
```

```bash
node app.js
```

### Módulos del core

```js
const fs   = require("node:fs");
const path = require("node:path");
const http = require("node:http");
const url  = require("node:url");
const os   = require("node:os");

fs.readFile("archivo.txt", "utf8", (err, data) => console.log(data));
const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Hola mundo");
});
server.listen(3000);
```

### Event Loop (resumen)

```
┌──────────────────┐
│  Call Stack      │
├──────────────────┤
│  Microtasks      │
├──────────────────┤
│  Macrotasks      │
│  (I/O async)     │
└──────────────────┘
```

### CommonJS vs ESM

```js
// CommonJS
const express = require("express");
module.exports = { miFn };

// ESM (Node 14+, recomendado)
import express from "express";
export const miFn = () => {};
```

Activar ESM:
```json
// package.json
{ "type": "module" }
```

### Actividad práctica

- Crea un script Node que lea 3 archivos en paralelo con `fs.promises.readFile` y loguee sus tamaños.

---

## 2. npm (Node Package Manager)

`npm` es el gestor de paquetes de Node, instalado junto con Node.js.

### Inicializar proyecto

```bash
mkdir mi-app
cd mi-app
npm init -y
```

Genera `package.json`:

```json
{
  "name": "mi-app",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "nodemon src/index.js",
    "test":  "jest"
  },
  "dependencies": {},
  "devDependencies": {},
  "engines": { "node": ">=20" }
}
```

### Instalar dependencias

```bash
npm install express            # dep de producción
npm install -D nodemon          # dep de desarrollo
npm install express@4           # versión específica
npm install express@^4.18       # rango
npm i -g pm2                    # global
```

`package.json` y `package-lock.json` se actualizan automáticamente.

### Scripts

```bash
npm start       # producción
npm run dev     # dev con nodemon
npm test
npm run build
```

### `npx`

Ejecuta binarios sin instalar:

```bash
npx create-express-api mi-proyecto
npx eslint .
npx prettier --check .
```

### Versionado semántico (SemVer)

`MAJOR.MINOR.PATCH`:
- MAJOR: cambios incompatibles.
- MINOR: features retrocompatibles.
- PATCH: bugfixes.

`^4.18.2` (minor/patch), `~4.18.2` (solo patch), `4.18.2` (exacto).

### Auditoría

```bash
npm audit
npm audit fix
```

### Herramientas complementarias

- **pnpm**: ahorra espacio, más rápido.
- **yarn**: alternativa a npm.
- **bun**: runtime + package manager todo-en-uno.
- **npm-check-updates**: actualiza dependencias.

### Actividad práctica

- Inicializa un proyecto con `npm init -y`, instala Express como dependencia, y configura scripts `dev` y `start`.

---

## 3. Express

**Express** es el framework HTTP minimalista más popular para Node.js.

### Instalación

```bash
npm install express
npm install -D nodemon
```

### Hola mundo

```js
// src/index.js
import express from "express";

const app = express();
const PORT = process.env.PORT || 3000;

app.get("/", (req, res) => res.send("Hola Express"));

app.listen(PORT, () => console.log(`http://localhost:${PORT}`));
```

### Estructura básica

```
src/
├── index.js        # arranque
├── app.js          # configuración del server
├── routes/
├── controllers/
├── services/
├── middlewares/
├── models/
└── config/
```

### `app.js` vs `index.js`

```js
// src/app.js
import express from "express";

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// rutas
app.get("/", (req, res) => res.json({ ok: true }));

export default app;
```

```js
// src/index.js
import app from "./app.js";

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server on ${PORT}`));
```

### Métodos HTTP

```js
app.get("/", h);
app.post("/", h);
app.put("/", h);
app.patch("/", h);
app.delete("/", h);
app.all("/", h);

app.use("/api", router);
```

### Objeto `Request` y `Response`

```js
app.get("/users/:id", (req, res) => {
  req.params;   // { id: "42" }
  req.query;    // { search: "ana", page: "2" }
  req.body;     // body parseado (con express.json())
  req.headers;  // cabeceras
  req.ip;
  req.method;
  req.path;

  res.status(201).json({ ok: true, data: [] });
  res.send("texto");
  res.sendFile(path);
  res.redirect("/login");
  res.setHeader("X-Powered-By", "Express");
  res.cookie("sid", "abc", { httpOnly: true });
});
```

### `express.Router`

```js
// routes/users.routes.js
import { Router } from "express";
import { getUsers, getUser, createUser, updateUser, deleteUser } from "../controllers/users.controller.js";
import { authMiddleware } from "../middlewares/auth.js";

const router = Router();

router.get("/",           authMiddleware, getUsers);
router.get("/:id",        authMiddleware, getUser);
router.post("/",          createUser);
router.put("/:id",        authMiddleware, updateUser);
router.patch("/:id",      authMiddleware, updateUser);
router.delete("/:id",     authMiddleware, deleteUser);

export default router;
```

```js
// src/app.js
import userRoutes from "./routes/users.routes.js";
app.use("/api/users", userRoutes);
```

### Respuestas convenientes

```js
res.json({ status: "ok", data });
res.status(404).json({ error: "Not Found" });
res.sendFile(path.resolve("public/index.html"));
res.download("./archivo.pdf");
```

### Actividad práctica

- Crea un servidor Express con las rutas `GET /`, `GET /api/productos`, `POST /api/productos`.

---

## 4. Routing (Enrutamiento)

### Rutas estáticas

```js
app.get("/", handler);
app.get("/api/health", (req, res) => res.json({ ok: true }));
```

### Parámetros de ruta

```js
app.get("/users/:id", (req, res) => {
  res.json({ id: req.params.id });
});

app.get("/posts/:slug/comments/:commentId", (req, res) => {
  res.json({ slug: req.params.slug, commentId: req.params.commentId });
});
```

Con expresiones regulares (vía `app.get` con prefijo o middlewares como `path-to-regexp`):

```js
app.get("/ab?cd",        (req, res) => ...); // acd o abcd
app.get("/ab(cd)?e",     (req, res) => ...); // abe o abcde
app.get("/a/:b",         (req, res) => ...); // /a/foo
app.get("/users/:id(\\d+)", (req, res) => ...); // solo dígitos
```

### Query strings

```js
app.get("/search", (req, res) => {
  const { q, page = 1, sort = "asc" } = req.query;
  res.json({ q, page, sort });
});
```

### `Router` modular

```js
// routes/auth.routes.js
import { Router } from "express";
import { login, register, refresh } from "../controllers/auth.controller.js";

export const authRouter = Router();
authRouter.post("/login",   login);
authRouter.post("/register", register);
authRouter.post("/refresh", refresh);
```

```js
// app.js
import { authRouter } from "./routes/auth.routes.js";
app.use("/api/auth", authRouter);
// POST /api/auth/login, /api/auth/register, /api/auth/refresh
```

### Restful CRUD ejemplo

```js
import { Router } from "express";
import * as ctrl from "../controllers/products.controller.js";

export const productsRouter = Router();
productsRouter.get("/",           ctrl.list);
productsRouter.get("/:id",        ctrl.getOne);
productsRouter.post("/",          ctrl.create);
productsRouter.put("/:id",        ctrl.replace);
productsRouter.patch("/:id",      ctrl.update);
productsRouter.delete("/:id",     ctrl.remove);
```

### `route()` con varios verbos

```js
router.route("/")
  .get(list)
  .post(create);

router.route("/:id")
  .get(getOne)
  .put(replace)
  .patch(update)
  .delete(remove);
```

### Actividad práctica

- Crea un router para `products` con endpoints CRUD, validación de query params y un sub-router para `/products/:id/reviews`.

---

## 5. Middleware

Funciones que se ejecutan **en cadena** durante el ciclo petición-respuesta.

### Anatomía

```js
function logger(req, res, next) {
  console.log(`${req.method} ${req.url}`);
  next(); // obligatorio
}
```

### Tipos

- **De aplicación**: `app.use(logger)` se aplica a todas.
- **De router**: `router.use(onlyForUsers)`.
- **De ruta**: `app.get("/admin", adminMiddleware, handler)`.
- **De error**: 4 argumentos (`err, req, res, next`).
- **Built-in**: `express.json`, `express.static`, `express.urlencoded`.
- **De terceros**: `cors`, `helmet`, `morgan`, `cookie-parser`.

### Built-in

```js
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static("public"));
```

### Custom de aplicación

```js
app.use((req, res, next) => {
  req.startTime = Date.now();
  next();
});

app.use((req, res, next) => {
  res.on("finish", () => {
    const ms = Date.now() - req.startTime;
    console.log(`${req.method} ${req.url} -> ${res.statusCode} (${ms}ms)`);
  });
  next();
});
```

### Custom de ruta

```js
const requireRole = (role) => (req, res, next) => {
  if (req.user?.role !== role) return res.status(403).json({ error: "Forbidden" });
  next();
};

router.get("/admin", requireRole("admin"), adminHandler);
```

### De error

```js
app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status || 500).json({ error: err.message });
});
```

### `next` parámetros

```js
next();         // siguiente middleware
next("route");  // saltar al siguiente handler de la ruta
next(err);      // saltar a middleware de error
```

### 404

```js
app.use((req, res) => {
  res.status(404).json({ error: "Not Found" });
});
```

### Orden importa

```js
// 1. globales primero
app.use(express.json());
app.use(cors());
app.use(morgan("dev"));

// 2. routers específicos
app.use("/api/users", userRoutes);
app.use("/api/products", productRoutes);

// 3. errores al final
app.use(notFound);
app.use(errorHandler);
```

### Librerías populares

```bash
npm install morgan       # logging
npm install helmet       # cabeceras de seguridad
npm install cors         # CORS
npm install compression  # gzip
npm install cookie-parser
```

```js
import morgan from "morgan";
import helmet from "helmet";
import cors from "cors";

app.use(morgan("dev"));
app.use(helmet());
app.use(cors({ origin: "https://miapp.com", credentials: true }));
```

### Actividad práctica

- Crea un middleware que mida el tiempo de respuesta, un logger de requests y un handler de errores JSON.

---

## 6. Controladores y servicios

### Capas

```
Ruta (Router)
   ↓
Middleware (validación, auth)
   ↓
Controlador (request/response, status codes)
   ↓
Servicio (lógica de negocio)
   ↓
Modelo / ORM (datos)
```

### Controlador

```js
// controllers/products.controller.js
import * as service from "../services/products.service.js";

export async function list(req, res, next) {
  try {
    const { page = 1, limit = 20, q = "" } = req.query;
    const data = await service.list({ page: +page, limit: +limit, q });
    res.json(data);
  } catch (err) { next(err); }
}

export async function getOne(req, res, next) {
  try {
    const product = await service.getById(+req.params.id);
    if (!product) return res.status(404).json({ error: "No existe" });
    res.json(product);
  } catch (err) { next(err); }
}

export async function create(req, res, next) {
  try {
    const nuevo = await service.create(req.body);
    res.status(201).json(nuevo);
  } catch (err) { next(err); }
}

export async function update(req, res, next) {
  try {
    const actualizado = await service.update(+req.params.id, req.body);
    res.json(actualizado);
  } catch (err) { next(err); }
}

export async function remove(req, res, next) {
  try {
    await service.remove(+req.params.id);
    res.status(204).end();
  } catch (err) { next(err); }
}
```

### Servicio

```js
// services/products.service.js
import { Product } from "../models/index.js";

export async function list({ page, limit, q }) {
  const offset = (page - 1) * limit;
  const where = q ? { nombre: { [Op.iLike]: `%${q}%` } } : {};
  const { count, rows } = await Product.findAndCountAll({ where, limit, offset });
  return { data: rows, total: count, page, limit };
}

export async function getById(id) {
  return Product.findByPk(id);
}

export async function create(data) {
  return Product.create(data);
}

export async function update(id, data) {
  const p = await Product.findByPk(id);
  if (!p) return null;
  return p.update(data);
}

export async function remove(id) {
  const p = await Product.findByPk(id);
  if (!p) return null;
  await p.destroy();
  return true;
}
```

### Ventajas de separar

- **Testabilidad**: el servicio se prueba aislado.
- **Reutilización**: varios controladores pueden llamar al mismo servicio.
- **Claridad**: cada capa tiene una responsabilidad única.

### Async error handling

```js
const asyncH = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

export const list = asyncH(async (req, res) => { ... });
```

Librería: `express-async-errors` (añade soporte para sync errors automáticamente).

### Actividad práctica

- Refactoriza un endpoint monolítico en controlador + servicio + ruta + middleware.

---

## 7. JWT (Json Web Token)

(Ver Módulo 8 para fundamentos). Aquí lo aplicamos al backend.

### Instalación

```bash
npm install jsonwebtoken bcrypt cookie-parser
```

### Generar token

```js
// utils/jwt.js
import jwt from "jsonwebtoken";

const SECRET = process.env.JWT_SECRET;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;
const ACCESS_TTL  = "15m";
const REFRESH_TTL = "7d";

export const signAccess = (payload) =>
  jwt.sign(payload, SECRET, { expiresIn: ACCESS_TTL });

export const signRefresh = (payload) =>
  jwt.sign(payload, REFRESH_SECRET, { expiresIn: REFRESH_TTL });

export const verifyAccess  = (token) => jwt.verify(token, SECRET);
export const verifyRefresh = (token) => jwt.verify(token, REFRESH_SECRET);
```

### Login

```js
// controllers/auth.controller.js
import bcrypt from "bcrypt";
import { User } from "../models/index.js";
import { signAccess, signRefresh } from "../utils/jwt.js";

export async function login(req, res) {
  const { email, password } = req.body;
  const user = await User.findOne({ where: { email } });
  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ error: "Credenciales inválidas" });
  }

  const payload = { sub: user.id, role: user.role };
  const accessToken = signAccess(payload);
  const refreshToken = signRefresh({ sub: user.id });

  res
    .cookie("refreshToken", refreshToken, {
      httpOnly: true,
      sameSite: "strict",
      secure: process.env.NODE_ENV === "production",
      maxAge: 7 * 24 * 60 * 60 * 1000,
    })
    .json({ accessToken, user: { id: user.id, email: user.email, role: user.role } });
}
```

### Refresh

```js
export async function refresh(req, res) {
  const token = req.cookies.refreshToken;
  if (!token) return res.status(401).json({ error: "Falta token" });

  try {
    const payload = verifyRefresh(token);
    const accessToken = signAccess({ sub: payload.sub });
    res.json({ accessToken });
  } catch {
    res.status(401).json({ error: "Token inválido" });
  }
}
```

### Logout

```js
export async function logout(req, res) {
  res.clearCookie("refreshToken").json({ ok: true });
}
```

### Actividad práctica

- Implementa login + refresh + logout con tokens en cookie y accessToken en JSON.

---

## 8. CRUD con Express + ORM

Veremos la **estructura típica** de un CRUD completo con Sequelize (alternativa con Prisma más adelante).

### Validación con `zod`

```js
// schemas/product.js
import { z } from "zod";

export const CreateProduct = z.object({
  nombre: z.string().min(2),
  precio: z.number().positive(),
  stock: z.number().int().min(0),
  categoria: z.string().min(2),
});

export const UpdateProduct = CreateProduct.partial();
```

### Middleware de validación

```js
// middlewares/validate.js
export const validate = (schema, target = "body") => (req, res, next) => {
  const result = schema.safeParse(req[target]);
  if (!result.success) {
    return res.status(400).json({
      error: "ValidationError",
      issues: result.error.flatten(),
    });
  }
  req[target] = result.data;
  next();
};
```

### Rutas

```js
// routes/products.routes.js
import { Router } from "express";
import * as c from "../controllers/products.controller.js";
import { validate } from "../middlewares/validate.js";
import { authMiddleware } from "../middlewares/auth.js";
import { CreateProduct, UpdateProduct } from "../schemas/product.js";

export const productsRouter = Router();
productsRouter.get("/",         authMiddleware, c.list);
productsRouter.get("/:id",      authMiddleware, c.getOne);
productsRouter.post("/",        authMiddleware, validate(CreateProduct), c.create);
productsRouter.patch("/:id",    authMiddleware, validate(UpdateProduct), c.update);
productsRouter.delete("/:id",   authMiddleware, c.remove);
```

### `app.js`

```js
import express from "express";
import cors from "cors";
import helmet from "helmet";
import morgan from "morgan";
import cookieParser from "cookie-parser";

import { authRouter }      from "./routes/auth.routes.js";
import { productsRouter }  from "./routes/products.routes.js";

const app = express();
app.use(helmet());
app.use(cors({ origin: true, credentials: true }));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());
app.use(morgan("dev"));

app.get("/api/health", (req, res) => res.json({ ok: true }));

app.use("/api/auth",     authRouter);
app.use("/api/products", productsRouter);

app.use((req, res) => res.status(404).json({ error: "Not Found" }));

app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status || 500).json({ error: err.message });
});

export default app;
```

### Actividad práctica

- Implementa CRUD completo de `tasks` con `zod` para validar el body.

---

## 9. Base de datos (con Sequelize)

PostgreSQL como gestor, Sequelize como ORM.

### Instalación

```bash
npm install sequelize pg pg-hstore dotenv
npm install -D sequelize-cli
```

### `.env`

```env
DATABASE_URL=postgres://user:pass@localhost:5432/miapp
JWT_SECRET=supersecret
JWT_REFRESH_SECRET=anothersecret
NODE_ENV=development
PORT=4000
```

### Conexión

```js
// src/db.js
import { Sequelize } from "sequelize";
import "dotenv/config";

export const sequelize = new Sequelize(process.env.DATABASE_URL, {
  dialect: "postgres",
  logging: process.env.NODE_ENV === "production" ? false : console.log,
});
```

### Modelo

```js
// models/User.js
import { DataTypes } from "sequelize";
import { sequelize } from "../db.js";

export const User = sequelize.define("User", {
  id:    { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  email: { type: DataTypes.STRING, unique: true, allowNull: false, validate: { isEmail: true } },
  passwordHash: { type: DataTypes.STRING, allowNull: false },
  role:  { type: DataTypes.ENUM("admin", "user"), defaultValue: "user" },
});
```

```js
// models/Product.js
import { DataTypes } from "sequelize";
import { sequelize } from "../db.js";

export const Product = sequelize.define("Product", {
  id:       { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  nombre:   { type: DataTypes.STRING, allowNull: false },
  precio:   { type: DataTypes.DECIMAL(10, 2), allowNull: false },
  stock:    { type: DataTypes.INTEGER, defaultValue: 0 },
  categoria:{ type: DataTypes.STRING },
});
```

```js
// models/Order.js
import { DataTypes } from "sequelize";
import { sequelize } from "../db.js";

export const Order = sequelize.define("Order", {
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  total: { type: DataTypes.DECIMAL(10, 2), allowNull: false },
  status: { type: DataTypes.ENUM("pending", "paid", "shipped"), defaultValue: "pending" },
});
```

### Asociaciones

```js
// models/index.js
import { sequelize } from "../db.js";
import "./User.js";
import "./Product.js";
import "./Order.js";
import "./OrderItem.js";

import { User }    from "./User.js";
import { Product } from "./Product.js";
import { Order }   from "./Order.js";
import { OrderItem } from "./OrderItem.js";

User.hasMany(Order);
Order.belongsTo(User);

Order.belongsToMany(Product, { through: OrderItem });
Product.belongsToMany(Order, { through: OrderItem });

export { sequelize, User, Product, Order, OrderItem };
```

```bash
# Migraciones
npx sequelize-cli init
npx sequelize-cli model:generate --name Product --attributes nombre:string,precio:decimal,stock:integer
npx sequelize-cli db:migrate
```

### Uso en el servicio

```js
import { Op } from "sequelize";
import { Product } from "../models/index.js";

export async function list({ page = 1, limit = 20, q = "" }) {
  const offset = (page - 1) * limit;
  const where = q ? { nombre: { [Op.iLike]: `%${q}%` } } : {};
  const { count, rows } = await Product.findAndCountAll({ where, limit, offset });
  return { data: rows, total: count, page, limit };
}
```

### `bcrypt`

```js
import bcrypt from "bcrypt";

export async function register(req, res) {
  const { email, password } = req.body;
  const exists = await User.findOne({ where: { email } });
  if (exists) return res.status(409).json({ error: "Email registrado" });

  const passwordHash = await bcrypt.hash(password, 12);
  const user = await User.create({ email, passwordHash });
  res.status(201).json({ id: user.id, email: user.email });
}
```

### Actividad práctica

- Conecta con PostgreSQL, define 2 modelos y haz un CRUD contra ellos.

---

## 10. ORM (Prisma como alternativa)

**Prisma** es un ORM moderno con schema declarativo, gran DX y soporte para muchas DBs.

### Instalación

```bash
npm install prisma @prisma/client
npx prisma init
```

### `schema.prisma`

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        Int     @id @default(autoincrement())
  email     String  @unique
  password  String
  role      Role    @default(USER)
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int     @id @default(autoincrement())
  title     String
  content   String?
  published Boolean @default(false)
  author    User    @relation(fields: [authorId], references: [id])
  authorId  Int
  createdAt DateTime @default(now())
}

enum Role {
  USER
  ADMIN
}
```

### Cliente

```js
// db.js
import { PrismaClient } from "@prisma/client";
export const prisma = new PrismaClient();
```

### CRUD

```js
export async function list() {
  return prisma.user.findMany({ select: { id: true, email: true, role: true } });
}

export async function create({ email, password }) {
  return prisma.user.create({
    data: { email, password: await bcrypt.hash(password, 12) },
  });
}

export async function getById(id) {
  return prisma.user.findUnique({ where: { id } });
}

export async function update(id, data) {
  return prisma.user.update({ where: { id }, data });
}

export async function remove(id) {
  return prisma.user.delete({ where: { id } });
}
```

### Migración

```bash
npx prisma migrate dev --name init
npx prisma generate
npx prisma studio        # GUI
```

### Ventajas Prisma

- Tipos generados.
- Migraciones automáticas.
- Buen manejo de relaciones.
- `studio` para inspeccionar datos.

### Desventajas

- Menos flexible para queries muy custom que SQL puro.
- Mayor tamaño de bundle cliente (menos relevante en backend).

### ORM vs query builder vs SQL

| Enfoque             | Pros                                | Contras                          |
| ------------------- | ----------------------------------- | -------------------------------- |
| **SQL puro**        | Máximo control, performance         | Mucho código, sin tipos           |
| **Query builder** (Knex) | Flexible, sin magia              | Verboso                          |
| **ORM tradicional** (Sequelize) | Rápido de aprender          | Sintaxis peculiar                |
| **Prisma**          | DX excelente, types                 | Menos control de SQL crudo       |
| **Drizzle**         | Types, SQL-like, ligero             | Ecosistema más pequeño           |

### Actividad práctica

- Migra uno de los modelos del proyecto a Prisma y prueba `prisma studio`.

---

## 11. Autenticación completa

Aquí se integra todo: hashing, JWT, middleware, rutas protegidas, refresh.

### Estructura

```
src/
├── config/
│   └── env.js
├── db/
├── models/
├── schemas/
├── middlewares/
│   ├── auth.js
│   ├── validate.js
│   └── error.js
├── controllers/
├── services/
├── routes/
├── utils/
│   ├── jwt.js
│   └── password.js
└── app.js
```

### `utils/password.js`

```js
import bcrypt from "bcrypt";

export const hash = (plain) => bcrypt.hash(plain, 12);
export const verify = (plain, hash) => bcrypt.compare(plain, hash);
```

### `middlewares/auth.js`

```js
import { verifyAccess } from "../utils/jwt.js";

export function authMiddleware(req, res, next) {
  const header = req.headers.authorization;
  if (!header || !header.startsWith("Bearer ")) {
    return res.status(401).json({ error: "Falta token" });
  }
  try {
    req.user = verifyAccess(header.split(" ")[1]);
    next();
  } catch {
    res.status(401).json({ error: "Token inválido" });
  }
}

export const requireRole = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user?.role)) {
    return res.status(403).json({ error: "Sin permisos" });
  }
  next();
};
```

### Rutas protegidas

```js
import { Router } from "express";
import { authMiddleware, requireRole } from "../middlewares/auth.js";
import * as ctrl from "../controllers/admin.controller.js";

export const adminRouter = Router();
adminRouter.get("/users",       authMiddleware, requireRole("admin"), ctrl.listUsers);
adminRouter.delete("/users/:id",authMiddleware, requireRole("admin"), ctrl.deleteUser);
```

### `controllers/auth.controller.js`

```js
import { User } from "../models/index.js";
import { hash, verify } from "../utils/password.js";
import { signAccess, signRefresh, verifyRefresh } from "../utils/jwt.js";

export async function register(req, res) {
  const { email, password } = req.body;
  const exists = await User.findOne({ where: { email } });
  if (exists) return res.status(409).json({ error: "Email registrado" });

  const passwordHash = await hash(password);
  const user = await User.create({ email, passwordHash, role: "user" });
  res.status(201).json({ id: user.id, email: user.email, role: user.role });
}

export async function login(req, res) {
  const { email, password } = req.body;
  const user = await User.findOne({ where: { email } });
  if (!user || !(await verify(password, user.passwordHash))) {
    return res.status(401).json({ error: "Credenciales inválidas" });
  }

  const accessToken  = signAccess({ sub: user.id, role: user.role });
  const refreshToken = signRefresh({ sub: user.id });

  res
    .cookie("refreshToken", refreshToken, {
      httpOnly: true,
      sameSite: "lax",
      secure: process.env.NODE_ENV === "production",
      maxAge: 7 * 24 * 60 * 60 * 1000,
    })
    .json({ accessToken, user: { id: user.id, email: user.email, role: user.role } });
}

export async function refresh(req, res) {
  const token = req.cookies.refreshToken;
  if (!token) return res.status(401).json({ error: "No refresh token" });
  try {
    const data = verifyRefresh(token);
    res.json({ accessToken: signAccess({ sub: data.sub, role: data.role }) });
  } catch {
    res.status(401).json({ error: "Refresh inválido" });
  }
}

export async function me(req, res) {
  const user = await User.findByPk(req.user.sub, {
    attributes: ["id", "email", "role", "createdAt"],
  });
  if (!user) return res.status(404).json({ error: "No existe" });
  res.json(user);
}

export async function logout(req, res) {
  res.clearCookie("refreshToken").json({ ok: true });
}
```

### Frontend cliente (resumen)

```js
// api.js
const api = {
  async request(method, url, body) {
    const headers = { "Content-Type": "application/json" };
    const token = sessionStorage.getItem("accessToken");
    if (token) headers.Authorization = `Bearer ${token}`;

    const res = await fetch(`/api${url}`, {
      method, headers,
      body: body && JSON.stringify(body),
      credentials: "include",
    });

    if (res.status === 401) {
      const refreshed = await api.refresh();
      if (refreshed) return api.request(method, url, body);
      location.href = "/login";
    }
    if (!res.ok) throw await res.json();
    return res.json();
  },

  async refresh() {
    const res = await fetch("/api/auth/refresh", { method: "POST", credentials: "include" });
    if (!res.ok) return false;
    const { accessToken } = await res.json();
    sessionStorage.setItem("accessToken", accessToken);
    return true;
  },
};
```

### Buenas prácticas de autenticación

- HTTPS siempre.
- JWT_SECRET largo y aleatorio (256 bits).
- Access token corto (15-30 min).
- Refresh token en cookie HttpOnly + Secure.
- Logout que invalide tokens (lista negra o rotación).
- bcrypt con **12** rondas (ajustable según CPU).
- Rate limiting en `/login`.

### Rate limiting

```bash
npm install express-rate-limit
```

```js
import rateLimit from "express-rate-limit";

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: "Demasiados intentos. Intenta en 15 minutos.",
});

router.post("/login", loginLimiter, login);
```

### Helmet + CORS

```js
app.use(helmet());
app.use(cors({ origin: "https://mi-frontend.com", credentials: true }));
```

### CSRF (si usas cookies para auth)

```bash
npm install csurf
```

Para SPAs con JWT, normalmente se evita CSRF usando:
- Tokens en headers (no cookies).
- Cookie `SameSite=Lax` o `Strict`.
- Cabecera `X-CSRF-Token` sincronizada.

### Actividad práctica

- Construye un sistema completo: register, login, refresh, logout y un endpoint `/me` protegido.

---

## Proyecto integrador del módulo

```
mi-backend/
├── package.json
├── .env
├── prisma/
│   └── schema.prisma
└── src/
    ├── index.js
    ├── app.js
    ├── config/
    │   └── env.js
    ├── db.js
    ├── prisma.js
    ├── middlewares/
    │   ├── auth.js
    │   ├── validate.js
    │   └── error.js
    ├── schemas/
    │   ├── auth.js
    │   └── product.js
    ├── controllers/
    │   ├── auth.controller.js
    │   └── products.controller.js
    ├── services/
    │   ├── auth.service.js
    │   └── products.service.js
    ├── routes/
    │   ├── auth.routes.js
    │   └── products.routes.js
    └── utils/
        ├── jwt.js
        └── password.js
```

### `package.json`

```json
{
  "type": "module",
  "scripts": {
    "dev": "nodemon -w src src/index.js",
    "start": "node src/index.js",
    "db:migrate": "prisma migrate dev",
    "db:studio": "prisma studio"
  }
}
```

### `src/app.js`

```js
import express from "express";
import cors from "cors";
import helmet from "helmet";
import morgan from "morgan";
import cookieParser from "cookie-parser";

import { authRouter }     from "./routes/auth.routes.js";
import { productsRouter } from "./routes/products.routes.js";
import { errorHandler }   from "./middlewares/error.js";

const app = express();
app.use(helmet());
app.use(cors({ origin: "http://localhost:5173", credentials: true }));
app.use(express.json());
app.use(cookieParser());
app.use(morgan("dev"));

app.get("/api/health", (req, res) => res.json({ ok: true }));

app.use("/api/auth",     authRouter);
app.use("/api/products", productsRouter);

app.use((req, res) => res.status(404).json({ error: "Not Found" }));
app.use(errorHandler);

export default app;
```

### `src/index.js`

```js
import "dotenv/config";
import app from "./app.js";

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`API on http://localhost:${PORT}`));
```

### Endpoints disponibles

| Método | Ruta                  | Auth | Descripción                       |
| ------ | --------------------- | ---- | --------------------------------- |
| POST   | /api/auth/register    | No   | Registrar usuario                 |
| POST   | /api/auth/login       | No   | Iniciar sesión                    |
| POST   | /api/auth/refresh     | No   | Refrescar access token            |
| POST   | /api/auth/logout      | No   | Limpiar cookie                    |
| GET    | /api/auth/me          | Sí   | Perfil del usuario autenticado    |
| GET    | /api/products         | Sí   | Listar productos paginados        |
| GET    | /api/products/:id     | Sí   | Obtener producto                  |
| POST   | /api/products         | Admin | Crear producto                   |
| PATCH  | /api/products/:id     | Admin | Actualizar producto              |
| DELETE | /api/products/:id     | Admin | Eliminar producto                |

---

## Recursos complementarios

### Lecturas

- [Express Docs](https://expressjs.com/)
- [Node.js Docs](https://nodejs.org/)
- [Sequelize Docs](https://sequelize.org/)
- [Prisma Docs](https://www.prisma.io/docs)
- [Zod Docs](https://zod.dev/)

### Buenas prácticas

- [Node Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [OWASP API Top 10](https://owasp.org/API-Security/editions/2023/)

### Testing (intro)

```bash
npm install -D jest supertest
```

```js
import request from "supertest";
import app from "../src/app.js";

test("GET /api/health", async () => {
  const res = await request(app).get("/api/health");
  expect(res.status).toBe(200);
  expect(res.body.ok).toBe(true);
});
```

---

## Evaluación sugerida

1. **CRUD (30%)**: API completa con validaciones y paginación.
2. **Autenticación (25%)**: register/login con JWT + bcrypt.
3. **Rutas protegidas (15%)**: middleware y roles.
4. **ORM (20%)**: modelar 3 entidades con relaciones y CRUD coherente.
5. **Refactor (10%)**: pasar de un solo archivo a estructura de capas.

---

> **Nota para el estudiante**: un buen backend no es solo "que funcione": es **capas claras**, **validación sistemática**, **manejo de errores coherente**, **seguridad por defecto** y **tests automatizados**. Invierte en esto desde el primer proyecto.
