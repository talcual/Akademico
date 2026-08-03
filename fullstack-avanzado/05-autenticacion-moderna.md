# Autenticación Moderna

## Objetivo de aprendizaje

La autenticación es **la pieza con más costo histórico** de una aplicación. Una elección mal hecha (típicamente "JWT para todo, sin refresh, sin revocación") se descubre meses después, cuando hay que migrar sin romper a 200,000 usuarios.

Aquí se estudian los **mecanismos reales** usados en producción: sesiones, tokens, refresh con rotación, OAuth/OIDC, Passkeys y Magic Links. El objetivo es que el estudiante entienda **qué problema resuelve cada uno y cuándo combinarlos**.

---

## Mapa general

| Mecanismo          | Estado vive en   | Caso típico                                |
| ------------------ | ---------------- | ------------------------------------------ |
| Session + Cookie   | Servidor         | Apps web tradicionales, admin panels       |
| JWT                | Cliente (header) | APIs stateless, microservicios             |
| Refresh Token      | Cliente (storage) | Renovar access tokens sin re-login         |
| Token Rotation     | Servidor         | Mitigar robo de refresh tokens             |
| OAuth 2.0          | Servidor         | "Login con Google/GitHub"                  |
| OpenID Connect     | Servidor         | OAuth + identidad verificada               |
| Passkeys (WebAuthn) | Dispositivo    | Reemplazo moderno de contraseñas           |
| Magic Links        | Email            | Login sin contraseña para apps de bajo riesgo |

---

## 1. Session + Cookie

### Idea central

El servidor mantiene un **estado de sesión** (un ID ↔ datos del usuario). El cliente solo guarda un **identificador opaco** en una cookie. Cada请求 lo envía y el servidor lo resuelve.

### Flujo

```
[1] POST /login { email, password }
    servidor valida, genera sessionId = "abc123"
    guarda { userId: 42, expira: ... } asociado a "abc123"
    responde Set-Cookie: sid=abc123; HttpOnly; Secure; SameSite=Lax

[2] GET /perfil  (Cookie: sid=abc123)
    servidor busca "abc123" en su store
    identifica al usuario
    responde { nombre, email }
```

### Atributos de la cookie

| Atributo    | Función                                                      |
| ----------- | ------------------------------------------------------------ |
| `HttpOnly`  | JavaScript no puede leerla. Mitiga XSS.                      |
| `Secure`    | Solo se envía por HTTPS.                                     |
| `SameSite`  | Controla envío cross-site. `Lax` o `Strict`. Mitiga CSRF.    |
| `Domain`/`Path` | Alcance de la cookie.                                     |
| `Max-Age`/`Expires` | Cuándo expira.                                       |

### Almacenamiento de sesiones

- **Memoria**: rápido, no escala entre instancias.
- **Redis**: estándar de facto. TTL, replicable, rápido.
- **Base de datos**: válido pero más lento.

```ts
// servidor (Node + Express + Redis)
import session from "express-session";
import RedisStore from "connect-redis";
import { createClient } from "redis";

const redis = createClient();
await redis.connect();

app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 1000 * 60 * 60 * 24, // 1 día
  },
}));
```

### Cuándo usarlo

- Apps web tradicionales.
- Admin panels, intranets.
- Cuando necesitas **revocación inmediata** (borrar la sesión = cerrar todas las pestañas del usuario).
- Cuando el cliente es un navegador que soporta cookies sin fricción.

### Limitaciones

- Requiere que el cliente **coopere con cookies** (no ideal para APIs consumidas por móviles o terceros).
- No escala bien en arquitecturas sin estado si no usas un store compartido.

---

## 2. JWT (JSON Web Token)

### Idea central

Token **autosuficiente**: contiene los claims del usuario firmados criptográficamente. El servidor **no guarda estado**; verifica la firma y confía en los claims.

### Estructura

```
header.payload.signature
```

- **Header**: algoritmo y tipo.
- **Payload**: claims (`sub`, `iat`, `exp`, `name`, `roles`).
- **Signature**: HMAC o RSA/ECDSA sobre header+payload.

### Ejemplo

