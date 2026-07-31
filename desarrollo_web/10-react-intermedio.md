# Módulo 10: React Intermedio

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Implementar enrutamiento SPA con React Router v6+.
- Crear y consumir Context API para estado global.
- Construir custom hooks reutilizables.
- Manejar estado complejo con `useReducer`.
- Optimizar rendimiento de componentes y hooks.
- Aplicar code splitting con `lazy` y `Suspense`.
- Capturar errores de árbol con Error Boundaries.
- Renderizar fuera del árbol DOM con Portales.

---

## 1. Routing (Enrutamiento)

Una **SPA** (Single Page Application) tiene una sola URL raíz y rutas internas que actualizan la vista sin recargar la página.

### Conceptos

- **Ruta**: emparejamiento entre URL e UI.
- **Path parameters**: `/productos/:id`.
- **Query parameters**: `?ordenar=precio`.
- **Rutas protegidas**: requieren autenticación.
- **Layouts**: rutas comparten layout (header, footer).
- **Lazy loading**: dividir bundle por ruta.

### Ejemplo conceptual sin librería

```jsx
import { useState, useEffect } from "react";

function useHashLocation() {
  const [path, setPath] = useState(window.location.hash || "#/");
  useEffect(() => {
    const onChange = () => setPath(window.location.hash || "#/");
    window.addEventListener("hashchange", onChange);
    return () => window.removeEventListener("hashchange", onChange);
  }, []);
  return path;
}
```

> En la práctica usamos **React Router** o alternativas (TanStack Router, Next.js, etc.).

### Actividad práctica

- Dibuja el mapa de rutas de una aplicación de blog (home, post, login, dashboard, 404).

---

## 2. React Router (v6+)

La librería estándar para enrutamiento en React.

### Instalación

```bash
npm install react-router-dom
```

### Tipos de routers

- **BrowserRouter** (URLs "limpias"): `https://app.com/users/42`.
- **HashRouter** (con `#`): útil en hosting estático sin rewrite de paths.
- **MemoryRouter**: para testing o React Native.

### Estructura básica

```jsx
import { BrowserRouter, Routes, Route, Link } from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Inicio</Link>{" | "}
        <Link to="/about">About</Link>{" | "}
        <Link to="/users/42">Usuario 42</Link>
      </nav>

      <Routes>
        <Route path="/"             element={<Home />} />
        <Route path="/about"        element={<About />} />
        <Route path="/users/:id"    element={<User />} />
        <Route path="*"             element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}
```

### `Link` y `NavLink`

```jsx
<Link to="/">Inicio</Link>
<NavLink to="/about" className={({ isActive }) => isActive ? "active" : ""}>
  About
</NavLink>
```

### `useNavigate`

```jsx
import { useNavigate } from "react-router-dom";

function Login() {
  const navigate = useNavigate();
  const onSuccess = () => navigate("/dashboard", { replace: true });
  return <button onClick={onSuccess}>Entrar</button>;
}
```

### `useParams`

```jsx
import { useParams } from "react-router-dom";

function User() {
  const { id } = useParams();
  return <h1>Usuario #{id}</h1>;
}
```

### `useSearchParams`

```jsx
import { useSearchParams } from "react-router-dom";

function Lista() {
  const [params, setParams] = useSearchParams();
  const ordenar = params.get("ordenar") ?? "asc";
  return (
    <select onChange={(e) => setParams({ ordenar: e.target.value })}>
      <option value="asc">Asc</option>
      <option value="desc">Desc</option>
    </select>
  );
}
```

### `useLocation`

```jsx
import { useLocation } from "react-router-dom";

function Layout() {
  const location = useLocation();
  useEffect(() => {
    console.log("ruta cambió:", location.pathname);
  }, [location]);
}
```

### Rutas anidadas y layouts

