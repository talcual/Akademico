# Módulo 11: React Avanzado

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Aplicar arquitecturas escalables como Atomic Design y Feature-Based.
- Decidir e implementar estrategias de state management global.
- Usar Redux Toolkit para estado complejo.
- Adoptar Zustand para estado global ligero.
- Implementar caching y sincronización con TanStack Query.
- Construir formularios robustos con React Hook Form.
- Validar datos con Zod y compartir esquemas cliente/servidor.

---

## 1. Arquitectura de aplicaciones React

A medida que una aplicación crece, la **organización** se vuelve tan importante como el código mismo.

### Decisiones clave

- **Cómo agrupar archivos** (capas vs features).
- **Cómo manejar el estado** (local, contexto, store global, servidor).
- **Cómo manejar efectos secundarios** (data fetching, caché, sync).
- **Cómo separar concerns** (UI vs lógica de negocio).

### Principios

- **Cohesión alta, acoplamiento bajo**: módulos independientes.
- **Convenciones compartidas**: naming, estructura, imports.
- **Capas claras**: `ui`, `features`, `pages`, `services`.
- **Barrel files** (`index.ts`) para exponer la API pública.

### Errores comunes

- "Mega componentes" con UI, lógica y llamadas al API dentro.
- Sobre-ingeniería con state global desde el día 1.
- Acoplamiento entre `features` (una feature importa el detalle de otra).
- Exceso de carpetas con un solo archivo cada una.

### Actividad práctica

- Audita una app existente y clasifica cada archivo en `ui`, `feature`, `lib`, `page`.

---

## 2. Atomic Design

Metodología de **Brad Frost** (2013) que organiza los componentes en niveles crecientes de complejidad.

### Niveles

```
┌──────────────────────────────────────┐
│ Pages        (LoginPage, HomePage)   │
├──────────────────────────────────────┤
│ Templates    (AuthLayout, MainLayout)│
├──────────────────────────────────────┤
│ Organisms    (Header, ProductCard)   │
├──────────────────────────────────────┤
│ Molecules    (SearchBar, FormField)  │
├──────────────────────────────────────┤
│ Atoms        (Button, Input, Label)  │
└──────────────────────────────────────┘
```

### Definición

#### Atoms (Átomos)

Bloques más básicos, sin lógica.

```jsx
// components/atoms/Button.jsx
export default function Button({ children, variant = "primary", ...rest }) {
  return (
    <button className={`btn btn--${variant}`} {...rest}>{children}</button>
  );
}
```

```css
.btn { padding: .5rem 1rem; border: 0; border-radius: 6px; cursor: pointer; }
.btn--primary { background: #0a7d54; color: white; }
.btn--danger  { background: #c0392b; color: white; }
.btn--ghost   { background: transparent; color: #333; }
```

```jsx
// components/atoms/Input.jsx
export default function Input({ label, error, ...rest }) {
  return (
    <label className="input">
      {label}
      <input {...rest} />
      {error && <span className="input__error">{error}</span>}
    </label>
  );
}
```

#### Molecules (Moléculas)

Combinación de átomos con un propósito concreto.

```jsx
// components/molecules/FormField.jsx
import Input from "../atoms/Input.jsx";

export default function FormField({ label, error, ...props }) {
  return <Input label={label} error={error} {...props} />;
}
```

```jsx
// components/molecules/SearchBar.jsx
import { useState } from "react";
import Input from "../atoms/Input.jsx";
import Button from "../atoms/Button.jsx";

export default function SearchBar({ onSearch }) {
  const [q, setQ] = useState("");
  return (
    <form onSubmit={(e) => { e.preventDefault(); onSearch(q); }}>
      <Input value={q} onChange={(e) => setQ(e.target.value)} placeholder="Buscar..." />
      <Button type="submit">Buscar</Button>
    </form>
  );
}
```

#### Organisms (Organismos)

Componentes con más lógica y/o estructura compleja.