```ts
import jwt from "jsonwebtoken";

const token = jwt.sign(
  { sub: "user-42", name: "Ana", roles: ["admin"] },
  process.env.JWT_SECRET!,
  { algorithm: "HS256", expiresIn: "15m" },
);
```

### Verificación

```ts
const decoded = jwt.verify(token, process.env.JWT_SECRET!);
console.log(decoded.sub); // "user-42"
```

### Formas de enviarlo

| Mecanismo             | Consideraciones                                  |
| --------------------- | ----------------------------------------------- |
| `Authorization: Bearer ...` | Estándar en APIs REST. Sin estado.         |
| Cookie `HttpOnly`     | Similar a session pero sin store servidor.      |
| LocalStorage          | **Peligroso**. Vulnerable a XSS.                |

### Claims importantes

| Claim | Significado                         |
| ----- | ----------------------------------- |
| `sub` | Subject (id del usuario)            |
| `iat` | Issued at                           |
| `exp` | Expiration time                     |
| `iss` | Issuer                              |
| `aud` | Audience                            |
| `jti` | JWT ID (para revocación/refresh)    |

### Ventajas

- **Stateless**: no hay store de sesiones. Cualquier servicio con la clave puede verificar.
- **Escalabilidad horizontal trivial**.
- Portabilidad entre servicios.

### Riesgos y limitaciones

- **No se puede revocar fácilmente**: un JWT emitido es válido hasta `exp`. Para revocación necesitas una **lista negra** o un esquema de **refresh tokens**.
- **Más grande que un session id**: claims se envían en cada request.
- **Secret management**: filtrar la clave de firma es catastrófico. Mejor **asimétrico** (RS256) con claves pública/privada.

### Cuándo usarlo

- APIs REST/GraphQL consumidas por múltiples clientes.
- Microservicios que necesitan verificar identidad sin llamar a un servicio central cada vez.
- Sistemas donde la **revocación inmediata no es crítica**.

### Cuándo **no** usarlo (solo)

- Cuando necesitas poder **cerrar la sesión de un usuario en todas sus pestañas** ahora mismo.
- Cuando el token va a vivir mucho tiempo (mejor con refresh).

---

## 3. Refresh Token

### Idea central

Separar dos tokens con **vidas distintas**:

- **Access token**: corto (5-15 min). Se envía en cada request.
- **Refresh token**: largo (días/semanas). Solo se usa para pedir un access token nuevo.

```
[1] POST /login → { accessToken: "...15m...", refreshToken: "...30d..." }

[2] GET /api  Authorization: Bearer accessToken
    si accessToken expiró → 401

[3] POST /refresh { refreshToken }
    → { accessToken: "nuevo..." }
```

### Almacenamiento del refresh token

| Lugar               | Riesgo      | Mitigación                              |
| ------------------- | ----------- | --------------------------------------- |
| Cookie `HttpOnly`   | CSRF        | `SameSite=Strict` + rotación            |
| LocalStorage        | XSS         | Evitar; o cifrar                        |
| Service Worker      | Moderado    | Útil para background sync               |
| Secure storage (móvil) | Bajo     | Estándar para apps nativas              |

### Por qué importa

- Si te roban el access token, **expira pronto**.
- Si te roban el refresh token, lo detectas por **rotación** o **reuso**.

---

## 4. Token Rotation (Rotación de Tokens)

### Idea central

Cada vez que se usa un refresh token, **se invalida y se emite uno nuevo**. Si el servidor detecta que un refresh token ya usado se vuelve a presentar, asume **robo** y revoca toda la familia.

### Flujo

```
[1] Login
    refreshA = "rt-001"
    familia = "fam-001"

[2] Cliente A usa refreshA → recibe refreshB = "rt-002", refreshA queda revocado
    familia = "fam-001", actual = "rt-002"

[3] Cliente A usa refreshB → recibe refreshC = "rt-003"
    familia = "fam-001", actual = "rt-003"

[4] Atacante presenta refreshA (robado y reusado) → DETECTADO
    → revocar TODA la familia fam-001
    → forzar re-login en todos los dispositivos del usuario
```

### Implementación (esquema)

