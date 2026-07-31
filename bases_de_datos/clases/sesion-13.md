# Sesión 13 — Módulo 11: Normalización

**Semana:** 13
**Corte:** 3
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema de Pedidos desnormalizado

---

## 1. ¿Por qué normalizar?

La normalización es el proceso de organizar las columnas y tablas de una base de datos para:

- **Evitar redundancia** (datos repetidos).
- **Evitar anomalías** de inserción, actualización y eliminación.
- **Garantizar la integridad** de los datos.

---

## 2. Dependencias funcionales

Una **dependencia funcional** X → Y significa que si conozco X, puedo determinar Y. Por ejemplo:

- `cedula → nombre` (la cédula determina el nombre).
- `id_producto → precio` (el id determina el precio).
- `codigo_postal → ciudad` (el CP determina la ciudad).

### Dependencias triviales vs no triviales

- Trivial: `X → X` siempre se cumple.
- No trivial: `X → Y` donde Y no es subconjunto de X (es la interesante).

---

## 3. Primera Forma Normal (1FN)

**Regla:** todos los atributos deben ser **atómicos** (un solo valor por celda). No hay atributos repetidos ni multivaluados.

### ❌ Tabla NO en 1FN

| id | nombre | telefonos |
|----|--------|-----------|
| 1 | Ana | 300111, 300222, 300333 |
| 2 | Beto | 311444 |

### ✅ En 1FN (dos opciones)

**Opción A:** varias filas

| id | nombre | telefono |
|----|--------|----------|
| 1 | Ana | 300111 |
| 1 | Ana | 300222 |
| 1 | Ana | 300333 |
| 2 | Beto | 311444 |

**Opción B:** tabla aparte

```
CLIENTE(id, nombre)
TELEFONO_CLIENTE(cliente_id, telefono)
```

---

## 4. Segunda Forma Normal (2FN)

**Regla:** estar en 1FN **+** todo atributo no clave debe depender de **toda** la PK (no de una parte de ella). Aplica a tablas con PK compuesta.

### ❌ Tabla NO en 2FN

```
INSCRIPCION(estudiante_id, curso_id, nombre_estudiante, nombre_curso, fecha)
```

- PK: (estudiante_id, curso_id)
- `nombre_estudiante` depende solo de `estudiante_id` ❌
- `nombre_curso` depende solo de `curso_id` ❌

### ✅ En 2FN

```
ESTUDIANTE(id, nombre)
CURSO(id, nombre)
INSCRIPCION(estudiante_id, curso_id, fecha)
```

---

## 5. Tercera Forma Normal (3FN)

**Regla:** estar en 2FN **+** ningún atributo no clave depende de **otro atributo no clave** (es decir, no hay dependencias transitivas).

### ❌ Tabla NO en 3FN

```
EMPLEADO(id, nombre, departamento_id, nombre_departamento)
```

- `departamento_id → nombre_departamento` (dependencia transitiva a través de `departamento_id`)

### ✅ En 3FN

```
EMPLEADO(id, nombre, departamento_id)
DEPARTAMENTO(id, nombre)
```

---

## 6. Forma Normal de Boyce-Codd (BCNF)

**Regla:** estar en 3FN **+** para toda dependencia funcional X → Y, **X debe ser superclave** (candidata a PK).

Es una versión más estricta de la 3FN. La mayoría de casos prácticos quedan en BCNF con 3FN, pero hay excepciones.

### Ejemplo donde 3FN NO basta

```
HORARIO(aula, hora, curso)
```

Dependencias:
- (aula, hora) → curso
- curso → aula (un curso siempre usa el mismo aula)

Aquí `curso → aula` viola BCNF porque `curso` no es superclave.

---

## 7. Ejemplo guiado: Sistema de Pedidos desnormalizado

```sql
CREATE TABLE pedido_malo (
    pedido_id        INTEGER,
    fecha            DATE,
    cliente_cedula   VARCHAR(20),
    cliente_nombre   VARCHAR(120),
    cliente_email    VARCHAR(120),
    producto_sku     VARCHAR(20),
    producto_nombre  VARCHAR(200),
    producto_precio  NUMERIC(10,2),
    cantidad         INTEGER,
    categoria_nombre VARCHAR(80)
);
```