```jsx
// components/organisms/ProductCard.jsx
import Button from "../atoms/Button.jsx";
import Badge from "../atoms/Badge.jsx";

export default function ProductCard({ product, onAdd }) {
  return (
    <article className="card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      {product.stock === 0 && <Badge variant="danger">Agotado</Badge>}
      <Button onClick={() => onAdd(product)} disabled={product.stock === 0}>
        Añadir al carrito
      </Button>
    </article>
  );
}
```

#### Templates

Estructura de página sin datos concretos (placeholders).

```jsx
// components/templates/AuthLayout.jsx
import Header from "../organisms/Header.jsx";

export default function AuthLayout({ children }) {
  return (
    <div className="auth-layout">
      <Header />
      <main>{children}</main>
    </div>
  );
}
```

#### Pages

Templates con datos reales.

```jsx
// pages/Login.jsx
import AuthLayout from "../components/templates/AuthLayout.jsx";
import LoginForm from "../features/auth/LoginForm.jsx";

export default function LoginPage() {
  return (
    <AuthLayout>
      <LoginForm />
    </AuthLayout>
  );
}
```

### Estructura de carpetas

```
src/
├── components/
│   ├── atoms/
│   ├── molecules/
│   ├── organisms/
│   └── templates/
├── features/
├── pages/
└── lib/
```

### Ventajas

- **Consistencia visual**.
- **Reutilización** clara.
- **Onboarding** más simple.

### Críticas

- A veces rígido para apps no UI-centric.
- Líneas entre niveles se difuminan.
- Mejor combinado con **feature-based**.

### Actividad práctica

- Construye 1 átomo (`Avatar`), 1 molécula (`UserBadge = Avatar + Nombre`), 1 organismo (`UserCard = UserBadge + stats + actions`).

---

## 3. Feature-Based Architecture

Organiza el código **por feature (caso de uso)** y no por tipo técnico. Cada feature tiene su propia carpeta con todo lo necesario.

### Estructura

```
src/
├── app/                (providers, router)
│   ├── App.jsx
│   └── providers.jsx
├── pages/              (rutas)
│   └── Home.jsx
├── features/           (una carpeta por feature)
│   ├── auth/
│   │   ├── LoginForm.jsx
│   │   ├── useAuth.js
│   │   ├── auth.api.js
│   │   ├── auth.slice.js (si Redux)
│   │   ├── auth.types.js
│   │   └── index.js     (barrel)
│   ├── products/
│   │   ├── ProductList.jsx
│   │   ├── ProductCard.jsx
│   │   ├── products.api.js
│   │   ├── useProducts.js (TanStack Query)
│   │   └── index.js
│   └── cart/
│       ├── Cart.jsx
│       ├── cart.store.js
│       └── index.js
├── components/         (átomos/moléculas compartidos)
│   ├── ui/
│   └── layout/
├── hooks/              (hooks genéricos compartidos)
│   ├── useDebounce.js
│   └── useLocalStorage.js
├── lib/                (utilidades, api client)
└── styles/
```

### Barrel file (`index.js`)

```js
// features/products/index.js
export { default as ProductList } from "./ProductList";
export { default as ProductCard } from "./ProductCard";
export { useProducts } from "./useProducts";
export { fetchProducts } from "./products.api";
```

Consumidores:

```jsx
import { ProductList, useProducts } from "@/features/products";
```

### Reglas

- Cada feature puede tener su **propio store** (Zustand, Redux slice).
- Las features pueden **comunicarse vía eventos** o shared state.
- **No importes** detalles internos desde fuera: solo el barrel.
- **`@/`** como alias a `src/`.

### Comunicación entre features

#### Shared UI

```jsx
// hooks/useAuth.js
export function useAuth() { /* ... */ }

// features/cart/CartButton.jsx
import { useAuth } from "@/hooks/useAuth";
```

#### Eventos (event bus)

```js
// lib/bus.js
import mitt from "mitt";
export const bus = mitt();
```

```jsx
// features/auth/onLogin.js
bus.emit("auth:login", user);

// features/cart/listener.js
bus.on("auth:login", user => cart.attach(user));
```

#### Stores compartidos

