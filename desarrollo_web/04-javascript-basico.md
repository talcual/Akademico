# Módulo 4: JavaScript Básico

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Comprender la historia y evolución de JavaScript y ECMAScript.
- Declarar variables con `let`, `const` y entender `var`.
- Trabajar con los tipos de datos primitivos y objetos.
- Operar con operadores aritméticos, de comparación y lógicos.
- Controlar el flujo con condicionales y ciclos.
- Definir y usar funciones tradicionales, anónimas y arrow.
- Entender scope, hoisting y closures.
- Manipular arrays y objetos con métodos modernos.
- Aplicar destructuring, spread y rest.
- Construir strings dinámicos con template literals.

---

## 1. Historia de JavaScript

- **1995**: Brendan Eich crea JavaScript en **10 días** en Netscape. Inicialmente llamado Mocha, luego LiveScript y finalmente JavaScript (por marketing, ya que Java era popular en la época).
- **1996**: Microsoft lanza JScript (en IE3).
- **1997**: ECMA International estandariza el lenguaje como **ECMAScript** (ES1).
- **1999**: ES3 (expresiones regulares, try/catch).
- **2009**: ES5 (JSON nativo, `strict mode`, `Array.isArray`, `Object.keys`).
- **2015**: ES6 / ES2015 (la mayor actualización: `let`, `const`, arrow functions, clases, promesas, módulos, template literals).
- **2016 en adelante**: actualizaciones anuales (ES2016, ES2017, ES2018, ...).
- **Hoy**: JavaScript corre en navegadores, servidores (Node.js, Deno, Bun), dispositivos IoT, móviles (React Native), e incluso en bases de datos.

### ¿Qué puede hacer?

- Manipular el DOM y responder a eventos.
- Enviar peticiones HTTP (fetch, AJAX).
- Almacenar datos localmente (localStorage, IndexedDB).
- Crear gráficos, animaciones, juegos.
- Ejecutarse en el servidor (Node.js).
- Aplicaciones móviles, desktop, IoT.

### Actividad práctica

- Investiga qué especificación de ECMAScript soporta tu navegador desde `chrome://version` o ejecutando `console.log(navigator.userAgent)` y comparando con la tabla de compatibilidad.

---

## 2. ECMAScript

**ECMAScript** es la **especificación** del lenguaje. JavaScript es la **implementación** más conocida. Otros dialectos: JScript (Microsoft, histórico), ActionScript (Flash, histórico).

### Versiones importantes

| Versión     | Año | Características clave                                       |
| ----------- | --- | ----------------------------------------------------------- |
| ES1         | 1997 | Primera versión                                            |
| ES3         | 1999 | Expresiones regulares, try/catch                            |
| ES5         | 2009 | `strict`, `JSON`, métodos de array, getters/setters        |
| ES6 / 2015  | 2015 | let, const, arrow, class, promesa, import/export, template  |
| ES2016      | 2016 | `**`, `Array.includes`, `async/await` parcial              |
| ES2017      | 2017 | `async/await`, `Object.values`, `Object.entries`           |
| ES2018      | 2018 | Spread en objetos, async iteradores                        |
| ES2019      | 2019 | `Array.flat`, `Array.flatMap`, `Object.fromEntries`         |
| ES2020      | 2020 | Optional chaining (`?.`), nullish coalescing (`??`), BigInt |
| ES2021      | 2021 | Logical assignment (`&&=`, `||=`), `String.replaceAll`     |
| ES2022      | 2022 | Class fields, top-level await, `Array.at()`                |
| ES2023      | 2023 | `Array.findLast`, `Array.toSorted`, hashbang               |
| ES2024      | 2024 | `Object.groupBy`, `Promise.withResolvers`                  |

### Cómo probar nuevas features

```js
// Babel o esbuild transpilan código moderno a compatible con navegadores viejos.
```

### Actividad práctica

- Ejecuta en la consola de tu navegador: `1n ** 100n` (BigInt) y `Object.groupBy([1,2,3,4,5], n => n%2)`.

---

## 3. Variables