```jsx
<Routes>
  <Route path="/" element={<MainLayout />}>
    <Route index              element={<Home />} />
    <Route path="blog"        element={<Blog />} />
    <Route path="blog/:slug"  element={<Post />} />
    <Route path="dashboard"   element={<DashboardLayout />}>
      <Route index                element={<DashboardHome />} />
      <Route path="perfil"        element={<Perfil />} />
      <Route path="ajustes"       element={<Ajustes />} />
    </Route>
    <Route path="*" element={<NotFound />} />
  </Route>
</Routes>
```

```jsx
import { Outlet } from "react-router-dom";

function MainLayout() {
  return (
    <>
      <Header />
      <main><Outlet /></main>
      <Footer />
    </>
  );
}
```

### Rutas protegidas

```jsx
function RutaProtegida({ children }) {
  const { user } = useAuth();
  const location = useLocation();
  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  return children;
}

<Route path="/dashboard" element={<RutaProtegida><Dashboard /></RutaProtegida>} />
```

### Data Routers (v6.4+)

```jsx
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/",
    element: <MainLayout />,
    children: [
      { index: true, element: <Home /> },
      { path: "about", element: <About /> },
      {
        path: "users",
        loader: () => fetch("/api/users").then(r => r.json()),
        element: <Users />,
        errorElement: <Error />,
      },
    ],
  },
]);

<RouterProvider router={router} />
```

### `loader` y `useLoaderData`

```jsx
function Users() {
  const users = useLoaderData();
  return <UserList users={users} />;
}
```

### `errorElement` y `useRouteError`

```jsx
function Error() {
  const error = useRouteError();
  return <p>Error: {error.message}</p>;
}
```

### Route API (loaders, actions, form-based)

```jsx
{
  path: "/login",
  action: async ({ request }) => {
    const form = await request.formData();
    return login(Object.fromEntries(form));
  },
  element: <Login />,
}
```

### Actividad práctica

- Crea una app con rutas: home, productos, detalle, contacto, 404. Usa `<NavLink>` con clase activa.

---

## 3. Context API

Permite compartir estado entre componentes **sin pasar props manualmente** por cada nivel.

### Crear contexto

```jsx
import { createContext } from "react";

export const TemaContext = createContext("light");
```

### Proveer (`Provider`)

```jsx
import { useState } from "react";
import { TemaContext } from "./TemaContext";

function App() {
  const [tema, setTema] = useState("light");
  const value = useMemo(() => ({ tema, setTema }), [tema]);

  return (
    <TemaContext.Provider value={value}>
      <Main />
    </TemaContext.Provider>
  );
}
```

### Consumir (`useContext`)

```jsx
import { useContext } from "react";
import { TemaContext } from "./TemaContext";

function Header() {
  const { tema, setTema } = useContext(TemaContext);
  return (
    <header className={`theme--${tema}`}>
      <button onClick={() => setTema(tema === "light" ? "dark" : "light")}>
        Cambiar tema
      </button>
    </header>
  );
}
```

### `displayName` y custom hook

```jsx
TemaContext.displayName = "TemaContext";

export const useTema = () => {
  const ctx = useContext(TemaContext);
  if (!ctx) throw new Error("useTema debe estar dentro de TemaProvider");
  return ctx;
};
```

### Context para autenticación

```jsx
const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(() => {
    const raw = localStorage.getItem("user");
    return raw ? JSON.parse(raw) : null;
  });

  const login  = async (email, pwd) => { /* ... */ setUser(usuario); };
  const logout = () => { localStorage.removeItem("token"); setUser(null); };

  const value = useMemo(() => ({ user, login, logout }), [user]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export const useAuth = () => useContext(AuthContext);
```

```jsx
function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>...</Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

### Mejores prácticas

- Un context por **concern** (auth, theme, carrito) en lugar de uno global.
- Memorizar `value` con `useMemo` para evitar renders innecesarios.
- **Dividir Provider y Consumer contexts** para optimizar.

```jsx
const UserStateContext = createContext(null);
const UserDispatchContext = createContext(null);
```

### Alternativas al Context

- **Zustand**: simple, sin provider, selectores.
- **Jotai**: atómico.
- **Redux Toolkit**: para apps grandes.
- **Recoil**: ya en desuso.

### Actividad práctica

- Implementa `CarritoContext` con `items`, `total`, `agregar`, `eliminar` y persista en localStorage.

---

## 4. Custom Hooks

Hooks personalizados que encapsulan lógica reutilizable. Convención: prefijo `use`.

### Ejemplo: `useToggle`

```jsx
import { useState } from "react";

