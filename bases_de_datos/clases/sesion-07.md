# Sesión 7 — Módulo 5: SQL Básico (DDL)

**Semana:** 7
**Corte:** 2
**Duración estimada:** 2 horas
**Caso de estudio:** Clínica Veterinaria "Patitas Felices"

---

## 1. SQL: el lenguaje de las bases de datos

**SQL** (*Structured Query Language*) es el lenguaje estándar para gestionar bases de datos relacionales. Se divide en sublenguajes:

| Sublenguaje | Significado | Comandos |
|-------------|-------------|----------|
| DDL | Data Definition Language | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| DML | Data Manipulation Language | `INSERT`, `UPDATE`, `DELETE` |
| DQL | Data Query Language | `SELECT` |
| DCL | Data Control Language | `GRANT`, `REVOKE` |
| TCL | Transaction Control Language | `BEGIN`, `COMMIT`, `ROLLBACK` |

En esta sesión cubrimos el **DDL**.

---

## 2. La base de datos: Veterinaria

### Requerimientos resumidos

- Mascotas: perros, gatos y otros. Cada mascota tiene un dueño.
- Dueños (clientes) con datos de contacto.
- Veterinarios con especialidad.
- Consultas (citas) con fecha, mascota, veterinario y diagnóstico.
- Recetas (medicamentos asociados a una consulta).
- Catálogo de medicamentos.

---

## 3. CREATE DATABASE

Crea una base de datos nueva.

```sql
CREATE DATABASE veterinaria
    WITH ENCODING 'UTF8'
    LC_COLLATE 'es_CO.UTF-8';
```

```sql
-- Conectarse a ella
\c veterinaria
-- O en script
\connect veterinaria
```

Para listar las bases de datos:

```sql
\l
```

---

## 4. CREATE TABLE

```sql
CREATE TABLE dueno (
    id              SERIAL PRIMARY KEY,
    cedula          VARCHAR(20) NOT NULL UNIQUE,
    nombre          VARCHAR(120) NOT NULL,
    telefono        VARCHAR(20),
    email           VARCHAR(120) UNIQUE,
    fecha_registro  TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### Restricciones inline vs. al final

```sql
-- Inline
CREATE TABLE mascota (
    id          SERIAL PRIMARY KEY,
    nombre      VARCHAR(80) NOT NULL,
    especie     VARCHAR(20) NOT NULL CHECK (especie IN ('perro','gato','otro')),
    raza        VARCHAR(60),
    fecha_nac   DATE,
    peso_kg     NUMERIC(5,2) CHECK (peso_kg > 0),
    dueno_id    INTEGER NOT NULL REFERENCES dueno(id) ON DELETE CASCADE
);

-- Al final (más legible)
CREATE TABLE veterinario (
    id          SERIAL PRIMARY KEY,
    nombre      VARCHAR(120) NOT NULL,
    especialidad VARCHAR(80),
    email       VARCHAR(120) UNIQUE,
    CONSTRAINT chk_email CHECK (email LIKE '%@%')
);
```

---

## 5. Tipos de datos relevantes (PostgreSQL)

```sql
CREATE TABLE consulta (
    id              SERIAL PRIMARY KEY,
    fecha           TIMESTAMP NOT NULL DEFAULT NOW(),
    motivo          VARCHAR(200) NOT NULL,
    diagnostico     TEXT,
    costo           NUMERIC(10,2) CHECK (costo >= 0),
    mascota_id      INTEGER NOT NULL REFERENCES mascota(id) ON DELETE CASCADE,
    veterinario_id  INTEGER NOT NULL REFERENCES veterinario(id) ON DELETE RESTRICT
);
```

### SERIAL vs INT con secuencia

```sql
-- Forma moderna (PostgreSQL 10+)
CREATE TABLE t (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre VARCHAR(50)
);
```

---

## 6. ALTER TABLE

Modifica una tabla existente.

```sql
-- Agregar columna
ALTER TABLE dueno ADD COLUMN direccion VARCHAR(200);

-- Eliminar columna
ALTER TABLE dueno DROP COLUMN direccion;

-- Cambiar tipo
ALTER TABLE mascota ALTER COLUMN raza TYPE TEXT;

-- Renombrar columna
ALTER TABLE dueno RENAME COLUMN nombre TO nombre_completo;

-- Agregar restricción
ALTER TABLE mascota ADD CONSTRAINT chk_peso_max CHECK (peso_kg < 200);

-- Eliminar restricción
ALTER TABLE mascota DROP CONSTRAINT chk_peso_max;