### Análisis

1. **1FN:** ✅ atómico.
2. **2FN:** ❌ si la PK es (pedido_id, producto_sku), entonces `cliente_nombre` depende solo de `cliente_cedula` (que a su vez depende solo de `pedido_id`).
3. **3FN:** ❌ `producto_nombre` y `producto_precio` dependen de `producto_sku`; `categoria_nombre` depende transitivamente.

### Normalización paso a paso

```sql
-- Tabla cliente
CREATE TABLE cliente (
    cedula  VARCHAR(20) PRIMARY KEY,
    nombre  VARCHAR(120) NOT NULL,
    email   VARCHAR(120) UNIQUE
);

-- Tabla categoría
CREATE TABLE categoria (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(80) NOT NULL UNIQUE
);

-- Tabla producto
CREATE TABLE producto (
    sku            VARCHAR(20) PRIMARY KEY,
    nombre         VARCHAR(200) NOT NULL,
    precio         NUMERIC(10,2) NOT NULL,
    categoria_id   INTEGER REFERENCES categoria(id)
);

-- Tabla pedido
CREATE TABLE pedido (
    id          SERIAL PRIMARY KEY,
    fecha       DATE NOT NULL,
    cliente_cedula VARCHAR(20) NOT NULL REFERENCES cliente(cedula)
);

-- Tabla línea de pedido
CREATE TABLE linea_pedido (
    pedido_id   INTEGER REFERENCES pedido(id) ON DELETE CASCADE,
    producto_sku VARCHAR(20) REFERENCES producto(sku),
    cantidad    INTEGER NOT NULL CHECK (cantidad > 0),
    PRIMARY KEY (pedido_id, producto_sku)
);
```

---

## 8. Desnormalización

A veces, **violar** las formas normales es intencional para mejorar rendimiento. Ejemplos:

- **Cachear totales:** una columna `total` en `pedido` evita recalcularlo en cada consulta.
- **Cachear contadores:** una columna `num_seguidores` en `usuario` evita un `COUNT` costoso.
- **Datos redundantes para reportes:** tablas pre-agregadas para data warehouse.

### Reglas de oro

- Normaliza primero, desnormaliza después.
- Desnormaliza solo cuando el rendimiento lo justifique y mides el impacto.
- Mantén la integridad con triggers o procesos batch.

---

## Ejercicios

### Identificar formas normales
1. Para la siguiente tabla, identifica en qué FN está y por qué:

```
VENTA(id, fecha, cliente_id, cliente_nombre, vendedor_id, vendedor_nombre, monto)
```

### Normalizar paso a paso
2. Normaliza la siguiente tabla hasta 3FN:

```
PROYECTO(id, nombre, empleado_id, empleado_nombre, empleado_cargo, cliente_id, cliente_nombre, horas_trabajadas)
```

3. Normaliza:

```
MATRICULA(estudiante_id, estudiante_nombre, programa_id, programa_nombre, facultad_id, facultad_nombre, asignatura_id, asignatura_nombre, nota)
```

4. Normaliza:

```
PEDIDO(id, fecha, cliente_id, cliente_nombre, productos_ids, productos_nombres, total)
```

Donde `productos_ids` y `productos_nombres` son listas separadas por comas.

### Decisiones de diseño
5. Para un sistema de e-commerce, ¿qué tablas mantendrías desnormalizadas y por qué?
6. ¿Cuándo NO aplicarías 3FN? Da 2 ejemplos reales.

### BCNF
7. Dada la tabla `ASIGNATURA_PROFESOR(profesor_id, asignatura_id, aula)`, donde cada profesor solo dicta en un aula específica:
   - Identifica las dependencias funcionales.
   - ¿Está en 3FN? ¿En BCNF? Justifica.

### Reflexión
8. ¿Por qué crees que la normalización excesiva también puede ser perjudicial? Da un ejemplo.

---

## Recursos

- Elmasri & Navathe — Capítulo 14.
- https://es.wikipedia.org/wiki/Normalizaci%C3%B3n_de_bases_de_datos
- https://www.guru99.com/database-normalization.html

---

**Próxima sesión:** Módulo 12 — Índices y EXPLAIN.