export function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  return [value, () => setValue(v => !v), setValue];
}

// Uso
const [open, toggleOpen] = useToggle();
```

### `useLocalStorage`

```jsx
import { useState, useEffect } from "react";

export function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try {
      const raw = window.localStorage.getItem(key);
      return raw ? JSON.parse(raw) : initial;
    } catch {
      return initial;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch {}
  }, [key, value]);

  return [value, setValue];
}
```

### `useFetch`

```jsx
import { useState, useEffect } from "react";

export function useFetch(url, options) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) return;
    let alive = true;
    setLoading(true);

    fetch(url, options)
      .then(r => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
      })
      .then(d => alive && setData(d))
      .catch(e => alive && setError(e))
      .finally(() => alive && setLoading(false));

    return () => { alive = false; };
  }, [url]);

  return { data, loading, error };
}
```

### `useDebounce`

```jsx
import { useState, useEffect } from "react";

export function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}
```

### `useMediaQuery`

```jsx
import { useState, useEffect } from "react";

export function useMediaQuery(query) {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);
  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = () => setMatches(mql.matches);
    mql.addEventListener("change", handler);
    return () => mql.removeEventListener("change", handler);
  }, [query]);
  return matches;
}

// const isMobile = useMediaQuery("(max-width: 600px)");
```

### `usePrevious`

```jsx
import { useRef, useEffect } from "react";

export function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; }, [value]);
  return ref.current;
}
```

### `useOnClickOutside`

```jsx
import { useEffect } from "react";

export function useOnClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (e) => {
      if (!ref.current || ref.current.contains(e.target)) return;
      handler(e);
    };
    document.addEventListener("mousedown", listener);
    document.addEventListener("touchstart", listener);
    return () => {
      document.removeEventListener("mousedown", listener);
      document.removeEventListener("touchstart", listener);
    };
  }, [ref, handler]);
}
```

### Reglas

- Empezar con `use`.
- Solo llamar hooks en el top-level.
- Se pueden componer otros hooks.

### Actividad práctica

- Construye `useIntersectionObserver(ref)` que devuelva `isIntersecting`.

---

## 5. `useReducer`

Alternativa a `useState` para **estado complejo** con múltiples sub-valores o transiciones.

### Sintaxis

```jsx
const [state, dispatch] = useReducer(reducer, initialArg, init?);
```

### Ejemplo: carrito

```jsx
const initial = { items: [], total: 0 };

function reducer(state, action) {
  switch (action.type) {
    case "ADD": {
      const existing = state.items.find(i => i.id === action.payload.id);
      const items = existing
        ? state.items.map(i => i.id === action.payload.id ? { ...i, qty: i.qty + 1 } : i)
        : [...state.items, { ...action.payload, qty: 1 }];
      const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
      return { items, total };
    }
    case "REMOVE":
      return {
        items: state.items.filter(i => i.id !== action.payload),
        total: state.total, // simplificado
      };
    case "CLEAR":
      return initial;
    default:
      return state;
  }
}

function Cart() {
  const [state, dispatch] = useReducer(reducer, initial);
  return (
    <>
      <button onClick={() => dispatch({ type: "ADD", payload: { id: 1, price: 10 } })}>
        Añadir
      </button>
      <p>{state.total}</p>
    </>
  );
}
```

### Función `init` (lazy)

```jsx
function init(initial) {
  return { count: initial };
}