JavaScript tiene tres formas de declarar variables: `var`, `let`, `const`.

### `var` (legacy)

```js
var nombre = "Ana";
var nombre = "Luis"; // se puede redeclarar
console.log(nombre); // "Luis"
```

### `let` (preferido)

```js
let edad = 25;
edad = 26;       // reasignable
// let edad = 30; // ❌ no se puede redeclarar en el mismo scope
```

### `const` (constante de referencia)

```js
const PI = 3.1416;
// PI = 3.14;        // ❌ no se puede reasignar
const usuario = { nombre: "Ana" };
usuario.nombre = "Luis"; // ✅ se puede mutar el objeto
// usuario = {};          // ❌ no se puede reasignar
```

### Convención de nombres

```js
const MAX_USUARIOS = 100;     // constantes globales: MAYÚSCULAS
let nombreCompleto = "Ana";   // variables: camelCase
class SuperUsuario { ... }    // clases: PascalCase
```

### Actividad práctica

- Crea una variable por cada forma (`var`, `let`, `const`) e intenta redeclarar y reasignar para ver las restricciones.

---

## 4. Tipos de datos

JavaScript tiene **8 tipos** de datos. Se dividen en **primitivos** (inmutables) y **objetos** (mutables).

### Primitivos

```js
let texto      = "Hola";            // string
let numero     = 42;                // number
let decimal    = 3.14;              // number
let enteroGrande = 9007199254740991n; // bigint
let esVerdad   = true;              // boolean
let indefinido = undefined;         // undefined
let vacio      = null;              // null
let simbolo    = Symbol("id");      // symbol (único)
```

### typeof

```js
typeof "Hola"      // "string"
typeof 42          // "number"
typeof 42n         // "bigint"
typeof true        // "boolean"
typeof undefined   // "undefined"
typeof null        // "object"  ← quirk histórico
typeof {}          // "object"
typeof []          // "object"
typeof function(){} // "function"
```

### Diferencia entre `null` y `undefined`

```js
let x;            // undefined: declarado pero no asignado
let y = null;     // null: intencionalmente "sin valor"
```

### Conversión de tipos

```js
Number("42")     // 42
Number("")       // 0
Number("abc")    // NaN
parseInt("3.14") // 3
parseFloat("3.14")// 3.14
String(42)        // "42"
Boolean(0)        // false
Boolean("")       // false
Boolean("hola")   // true
```

### Truthy y Falsy

```js
// Falsy: 0, "", null, undefined, NaN, false, 0n, document.all
// Todo lo demás es truthy.
if ("hola") console.log("truthy");
if (0)      /* nada */;
```

### Coerción

```js
"5" + 3    // "53"     (concatenación)
"5" - 3    // 2        (numérico)
"5" * "2"  // 10
"a" * 2    // NaN
"5" == 5   // true     (==)
"5" === 5  // false    (===)
```

> ⚠️ Usa siempre `===` y `!==` salvo que tengas una razón clara para usar `==`.

### NaN

```js
isNaN("abc")     // true
Number.isNaN(NaN)// true (más estricto)
```

### Actividad práctica

- Crea una función `tipoDe(valor)` que retorne un string con el tipo detectado, evitando el quirk de `typeof null`.

---

## 5. Operadores

### Aritméticos

```js
let a = 10, b = 3;

a + b   // 13
a - b   // 7
a * b   // 30
a / b   // 3.333...
a % b   // 1     (módulo/resto)
a ** b  // 1000  (potencia, ES2016)

a++     // post-incremento
++a     // pre-incremento
a--     // post-decremento
```

### Asignación

```js
let x = 5;
x += 3;   // 8
x -= 2;   // 6
x *= 2;   // 12
x /= 3;   // 4
x **= 2;  // 16
x %= 3;   // 1
```

### Asignación lógica (ES2021)

```js
let a = 0;
a ||= 10;  // 10  (si a es falsy)
a &&= 5;   // 5
a ??= 7;   // 7  (si a es null o undefined)
```

### Comparación

