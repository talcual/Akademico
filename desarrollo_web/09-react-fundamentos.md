# Módulo 9: React.js Fundamentos

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Comprender qué es React y por qué existe.
- Crear aplicaciones con Vite + React.
- Escribir componentes funcionales usando JSX.
- Pasar y recibir datos mediante props.
- Gestionar estado local con `useState`.
- Ejecutar efectos secundarios con `useEffect`.
- Renderizar listas y aplicar renderizado condicional.
- Manejar eventos y formularios controlados.

---

## 1. ¿Qué es React?

**React** es una **biblioteca JavaScript** de código abierto, creada por **Jordan Walke** en Facebook (2011) y liberada en 2013, para construir **interfaces de usuario** declarativas y basadas en componentes.

### Características clave

- **Declarativo**: describes **qué** quieres, no **cómo** lograrlo.
- **Basado en componentes**: UI dividida en piezas reutilizables.
- **Virtual DOM**: diff mínimo contra el DOM real.
- **Unidireccional**: el flujo de datos va de padres a hijos (props).
- **Aprendizaje progresivo**: puedes usarlo en partes de una página o para SPAs completas.
- **Ecosistema enorme**: routers, state managers, UI kits, SSR, mobile (React Native).

### ¿Por qué React y no solo JS?

Con JS puro y DOM, sincronizar el estado con la UI **manualmente** es propenso a errores. React lo hace declarativamente.

```jsx
// JSX declarativo
function Saludo({ nombre, activo }) {
  return activo ? <h1>Hola {nombre}</h1> : null;
}
```

### Virtual DOM

```
Estado cambia
   │
   ▼
React construye nuevo Virtual DOM
   │
   ▼
Diff contra el Virtual DOM anterior
   │
   ▼
Aplica cambios mínimos al DOM real
```

El Virtual DOM es una **representación JS del DOM** (un objeto `{}` con hijos, atributos, etc.). React usa el algoritmo de **reconciliación** para aplicar solo lo necesario.

### Ecosistema

- **React Router**: enrutamiento.
- **Redux Toolkit**, **Zustand**, **Jotai**: estado global.
- **React Query (TanStack Query)**, **SWR**: data fetching.
- **Next.js**, **Remix**, **Astro**: frameworks full-stack.
- **React Native**: aplicaciones nativas iOS/Android.
- **React Three Fiber**: 3D.

### Herramientas recomendadas

- **Node.js** ≥ 18.
- **Vite** (build tool recomendado moderno).
- **TypeScript** muy recomendado.

### Crear un proyecto con Vite

```bash
npm create vite@latest mi-app -- --template react
# o con TS:
npm create vite@latest mi-app -- --template react-ts
cd mi-app
npm install
npm run dev
```

### Primer componente

```jsx
// src/main.jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
```

```jsx
// src/App.jsx
export default function App() {
  return (
    <div>
      <h1>Hola, React</h1>
      <p>Mi primera aplicación.</p>
    </div>
  );
}
```

### Actividad práctica

- Crea un proyecto React con Vite y reemplaza el contenido del `App` por un componente con tu nombre.

---

## 2. JSX

**JSX** (JavaScript XML) es una extensión de sintaxis para JavaScript que permite escribir **HTML dentro de JS**.

### Reglas fundamentales

```jsx
function App() {
  return (
    <div className="container">
      <h1>Hola</h1>
      <p>Aprendiendo JSX.</p>
    </div>
  );
}
```

- **Un solo elemento raíz**: usa `<div>`, `<>`, o `React.Fragment`.
- **Cerrar siempre las etiquetas**: `<br />`, `<img />`.
- **camelCase en atributos**: `className`, `tabIndex`, `onClick`, `htmlFor`.
- **Estilos en objeto**: `style={{ color: "red", fontSize: "16px" }}`.
- **Expresiones JS con `{}`**: `{nombre}`, `{2 + 2}`, `{fn()}`.