```ts
async function rotate(refreshToken: string) {
  const stored = await db.refreshTokens.findOne({ token: hash(refreshToken) });
  if (!stored) throw new Error("invalid");
  if (stored.usedAt) {
    // ¡reuso! Posible robo
    await db.refreshTokens.updateMany(
      { familyId: stored.familyId },
      { revoked: true },
    );
    throw new Error("family_revoked");
  }
  if (stored.expiresAt < new Date()) throw new Error("expired");

  // marcar como usado y emitir uno nuevo
  await db.refreshTokens.updateOne(
    { _id: stored._id },
    { usedAt: new Date() },
  );

  const newRefresh = crypto.randomUUID();
  await db.refreshTokens.insert({
    token: hash(newRefresh),
    familyId: stored.familyId,
    expiresAt: addDays(new Date(), 30),
  });

  const newAccess = signAccessToken(stored.userId);
  return { access: newAccess, refresh: newRefresh };
}
```

### Ventajas

- Detecta robo de refresh tokens.
- Permite revocar sesiones por dispositivo.

### Cuidados

- **Concurrencia**: si dos pestañas usan el refresh "a la vez", una fallará. Solución: pequeña ventana de gracia o reintento transparente.
- **Persistencia**: necesitas una tabla de refresh tokens con `familyId`, `usedAt`, `revoked`.

---

## 5. OAuth 2.0

### Idea central

Protocolo de **autorización delegada**. Un usuario delega a una app el acceso a recursos en otro servicio, sin compartir sus credenciales.

### Roles

- **Resource Owner**: el usuario.
- **Client**: la app que quiere acceder.
- **Authorization Server**: emite tokens (Google, Auth0, GitHub).
- **Resource Server**: la API que tiene los datos.

### Flujo principal: Authorization Code

```
[1] Usuario hace clic en "Login con Google"
[2] App redirige a Google:
      https://accounts.google.com/o/oauth2/v2/auth?
        client_id=...
        &redirect_uri=https://miapp.com/callback
        &response_type=code
        &scope=openid email profile
[3] Usuario se autentica en Google y aprueba
[4] Google redirige a redirect_uri con ?code=AUTH_CODE
[5] Backend intercambia code por tokens:
      POST https://oauth2.googleapis.com/token
        { code, client_id, client_secret, redirect_uri, grant_type: "authorization_code" }
[6] Backend recibe { access_token, refresh_token?, id_token? }
[7] Backend usa access_token para llamar a las APIs de Google
```

### Grants (flujos) principales

| Grant                          | Uso                                        |
| ------------------------------ | ------------------------------------------ |
| Authorization Code             | Apps web y móviles (el más seguro)         |
| Authorization Code + PKCE      | SPAs y móviles (sin client secret)         |
| Client Credentials             | Server-to-server, sin usuario              |
| Device Code                    | TVs, dispositivos sin navegador            |

### PKCE (Proof Key for Code Exchange)

Extensión que evita el **robo del `code`** añadiendo un `code_verifier` y `code_challenge`. **Obligatorio en OAuth 2.1 para apps públicas**.

```ts
// cliente
const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
const challenge = await sha256(verifier);

// paso 1: enviar challenge
authUrl.searchParams.set("code_challenge", challenge);
authUrl.searchParams.set("code_challenge_method", "S256");

// paso 5: enviar verifier
tokenResponse = await fetch(tokenUrl, {
  method: "POST",
  body: { code, code_verifier: verifier, ... },
});
```

### Cuándo usarlo

- "Login con Google/GitHub/Microsoft/Apple".
- Integración con APIs de terceros (Stripe, Slack, Notion).
- B2B: tu app accede a datos del cliente en su CRM, su存储, etc.

### Lo que OAuth **no** es

- OAuth **no es autenticación**. Es **autorización**. Para identidad necesitas OpenID Connect.

---

## 6. OpenID Connect (OIDC)

### Idea central

Capa de **identidad** sobre OAuth 2.0. Añade un `id_token` (JWT) que el cliente puede verificar para saber **quién es el usuario**.

### Diferencia clave

