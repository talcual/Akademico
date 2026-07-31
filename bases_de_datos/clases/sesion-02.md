# Sesión 2 — Módulo 2 (parte 1): Modelado de Datos

**Semana:** 2
**Corte:** 1
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema de Biblioteca

---

## 1. Análisis de requerimientos

Antes de modelar, hay que **entender el problema**. El análisis de requerimientos consiste en entrevistar al cliente, leer documentos y responder:

- ¿Qué información se necesita guardar?
- ¿Quién la va a usar?
- ¿Qué operaciones se realizarán?
- ¿Qué reglas de negocio aplican?

### Técnicas

- Entrevistas con el cliente.
- Revisión de documentos (facturas, formatos, planillas).
- Observación del proceso real.
- Lluvia de ideas con el equipo.

### Ejemplo: Biblioteca

> "Necesitamos un sistema para registrar los libros de la biblioteca, los socios que se inscriben, y los préstamos que se realizan. Cada libro puede tener varios ejemplares. Un socio puede tener máximo 3 préstamos activos. Los préstamos duran 15 días."

De aquí extraemos:
- **Entidades candidatas:** Libro, Ejemplar, Socio, Préstamo.
- **Reglas:** máximo 3 préstamos por socio, 15 días de duración.

---

## 2. Entidades

Una **entidad** representa un objeto del mundo real con existencia independiente. En un diagrama ER se representa con un **rectángulo**.

### Tipos

- **Entidad fuerte:** existe por sí misma. Ej: `Libro`, `Socio`.
- **Entidad débil:** depende de otra para existir. Ej: `Ejemplar` (depende de `Libro`), `Préstamo` (depende de `Socio` y `Ejemplar`).

### Ejemplo — Biblioteca

```
┌────────┐         ┌──────────┐
│ Libro  │         │  Socio   │
└────────┘         └──────────┘
      │                  │
      │ 1            1   │
      ▼ N              ▼ N
┌──────────────┐   ┌───────────┐
│  Ejemplar    │   │ Préstamo  │
│  (débil)     │   │  (débil)  │
└──────────────┘   └───────────┘
```

---

## 3. Atributos

Un **atributo** es una propiedad o característica de una entidad. En un diagrama ER se representa con un **óvalo** conectado a la entidad.

### Tipos de atributos

| Tipo | Ejemplo | Notas |
|------|---------|-------|
| Simple | `nombre` | Atómico, no se divide |
| Compuesto | `dirección` (calle, ciudad, CP) | Se puede descomponer |
| Multivalorado | `teléfonos` | Varios valores (se representa con doble óvalo) |
| Derivado | `edad` (a partir de `fecha_nac`) | Se calcula, no se guarda |
| Clave | `isbn` | Identifica unívocamente (subrayado) |
| Opcional | `segundo_nombre` | Puede ser NULL |

### Ejemplo

```
           isbn (clave)
              │
   ┌──────────┴──────────┐
   │       Libro         │
   │                     │
   │  título              │
   │  año_publicacion     │
   │  idioma              │
   │  páginas (opcional)  │
   │  edad_recomendada (derivado)  │
   └─────────────────────┘
```

---

## 4. Relaciones

Una **relación** es una asociación entre dos o más entidades. Se representa con un **rombo**.

### Ejemplo

```
   ┌────────┐                ┌──────────┐
   │ Socio  │──── realiza ────│ Préstamo │
   └────────┘                └──────────┘
```

### Grado

- **Binaria:** entre 2 entidades (lo más común).
- **Ternaria:** entre 3 entidades.
- **Reflexiva (unaria):** una entidad se relaciona consigo misma. Ej: `Empleado` → `jefe_de` → `Empleado`.

---

## 5. Cardinalidad

La **cardinalidad** indica cuántas instancias de una entidad se relacionan con cuántas de la otra.

### Tipos principales

| Cardinalidad | Significado | Ejemplo |
|--------------|-------------|---------|
| 1:1 | Uno a uno | Persona ↔ Pasaporte |
| 1:N | Uno a muchos | Autor ↔ Libros |
| N:M | Muchos a muchos | Estudiantes ↔ Cursos |

### Notación (pata de cuervo / crow's foot)

```
   ┌─────┐                  ┌─────┐
   │  1  │──────||<─────────│  N  │     (1 a N)
   └─────┘                  └─────┘

   ┌─────┐                  ┌─────┐
   │  1  │──────||──────||──│  1  │     (1 a 1)
   └─────┘                  └─────┘

   ┌─────┐                  ┌─────┐
   │  N  │──────>┼──────<───│  M  │     (N a M)
   └─────┘                  └─────┘
```

### Ejemplo — Biblioteca

- Un `Libro` tiene **muchos** `Ejemplares` (1:N).
- Un `Socio` realiza **muchos** `Préstamos` (1:N).
- Un `Ejemplar` puede estar en **muchos** `Préstamos` a lo largo del tiempo (1:N).

---

## Ejercicios

### Ejercicio 1 — Análisis de requerimientos
Lee el siguiente caso y elabora una lista de:
- Posibles entidades
- Posibles atributos
- Posibles relaciones
- Reglas de negocio

> "La clínica veterinaria 'Patitas Felices' atiende perros y gatos. De cada mascota se registra nombre, especie, raza, fecha de nacimiento y peso. Cada mascota tiene un dueño (cliente) con nombre, teléfono y email. Cuando un cliente llega a consulta, se genera una cita con fecha, hora, motivo y el veterinario asignado. El veterinario emite un diagnóstico y puede recetar medicamentos."

### Ejercicio 2 — Entidades y atributos
Para el caso del ejercicio 1:
- Dibuja las entidades como rectángulos.
- Agrega al menos 4 atributos por entidad, marca el atributo clave con subrayado.
- Identifica qué entidades son débiles y por qué.

### Ejercicio 3 — Relaciones y cardinalidad
Para el mismo caso:
- Dibuja las relaciones entre entidades.
- Asigna cardinalidad a cada relación.
- Justifica cada cardinalidad con una frase.

### Ejercicio 4 — Atributos especiales
Para un sistema de **tienda virtual**, identifica:
- 2 atributos simples
- 1 atributo compuesto (y descompón sus partes)
- 1 atributo multivalorado
- 1 atributo derivado
- El atributo clave de cada entidad

### Ejercicio 5 — Reflexión
¿Por qué crees que es importante modelar antes de crear las tablas en la base de datos? Da 3 razones concretas.

---

## Recursos

- Elmasri & Navathe — Capítulo 3 (Modelado ER).
- https://www.lucidchart.com/pages/es/que-es-un-diagrama-entidad-relacion
- https://dbdiagram.io/ (herramienta online para diagramar).

---

**Próxima sesión:** Módulo 2 (parte 2) — Diagramas ER completos y casos de estudio.