const [state, dispatch] = useReducer(reducer, 0, init);
```

### `useReducer` vs `useState`

Usa `useReducer` cuando:

- Hay **transiciones** complejas.
- El siguiente estado depende del anterior en múltiples formas.
- Quieres **separar la lógica del componente**.

### Patrón "actions" tipados (TypeScript)

```ts
type State = { count: number };
type Action =
  | { type: "inc" }
  | { type: "dec" }
  | { type: "set"; payload: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "inc": return { count: state.count + 1 };
    case "dec": return { count: state.count - 1 };
    case "set": return { count: action.payload };
  }
}
```

### Actividad práctica

- Implementa un `useReducer` para un formulario multi-step o una lista de tareas con añadir/marcar/eliminar/editar.

---

## 6. Optimización

React es rápido por defecto, pero hay cuellos de botella comunes.

### `React.memo`

Compara props y evita re-render si las props no cambiaron.

```jsx
const Item = React.memo(function Item({ user, onSelect }) {
  return <li onClick={() => onSelect(user.id)}>{user.name}</li>;
});
```

Personalizar la comparación:

```jsx
React.memo(Item, (prev, next) => prev.user.id === next.user.id);
```

### `useMemo`

Memoriza **valores** calculados.

```jsx
const sorted = useMemo(() => [...items].sort(by), [items]);
const fullName = useMemo(() => `${first} ${last}`, [first, last]);
```

### `useCallback`

Memoriza **funciones**.

```jsx
const handleClick = useCallback((id) => doThing(id), [doThing]);
```

### Lazy initialization

```jsx
const [data, setData] = useState(() => expensiveInit());
```

### Keys estables

```jsx
// ✅ Estable
items.map(i => <Item key={i.id} {...i} />)
// ❌ Inestable
items.map((i, idx) => <Item key={idx} {...i} />)
```

### Renderizado controlado

```jsx
// Si solo cambia `user.id` pero `Item` solo usa `user.name`,
// pasar el id como key fuerza remontar
<Item key={user.id} user={user} />
```

### Herramientas de diagnóstico

- **React DevTools → Profiler** → graba interacciones.
- **why-did-you-render** (librería) → detecta re-renders innecesarios.
- **Virtualización** para listas largas (`react-window`).
- **Throttling** y **debouncing** en handlers de scroll/resize.

### Virtualización

```jsx
import { FixedSizeList } from "react-window";

<FixedSizeList height={500} itemSize={40} itemCount={items.length}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</FixedSizeList>
```

### Concurrent features (React 18+)

- `useTransition`: marca actualizaciones como no urgentes.
- `useDeferredValue`: "delay" intencional de un valor.
- `<Suspense>` y lazy.
- Server Components (Next.js).

### `useTransition`

```jsx
const [isPending, startTransition] = useTransition();

function handleChange(e) {
  setQuery(e.target.value);          // urgente (input)
  startTransition(() => {
    setResults(search(e.target.value)); // no urgente
  });
}
```

### Actividad práctica

- Toma una lista de 1000 elementos y aplica `memo`, `useMemo` y virtualización con `react-window`.

---

## 7. Lazy Loading (Code splitting)

Divide el bundle en chunks que se cargan bajo demanda.

### `React.lazy` + `Suspense`

```jsx
import { lazy, Suspense } from "react";

