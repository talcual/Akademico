# Sesión 4 — Módulo 3: Modelo Relacional

**Semana:** 4
**Corte:** 1
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema Universitario → Tienda Virtual

---

## 1. De ER a Relacional: reglas de transformación

El modelo relacional representa los datos como **tablas** (relaciones). Para transformar un diagrama ER:

| Elemento ER | Equivalente Relacional |
|-------------|------------------------|
| Entidad fuerte | Tabla con sus atributos |
| Entidad débil | Tabla con PK compuesta (propia + FK de la fuerte) |
| Relación 1:1 | FK en cualquiera de las dos tablas |
| Relación 1:N | FK en la tabla del lado "muchos" |
| Relación N:M | Nueva tabla (tabla puente) con dos FKs |
| Atributo multivalorado | Nueva tabla con FK de la entidad |
| Atributo compuesto | Columnas separadas por cada parte |

---

## 2. Tablas, filas y columnas

- **Tabla (relación):** estructura bidimensional con nombre.
- **Fila (tupla):** cada ocurrencia individual.
- **Columna (atributo):** cada propiedad de la entidad.
- **Celda:** intersección fila-columna, contiene un solo valor atómico.
- **Esquema:** nombre de la tabla + sus columnas + tipos.

### Propiedades de una tabla relacional

1. Cada fila es única (por la PK).
2. El orden de filas y columnas no importa.
3. Cada celda contiene un solo valor (atomicidad).
4. Los nombres de columnas son únicos dentro de la tabla.

---

## 3. Tipos de datos en PostgreSQL

| Categoría | Tipos | Ejemplo |
|-----------|-------|---------|
| Enteros | `SMALLINT`, `INTEGER`, `BIGINT` | `42`, `-7` |
| Numéricos exactos | `NUMERIC(p,s)`, `DECIMAL` | `9999.99` |
| Numéricos aproximados | `REAL`, `DOUBLE PRECISION` | `3.14` |
| Texto | `VARCHAR(n)`, `CHAR(n)`, `TEXT` | `'Hola'` |
| Fecha/hora | `DATE`, `TIME`, `TIMESTAMP` | `2026-07-31` |
| Booleano | `BOOLEAN` | `TRUE`, `FALSE` |
| Serial | `SERIAL`, `BIGSERIAL` | `1, 2, 3, ...` |
| UUID | `UUID` | `'a0eebc99-9c0b-...'` |
| JSON | `JSON`, `JSONB` | `'{"k":"v"}'` |
| Binario | `BYTEA` | archivos |

---

## 4. Claves primarias (PK)

La **clave primaria** identifica unívocamente cada fila de una tabla. Reglas:
- Única.
- No nula (`NOT NULL`).
- Estable (no debería cambiar).

### Tipos de PK

- **Natural:** basada en datos reales (`isbn`, `cédula`).
- **Sintética / surrogate:** generada por el sistema (`SERIAL`, `UUID`).

```sql
CREATE TABLE autor (
    id        SERIAL PRIMARY KEY,
    nombre    VARCHAR(100) NOT NULL,
    email     VARCHAR(120) UNIQUE
);
```

### Llaves compuestas

```sql
CREATE TABLE matricula (
    estudiante_id  INTEGER,
    asignatura_id  INTEGER,
    nota_final     DECIMAL(3,2),
    PRIMARY KEY (estudiante_id, asignatura_id)
);
```

---

## 5. Claves foráneas (FK)

Una **clave foránea** es un campo (o conjunto) en una tabla que referencia la PK de otra tabla. Garantiza la **integridad referencial**.

```sql
CREATE TABLE libro (
    isbn        VARCHAR(13) PRIMARY KEY,
    titulo      VARCHAR(200) NOT NULL,
    autor_id    INTEGER REFERENCES autor(id) ON DELETE SET NULL
);
```

Acciones referenciales (vistas en detalle en módulos posteriores):
- `NO ACTION` (por defecto)
- `RESTRICT`
- `CASCADE`
- `SET NULL`
- `SET DEFAULT`

---

## 6. Restricciones (Constraints)