-- Renombrar tabla
ALTER TABLE dueno RENAME TO cliente;
```

---

## 7. DROP

Elimina objetos. **¡Cuidado, no se puede deshacer!**

```sql
DROP TABLE consulta;          -- Elimina la tabla
DROP TABLE cliente CASCADE;   -- Elimina en cascada
DROP DATABASE veterinaria;    -- Elimina toda la BD
```

> `CASCADE` elimina objetos que dependen del actual (FKs, vistas, etc.). Usar con precaución.

---

## 8. TRUNCATE

Vacía una tabla (elimina todas las filas) **sin** disparar triggers y resetea la secuencia (con `RESTART IDENTITY`).

```sql
TRUNCATE TABLE consulta RESTART IDENTITY CASCADE;
```

> A diferencia de `DELETE`, `TRUNCATE` es mucho más rápido en tablas grandes.

---

## 9. Script completo de la Veterinaria

```sql
-- ========================================
-- Base de datos: Veterinaria Patitas Felices
-- ========================================

DROP TABLE IF EXISTS receta CASCADE;
DROP TABLE IF EXISTS consulta CASCADE;
DROP TABLE IF EXISTS medicamento CASCADE;
DROP TABLE IF EXISTS mascota CASCADE;
DROP TABLE IF EXISTS veterinario CASCADE;
DROP TABLE IF EXISTS dueno CASCADE;

CREATE TABLE dueno (
    id              SERIAL PRIMARY KEY,
    cedula          VARCHAR(20) NOT NULL UNIQUE,
    nombre          VARCHAR(120) NOT NULL,
    telefono        VARCHAR(20),
    email           VARCHAR(120) UNIQUE,
    fecha_registro  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE mascota (
    id          SERIAL PRIMARY KEY,
    nombre      VARCHAR(80) NOT NULL,
    especie     VARCHAR(20) NOT NULL CHECK (especie IN ('perro','gato','otro')),
    raza        VARCHAR(60),
    fecha_nac   DATE,
    peso_kg     NUMERIC(5,2) CHECK (peso_kg > 0),
    dueno_id    INTEGER NOT NULL REFERENCES dueno(id) ON DELETE CASCADE
);

CREATE TABLE veterinario (
    id            SERIAL PRIMARY KEY,
    nombre        VARCHAR(120) NOT NULL,
    especialidad  VARCHAR(80),
    email         VARCHAR(120) UNIQUE
);

CREATE TABLE consulta (
    id              SERIAL PRIMARY KEY,
    fecha           TIMESTAMP NOT NULL DEFAULT NOW(),
    motivo          VARCHAR(200) NOT NULL,
    diagnostico     TEXT,
    costo           NUMERIC(10,2) CHECK (costo >= 0) DEFAULT 0,
    mascota_id      INTEGER NOT NULL REFERENCES mascota(id) ON DELETE CASCADE,
    veterinario_id  INTEGER NOT NULL REFERENCES veterinario(id) ON DELETE RESTRICT
);

CREATE TABLE medicamento (
    id          SERIAL PRIMARY KEY,
    nombre      VARCHAR(120) NOT NULL,
    precio      NUMERIC(10,2) NOT NULL CHECK (precio > 0),
    stock       INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0)
);

CREATE TABLE receta (
    id              SERIAL PRIMARY KEY,
    consulta_id     INTEGER NOT NULL REFERENCES consulta(id) ON DELETE CASCADE,
    medicamento_id  INTEGER NOT NULL REFERENCES medicamento(id) ON DELETE RESTRICT,
    dosis           VARCHAR(80) NOT NULL,
    cantidad        INTEGER NOT NULL CHECK (cantidad > 0)
);
```

---

## Ejercicios

### Ejercicio 1
Conéctate a PostgreSQL y crea una base de datos `veterinaria`. Ejecuta el script completo.

### Ejercicio 2
Crea las siguientes tablas adicionales para el sistema de veterinaria:
- `vacuna` (id, nombre, fabricante, lote, fecha_vencimiento).
- `vacunacion` (id, mascota_id, vacuna_id, fecha, veterinario_id).

### Ejercicio 3
Con `ALTER TABLE`:
- Agrega a `mascota` la columna `color` (VARCHAR 50).
- Cambia el tipo de `raza` a TEXT.
- Renombra `dueno` por `cliente` (reflejando una nueva política).
- Elimina la columna `email` de `veterinario`.

### Ejercicio 4
Crea una tabla `historia_clinica` con PK autoincrement, columna `descripcion` TEXT no nula, FK a `mascota` con borrado en cascada, y una restricción CHECK de que `descripcion` tenga al menos 10 caracteres.

### Ejercicio 5
Investiga: ¿qué diferencia hay entre `DROP TABLE X` y `DROP TABLE X CASCADE`? ¿Cuándo usarías cada uno?

### Ejercicio 6
Vacía la tabla `consulta` con `TRUNCATE RESTART IDENTITY`. ¿Qué pasa con el próximo `INSERT`? Explica.

---

## Recursos

- https://www.postgresql.org/docs/current/sql-createtable.html
- https://www.postgresql.org/docs/current/sql-altertable.html
- https://www.postgresqltutorial.com/postgresql-create-table/

---

**Próxima sesión:** Módulo 6 — CRUD.