```js
5 == "5"    // true
5 === "5"   // false ✅ recomendado
5 != "5"    // false
5 !== "5"   // true ✅
5 > 3, 5 >= 5
3 < 5, 3 <= 3
```

### Lógicos

```js
true && false   // AND
true || false   // OR
!true           // NOT
```

### Nullish coalescing `??`

```js
let nombre = null ?? "Invitado"; // "Invitado"
let cero = 0 ?? 10;              // 0 (no se reemplaza con nullish)
```

### Optional chaining `?.`

```js
const usuario = { perfil: { nombre: "Ana" } };
usuario?.perfil?.nombre; // "Ana"
usuario?.direccion?.calle; // undefined (sin error)
usuario?.metodo?.();      // llamada segura
```

### Ternario

```js
let edad = 18;
let tipo = edad >= 18 ? "adulto" : "menor";
```

### Bitwise

```js
5 & 3   // AND  : 1
5 | 3   // OR   : 7
5 ^ 3   // XOR  : 6
~5      // NOT  : -6
5 << 1  // 10
5 >> 1  // 2
```

### Actividad práctica

- Crea una función que reciba edad y tenga un valor por defecto si es `null` usando `??`.

---

## 6. Condicionales

### `if / else if / else`

```js
let edad = 18;

if (edad < 12) {
  console.log("Niño");
} else if (edad < 18) {
  console.log("Adolescente");
} else if (edad < 60) {
  console.log("Adulto");
} else {
  console.log("Adulto mayor");
}
```

### `switch`

```js
let dia = 3;
let nombre;

switch (dia) {
  case 1: nombre = "Lunes"; break;
  case 2: nombre = "Martes"; break;
  case 3: nombre = "Miércoles"; break;
  default: nombre = "Desconocido";
}
```

### Switch moderno (sin `break` agrupado)

```js
let mes = 2;
let dias;
switch (mes) {
  case 1: case 3: case 5: case 7: case 8: case 10: case 12:
    dias = 31; break;
  case 4: case 6: case 9: case 11:
    dias = 30; break;
  case 2:
    dias = 28; break;
}
```

### Valores truthy directos

```js
if (usuario.nombre) {
  console.log(`Hola ${usuario.nombre}`);
}
```

### Actividad práctica

- Construye una calculadora simple con `switch` para +, -, *, /.

---

## 7. Ciclos

### `for`

```js
for (let i = 0; i < 5; i++) {
  console.log(i);
}
```

### `while`

```js
let i = 0;
while (i < 5) {
  console.log(i);
  i++;
}
```

### `do...while`

```js
let i = 0;
do {
  console.log(i);
  i++;
} while (i < 5);
```

### `for...of` (iterables: arrays, strings, Maps, Sets)

```js
const frutas = ["manzana", "pera", "uva"];
for (const fruta of frutas) {
  console.log(fruta);
}

for (const letra of "Hola") {
  console.log(letra);
}
```

### `for...in` (claves de objetos)

```js
const user = { nombre: "Ana", edad: 25 };
for (const clave in user) {
  console.log(clave, user[clave]);
}

// ⚠️ Evita for...in en arrays. Usa for...of.
```

### Control de flujo

```js
for (let i = 0; i < 10; i++) {
  if (i === 5) break;     // sale del ciclo
  if (i % 2 === 0) continue; // salta a la siguiente iteración
  console.log(i);
}
```

### `for await...of` (iteradores async)

```js
for await (const item of asyncIterable) {
  console.log(item);
}
```

### Ciclos con Arrays

```js
const nums = [1, 2, 3, 4];
nums.forEach(n => console.log(n));
nums.map(n => n * 2);
nums.filter(n => n > 2);
nums.reduce((acc, n) => acc + n, 0);
```

### Actividad práctica

- Imprime los números primos entre 1 y 100 usando un `for` y condicionales.

---

## 8. Funciones

### Declaración de función

```js
function saludar(nombre = "Invitado") {
  return `Hola, ${nombre}`;
}
saludar("Ana"); // "Hola, Ana"
```

### Expresión de función

```js
const saludar = function (nombre) {
  return `Hola, ${nombre}`;
};
```