| Restricción | Significado |
|-------------|-------------|
| `NOT NULL` | No permite valores nulos |
| `UNIQUE` | No permite valores duplicados |
| `PRIMARY KEY` | PK = UNIQUE + NOT NULL |
| `FOREIGN KEY` | Debe existir en la tabla referenciada |
| `CHECK` | Una condición debe cumplirse |
| `DEFAULT` | Valor por defecto si no se inserta |

```sql
CREATE TABLE producto (
    sku         VARCHAR(20) PRIMARY KEY,
    nombre      VARCHAR(150) NOT NULL,
    precio      NUMERIC(10,2) CHECK (precio > 0),
    stock       INTEGER DEFAULT 0 CHECK (stock >= 0),
    categoria   VARCHAR(50) NOT NULL
);
```

---

## 7. Índices (introducción)

Un **índice** es una estructura auxiliar que acelera las búsquedas, a costa de espacio extra y costo en inserciones/actualizaciones.

```sql
CREATE INDEX idx_producto_categoria ON producto(categoria);
```

Por defecto, PostgreSQL crea índices para las PK y las UNIQUE. Veremos índices en profundidad en la sesión 14.

---

## 8. Ejemplo completo: Tienda Virtual (transformación)

```sql
CREATE TABLE categoria (
    id    SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE producto (
    sku           VARCHAR(20) PRIMARY KEY,
    nombre        VARCHAR(150) NOT NULL,
    descripcion   TEXT,
    precio        NUMERIC(10,2) NOT NULL CHECK (precio > 0),
    stock         INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    categoria_id  INTEGER NOT NULL REFERENCES categoria(id) ON DELETE RESTRICT
);

CREATE TABLE cliente (
    id        SERIAL PRIMARY KEY,
    email     VARCHAR(120) NOT NULL UNIQUE,
    nombre    VARCHAR(120) NOT NULL,
    telefono  VARCHAR(20)
);

CREATE TABLE pedido (
    id          SERIAL PRIMARY KEY,
    fecha       TIMESTAMP NOT NULL DEFAULT NOW(),
    total       NUMERIC(12,2) NOT NULL,
    cliente_id  INTEGER NOT NULL REFERENCES cliente(id) ON DELETE CASCADE
);

CREATE TABLE linea_pedido (
    pedido_id    INTEGER REFERENCES pedido(id) ON DELETE CASCADE,
    producto_sku VARCHAR(20) REFERENCES producto(sku) ON DELETE RESTRICT,
    cantidad     INTEGER NOT NULL CHECK (cantidad > 0),
    subtotal     NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (pedido_id, producto_sku)
);
```

---

## Ejercicios

### Ejercicio 1 — Biblioteca
Transforma el diagrama ER de la biblioteca (sesión 3) a tablas SQL. Incluye PK, FK, restricciones y tipos de datos apropiados.

### Ejercicio 2 — Sistema universitario
Haz lo mismo con el sistema universitario. La tabla `matricula` debe tener PK compuesta y la FK a `asignatura` con `ON DELETE CASCADE`.

### Ejercicio 3 — Nuevas restricciones
Agrega las siguientes restricciones a la tabla `producto`:
- `nombre` no puede ser vacío (`CHECK (LENGTH(nombre) > 0)`).
- `stock` máximo 10000.
- `precio` entre 0.01 y 999999.99.

### Ejercicio 4 — Tabla puente
En la tienda virtual, modela una relación N:M entre `cliente` y `producto` llamada `favorito` (los productos favoritos de cada cliente). Define la tabla con PK compuesta.

### Ejercicio 5 — Atributo multivalorado
Un cliente puede tener varios teléfonos. Modela esto con una tabla aparte `cliente_telefono` con FK a `cliente`.

### Ejercicio 6 — Investigación
Consulta la documentación oficial de PostgreSQL. ¿Qué diferencia hay entre `VARCHAR(n)`, `CHAR(n)` y `TEXT`?

---

## Recursos

- https://www.postgresql.org/docs/current/ddl.html
- https://www.postgresql.org/docs/current/datatype.html
- https://www.postgresql.org/docs/current/ddl-constraints.html

---

**Próxima sesión:** Módulo 4 — Álgebra Relacional.