| Pregunta                  | OAuth 2.0  | OIDC                |
| ------------------------- | ---------- | ------------------- |
| ¿El usuario me autorizó?  | ✅         | ✅                  |
| ¿Quién es el usuario?     | ❌         | ✅ (id_token)       |
| ¿Qué datos tiene?         | Con API    | En claims estándar  |

### Claims estándar del id_token

```json
{
  "iss": "https://accounts.google.com",
  "sub": "109283746",
  "aud": "client-id",
  "iat": 1700000000,
  "exp": 1700003600,
  "email": "ana@example.com",
  "email_verified": true,
  "name": "Ana López",
  "picture": "https://..."
}
```

### Flujo (resumido)

1. Mismo Authorization Code que OAuth.
2. El token endpoint devuelve, además del `access_token`, un `id_token` (JWT firmado por el IdP).
3. El cliente **verifica la firma** del id_token con las claves públicas del IdP (descubiertas por `/.well-known/jwks.json`).
4. Lee los claims y obtiene la identidad verificada.

### Cuándo usarlo

- Single Sign-On (SSO) empresarial.
- Cualquier escenario donde necesitas **identidad verificada por un tercero de confianza**.
- Sustituto moderno de "implementar login tú mismo".

---

## 7. Passkeys (WebAuthn)

### Idea central

**Reemplazo de la contraseña** basado en criptografía de clave pública. El dispositivo genera un par de claves; el servidor guarda la pública. Para autenticar, el dispositivo **firma un challenge** y el servidor verifica.

### Cómo se ve para el usuario

```
[1] Click en "Iniciar sesión"
[2] El sistema operativo muestra:
      "Usa tu Face ID / Touch ID / Windows Hello / llave de seguridad"
[3] El usuario aprueba
[4] Login completo
```

No hay contraseña. No hay phishing. No hay database de contraseñas para filtrar.

### Registro

```ts
// servidor genera challenge
const challenge = crypto.randomUUID();
const options = {
  challenge,
  rp: { name: "Mi App" },
  user: { id: userId, name, displayName },
  pubKeyCredParams: [{ type: "public-key", alg: -7 }], // ES256
};
// envía al cliente → navigator.credentials.create({ publicKey: options })
// recibe { id, rawId, response: { attestationObject, clientDataJSON } }
// servidor verifica y guarda { credentialId, publicKey }
```

### Autenticación

```ts
const challenge = crypto.randomUUID();
const options = { challenge, allowCredentials: [{ id: credentialId, type: "public-key" }] };
// cliente → navigator.credentials.get({ publicKey: options })
// recibe assertion firmada
// servidor verifica firma con la public key guardada
```

### Cuándo usarlo

- Sustituir contraseñas en cualquier app moderna.
- Requisito creciente en enterprise (empresas ya no permiten contraseñas simples).
- Apps móviles y web con foco en UX y seguridad.

### Estado del ecosistema (2026)

- Soporte masivo: Safari, Chrome, Firefox, Edge, iOS, Android.
- Sincronización entre dispositivos via iCloud Keychain / Google Password Manager.
- Es la recomendación de FIDO Alliance y la dirección de la industria.

---

## 8. Magic Links

### Idea central

El usuario introduce su email. El servidor **envía un enlace único de un solo uso** al email. Hacer click en el enlace **inicia sesión**.

### Flujo

```
[1] POST /auth/magic { email }
    servidor genera token = "abc123" con TTL 15min
    guarda { email, token, expira, usado: false }
    envía email: "https://app.com/auth/magic?token=abc123"

[2] Usuario hace click
    GET /auth/magic?token=abc123
    servidor valida (no expirado, no usado)
    marca como usado
    crea sesión / emite tokens
    redirige a la app
```

### Consideraciones

- **El email es la credencial**. Si alguien tiene acceso al email, tiene acceso a la cuenta. → MFA recomendado.
- **Token de un solo uso**: marcar como usado inmediatamente.
- **TTL corto**: 10-15 min.
- **Rate limit**: evitar spam y enumeración de emails.

### Cuándo usarlo

- Apps de bajo riesgo (newsletters, wikis internos, herramientas internas).
- Reducir fricción de onboarding (sin contraseña que recordar).
- Combinable con SSO o passkeys.

### Cuándo **no** usarlo