```js
// features/cart/cart.store.js
import { create } from "zustand";
import { bus } from "@/lib/bus";

export const useCart = create((set, get) => ({
  items: [],
  add: (p) => set(s => ({ items: [...s.items, p] })),
  clear: () => set({ items: [] }),
}));

bus.on("auth:logout", () => get().clear());
```

### Actividad práctica

- Migra una app de "organización por tipo" a "organización por feature" identificando los límites.

---

## 4. State Management (visión general)

El **estado** es cualquier valor que cambia con el tiempo y afecta la UI.

### Categorías

| Tipo                | Ejemplos                          | Solución típica                  |
| ------------------- | --------------------------------- | -------------------------------- |
| **Local UI**        | Input, modal abierto, hover       | `useState`, `useReducer`         |
| **Compartido**      | Tema, usuario, carrito            | Context, Zustand, Redux          |
| **Servidor**        | Productos, posts, user            | TanStack Query, SWR, RTK Query  |
| **URL**             | Filtros, paginación, sort         | React Router, nuqs               |
| **Formulario**      | Estado de inputs, validación      | React Hook Form                  |
| **Global efímero**  | Toasts, dialogs                   | Zustand, Context con reducer     |
| **Persistente**     | Drafts, preferencias              | localStorage + Zustand persist   |

### Regla de oro

> Empieza con `useState` y `useReducer`. Sube a **Context/Zustand** cuando el estado sea compartido entre muchos componentes. Usa **TanStack Query** para datos del servidor. Lleva a **Redux Toolkit** solo cuando necesites time-travel, middlewares, ecosystem o app muy grande.

### Cuándo evitar el estado global

- Solo lo necesita **un componente**: local.
- Solo lo lee un componente pero viene de una prop: **levantar el estado**.
- Datos del servidor: usar TanStack Query (que ya tiene caché).

### Actividad práctica

- Dado un caso (tema oscuro, lista de productos, formulario, modal global), decide qué solución aplicarías.

---

## 5. Redux Toolkit

Redux Toolkit (RTK) es la **forma oficial y moderna** de usar Redux.

### Instalación

```bash
npm install @reduxjs/toolkit react-redux
```

### Conceptos

- **Store** único.
- **Slices** (reducers + actions combinados).
- **createSlice** (sintaxis moderna).
- **createAsyncThunk** (peticiones asíncronas).
- **RTK Query** (alternativa a TanStack Query).
- **Provider** + **hooks** (`useSelector`, `useDispatch`).

### Ejemplo: slice de autenticación

```js
// app/store.js
import { configureStore } from "@reduxjs/toolkit";
import authReducer from "../features/auth/auth.slice";
import productsReducer from "../features/products/products.slice";

export const store = configureStore({
  reducer: { auth: authReducer, products: productsReducer },
});
```

```js
// features/auth/auth.slice.js
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

export const login = createAsyncThunk(
  "auth/login",
  async ({ email, password }) => {
    const res = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });
    if (!res.ok) throw new Error("Login failed");
    return res.json();
  }
);

const slice = createSlice({
  name: "auth",
  initialState: { user: null, token: null, status: "idle", error: null },
  reducers: {
    logout(state) {
      state.user = null;
      state.token = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(login.pending, (state) => { state.status = "loading"; state.error = null; })
      .addCase(login.fulfilled, (state, action) => {
        state.status = "succeeded";
        state.user = action.payload.user;
        state.token = action.payload.token;
      })
      .addCase(login.rejected, (state, action) => {
        state.status = "failed";
        state.error = action.error.message;
      });
  },
});

export const { logout } = slice.actions;
export default slice.reducer;
```

### Provider

```jsx
// main.jsx
import { Provider } from "react-redux";
import { store } from "./app/store.js";

createRoot(document.getElementById("root")).render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

### Hooks

```jsx
// features/auth/Login.jsx
import { useDispatch, useSelector } from "react-redux";
import { login, logout } from "./auth.slice";