### Arrow function (ES6)

```js
const saludar = (nombre = "Invitado") => `Hola, ${nombre}`;
const sumar  = (a, b) => a + b;
const doble  = x => x * 2;
const imprimir = () => console.log("Listo");

// Bloque completo:
const procesar = (x) => {
  const y = x * 2;
  return y + 1;
};
```

### Argumentos y parámetros

```js
function ejemplo(a, b = 2, ...rest) {
  console.log(a, b, rest);
}
ejemplo(1, 3, 4, 5, 6);
// 1, 3, [4, 5, 6]
```

### `arguments` (en funciones tradicionales)

```js
function demo() {
  console.log(arguments.length);
}
demo(1, 2, 3); // 3
```

### Diferencia clave: `this`

```js
const obj = {
  nombre: "Ana",
  tradicional: function () { console.log(this.nombre); },     // "Ana"
  arrow:      ()      => { console.log(this.nombre); }       // undefined (this es léxico)
};
```

### Funciones como valores

```js
const ops = {
  sumar:  (a, b) => a + b,
  restar: (a, b) => a - b,
};
ops.sumar(2, 3); // 5
```

### Callbacks

```js
const procesar = (arr, fn) => fn(arr);
procesar([1,2,3], arr => arr.map(x => x * 2)); // [2,4,6]
```

### Funciones inmediatamente invocadas (IIFE)

```js
(function () {
  console.log("Ejecutada al definir");
})();

(() => {
  console.log("IIFE con arrow");
})();
```

### Actividad práctica

- Crea 3 funciones: una tradicional, una expresión y una arrow, que conviertan grados Celsius a Fahrenheit.

---

## 9. Scope

El **scope** es el alcance de una variable: dónde puede ser accedida.

### Tipos

- **Global**: declarado fuera de cualquier función o bloque. Accesible en todo el archivo.
- **De función**: con `var` dentro de `function`.
- **De bloque**: con `let` y `const` dentro de `{ }`.
- **Léxico (lexical)**: una función interna accede a variables de su función externa.

```js
const global = "global";

function externa() {
  const externaVar = "externa";

  function interna() {
    const internaVar = "interna";
    console.log(global, externaVar, internaVar); // todas accesibles
  }
  // interna(); // llamada aquí
}

// console.log(externaVar); // ❌ no accesible
```

### Bloque con `let`

```js
if (true) {
  let bloque = "solo aquí dentro";
  var noBloque = "sale del bloque";
}
// console.log(bloque);   // ❌
console.log(noBloque);   // ✅
```

### Shadowing

```js
let nombre = "global";
function saludar() {
  let nombre = "local";
  console.log(nombre); // "local"
}
saludar();
console.log(nombre);   // "global"
```

### Actividad práctica

- Crea una función anidada que demuestre cómo una variable del scope padre es accesible desde el hijo.

---

## 10. Hoisting

Las declaraciones son **izadas** (hoisted) al inicio de su scope antes de ejecutarse.

### Con `var`

```js
console.log(a); // undefined (no error)
var a = 5;
// Equivale a:
// var a;
// console.log(a);
// a = 5;
```

### Con `let` y `const`

```js
console.log(b); // ❌ ReferenceError (TDZ)
let b = 5;
```

> `let` y `const` están en la **Temporal Dead Zone (TDZ)** desde el inicio del scope hasta su declaración.

### Hoisting de funciones

```js
saludar(); // ✅ funciona
function saludar() {
  console.log("Hola");
}
```

Las **declaraciones** de funciones se hoistean completamente. Las **expresiones** no.

```js
saludar(); // ❌ TypeError
const saludar = function () { console.log("Hola"); };
```

### `function` dentro de un bloque

```js
console.log(hola); // ✅ accesible (var)
if (true) {
  function hola() { return "mundo"; }
}
```

### Actividad práctica

- Predice la salida de los siguientes 5 snippets con hoisting y luego compruébalos en la consola.

---

## 11. Closures

Una **closure** es una función que **recuerda** las variables de su scope léxico aunque ese scope ya haya terminado de ejecutarse.