### Fragmentos

```jsx
function App() {
  return (
    <>
      <Header />
      <Main />
      <Footer />
    </>
  );
}

// o explícito
import { Fragment } from "react";
return (
  <Fragment>
    <Header />
    <Main />
  </Fragment>
);
```

### Interpolación

```jsx
const nombre = "Ana";
const clase = "card";

return <h1 className={clase}>Hola {nombre}</h1>;
```

### Estilos en línea

```jsx
const estilo = {
  background: "#0a7d54",
  color: "white",
  padding: "1rem",
};

return <div style={estilo}>Tarjeta</div>;
```

### Comentarios

```jsx
{/* Comentario en JSX */}
{/* Otro comentario */}
```

### Ternarios y condicionales

```jsx
{isLoggedIn ? <Dashboard /> : <Login />}
{loading && <Spinner />}
```

### Componentes en mayúscula

```jsx
// ❌ JSX trata <hola> como elemento HTML
function hola() { return <p>Hola</p>; }

// ✅ Correcto (mayúscula = componente)
function Hola() { return <p>Hola</p>; }
```

### Spread de props

```jsx
const props = { id: 1, className: "card", children: "Texto" };
return <div {...props} />;
```

### Strings vs children

```jsx
<Hello name="Ana" />            // string prop
<Hello>{name}</Hello>            // children
<Hello>{[<A />, <B />]}</Hello>  // children array
```

### `dangerouslySetInnerHTML`

```jsx
const html = "<strong>Hola</strong>";
return <div dangerouslySetInnerHTML={{ __html: html }} />;
```

> ⚠️ Riesgo de XSS. Solo con datos confiables.

### Actividad práctica

- Crea un componente `TarjetaUsuario` con JSX que use variables, ternarios, estilos en objeto y un fragmento.

---

## 3. Componentes

Un **componente** es una pieza reutilizable de UI. En React moderno, son **funciones**.

### Función componente

```jsx
function Saludo(props) {
  return <h1>Hola, {props.nombre}</h1>;
}
```

### Con destructuring (preferido)

```jsx
function Saludo({ nombre }) {
  return <h1>Hola, {nombre}</h1>;
}
```

### `export` y `import`

```jsx
// Saludo.jsx
export default function Saludo({ nombre }) {
  return <h1>Hola, {nombre}</h1>;
}
```

```jsx
// App.jsx
import Saludo from "./Saludo";

export default function App() {
  return <Saludo nombre="Ana" />;
}
```

### Composición con `children`

```jsx
function Card({ children, titulo }) {
  return (
    <article className="card">
      <header><h3>{titulo}</h3></header>
      <div className="card-body">{children}</div>
    </article>
  );
}

function App() {
  return (
    <Card titulo="Aprende React">
      <p>Contenido que se muestra dentro.</p>
      <button>Ver más</button>
    </Card>
  );
}
```

### Componentes compuestos

```jsx
Card.Header = ({ children }) => <header>{children}</header>;
Card.Body   = ({ children }) => <div>{children}</div>;
```

```jsx
<Card>
  <Card.Header>Título</Card.Header>
  <Card.Body>Contenido</Card.Body>
</Card>
```

### Componentes puros vs impuros

- **Función pura**: mismas props → mismo JSX (ideal).
- **Con hooks**: pueden tener estado y efectos.

### Estructura de carpetas recomendada

```
src/
├── components/
│   ├── ui/         (Button, Card, Modal)
│   ├── layout/     (Header, Footer, Sidebar)
│   └── features/   (LoginForm, ProductList)
├── pages/          (Home, About, Profile)
├── hooks/          (useAuth, useFetch)
├── context/        (ThemeContext, AuthContext)
├── lib/            (api, utils)
├── assets/
├── App.jsx
└── main.jsx
```

### Naming convention