export default function Login() {
  const dispatch = useDispatch();
  const { user, status, error } = useSelector(s => s.auth);

  const submit = (e) => {
    e.preventDefault();
    const fd = new FormData(e.target);
    dispatch(login(Object.fromEntries(fd)));
  };

  if (user) return <button onClick={() => dispatch(logout())}>Logout ({user.email})</button>;

  return (
    <form onSubmit={submit}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button disabled={status === "loading"}>
        {status === "loading" ? "Cargando..." : "Entrar"}
      </button>
      {error && <p className="error">{error}</p>}
    </form>
  );
}
```

### RTK Query

```js
// features/products/products.api.js
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const productsApi = createApi({
  reducerPath: "productsApi",
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  tagTypes: ["Producto"],
  endpoints: (builder) => ({
    getProductos: builder.query({
      query: () => "/productos",
      providesTags: ["Producto"],
    }),
    addProducto: builder.mutation({
      query: (body) => ({ url: "/productos", method: "POST", body }),
      invalidatesTags: ["Producto"],
    }),
  }),
});

export const { useGetProductosQuery, useAddProductoMutation } = productsApi;
```

```js
// app/store.js
import { productsApi } from "../features/products/products.api";

export const store = configureStore({
  reducer: {
    [productsApi.reducerPath]: productsApi.reducer,
    // otros reducers
  },
  middleware: (getDefault) => getDefault().concat(productsApi.middleware),
});
```

```jsx
// features/products/Productos.jsx
import { useGetProductosQuery, useAddProductoMutation } from "./products.api";

export default function Productos() {
  const { data, isLoading, error } = useGetProductosQuery();
  const [add] = useAddProductoMutation();

  if (isLoading) return <p>Cargando...</p>;
  if (error)     return <p>Error</p>;

  return (
    <ul>
      {data.map(p => <li key={p.id}>{p.nombre} - ${p.precio}</li>)}
      <button onClick={() => add({ nombre: "Nuevo", precio: 10 })}>Añadir</button>
    </ul>
  );
}
```

### DevTools y middleware

```js
middleware: (gDM) => gDM().concat(loggerMiddleware)
```

### Cuándo usar Redux Toolkit

- App muy grande con mucho estado global compartido.
- Necesitas **time-travel debugging**, **middlewares**, persistencia avanzada.
- Equipo que ya conoce Redux.

### Actividad práctica

- Implementa un slice `cart` con add/remove/clear usando `createSlice` y `useDispatch`.

---

## 6. Zustand

Store global **ligero** (~1KB), sin providers obligatorios, basado en hooks.

### Instalación

```bash
npm install zustand
```

### Conceptos

- `create((set, get) => ({ ... }))`.
- Selectores para optimizar renders.
- Middlewares: `persist`, `devtools`, `immer`, `subscribeWithSelector`.

### Ejemplo: store de carrito

```js
// features/cart/cart.store.js
import { create } from "zustand";
import { persist } from "zustand/middleware";

export const useCart = create(
  persist(
    (set, get) => ({
      items: [],
      add: (product) => set((s) => ({
        items: [...s.items, { ...product, qty: 1 }],
      })),
      remove: (id) => set((s) => ({
        items: s.items.filter((i) => i.id !== id),
      })),
      clear: () => set({ items: [] }),
      total: () => get().items.reduce((s, i) => s + i.price * i.qty, 0),
    }),
    { name: "cart-storage" }
  )
);
```

```jsx
// features/cart/CartButton.jsx
import { useCart } from "./cart.store";

export default function CartButton({ product }) {
  const add = useCart((s) => s.add);
  return <button onClick={() => add(product)}>Añadir</button>;
}

// features/cart/CartCount.jsx
export default function CartCount() {
  const count = useCart((s) => s.items.length);
  return <span>{count}</span>;
}
```

### Selectores finos

```jsx
const qty = useCart((s) => s.items.find(i => i.id === id)?.qty ?? 0);
```

### Acciones asíncronas

```js
export const useCart = create((set) => ({
  items: [],
  loading: false,
  error: null,
  fetchAll: async () => {
    set({ loading: true });
    try {
      const res = await fetch("/api/cart");
      const items = await res.json();
      set({ items, loading: false });
    } catch (e) {
      set({ error: e.message, loading: false });
    }
  },
  add: (p) => set((s) => ({ items: [...s.items, p] })),
}));
```

### Suscripciones fuera de React

```js
const unsub = useCart.subscribe(
  (state) => state.items,
  (items, prev) => console.log("items cambió", items)
);
unsub();
```

### `devtools` y `immer`

```js
import { create } from "zustand";
import { devtools } from "zustand/middleware";
import { immer } from "zustand/middleware/immer";