```js
function crearContador() {
  let count = 0;
  return function () {
    count++;
    return count;
  };
}

const counter = crearContador();
counter(); // 1
counter(); // 2
counter(); // 3
```

### Otro ejemplo

```js
function saludar(nombre) {
  return function () {
    console.log(`Hola, ${nombre}`);
  };
}
const saludoAna = saludar("Ana");
saludoAna(); // "Hola, Ana"
```

### Caso clásico: fábrica de funciones

```js
function multiplicador(factor) {
  return function (n) {
    return n * factor;
  };
}
const por5 = multiplicador(5);
por5(3); // 15
por5(10); // 50
```

### Loop y closures (clásico bug)

```js
// ❌ Error común
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// 3, 3, 3

// ✅ Con let
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// 0, 1, 2
```

### Encapsulación de datos

```js
function crearCuenta() {
  let saldo = 0;
  return {
    depositar: (cantidad) => (saldo += cantidad),
    retirar:   (cantidad) => (saldo -= cantidad),
    obtenerSaldo: () => saldo,
  };
}
const cuenta = crearCuenta();
cuenta.depositar(100);
cuenta.obtenerSaldo(); // 100
```

### Actividad práctica

- Crea una función `crearSumador(x)` que retorne una función que sume `x` a cualquier número.

---

## 12. Arrays

Arrays en JS son **listas ordenadas** y pueden contener cualquier tipo de dato.

### Creación

```js
const vacio = [];
const nums  = [1, 2, 3];
const mix   = [1, "dos", true, null, { a: 1 }, [1, 2]];
```

### Acceso y modificación

```js
const frutas = ["manzana", "pera", "uva"];
frutas[0];          // "manzana"
frutas[1] = "mango";
frutas.length;      // 3
frutas[10] = "kiwi"; // crea huecos (sparse array)
```

### Métodos mutables (modifican el array)

```js
const a = [1, 2, 3];
a.push(4);        // [1,2,3,4]       (añade al final)
a.pop();          // [1,2,3]         (quita el último)
a.unshift(0);     // [0,1,2,3]       (añade al inicio)
a.shift();        // [1,2,3]         (quita el primero)
a.splice(1, 1);   // [1,3]           (elimina 1 desde índice 1)
a.splice(1, 0, 99); // [1,99,2,3]   (inserta sin eliminar)
a.reverse();      // [3,2,1]
a.sort();         // [1,2,3]
a.fill(0);        // [0,0,0]
a.copyWithin(0, 1); // copia parte del array sobre sí mismo
```

### Métodos no mutables (devuelven uno nuevo)

```js
const nums = [3, 1, 4, 1, 5, 9, 2, 6];

nums.map(n => n * 2);        // [6,2,8,2,10,18,4,12]
nums.filter(n => n > 3);     // [4,5,9,6]
nums.reduce((acc, n) => acc + n, 0); // 31
nums.find(n => n > 3);       // 4
nums.findIndex(n => n > 3);  // 2
nums.findLast(n => n > 3);   // 6 (ES2023)
nums.some(n => n > 5);       // true
nums.every(n => n > 0);      // true
nums.includes(4);            // true
nums.indexOf(1);             // 1
nums.lastIndexOf(1);         // 3
nums.includes(4, 2);         // true (desde índice 2)
nums.flat();                 // [3,1,4,1,5,9,2,6]
nums.join(" - ");            // "3 - 1 - 4..."
nums.slice(1, 4);            // [1,4,1]
nums.concat([7, 8]);         // [3,1,4,1,5,9,2,6,7,8]
nums.toSorted();             // [1,1,2,3,4,5,6,9] (no muta, ES2023)
nums.toReversed();           // [6,2,9,5,1,4,1,3]
```

### `flat` y `flatMap`

```js
const anidado = [1, [2, [3, [4]]]];
anidado.flat();       // [1,2,[3,[4]]]
anidado.flat(2);      // [1,2,3,[4]]
anidado.flat(Infinity);// [1,2,3,4]

[1, 2, 3].flatMap(n => [n, n * 2]);
// [1,2,2,4,3,6]
```