const Home    = lazy(() => import("./pages/Home"));
const About   = lazy(() => import("./pages/About"));
const Profile = lazy(() => import("./pages/Profile"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/"       element={<Home />} />
        <Route path="/about"  element={<About />} />
        <Route path="/perfil" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}
```

### Lazy de componentes secundarios

```jsx
const Modal = lazy(() => import("./Modal"));

function App() {
  const [abierto, setAbierto] = useState(false);
  return (
    <>
      <button onClick={() => setAbierto(true)}>Abrir</button>
      {abierto && (
        <Suspense fallback={<Spinner />}>
          <Modal onClose={() => setAbierto(false)} />
        </Suspense>
      )}
    </>
  );
}
```

### Webpack chunk names

```jsx
const Home = lazy(() => import(/* webpackChunkName: "home" */ "./pages/Home"));
```

Vite usa magic comments similares.

### Actividad práctica

- Divide tu app en al menos 3 chunks y observa el uso de red en DevTools.

---

## 8. Suspense

`<Suspense>` muestra un fallback mientras los hijos esperan resolver algo (lazy, data fetching, etc.).

### Con lazy

```jsx
<Suspense fallback={<Spinner />}>
  <LazyComponent />
</Suspense>
```

### Múltiples niveles

```jsx
<Suspense fallback={<SpinnerGlobal />}>
  <Layout>
    <Suspense fallback={<Spinner />}><Home /></Suspense>
    <Suspense fallback={<CardSkeleton />}><Profile /></Suspense>
  </Layout>
</Suspense>
```

### `useTransition` para evitar "spinners"

```jsx
const [isPending, startTransition] = useTransition();

function cambiar(vista) {
  startTransition(() => setVista(vista));
}

return (
  <>
    <button onClick={() => cambiar("about")}>About</button>
    {isPending && <BarraProgreso />}
    <Suspense fallback={<Spinner />}>{vista === "home" ? <Home /> : <About />}</Suspense>
  </>
);
```

### React Suspense y data fetching (experimental)

En React 18+, frameworks como Next.js, Remix o Relay ya aprovechan Suspense para data fetching.

```jsx
import { use, Suspense } from "react";

function User({ id }) {
  const data = use(fetch(`/api/users/${id}`).then(r => r.json()));
  return <p>{data.name}</p>;
}

<Suspense fallback={<Spinner />}>
  <User id="42" />
</Suspense>
```

### Actividad práctica

- Crea un árbol de rutas donde cada nivel tenga su propio `<Suspense>` y fallback específico.

---

## 9. Error Boundaries

Componentes que capturan errores de render en su árbol hijo y muestran una UI alternativa.

### Clase ErrorBoundary

```jsx
import { Component } from "react";

export default class ErrorBoundary extends Component {
  state = { error: null };

  static getDerivedStateFromError(error) {
    return { error };
  }

  componentDidCatch(error, info) {
    console.error("ErrorBoundary:", error, info.componentStack);
    // enviar a Sentry/Datadog
  }

  reset = () => this.setState({ error: null });

  render() {
    if (this.state.error) {
      return this.props.fallback ? (
        this.props.fallback(this.state.error, this.reset)
      ) : (
        <div role="alert">
          <h2>Algo salió mal</h2>
          <p>{this.state.error.message}</p>
          <button onClick={this.reset}>Reintentar</button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

### Uso

```jsx
<ErrorBoundary fallback={(err, reset) => (
  <ErrorPage error={err} onRetry={reset} />
)}>
  <App />
</ErrorBoundary>
```

### React Router `errorElement`

```jsx
{
  path: "/users",
  loader: async () => {
    const r = await fetch("/api/users");
    if (!r.ok) throw new Error("Falló");
    return r.json();
  },
  element: <Users />,
  errorElement: <ErrorPage />,
}
```

### `useRouteError` (interno a React Router)

```jsx
import { useRouteError, isRouteErrorResponse } from "react-router-dom";

function ErrorPage() {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) return <p>{error.status} {error.statusText}</p>;
  return <p>{error.message}</p>;
}
```

### Resetear tras render

```jsx
function Safe({ children, fallback = <p>Algo falló</p> }) {
  return (
    <ErrorBoundary
      fallback={(_, reset) => (
        <div>
          {fallback}
          <button onClick={reset}>Reintentar</button>
        </div>
      )}
    >
      {children}
    </ErrorBoundary>
  );
}
```

### `react-error-boundary` (librería)

```jsx
import { ErrorBoundary } from "react-error-boundary";

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onError={(error, info) => log(error, info)}
  onReset={() => clearCache()}
>
  <App />
</ErrorBoundary>
```

### Limitaciones

- No capturan errores en **handlers de eventos** (usa `try/catch`).
- No capturan en **código asíncrono fuera de React**.
- No capturan en **server-side rendering**.

### Actividad práctica

- Envuelve tu app en un `ErrorBoundary`. Lanza un error a propósito en un componente y verifica la UI de fallback.

---

## 10. Portals

Renderizan hijos en un **nodo DOM distinto** del padre en el árbol React.

### Sintaxis

```jsx
import { createPortal } from "react-dom";

function Modal({ children, onClose }) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {children}
        <button onClick={onClose}>×</button>
      </div>
    </div>,
    document.getElementById("modal-root")
  );
}
```

### Caso típico: Modal global

```html
<body>
  <div id="root"></div>
  <div id="modal-root"></div>
  <div id="toast-root"></div>
</body>
```

```jsx
function App() {
  const [abierto, setAbierto] = useState(false);
  return (
    <>
      <button onClick={() => setAbierto(true)}>Abrir modal</button>
      {abierto && (
        <Modal onClose={() => setAbierto(false)}>
          <h2>Título</h2>
          <p>Contenido</p>
        </Modal>
      )}
    </>
  );
}
```

### Toasts globales

```jsx
function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);

  const show = (msg) => setToasts(t => [...t, { id: Date.now(), msg }]);

  return (
    <ToastContext.Provider value={{ show }}>
      {children}
      {createPortal(
        <div className="toast-stack">
          {toasts.map(t => <Toast key={t.id} {...t} />)}
        </div>,
        document.getElementById("toast-root")
      )}
    </ToastContext.Provider>
  );
}
```

### Eventos burbujean al árbol React, no al DOM

```jsx
<div onClick={onClick}>
  {createPortal(<button>Click</button>, document.body)}
</div>
// El click en el botón sigue disparando onClick del div.
```

### ¿Por qué usar Portals?

- Escapar `overflow: hidden` o `z-index` del padre.
- Evitar problemas de eventos.
- Renderizar fuera de la app (modales, tooltips, dropdowns).

### `Suspense` dentro de Portales

```jsx
{createPortal(
  <Suspense fallback={<Spinner />}>
    <LazyModal />
  </Suspense>,
  document.getElementById("modal-root")
)}
```

### Actividad práctica

- Implementa un sistema de toasts (Portal en `toast-root`) usando Context API y un custom hook `useToast`.

---

## Proyecto integrador

```jsx
// src/main.jsx
import { lazy, Suspense } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";

import App from "./App.jsx";
import ErrorBoundary from "./ErrorBoundary.jsx";
import { AuthProvider } from "./context/AuthContext.jsx";
import { ToastProvider } from "./context/ToastContext.jsx";
import Spinner from "./components/Spinner.jsx";

createRoot(document.getElementById("root")).render(
  <ErrorBoundary>
    <Suspense fallback={<Spinner />}>
      <ToastProvider>
        <AuthProvider>
          <BrowserRouter>
            <App />
          </BrowserRouter>
        </AuthProvider>
      </ToastProvider>
    </Suspense>
  </ErrorBoundary>
);
```

```jsx
// src/App.jsx
import { Routes, Route } from "react-router-dom";
import MainLayout from "./layouts/MainLayout.jsx";
import RutaProtegida from "./components/RutaProtegida.jsx";
import { lazy } from "react";

const Home      = lazy(() => import("./pages/Home.jsx"));
const Productos = lazy(() => import("./pages/Productos.jsx"));
const Producto  = lazy(() => import("./pages/Producto.jsx"));
const Login     = lazy(() => import("./pages/Login.jsx"));
const Dashboard = lazy(() => import("./pages/Dashboard.jsx"));
const NotFound  = lazy(() => import("./pages/NotFound.jsx"));

export default function App() {
  return (
    <Routes>
      <Route element={<MainLayout />}>
        <Route index              element={<Home />} />
        <Route path="productos"   element={<Productos />} />
        <Route path="productos/:id" element={<Producto />} />
        <Route path="login"        element={<Login />} />
        <Route path="dashboard"    element={<RutaProtegida><Dashboard /></RutaProtegida>} />
        <Route path="*"            element={<NotFound />} />
      </Route>
    </Routes>
  );
}
```

```jsx
// src/components/RutaProtegida.jsx
import { Navigate, useLocation } from "react-router-dom";
import { useAuth } from "../context/AuthContext.jsx";

export default function RutaProtegida({ children }) {
  const { user } = useAuth();
  const location = useLocation();
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />;
  return children;
}
```

```jsx
// src/context/AuthContext.jsx
import { createContext, useState, useMemo, useContext } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(() => {
    const raw = localStorage.getItem("user");
    return raw ? JSON.parse(raw) : null;
  });

  const login  = async (email, pwd) => {
    await new Promise(r => setTimeout(r, 800));
    const u = { id: 1, name: email.split("@")[0], email };
    localStorage.setItem("user", JSON.stringify(u));
    setUser(u);
    return u;
  };

  const logout = () => {
    localStorage.removeItem("user");
    setUser(null);
  };

  const value = useMemo(() => ({ user, login, logout }), [user]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export const useAuth = () => useContext(AuthContext);
```

```jsx
// src/context/ToastContext.jsx
import { createContext, useState, useCallback, useContext } from "react";
import { createPortal } from "react-dom";

const ToastContext = createContext(null);

export function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);

  const show = useCallback((message, type = "info") => {
    const id = Date.now();
    setToasts(t => [...t, { id, message, type }]);
    setTimeout(() => setToasts(t => t.filter(x => x.id !== id)), 3000);
  }, []);

  const value = { show };
  return (
    <ToastContext.Provider value={value}>
      {children}
      {createPortal(
        <div className="toast-stack">
          {toasts.map(t => (
            <div key={t.id} className={`toast toast--${t.type}`}>{t.message}</div>
          ))}
        </div>,
        document.getElementById("toast-root")
      )}
    </ToastContext.Provider>
  );
}

export const useToast = () => useContext(ToastContext);
```

```jsx
// src/pages/Productos.jsx
import { useState, useMemo } from "react";
import { useFetch } from "../hooks/useFetch.js";
import { useDebounce } from "../hooks/useDebounce.js";

export default function Productos() {
  const [query, setQuery] = useState("");
  const debounced = useDebounce(query, 400);
  const { data, loading, error } = useFetch(
    `/api/productos?q=${encodeURIComponent(debounced)}`
  );

  const sorted = useMemo(
    () => [...(data ?? [])].sort((a, b) => a.nombre.localeCompare(b.nombre)),
    [data]
  );

  return (
    <section>
      <input
        placeholder="Buscar..."
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />

      {loading && <p>Cargando...</p>}
      {error   && <p className="error">{error.message}</p>}

      <ul>
        {sorted.map(p => <li key={p.id}>{p.nombre} - ${p.precio}</li>)}
      </ul>
    </section>
  );
}
```

---

## Recursos complementarios

### Lecturas

- [React Router Docs](https://reactrouter.com/)
- [Patterns.dev](https://www.patterns.dev/)
- [Kent C. Dodds - Blog](https://kentcdodds.com/blog)
- [React Docs - Performance](https://react.dev/learn/render-and-commit)
- [TanStack](https://tanstack.com/)

### Librerías útiles

- **React Router** (enrutamiento).
- **Zustand** (estado simple).
- **TanStack Query** (data fetching).
- **react-hook-form** + **zod** (formularios).
- **react-error-boundary**.
- **react-window**, **react-virtuoso** (virtualización).
- **framer-motion**, **React Spring** (animaciones).

### Cursos

- [Epic React](https://epicreact.dev/) (Kent C. Dodds).
- [Frontend Masters - React Path](https://frontendmasters.com/).

---

## Evaluación sugerida

1. **Routing (25%)**: app con React Router, rutas protegidas y layouts anidados.
2. **Custom hooks (25%)**: `useFetch`, `useLocalStorage`, `useDebounce`.
3. **Optimización (20%)**: refactor de una lista grande con `memo`, `useMemo` y virtualización.
4. **Error boundaries + Portals (15%)**: capturar errores y mostrar modales/toasts globales.
5. **useReducer (15%)**: estado complejo con acciones tipadas.

---

> **Nota para el estudiante**: cada feature de este módulo resuelve un problema real. Aprende a reconocer la necesidad: ¿tu estado es global? Context. ¿Tu lista es enorme? Virtualización. ¿Ruta protegida? Componente wrapper. Las herramientas son medios, no fines.