export const useAuth = create(
  devtools(
    immer((set) => ({
      user: null,
      login: (user) => set((s) => { s.user = user; }),
    })),
    { name: "auth" }
  )
);
```

### Cuándo usar Zustand

- Apps pequeñas/medianas.
- Quieres código simple, sin boilerplate.
- Prefieres TypeScript-friendly.

### Actividad práctica

- Crea `useTheme` con Zustand + `persist` y un toggle en la UI.

---

## 7. TanStack Query

Librería para **gestionar estado del servidor**: fetching, caching, sincronización, mutaciones, paginación.

### Instalación

```bash
npm install @tanstack/react-query
```

### Provider

```jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const client = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 5 * 60 * 1000, retry: 1, refetchOnWindowFocus: false },
  },
});

createRoot(document.getElementById("root")).render(
  <QueryClientProvider client={client}>
    <App />
  </QueryClientProvider>
);
```

### Query

```jsx
import { useQuery } from "@tanstack/react-query";

function Productos() {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ["productos"],
    queryFn: () => fetch("/api/productos").then(r => r.json()),
  });

  if (isLoading) return <p>Cargando...</p>;
  if (error)     return <p>Error: {error.message}</p>;
  return (
    <>
      <button onClick={() => refetch()}>Refrescar</button>
      <ul>{data.map(p => <li key={p.id}>{p.nombre}</li>)}</ul>
    </>
  );
}
```

### Con parámetros

```jsx
const { data } = useQuery({
  queryKey: ["productos", { q, page }],
  queryFn: ({ queryKey }) => {
    const [, { q, page }] = queryKey;
    const url = `/api/productos?q=${q}&page=${page}`;
    return fetch(url).then(r => r.json());
  },
  keepPreviousData: true,
});
```

### Mutations

```jsx
import { useMutation, useQueryClient } from "@tanstack/react-query";

function AddProducto() {
  const qc = useQueryClient();
  const mutation = useMutation({
    mutationFn: (nuevo) =>
      fetch("/api/productos", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(nuevo),
      }).then(r => r.json()),
    onSuccess: () => qc.invalidateQueries({ queryKey: ["productos"] }),
  });

  return <button onClick={() => mutation.mutate({ nombre: "Nuevo", precio: 10 })}>
    {mutation.isPending ? "Guardando..." : "Añadir"}
  </button>;
}
```

### Optimistic updates

```js
useMutation({
  mutationFn: (todo) => updateTodo(todo),
  onMutate: async (newTodo) => {
    await qc.cancelQueries({ queryKey: ["todos"] });
    const prev = qc.getQueryData(["todos"]);
    qc.setQueryData(["todos"], (old) => [...old, newTodo]);
    return { prev };
  },
  onError: (e, vars, ctx) => qc.setQueryData(["todos"], ctx.prev),
  onSettled: () => qc.invalidateQueries({ queryKey: ["todos"] }),
});
```

### Infinite queries (scroll infinito)

```jsx
import { useInfiniteQuery } from "@tanstack/react-query";

function Feed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ["feed"],
    queryFn: ({ pageParam = 1 }) =>
      fetch(`/api/feed?page=${pageParam}`).then(r => r.json()),
    getNextPageParam: (last) => last.nextPage ?? false,
  });

  return (
    <>
      {data?.pages.flatMap(p => p.items).map(i => <p key={i.id}>{i.text}</p>)}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          Cargar más
        </button>
      )}
    </>
  );
}
```

### Devtools

```jsx
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

<QueryClientProvider>
  <App />
  <ReactQueryDevtools initialIsOpen={false} />