### `Array.from`

```js
Array.from("hola");          // ["h","o","l","a"]
Array.from({length: 5}, (_, i) => i); // [0,1,2,3,4]
```

### Iteración

```js
nums.forEach((n, i) => console.log(i, n));

for (const n of nums) console.log(n);
```

### Desestructuración de arrays (anticipo)

```js
const [a, b, c] = [1, 2, 3];
const [primero, , tercero] = [1, 2, 3]; // saltar
const [x, ...resto] = [1, 2, 3];        // resto
```

### Spread (anticipo)

```js
const a = [1, 2];
const b = [...a, 3, 4];   // [1,2,3,4]
```

### Actividad práctica

- Dada una lista de productos `{ nombre, precio }`, devuelve los nombres de los 3 más caros.

---

## 13. Objetos

Los objetos son colecciones de **pares clave-valor** (propiedades).

### Creación

```js
// Literal
const user = {
  nombre: "Ana",
  edad: 25,
  activo: true,
  hobbies: ["leer", "correr"],
  direccion: { ciudad: "Caracas" },
  saludar() {
    return `Hola, soy ${this.nombre}`;
  },
};

// Constructor
const user2 = new Object();
user2.nombre = "Luis";

// Object.create
const prototipo = { tipo: "humano" };
const user3 = Object.create(prototipo);
```

### Acceso

```js
user.nombre;              // "Ana"
user["nombre"];           // "Ana"
user["edad"] = 26;
user.nuevoAtributo ??= "default";
```

### Métodos del objeto

```js
const user = { nombre: "Ana", edad: 25 };

Object.keys(user);     // ["nombre", "edad"]
Object.values(user);   // ["Ana", 25]
Object.entries(user);  // [["nombre","Ana"],["edad",25]]
Object.fromEntries([["a",1], ["b",2]]); // {a:1, b:2}

Object.assign(target, source1, source2); // copia
const copia = { ...user }; // spread copia superficial

Object.freeze(user);  // no se puede modificar
Object.isFrozen(user);// true
Object.seal(user);    // no se pueden añadir/eliminar claves
```

### Getters y setters

```js
const user = {
  nombre: "Ana",
  get nombreMayus() { return this.nombre.toUpperCase(); },
  set nombreVal(val) { this.nombre = val.trim(); },
};

user.nombreMayus; // "ANA"
user.nombreVal = "  Luis  ";
user.nombre;      // "Luis"
```

### `this`

```js
const user = {
  nombre: "Ana",
  saludar() { return `Hola, ${this.nombre}`; },
};
user.saludar();

const fn = user.saludar;
// fn();  // ❌ this es undefined

const fn2 = user.saludar.bind(user); // ✅
```

### Prototype chain

```js
function Persona(nombre) { this.nombre = nombre; }
Persona.prototype.saludar = function () {
  return `Hola, soy ${this.nombre}`;
};
const p = new Persona("Ana");
p.saludar(); // "Hola, soy Ana"
```

### Clases (azúcar sintáctico para prototipos)

```js
class Persona {
  constructor(nombre) {
    this.nombre = nombre;
  }
  saludar() {
    return `Hola, soy ${this.nombre}`;
  }
}
const p = new Persona("Ana");
```

### JSON

```js
const json = JSON.stringify({ a: 1, b: 2 });   // '{"a":1,"b":2}'
const obj  = JSON.parse('{"a":1,"b":2}');      // {a:1, b:2}
```

### Comparación

```js
{} === {} // false (referencias)
JSON.stringify({a:1}) === JSON.stringify({a:1}) // true (por valor)
```

### Actividad práctica

- Crea un objeto `coche` con marca, modelo, año, y métodos para `acelerar()` y `frenar()` que modifiquen un atributo `velocidad`.

---

## 14. Destructuring

Permite **desempacar** valores de arrays u objetos en variables.

### Arrays