- Apps financieras, salud, datos sensibles.
- Usuarios sin acceso confiable a email en el momento.

---

## Patrones de combinación

### 1. Cookie + Session para web tradicional

```
Login form → Session + Cookie HttpOnly
Todas las请求 futuras viajan con la cookie
Logout: borrar sesión del store
```

### 2. JWT + Refresh con rotación para API + SPA

```
Login → accessToken (15m) + refreshToken (30d) en cookies HttpOnly
Cada request: Authorization: Bearer accessToken
Renovar: POST /refresh con cookie de refresh
Rotación: cada refresh emite uno nuevo
```

### 3. OAuth/OIDC + Passkeys para apps modernas

```
"Login con Google" → OAuth/OIDC
Tras login, ofrecer registrar un Passkey para下次 sin password
Usuarios enterprise: SSO con OIDC
```

### 4. Magic Link como fallback o onboarding

```
Primer login: Magic Link
Login posterior: Passkey o password
```

---

## Ataques comunes y mitigaciones

| Ataque                    | Vector                              | Mitigación                                                |
| ------------------------- | ----------------------------------- | --------------------------------------------------------- |
| XSS (Cross-Site Scripting) | Script malicioso en la página     | Cookies `HttpOnly`, CSP, sanitización                     |
| CSRF (Cross-Site Request Forgery) | Petición forzada con cookies | `SameSite=Strict/Lax`, CSRF tokens, doble submit cookie |
| Session fixation          | Atacante fija el sessionId          | Regenerar sessionId al autenticar                         |
| Token theft (localStorage) | XSS roba el token                  | `HttpOnly` cookies o storage seguro en nativo             |
| Refresh token theft       | Robo físico o lógico               | Token rotation + detección de reuso                       |
| Phishing                  | Suplantación de login               | Passkeys son resistentes; passwords requieren educación   |
| Brute force               | Adivinar passwords                  | Rate limit, MFA, lockout, captcha                         |
| Replay                    | Reutilizar tokens/links             | TTL corto, `jti`, nonces, tokens de un solo uso          |
| Token leakage en logs     | Tokens en logs/error reports        | No loguear tokens; redactar; cifrar en tránsito          |

---

## Checklist de una implementación seria

- [ ] Cookies con `HttpOnly`, `Secure`, `SameSite` correcto.
- [ ] HTTPS obligatorio en producción.
- [ ] Access tokens con TTL corto (5-15 min).
- [ ] Refresh tokens con TTL largo y **rotación**.
- [ ] Detección de reuso de refresh tokens (revocar familia).
- [ ] Logout que invalida el refresh token en el servidor.
- [ ] CSRF protection si se usan cookies.
- [ ] Rate limit en login, refresh, magic link.
- [ ] MFA / Passkey como segunda capa cuando el riesgo lo justifique.
- [ ] No loguear tokens. No enviarlos a analytics.
- [ ] Manejo de时钟 (clock skew) al validar `exp`/`iat`.
- [ ] Almacenamiento de passwords con bcrypt/argon2 (si los hay).
- [ ] Política de password solo si la necesitas (NIST: longitud > complejidad).

---

## Ejercicios

### Ejercicio 1 — Elige el mecanismo

Para cada caso, justifica:

1. Admin panel interno de una empresa.
2. App móvil que consume una API REST.
3. Marketplace que permite "Login con Google" y también email/password.
4. App de notas colaborativas para 10,000 empresas (B2B).
5. App de banco con transacciones.
6. Newsletter para suscriptores.

### Ejercicio 2 — Sesión vs JWT

Dadas estas afirmaciones, di si son verdad y por qué:

1. JWT es siempre mejor que sesiones porque no tiene estado.
2. Con JWT puedes revocar tokens instantáneamente.
3. Con sesiones, cerrar sesión cierra todas las pestañas del usuario.
4. JWT en localStorage es seguro si la app no tiene XSS.
5. Las sesiones escalan peor porque requieren un store compartido.

### Ejercicio 3 — Implementa rotación de refresh

Modela la tabla de refresh tokens y escribe las funciones:

- `crear(userId)`
- `rotar(refreshToken)`
- `revocarFamilia(familyId)`