</QueryClientProvider>
```

### Actividad práctica

- Construye un CRUD completo contra una API con queries, mutations, y optimistic updates.

---

## 8. React Hook Form

Librería para formularios performantes (mínimos re-renders) y ergonomía superior.

### Instalación

```bash
npm install react-hook-form
```

### Básico

```jsx
import { useForm } from "react-hook-form";

function Login() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    defaultValues: { email: "", password: "" },
  });

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register("email", {
          required: "Requerido",
          pattern: { value: /@/, message: "Email inválido" },
        })}
      />
      {errors.email && <span>{errors.email.message}</span>}

      <input
        type="password"
        {...register("password", {
          required: "Requerido",
          minLength: { value: 8, message: "Mínimo 8" },
        })}
      />
      {errors.password && <span>{errors.password.message}</span>}

      <button>Enviar</button>
    </form>
  );
}
```

### Registro con `Controller` (para inputs no nativos)

```jsx
import { Controller } from "react-hook-form";

<Controller
  name="rol"
  control={control}
  rules={{ required: "Requerido" }}
  render={({ field }) => (
    <Select
      {...field}
      options={roles}
      onChange={(v) => field.onChange(v)}
      value={field.value}
    />
  )}
/>
```

### `watch` y `setValue`

```jsx
const password = watch("password");

<Controller
  name="confirm"
  rules={{
    validate: (v) => v === password || "No coincide",
  }}
  ...
/>
```

### `useFieldArray`

```jsx
import { useFieldArray } from "react-hook-form";

function Form() {
  const { control, register, handleSubmit } = useForm({
    defaultValues: { phones: [{ number: "" }] },
  });

  const { fields, append, remove } = useFieldArray({ name: "phones", control });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {fields.map((f, i) => (
        <div key={f.id}>
          <input {...register(`phones.${i}.number`)} />
          <button type="button" onClick={() => remove(i)}>×</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ number: "" })}>Añadir</button>
      <button>Enviar</button>
    </form>
  );
}
```

### `formState`

```jsx
const {
  isDirty, isSubmitting, isValid, errors,
  touchedFields, dirtyFields, submitCount,
} = formState;

<button disabled={!isDirty || isSubmitting}>Guardar</button>
```

### `mode` y `reValidateMode`

```js
useForm({ mode: "onBlur", reValidateMode: "onChange" });
```

- `onSubmit` (default): valida al enviar.
- `onChange`, `onBlur`, `onTouched`, `all`.

### Yup / Zod como resolver

```jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

const { register, handleSubmit, formState: { errors } } = useForm({
  resolver: zodResolver(schema),
});
```

### Actividad práctica

- Construye un registro de usuarios con validación, campos condicionales y `useFieldArray` para teléfonos múltiples.

---

## 9. Validaciones con Zod

Zod es una librería de **declaración y validación de esquemas** TypeScript-first.

### Instalación

```bash
npm install zod
npm install @hookform/resolvers  # si la usas con RHF
```

### Esquemas básicos

```js
import { z } from "zod";

const UserSchema = z.object({
  id: z.number().int().positive(),
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().int().min(18).max(120).optional(),
  active: z.boolean().default(true),
  role: z.enum(["admin", "user", "guest"]),
  tags: z.array(z.string()).min(1),
});
```

### Validar

```js
const result = UserSchema.safeParse({
  id: 1, name: "Ana", email: "ana@x.com", age: 25, role: "admin", tags: ["js"],
});

if (!result.success) console.log(result.error.issues);
else console.log(result.data);
```

### Tipos derivados

```js
export type User = z.infer<typeof UserSchema>;
```

### Validación de formularios

```js
const LoginSchema = z.object({
  email: z.string().email("Email inválido"),
  password: z.string().min(8, "Mínimo 8 caracteres"),
  remember: z.boolean().optional(),
});
```

### Validaciones complejas

```js
const SignupSchema = z.object({
  password: z.string().min(8),
  confirm:  z.string(),
}).refine((d) => d.password === d.confirm, {
  message: "Las contraseñas no coinciden",
  path: ["confirm"],
});