```js
const [a, b, c] = [1, 2, 3];
console.log(a, b, c); // 1 2 3

const [primero, , tercero] = [1, 2, 3]; // saltar
const [x, ...resto] = [1, 2, 3, 4];      // resto
console.log(x, resto); // 1 [2,3,4]

// Valores por defecto
const [uno = 0, dos = 0] = [, 5];
console.log(uno, dos); // 0 5

// Swap
let a = 1, b = 2;
[a, b] = [b, a];
```

### Objetos

```js
const user = { nombre: "Ana", edad: 25, email: "ana@x.com" };

const { nombre, edad } = user;
const { nombre: n, edad: e } = user; // renombrar

// Defaults
const { ciudad = "Desconocida" } = user;

// Profundo
const { direccion: { calle } } = user;

// Rest
const { nombre, ...resto } = user;
console.log(resto); // {edad:25, email:"ana@x.com"}
```

### En parámetros de funciones

```js
const saludar = ({ nombre, edad }) => `Hola, ${nombre} (${edad})`;
saludar({ nombre: "Ana", edad: 25 });
```

### Actividad práctica

- Dada una respuesta JSON de una API, extrae solo `id`, `name` y `email` usando destructuring, con valores por defecto.

---

## 15. Spread Operator (`...`)

Permite **expandir** un iterable o objeto en otro contexto.

### Spread en arrays

```js
const a = [1, 2];
const b = [3, 4];

const union  = [...a, ...b];        // [1,2,3,4]
const copia  = [...a];             // [1,2]
const conNum = [...a, 99, ...b];    // [1,2,99,3,4]
```

### Spread en strings

```js
[..."hola"]; // ["h","o","l","a"]
```

### Spread en objetos (ES2018)

```js
const user = { nombre: "Ana", edad: 25 };
const copia   = { ...user };                          // copia superficial
const merged  = { ...user, email: "ana@x.com" };      // merge
const over    = { ...user, nombre: "Luis" };          // override
```

### Casos de uso comunes

```js
// Pasar argumentos
function suma(a, b, c) { return a + b + c; }
suma(...[1, 2, 3]); // 6

// Combinar objetos inmutablemente
const actualizado = { ...user, edad: 26 };

// Quitar claves (con rest)
const { password, ...sinClave } = user;
```

### Diferencia con mutación

```js
const original = [3, 1, 4];
const copia = [...original];
copia.push(5);
console.log(original); // [3,1,4]
console.log(copia);    // [3,1,4,5]
```

### Actividad práctica

- Combina dos arrays (`a = [1,2,3]`, `b = [4,5,6]`) sin mutar ninguno y devuelve uno nuevo ordenado.

---

## 16. Rest Operator (`...`)

Reúne múltiples elementos en un **array**. La sintaxis es idéntica al spread; el contexto determina cuál es.

### En funciones

```js
function suma(...numeros) {
  return numeros.reduce((a, b) => a + b, 0);
}
suma(1, 2, 3, 4); // 10

const saludar = (mensaje, ...nombres) =>
  `${mensaje}: ${nombres.join(", ")}`;
saludar("Hola", "Ana", "Luis", "Eva");
```

### En destructuring

```js
const [primero, segundo, ...resto] = [1, 2, 3, 4, 5];
// primero=1, segundo=2, resto=[3,4,5]

const { nombre, ...demas } = { nombre: "Ana", edad: 25, email: "a@x" };
// nombre="Ana", demas={edad:25, email:"a@x"}
```

### Diferencia con el parámetro `arguments`

```js
function demo(...args) {
  console.log(args); // array real
}
demo(1, 2, 3); // [1,2,3]
```

### Actividad práctica

- Escribe una función `max(...nums)` que devuelva el mayor número pasado como argumento.

---

## 17. Template Strings (Template Literals)