Incluye el manejo de reuso.

### Ejercicio 4 — Flujo OAuth/OIDC con PKCE

Implementa un cliente OIDC con PKCE:

1. Genera `verifier` y `challenge`.
2. Redirige al authorization endpoint.
3. En el callback, intercambia `code` + `verifier` por tokens.
4. Verifica la firma del `id_token` con las claves del IdP (usa `jose`).

### Ejercicio 5 — Passkey end-to-end

Implementa registro y login con WebAuthn:

1. Backend genera challenge y opciones.
2. Cliente hace `navigator.credentials.create/get`.
3. Backend verifica la firma y maneja el credential store.
4. Prueba que el mismo passkey funciona en dos navegadores diferentes (sincronización).

### Ejercicio 6 — Magic link con un solo uso

Implementa:

1. Endpoint para pedir magic link con rate limit.
2. Envío de email (puedes mockear).
3. Endpoint de callback con validación de un solo uso.
4. Manejo de "token expirado", "ya usado", "email no encontrado" sin filtrar qué emails están registrados.

---

## Problema propuesto

### Plataforma SaaS multi-tenant con SSO

Vas a construir el sistema de autenticación de un **SaaS B2B** que:

- Tiene clientes empresariales (multi-tenant).
- Ofrece **SSO con Google, Microsoft y SAML/OIDC** propio del cliente.
- Tiene un panel admin donde cada empresa gestiona a sus usuarios.
- Soporta **passkeys como método principal**, con password como fallback opcional.
- Expone una **API REST** consumida por integraciones de terceros.

### Requisitos funcionales

1. **Login principal con Passkey** (WebAuthn), con password como fallback opcional.
2. **SSO empresarial**: el admin de cada tenant puede conectar su IdP (Google Workspace, Microsoft Entra, OIDC genérico, SAML).
3. **SSO social**: "Continuar con Google" y "Continuar con Microsoft" para usuarios individuales.
4. **API tokens para integraciones**: cada tenant puede emitir tokens de larga duración para consumir la API desde sus servidores.
5. **Sesiones revocables**: el admin puede cerrar la sesión de cualquier usuario de su tenant.
6. **Audit log**: cada login, logout, emisión de API token, cambio de método queda registrado.
7. **MFA opcional**: TOTP para cuentas que no usan passkey.

### Requisitos técnicos

Tu solución debe:

1. Modelar el concepto de **Tenant** y separar identidades por tenant.
2. Implementar **passkey registration y authentication** end-to-end.
3. Implementar **OIDC Authorization Code + PKCE** como cliente de Google y Microsoft.
4. Implementar **session + cookie** para el panel web, con revocación inmediata.
5. Implementar **access + refresh tokens con rotación** para la API REST.
6. **No** almacenar passwords en texto plano. Usar **argon2 o bcrypt**.
7. **Rate limit** en endpoints sensibles (login, magic link, refresh).
8. **Audit log** persistente e inmutable (append-only).

### Entregables

- Diagrama de componentes del sistema de autenticación.
- Modelo de datos (tablas: tenants, users, passkeys, sessions, refresh_tokens, api_tokens, audit_log, sso_connections).
- Implementación funcional de:
  - Registro y login con Passkey.
  - Flujo OIDC con PKCE.
  - Rotación de refresh tokens con detección de reuso.
  - Endpoint de revocación de sesión.
- Documento de 1 página justificando:
  - Por qué combinaste estos mecanismos.
  - Cómo resolviste multi-tenancy en la autenticación.
  - Qué pasa si el IdP de un cliente cae.
  - Qué harías distinto si necesitaras cumplir SOC2 o ISO 27001.

### Criterios de evaluación

| Criterio                                          | Peso |
| ------------------------------------------------- | ---- |
| Passkeys correctamente implementados              | 20%  |
| OIDC con PKCE funcional                           | 20%  |
| Rotación de refresh tokens con detección de reuso | 20%  |
| Multi-tenancy bien modelada                       | 10%  |
| Revocación inmediata de sesiones                  | 10%  |
| Audit log completo y append-only                  | 10%  |
| Justificación y trade-offs                        | 10%  |