const PromoSchema = z.object({
  birthdate: z.date().refine(
    (d) => d <= new Date(),
    "Fecha futura"
  ),
});
```

### Transformación y coerción

```js
const Coerced = z.object({
  age:    z.coerce.number().int().min(0),  // "30" -> 30
  active: z.coerce.boolean(),              // "true" -> true
  date:   z.coerce.date(),                 // ISO string -> Date
});
```

### `parse` vs `safeParse`

```js
try {
  UserSchema.parse(data);
} catch (err) {
  console.error(err.issues);
}

const result = UserSchema.safeParse(data);
if (!result.success) console.error(result.error.flatten());
```

### Con React Hook Form

```jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8, "Mínimo 8"),
  confirm: z.string(),
}).refine(d => d.password === d.confirm, {
  message: "No coincide",
  path: ["confirm"],
});

function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
  });
  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register("email")} />
      {errors.email?.message && <span>{errors.email.message}</span>}

      <input type="password" {...register("password")} />
      {errors.password?.message && <span>{errors.password.message}</span>}

      <input type="password" {...register("confirm")} />
      {errors.confirm?.message && <span>{errors.confirm.message}</span>}

      <button>Registrar</button>
    </form>
  );
}
```

### Compartir esquemas cliente/servidor

```js
// shared/schemas/user.js
import { z } from "zod";
export const UserSchema = z.object({ ... });
```

```js
// server routes
import { UserSchema } from "../../../shared/schemas/user";

app.post("/users", (req, res) => {
  const result = UserSchema.safeParse(req.body);
  if (!result.success) return res.status(400).json(result.error.flatten());
  // guardar
});
```

### Validación de queries

```js
const QuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  search: z.string().optional(),
});

router.get("/productos", (req, res) => {
  const { page, limit, search } = QuerySchema.parse(req.query);
  // ...
});
```

### Actividad práctica

- Crea un esquema para un registro de producto con nombre, precio (>0), stock (>=0), categoría y tags; úsalo con RHF.

---

## Proyecto integrador del módulo

```jsx
// src/main.jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { Provider } from "react-redux";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { store } from "./app/store.js";
import App from "./App.jsx";

const queryClient = new QueryClient();

export default function Root() {
  return (
    <Provider store={store}>
      <QueryClientProvider client={queryClient}>
        <App />
        <ReactQueryDevtools initialIsOpen={false} />
      </QueryClientProvider>
    </Provider>
  );
}
```

```jsx
// src/features/products/products.queries.js
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

const fetchProducts = () => fetch("/api/productos").then(r => r.json());
const addProduct = (p) =>
  fetch("/api/productos", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(p),
  }).then(r => r.json());

export function useProducts() {
  return useQuery({ queryKey: ["products"], queryFn: fetchProducts, staleTime: 60_000 });
}

export function useAddProduct() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: addProduct,
    onSuccess: () => qc.invalidateQueries({ queryKey: ["products"] }),
  });
}
```

```jsx
// src/features/cart/cart.store.js (Zustand)
import { create } from "zustand";
import { persist } from "zustand/middleware";

export const useCart = create(persist(
  (set, get) => ({
    items: [],
    add: (p) => set(s => {
      const existing = s.items.find(i => i.id === p.id);
      if (existing) {
        return { items: s.items.map(i => i.id === p.id ? { ...i, qty: i.qty + 1 } : i) };
      }
      return { items: [...s.items, { ...p, qty: 1 }] };
    }),
    remove: (id) => set(s => ({ items: s.items.filter(i => i.id !== id) })),
    clear: () => set({ items: [] }),
    count: () => get().items.reduce((acc, i) => acc + i.qty, 0),
    total: () => get().items.reduce((acc, i) => acc + i.price * i.qty, 0),
  }),
  { name: "cart" }
));
```

```jsx
// src/features/products/schema.js
import { z } from "zod";