Literales de string delimitados por **backticks** (`` ` ``), con interpolación y múltiples líneas.

### Interpolación

```js
const nombre = "Ana";
const edad = 25;
console.log(`Hola, soy ${nombre} y tengo ${edad} años.`);

// Expresiones
console.log(`2 + 3 = ${2 + 3}`);
console.log(`Hoy es ${new Date().toLocaleDateString()}`);

// Llamadas a funciones
console.log(`Nombre en mayúsculas: ${nombre.toUpperCase()}`);
```

### Multilínea

```js
const html = `
  <div class="card">
    <h2>${nombre}</h2>
    <p>${edad} años</p>
  </div>
`;
```

### Tagged templates

```js
function highlight(strings, ...values) {
  return strings.reduce((acc, str, i) => {
    return acc + str + (values[i] ? `<mark>${values[i]}</mark>` : "");
  }, "");
}

const nombre = "Ana";
const rol = "admin";
highlight`Hola, ${nombre}, tu rol es ${rol}`;
// "Hola, <mark>Ana</mark>, tu rol es <mark>admin</mark>"
```

### Escape

```js
const msg = `Esto es un backtick: \` y un \${variable}`;
```

### Anidación

```js
const a = 5, b = 10;
console.log(`Resultado: ${`Suma = ${a + b}`}`);
```

### Actividad práctica

- Construye una función `plantillaUsuario({nombre, edad, email})` que devuelva un HTML en template string listo para insertar.

---

## Ejemplo integrador

```js
// Módulo 4 - Ejemplo integrador

const productos = [
  { id: 1, nombre: "Laptop",   precio: 1200, stock: 5 },
  { id: 2, nombre: "Mouse",    precio: 25,   stock: 0 },
  { id: 3, nombre: "Teclado",  precio: 75,   stock: 12 },
  { id: 4, nombre: "Monitor",  precio: 350,  stock: 3 },
];

const disponibles = productos.filter(p => p.stock > 0);

const incrementarStock = (id, cantidad) => {
  const producto = productos.find(p => p.id === id);
  if (!producto) return `Producto ${id} no encontrado`;
  return { ...producto, stock: producto.stock + cantidad };
};

const nuevo = incrementarStock(2, 10);

const generarReporte = (lista) => {
  const total = lista.reduce((acc, p) => acc + p.precio * p.stock, 0);
  return `
    Reporte generado el ${new Date().toLocaleDateString()}

    Productos disponibles: ${lista.length}
    Valor total: $${total.toFixed(2)}

    ${lista.map((p, i) => `
      ${i + 1}. ${p.nombre}
         - Precio: $${p.precio}
         - Stock: ${p.stock} unidades
    `).join("\n")}
  `;
};

console.log(generarReporte(disponibles));
```

---

## Recursos complementarios

### Lecturas

- [MDN JavaScript](https://developer.mozilla.org/es/docs/Web/JavaScript)
- [JavaScript.info](https://javascript.info/)
- [Eloquent JavaScript](https://eloquentjavascript.net/) (gratis)
- [You Don't Know JS](https://github.com/getify/You-Dont-Know-JS)

### Práctica interactiva

- [freeCodeCamp - JS](https://www.freecodecamp.org/learn/javascript-algorithms-and-data-structures/)
- [Codewars](https://www.codewars.com/)
- [JavaScript30](https://javascript30.com/) (30 proyectos en 30 días)
- [Exercism - JS Track](https://exercism.org/tracks/javascript)

### Herramientas

- [Babel REPL](https://babeljs.io/repl)
- [ESLint](https://eslint.org/)
- [Prettier](https://prettier.io/)
- [Quokka.js](https://quokkajs.com/) (playground en VS Code)

---

## Evaluación sugerida

1. **Cuestionario (20%)**: tipos, coerción, scope, hoisting.
2. **Algoritmos con arrays (30%)**: resolver 5 ejercicios usando `map`, `filter`, `reduce`.
3. **Closures (20%)**: crear un contador privado y una fábrica de funciones.
4. **Refactor (30%)**: tomar código con `var` y reescribirlo a `let`/`const`, arrow functions y destructuring.

---

> **Nota para el estudiante**: JavaScript es el lenguaje más usado del mundo y uno de los más flexibles (y a veces confusos). Tómate el tiempo de entender bien la diferencia entre `var`, `let` y `const`, así como el concepto de scope y closures: te salvará de muchos bugs.