- **PascalCase** para componentes: `UserCard`, `LoginForm`.
- **Archivos** `.jsx` o `.tsx` igual.
- **Hooks** con `use*`: `useAuth`, `useLocalStorage`.

### Actividad práctica

- Crea 3 componentes: `Avatar`, `Badge`, `Card`. Compóngalos en un `Profile` que reciba datos de un usuario.

---

## 4. Props

Las **props** son argumentos inmutables que un padre pasa a un hijo.

### Pasar props

```jsx
function App() {
  return <Saludo nombre="Ana" edad={25} activo />;
}
```

### Recibir props

```jsx
function Saludo({ nombre, edad, activo }) {
  return (
    <p>
      Hola {nombre}, tienes {edad} años y {activo ? "estás" : "no estás"} activo.
    </p>
  );
}
```

### Default values

```jsx
function Saludo({ nombre = "Invitado", edad = 18 }) {
  return <p>Hola {nombre}</p>;
}
```

### `children`

```jsx
function Modal({ children }) {
  return <div className="modal">{children}</div>;
}

<Modal>
  <h2>Título</h2>
  <p>Contenido</p>
</Modal>
```

### Prop drilling

Pasar props por muchos niveles intermedios. Se evita con:
- **Context**.
- **State managers** (Zustand, Redux).
- **Component composition**.

### Funciones como props

```jsx
function Boton({ onClick, children }) {
  return <button onClick={onClick}>{children}</button>;
}

<Boton onClick={() => alert("Click!")}>Púlsame</Boton>;
```

### `React.ComponentProps`

```jsx
function Boton(props) {
  return <button {...props} />;
}
```

### Tipos (con TypeScript)

```tsx
interface UserCardProps {
  user: { id: number; name: string };
  onSelect?: (id: number) => void;
}

function UserCard({ user, onSelect }: UserCardProps) {
  return <h3 onClick={() => onSelect?.(user.id)}>{user.name}</h3>;
}
```

### `key` en listas

```jsx
items.map(item => <Card key={item.id} {...item} />);
```

### Actividad práctica

- Construye un `Card` reutilizable que reciba `title`, `subtitle`, `imageUrl` y `onAction`.

---

## 5. State

El **state** es información que cambia en el tiempo y que el componente "recuerda". Provoca un re-render cuando se actualiza.

### `useState` (preferido)

```jsx
import { useState } from "react";

function Contador() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Valor: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(c => c + 1)}>+1 funcional</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

### Reglas del state

- Es **inmutable**: nunca `state.x = 10`, usa `setState`.
- Es **asincrónico**: varios `setCount(c => c + 1)` se aplican correctamente.
- **Functional updates**: usar el callback cuando dependes del valor previo.

```jsx
setCount(prev => prev + 1);
```

### Estructura de estado

```jsx
const [user, setUser] = useState({ nombre: "", email: "" });

setUser({ ...user, nombre: "Ana" });
setUser(prev => ({ ...prev, email: "a@x.com" }));
```

### Levantar el estado (lift state up)

Si dos hermanos necesitan el mismo estado, se levanta al padre común:

```jsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <Display value={count} />
      <Buttons onIncrement={() => setCount(c => c + 1)} />
    </>
  );
}
```

### Estado por componente vs global

- Local: `useState`.
- Global: Context, Redux, Zustand, Jotai.

### Errores comunes

```jsx
// ❌ Mutación directa
const [user, setUser] = useState({ name: "Ana" });
user.name = "Luis";
setUser(user); // misma referencia → no re-render

// ✅ Nuevo objeto
setUser({ ...user, name: "Luis" });

// ❌ Depender del valor anterior
setCount(count + 1);
setCount(count + 1); // suma 1, no 2