export const ProductSchema = z.object({
  nombre: z.string().min(2, "Mínimo 2 caracteres"),
  precio: z.coerce.number().positive("Mayor a 0"),
  stock:  z.coerce.number().int().min(0, ">=0"),
  categoria: z.enum(["tecnologia", "hogar", "moda", "otros"]),
  tags: z.array(z.string()).default([]),
});
```

```jsx
// src/features/products/AddProductForm.jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { ProductSchema } from "./schema";
import { useAddProduct } from "./products.queries";

export default function AddProductForm({ onClose }) {
  const {
    register, handleSubmit, reset,
    formState: { errors, isSubmitting },
  } = useForm({ resolver: zodResolver(ProductSchema) });

  const { mutateAsync } = useAddProduct();

  const onSubmit = async (data) => {
    await mutateAsync(data);
    reset();
    onClose?.();
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input placeholder="Nombre" {...register("nombre")} />
      {errors.nombre && <span>{errors.nombre.message}</span>}

      <input type="number" step="0.01" placeholder="Precio" {...register("precio")} />
      {errors.precio && <span>{errors.precio.message}</span>}

      <input type="number" placeholder="Stock" {...register("stock")} />
      {errors.stock && <span>{errors.stock.message}</span>}

      <select {...register("categoria")}>
        <option value="">Selecciona...</option>
        <option value="tecnologia">Tecnología</option>
        <option value="hogar">Hogar</option>
        <option value="moda">Moda</option>
        <option value="otros">Otros</option>
      </select>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Guardando..." : "Guardar"}
      </button>
    </form>
  );
}
```

```jsx
// src/features/products/ProductList.jsx
import { useProducts } from "./products.queries";
import { useCart } from "../cart/cart.store";

export default function ProductList() {
  const { data, isLoading, error } = useProducts();
  const add = useCart(s => s.add);

  if (isLoading) return <p>Cargando...</p>;
  if (error)     return <p>Error: {error.message}</p>;

  return (
    <ul>
      {data.map(p => (
        <li key={p.id}>
          <h3>{p.nombre}</h3>
          <p>${p.precio}</p>
          <button onClick={() => add(p)}>Añadir</button>
        </li>
      ))}
    </ul>
  );
}
```

```jsx
// src/features/cart/CartBadge.jsx
import { useCart } from "./cart.store";

export default function CartBadge() {
  const count = useCart(s => s.items.reduce((acc, i) => acc + i.qty, 0));
  return <span className="badge">{count}</span>;
}
```

---

## Recursos complementarios

### Lecturas

- [Redux Toolkit Docs](https://redux-toolkit.js.org/)
- [Zustand Docs](https://github.com/pmndrs/zustand)
- [TanStack Query Docs](https://tanstack.com/query/latest)
- [React Hook Form Docs](https://react-hook-form.com/)
- [Zod Docs](https://zod.dev/)

### Cursos y libros

- **Mark Erikson - Redux Essentials / Redux Fundamentals**.
- **TkDodo's Blog** (mantenedor de TanStack Query).
- **Epic React** (Kent C. Dodds).
- **Bulletproof React** (bulletproof-react.com).

### Librerías adicionales

- **React Query**: caching.
- **SWR** (alternativa a TanStack Query).
- **Yup** (alternativa a Zod).
- **Immer** (mutaciones inmutables).
- **Jotai** (estado atómico).
- **Recoil** (estado atómico de Meta).
- **MobX** (reactivo).

---

## Evaluación sugerida

1. **Atomic Design + Feature Based (20%)**: app con jerarquía atómica + features aisladas.
2. **Redux Toolkit (20%)**: slice con `createAsyncThunk`, login, dashboard protegido.
3. **Zustand + persist (15%)**: carrito con persistencia en localStorage.
4. **TanStack Query (25%)**: CRUD con optimistic updates.
5. **React Hook Form + Zod (20%)**: formulario complejo con `useFieldArray` y validación derivada.

---

> **Nota para el estudiante**: cada herramienta es un trade-off. RTK es robusto pero verboso; Zustand es simple pero no tiene ecosistema de middlewares enorme; TanStack Query no es un estado global sino **del servidor**. Elige en función del problema real, no de la popularidad.
