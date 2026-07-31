# Sesión 14 — Módulo 12: Índices y EXPLAIN

**Semana:** 14
**Corte:** 3
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema de Facturación

---

## 1. ¿Qué es un índice?

Un índice es una **estructura de datos auxiliar** que permite a la base de datos encontrar filas rápidamente, sin tener que recorrer toda la tabla (lo que se llama *full table scan*).

**Analogía:** el índice de un libro. En lugar de leer todo el libro para encontrar un tema, consultas el índice y vas directo a la página.

### Desventajas

- Ocupan espacio en disco.
- Las inserciones/actualizaciones/borrados son más lentas (hay que actualizar el índice).
- No siempre se usan (el optimizador decide).

---

## 2. Tipos de índices en PostgreSQL

### B-Tree (por defecto)

- Estructura de árbol balanceado.
- Ideal para igualdad (`=`) y rangos (`<`, `>`, `BETWEEN`, `LIKE 'abc%'`).
- Es el índice por defecto y el más usado.

### Hash

- Basado en funciones hash.
- Solo soporta igualdad (`=`).
- Más compacto que B-Tree, pero menos versátil.
- En la práctica, B-Tree cubre casi todos los casos.

### GIN (Generalized Inverted Index)

- Ideal para datos multivaluados: JSONB, arrays, texto completo (`to_tsvector`).

### GiST (Generalized Search Tree)

- Para datos geométricos y búsqueda por similitud.

### BRIN (Block Range Index)

- Para tablas enormes con datos ordenados naturalmente (logs, series de tiempo).

---

## 3. Crear índices

```sql
-- B-Tree simple
CREATE INDEX idx_factura_fecha ON factura(fecha);

-- Índice compuesto (orden importa)
CREATE INDEX idx_factura_cliente_fecha ON factura(cliente_id, fecha);

-- Índice único
CREATE UNIQUE INDEX idx_cliente_email ON cliente(email);

-- Índice parcial (solo algunas filas)
CREATE INDEX idx_facturas_pendientes ON factura(fecha)
WHERE estado = 'pendiente';

-- Índice en expresión
CREATE INDEX idx_cliente_email_lower ON cliente(LOWER(email));
```

### Eliminar índice

```sql
DROP INDEX idx_factura_fecha;
```

### Listar índices

```sql
\di
-- O:
SELECT * FROM pg_indexes WHERE schemaname = 'public';
```

---

## 4. ¿Cuándo SÍ crear un índice?

- Columnas usadas frecuentemente en `WHERE`.
- Columnas usadas en `JOIN` (las FK deberían tener índice casi siempre).
- Columnas usadas en `ORDER BY` o `GROUP BY` para ordenamientos grandes.
- Columnas con alta cardinalidad (muchos valores distintos).

### Ejemplo típico

```sql
-- Búsqueda por fecha y por cliente
CREATE INDEX idx_factura_cliente_id ON factura(cliente_id);
CREATE INDEX idx_factura_fecha_emision ON factura(fecha_emision);

-- Búsqueda por estado
CREATE INDEX idx_factura_estado ON factura(estado);
```

---

## 5. ¿Cuándo NO crear un índice?

- Tablas pequeñas (full scan es más rápido).
- Columnas con muy poca cardinalidad (ej: booleanos).
- Tablas con muchas escrituras y pocas lecturas.
- Columnas casi nunca usadas en filtros.

### Anti-ejemplo

```sql
-- En una tabla de 100 filas, el índice no aporta
-- En una columna con solo 2 valores, mejor filtrar por otro criterio
```

---

## 6. EXPLAIN — leyendo el plan de ejecución

`EXPLAIN` muestra cómo PostgreSQL ejecutará una consulta.

```sql
EXPLAIN SELECT * FROM factura WHERE cliente_id = 5;
```

Resultado típico:
```
Seq Scan on factura  (cost=0.00..15.00 rows=5 width=100)
  Filter: (cliente_id = 5)
```

- `Seq Scan`: full table scan.
- `Index Scan`: usa índice.
- `Index Only Scan`: cubre la consulta solo con el índice.
- `Bitmap Index Scan`: usa índice para varias filas.

`EXPLAIN ANALYZE` ejecuta la consulta y muestra tiempos reales:

```sql
EXPLAIN ANALYZE
SELECT * FROM factura WHERE cliente_id = 5;
```

```
Seq Scan on factura  (cost=0.00..15.00 rows=5 width=100)
                    (actual time=0.020..0.025 rows=5.00 loops=1)
  Filter: (cliente_id = 5)
Planning Time: 0.1 ms
Execution Time: 0.05 ms
```

### Cómo leer el plan

- `cost=` estimación interna (no es tiempo real).
- `actual time=` tiempo real en ms.
- `rows=` estimación vs `actual rows=` (si difieren mucho, falta `ANALYZE`).
- Operaciones: de abajo hacia arriba se ejecutan.

---

## 7. Ejemplo práctico paso a paso

### Crear el esquema

```sql
CREATE TABLE cliente (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(100) NOT NULL
);

CREATE TABLE factura (
    id           SERIAL PRIMARY KEY,
    cliente_id   INTEGER REFERENCES cliente(id),
    fecha        DATE NOT NULL,
    monto        NUMERIC(10,2) NOT NULL,
    estado       VARCHAR(20) NOT NULL
);

-- Insertar 100.000 facturas
INSERT INTO cliente (nombre)
SELECT 'Cliente ' || g FROM generate_series(1, 1000) g;

INSERT INTO factura (cliente_id, fecha, monto, estado)
SELECT
    (random() * 999 + 1)::INT,
    CURRENT_DATE - (random() * 365)::INT,
    (random() * 1000000)::NUMERIC(10,2),
    CASE WHEN random() < 0.7 THEN 'pagada'
         WHEN random() < 0.5 THEN 'pendiente'
         ELSE 'vencida' END
FROM generate_series(1, 100000);
```

### Sin índice

```sql
EXPLAIN ANALYZE
SELECT * FROM factura WHERE cliente_id = 42;
```

Resultado esperado: **Seq Scan**, lento.

### Con índice

```sql
CREATE INDEX idx_factura_cliente ON factura(cliente_id);

EXPLAIN ANALYZE
SELECT * FROM factura WHERE cliente_id = 42;
```

Resultado esperado: **Index Scan**, mucho más rápido.

### Actualizar estadísticas

```sql
ANALYZE factura;
```

---

## 8. Índices compuestos — orden importa

```sql
-- Índice (a, b, c)
-- Útil para: WHERE a=?; WHERE a=? AND b=?; WHERE a=? AND b=? AND c=?
-- NO útil para: WHERE b=?; WHERE c=?; WHERE b=? AND c=?
```

Regla: **columna más selectiva primero** (la que más filtra).

---

## Ejercicios

### Práctica con índices
1. Crea el esquema de cliente/factura con 100k filas (script arriba).
2. Mide con `EXPLAIN ANALYZE` el tiempo de búsqueda por `cliente_id` sin índice.
3. Crea el índice y mide de nuevo. Compara tiempos.
4. Repite con búsqueda por rango de fechas (`fecha BETWEEN '...' AND '...'`).
5. Repite con búsqueda por estado.

### Diseño
6. Para el esquema del Corte 2 (EduOnline), identifica al menos 5 columnas candidatas a índice y justifícalo.
7. ¿Por qué casi siempre es buena idea indexar las FK?
8. Crea un índice parcial sobre facturas en estado 'vencida' y comprueba su uso.

### Análisis
9. Ejecuta `EXPLAIN` sobre una consulta que use `JOIN` y explica cada línea del plan.
10. ¿Qué hace `ANALYZE` y por qué es importante ejecutarlo periódicamente?
11. Investiga: ¿qué diferencia hay entre `Index Scan`, `Index Only Scan` y `Bitmap Index Scan`?

### Decisiones
12. Tu tabla recibe 10.000 inserciones por minuto pero solo 100 consultas. ¿Cuántos índices crearías? Justifica.
13. ¿Crearías un índice en una columna de tipo `BOOLEAN`? Justifica.

---

## Recursos

- https://www.postgresql.org/docs/current/indexes.html
- https://use-the-index-luke.com/
- https://www.postgresql.org/docs/current/using-explain.html
- https://explain.dalibo.com/ (visualizador de planes EXPLAIN)

---

**Próxima sesión:** Módulo 13 (Vistas) + Módulo 14 (Procedimientos Almacenados).