// ✅ Usar callback
setCount(c => c + 1);
setCount(c => c + 1); // suma 2
```

### `useReducer`

Para estado complejo:

```jsx
function reducer(state, action) {
  switch (action.type) {
    case "inc": return { count: state.count + 1 };
    case "dec": return { count: state.count - 1 };
    case "reset": return { count: 0 };
    default: return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "inc" })}>+</button>
      <button onClick={() => dispatch({ type: "dec" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </>
  );
}
```

### Actividad práctica

- Construye un componente `Todo` con un input controlado, una lista, y funciones para añadir/eliminar.

---

## 6. Hooks

Los **Hooks** permiten usar estado, efectos y otras características de React en componentes funcionales. Reglas:

1. Empiezan con `use`.
2. Se llaman en el **nivel superior** del componente (no en condicionales, loops o funciones anidadas).
3. Solo en componentes React o en otros hooks.

### Hooks principales

| Hook             | Uso                                  |
| ---------------- | ------------------------------------ |
| `useState`       | Estado local                         |
| `useEffect`      | Efectos secundarios                  |
| `useContext`     | Consumir contexto                    |
| `useReducer`     | Estado complejo con acciones         |
| `useRef`         | Referencia mutable, no causa re-render |
| `useMemo`        | Memorizar valores costosos           |
| `useCallback`    | Memorizar funciones                  |
| `useId`          | IDs únicos accesibles                |
| `useLayoutEffect`| Efectos síncronos antes del paint    |
| `useTransition`  | Marcar actualizaciones no urgentes   |
| `useDeferredValue` | Versión diferida de un valor       |
| `useSyncExternalStore` | Sincronizar con stores externos |
| `useImperativeHandle` | Personalizar ref hacia padre   |

### Hooks personalizados

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : initial;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

function App() {
  const [theme, setTheme] = useLocalStorage("theme", "light");
  return <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
    {theme}
  </button>;
}
```

### `useRef`

```jsx
function InputFocus() {
  const ref = useRef(null);
  return (
    <>
      <input ref={ref} />
      <button onClick={() => ref.current.focus()}>Enfocar</button>
    </>
  );
}
```

### `useMemo` y `useCallback`

```jsx
const sorted = useMemo(() => items.sort((a, b) => a.n - b.n), [items]);
const handleClick = useCallback((id) => doSomething(id), []);
```

### Actividad práctica

- Crea un hook `useFetch(url)` que devuelva `{ data, loading, error }`.

---

## 7. `useState` en detalle

### Forma extendida

```jsx
const [state, setState] = useState(initialValue);
```

`initialValue` puede ser:
- Un valor directo.
- Una **función lazy** (se ejecuta solo en el primer render):

```jsx
const [items, setItems] = useState(() => {
  return JSON.parse(localStorage.getItem("items")) || [];
});
```

### Múltiples `useState`

```jsx
const [name, setName] = useState("");
const [age, setAge] = useState(0);
```

### Objeto como state

```jsx
const [form, setForm] = useState({ name: "", email: "" });

const update = (key, value) =>
  setForm(prev => ({ ...prev, [key]: value }));
```

### Array como state

```jsx
const [todos, setTodos] = useState([]);

setTodos([...todos, nuevo]);
setTodos(todos.filter(t => t.id !== id));
setTodos(todos.map(t => (t.id === id ? { ...t, done: !t.done } : t)));
```

### `useState` vs `useRef` para valores mutables

- `useState`: causa re-render al cambiar.
- `useRef`: NO causa re-render (útil para timers, instancias externas).

```jsx
const timerRef = useRef(null);
useEffect(() => {
  timerRef.current = setInterval(() => console.log("tick"), 1000);
  return () => clearInterval(timerRef.current);
}, []);
```

### Actividad práctica

- Implementa un formulario multi-step con `useState` por paso y un objeto acumulativo.

---

## 8. `useEffect` en detalle

`useEffect` ejecuta código **después del render**. Permite sincronizar con sistemas externos.

### Sintaxis

```jsx
useEffect(() => {
  // efecto
  return () => {
    // cleanup (opcional)
  };
}, [deps]);
```

### Casos

#### Cada render (evitar)

```jsx
useEffect(() => {
  console.log("cada render");
});
```

#### Solo al montar (componentDidMount)

```jsx
useEffect(() => {
  console.log("solo al montar");
}, []);
```

#### Cuando cambian las dependencias

```jsx
useEffect(() => {
  document.title = `Tienes ${count} mensajes`;
}, [count]);
```

#### Cleanup

```jsx
useEffect(() => {
  const id = setInterval(() => console.log("tick"), 1000);
  return () => clearInterval(id);
}, []);
```

### Fetch con `useEffect`

```jsx
function User({ id }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let alive = true;
    setLoading(true);
    fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(data => {
        if (alive) setUser(data);
      })
      .catch(console.error)
      .finally(() => alive && setLoading(false));

    return () => { alive = false; };
  }, [id]);

  if (loading) return <Spinner />;
  return <p>{user?.name}</p>;
}
```

### Race condition

```jsx
const [id, setId] = useState(1);

useEffect(() => {
  let ignore = false;
  fetch(`/api/users/${id}`)
    .then(r => r.json())
    .then(data => { if (!ignore) setUser(data); });
  return () => { ignore = true; };
}, [id]);
```

### Reglas importantes

- **Dependencias** declaradas correctamente (usar ESLint `react-hooks/exhaustive-deps`).
- **No llamar efectos en loops/condicionales**.
- Si quieres ejecutar una sola vez y leer valores siempre actuales, usa `useRef`.

### Tres familias de efectos

| Hook               | Cuándo se ejecuta                            |
| ------------------ | -------------------------------------------- |
| `useEffect`        | Después del render (asíncrono, no bloquea paint) |
| `useLayoutEffect`  | Antes del paint (DOM ya actualizado, síncrono) |
| `useInsertionEffect`| Antes de que se inserten estilos (CSS-in-JS) |

### Actividad práctica

- Crea un `LiveClock` que muestre la hora y se actualice cada segundo, con cleanup correcto.

---

## 9. Renderizado condicional

Mostrar UI distinta según condiciones.

### Ternario

```jsx
{isLoggedIn ? <Dashboard /> : <Login />}
```

### Cortocircuito (AND)

```jsx
{loading && <Spinner />}
{error   && <Error message={error} />}
{empty   && <EmptyState />}
```

> Cuidado: si el primer valor es `0` o `""`, ¡lo renderiza! Evita `count && ...`.

```jsx
{count > 0 && <Badge>{count}</Badge>}   // ✅ seguro
{count      && <Badge>{count}</Badge>}   // ❌ peligroso
```

### If fuera del JSX

```jsx
function App() {
  if (!user) return <Login />;
  return <Dashboard user={user} />;
}
```

### Switch temprano

```jsx
function Estado({ status }) {
  switch (status) {
    case "loading": return <Spinner />;
    case "error":   return <Error />;
    case "empty":   return <Empty />;
    default:        return <List />;
  }
}
```

### `enum` con objeto

```jsx
const VARIANTS = {
  loading: <Spinner />,
  error:   <Error />,
  empty:   <Empty />,
};

function Estado({ status, data }) {
  return VARIANTS[status] || null;
}
```

### Variables de elementos

```jsx
function App() {
  let body;
  if (loading)      body = <Spinner />;
  else if (error)   body = <Error />;
  else              body = <List items={items} />;
  return <main>{body}</main>;
}
```

### Actividad práctica

- Construye un componente `ListaInteligente` con cuatro estados: cargando, error, vacío y lleno.

---

## 10. Renderizado de listas

### `.map`

```jsx
const productos = [
  { id: 1, nombre: "Laptop",   precio: 1200 },
  { id: 2, nombre: "Mouse",    precio: 25 },
  { id: 3, nombre: "Teclado",  precio: 75 },
];

function Lista() {
  return (
    <ul>
      {productos.map(p => (
        <li key={p.id}>
          {p.nombre} - ${p.precio}
        </li>
      ))}
    </ul>
  );
}
```

### `key`

React usa `key` para identificar cada elemento.

Reglas:
- **Único** entre hermanos.
- **Estable** (no usar `index` si la lista se reordena).
- **No único globalmente**, basta con que sea único entre hermanos.

```jsx
items.map((item, index) => <li key={item.id ?? index}>...</li>);
```

### Componente por item

```jsx
function Producto({ producto }) {
  return (
    <li className="producto">
      <h3>{producto.nombre}</h3>
      <p>${producto.precio}</p>
    </li>
  );
}

function Lista({ productos }) {
  return (
    <ul>
      {productos.map(p => <Producto key={p.id} producto={p} />)}
    </ul>
  );
}
```

### Listas vacías y condicionales

```jsx
function Lista({ items }) {
  if (!items?.length) return <p>No hay elementos</p>;
  return (
    <ul>
      {items.map(item => <Item key={item.id} item={item} />)}
    </ul>
  );
}
```

### Fragmentos en listas

```jsx
items.map(item => (
  <Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.def}</dd>
  </Fragment>
));
```

### Filtros y orden

```jsx
const visibles = items.filter(i => i.active);
const ordenados = [...items].sort((a, b) => a.n.localeCompare(b.n));
```

### `useMemo` para listas costosas

```jsx
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.n.localeCompare(b.n)),
  [items]
);
```

### Listas virtualizadas

Para miles de elementos, usa librerías como:
- `react-window`.
- `react-virtuoso`.
- `tanstack/react-virtual`.

### Actividad práctica

- Renderiza una lista de 20 usuarios con búsqueda por nombre usando `useState` + `filter`.

---

## 11. Eventos en React

### Eventos sintéticos

React envuelve los eventos nativos en un wrapper llamado **SyntheticEvent**, con interfaz consistente entre navegadores.

```jsx
<button onClick={() => alert("Click!")}>Click</button>
```

### Eventos comunes

- `onClick`, `onDoubleClick`.
- `onChange`, `onInput`, `onSubmit`.
- `onFocus`, `onBlur`.
- `onKeyDown`, `onKeyPress`, `onKeyUp`.
- `onMouseEnter`, `onMouseLeave`, `onMouseOver`, `onMouseOut`, `onMouseMove`.
- `onCopy`, `onCut`, `onPaste`.
- `onScroll`, `onWheel`.
- `onDragStart`, `onDrop`, `onDragOver`.

### Pasar argumentos al handler

```jsx
<button onClick={() => eliminar(id)}>×</button>

// ⚠️ Si solo pasas eliminar(id), se ejecuta al renderizar
<button onClick={eliminar(id)}>×</button> // ❌
```

### Manejo desde el padre

```jsx
function Lista({ items, onSelect }) {
  return items.map(item => (
    <button key={item.id} onClick={() => onSelect(item)}>
      {item.nombre}
    </button>
  ));
}
```

### `event.target` vs `event.currentTarget`

```jsx
<input onChange={(e) => {
  console.log(e.target.value);       // el input
  console.log(e.currentTarget.value);// mismo en este caso
}} />
```

### `preventDefault` y `stopPropagation`

```jsx
<form onSubmit={(e) => {
  e.preventDefault();
  handleSubmit();
}} />

<div onClick={(e) => e.stopPropagation()}>
  <button onClick={onClick}>Click</button>
</div>
```

### `useCallback` para handlers estables

```jsx
const onClick = useCallback((id) => {
  console.log(id);
}, []);

return <Item key={x.id} id={x.id} onClick={onClick} />;
```

### Captura vs burbuja

```jsx
<div onClickCapture={onCapture}>
  <button onClick={onBubble}>Click</button>
</div>
```

### Eventos de teclado

```jsx
<input onKeyDown={(e) => {
  if (e.key === "Enter") submit();
  if (e.ctrlKey && e.key === "k") openCommandK();
  if (e.key === "Escape") closeModal();
}} />
```

### `onPointerDown` / `onPointerMove`

```jsx
<div
  onPointerDown={(e) => { dragRef.current = { x: e.clientX, y: e.clientY }; }}
  onPointerMove={(e) => move(e)}
  onPointerUp={() => (dragRef.current = null)}>
  ...
</div>
```

### Actividad práctica

- Crea un modal que se abra al click y se cierre con Escape o click fuera.

---

## 12. Formularios en React

### Formulario controlado

Los inputs están controlados por React (estado).

```jsx
function Login() {
  const [email, setEmail] = useState("");
  const [pwd, setPwd]     = useState("");

  const submit = (e) => {
    e.preventDefault();
    console.log({ email, pwd });
  };

  return (
    <form onSubmit={submit}>
      <label>
        Email
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
      </label>

      <label>
        Contraseña
        <input
          type="password"
          value={pwd}
          onChange={(e) => setPwd(e.target.value)}
          minLength={8}
          required
        />
      </label>

      <button type="submit">Entrar</button>
    </form>
  );
}
```

### Formulario con objeto de estado

```jsx
function Register() {
  const [form, setForm] = useState({ name: "", email: "", pwd: "", terms: false });

  const update = (e) => {
    const { name, value, type, checked } = e.target;
    setForm(prev => ({ ...prev, [name]: type === "checkbox" ? checked : value }));
  };

  const submit = (e) => {
    e.preventDefault();
    console.log(form);
  };

  return (
    <form onSubmit={submit}>
      <input name="name"  value={form.name}  onChange={update} placeholder="Nombre" required />
      <input name="email" type="email" value={form.email} onChange={update} required />
      <input name="pwd"   type="password" value={form.pwd} onChange={update} minLength={8} required />
      <label>
        <input name="terms" type="checkbox" checked={form.terms} onChange={update} />
        Acepto los términos
      </label>
      <button disabled={!form.terms}>Registrar</button>
    </form>
  );
}
```

### Validación

```jsx
function validar(form) {
  const errors = {};
  if (!form.email) errors.email = "Requerido";
  else if (!form.email.includes("@")) errors.email = "Inválido";
  if (form.pwd.length < 8) errors.pwd = "Mínimo 8 caracteres";
  return errors;
}

const [errors, setErrors] = useState({});

const submit = (e) => {
  e.preventDefault();
  const e = validar(form);
  setErrors(e);
  if (Object.keys(e).length === 0) enviar(form);
};

return (
  <form onSubmit={submit}>
    <input name="email" value={form.email} onChange={update} />
    {errors.email && <span className="error">{errors.email}</span>}
    ...
  </form>
);
```

### `useRef` para inputs no controlados

```jsx
function Buscador({ onSearch }) {
  const ref = useRef(null);
  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      onSearch(ref.current.value);
    }}>
      <input ref={ref} defaultValue="" />
      <button>Buscar</button>
    </form>
  );
}
```

> Útil cuando no necesitas el valor en cada keystroke.

### `FormData`

```jsx
const submit = (e) => {
  e.preventDefault();
  const fd = new FormData(e.target);
  const data = Object.fromEntries(fd);
  console.log(data);
};
```

### Formularios con librerías

- **React Hook Form**: minimal re-renders, validación potente.
- **Formik**: tradicional, popular.
- **Zod** + **React Hook Form**: validación tipada.

### React Hook Form

```jsx
import { useForm } from "react-hook-form";

function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email", { required: "Requerido", pattern: /@/ })} />
      {errors.email && <p>{errors.email.message}</p>}

      <input type="password" {...register("password", { minLength: 8 })} />
      <button>Enviar</button>
    </form>
  );
}
```

### `useId`

```jsx
function Field({ label, ...props }) {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} {...props} />
    </>
  );
}
```

### Actividad práctica

- Construye un formulario de registro completo con validación, mensajes de error y deshabilitar submit hasta que sea válido.

---

## Proyecto integrador del módulo

```jsx
// src/App.jsx
import { useState, useEffect } from "react";

const initial = [
  { id: 1, text: "Aprender JSX", done: false },
  { id: 2, text: "Practicar useState", done: true },
  { id: 3, text: "Dominar useEffect", done: false },
];

function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : initial;
  });
  useEffect(() => localStorage.setItem(key, JSON.stringify(value)), [key, value]);
  return [value, setValue];
}

function TodoList() {
  const [todos, setTodos] = useLocalStorage("todos", initial);
  const [text, setText] = useState("");

  const add = (e) => {
    e.preventDefault();
    if (!text.trim()) return;
    setTodos([...todos, { id: Date.now(), text, done: false }]);
    setText("");
  };

  const toggle = (id) =>
    setTodos(todos.map(t => (t.id === id ? { ...t, done: !t.done } : t)));

  const remove = (id) =>
    setTodos(todos.filter(t => t.id !== id));

  const pendientes = todos.filter(t => !t.done).length;

  return (
    <main>
      <h1>Mis tareas ({pendientes} pendientes)</h1>

      <form onSubmit={add}>
        <input
          placeholder="Nueva tarea"
          value={text}
          onChange={(e) => setText(e.target.value)}
        />
        <button>Añadir</button>
      </form>

      {todos.length === 0 ? (
        <p>No hay tareas</p>
      ) : (
        <ul>
          {todos.map(t => (
            <li key={t.id}>
              <label>
                <input type="checkbox" checked={t.done} onChange={() => toggle(t.id)} />
                <span style={{ textDecoration: t.done ? "line-through" : "none" }}>
                  {t.text}
                </span>
              </label>
              <button onClick={() => remove(t.id)} aria-label="Eliminar">×</button>
            </li>
          ))}
        </ul>
      )}
    </main>
  );
}

export default function App() {
  return <TodoList />;
}
```

---

## Recursos complementarios

### Lecturas

- [React Docs (oficial, en español)](https://es.react.dev/)
- [React Docs (inglés)](https://react.dev/)
- [Patterns.dev](https://www.patterns.dev/) (patrones de React)
- [Mark Erikson - Blog sobre React](https://overreacted.io/)

### Cursos interactivos

- [React Beta Docs tutorial](https://react.dev/learn)
- [Scrimba - Learn React](https://scrimba.com/learn/learnreact)
- [Codecademy - React](https://www.codecademy.com/learn/react-101)

### Herramientas

- [React Developer Tools](https://react.dev/learn/react-developer-tools)
- [Vite](https://vitejs.dev/)
- [Vitest](https://vitest.dev/) (testing)
- [Storybook](https://storybook.js.org/) (UI components en aislamiento)

### Librerías y frameworks a explorar

- **React Router** (enrutamiento).
- **TanStack Query** (data fetching).
- **Zustand** / **Jotai** / **Redux Toolkit** (estado global).
- **Styled Components** / **Emotion** / **TailwindCSS** (estilos).
- **Next.js**, **Remix**, **Astro** (frameworks full-stack).
- **React Native** (móvil).

---

## Evaluación sugerida

1. **Cuestionario JSX/Props/State (20%)**.
2. **Lista con filtros (30%)**: app que liste y filtre elementos.
3. **Formulario controlado (30%)**: registro con validación y `useState`.
4. **Custom hook (20%)**: implementar `useFetch` con manejo de carga y error.

---

> **Nota para el estudiante**: React cambia constantemente. Aprende los **fundamentos** (componentes, props, estado, hooks) más profundo que cada API individual. Eso te permite adaptarte a cualquier nueva versión.
