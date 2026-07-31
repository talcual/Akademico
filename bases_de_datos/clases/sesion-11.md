# Sesión 11 — Módulo 9: JOINs

**Semana:** 11
**Corte:** 2
**Duración estimada:** 2 horas
**Caso de estudio:** Empresa de Logística "EnvíoYa"

---

## 1. Sistema de ejemplo: EnvíoYa

```sql
CREATE DATABASE envioya;
\c envioya

CREATE TABLE ciudad (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(100) NOT NULL
);

CREATE TABLE cliente (
    id          SERIAL PRIMARY KEY,
    cedula      VARCHAR(20) NOT NULL UNIQUE,
    nombre      VARCHAR(120) NOT NULL,
    email       VARCHAR(120),
    ciudad_id   INTEGER REFERENCES ciudad(id) ON DELETE SET NULL
);

CREATE TABLE repartidor (
    id          SERIAL PRIMARY KEY,
    cedula      VARCHAR(20) NOT NULL UNIQUE,
    nombre      VARCHAR(120) NOT NULL,
    vehiculo    VARCHAR(40) CHECK (vehiculo IN ('moto','bicicleta','carro')),
    ciudad_id   INTEGER NOT NULL REFERENCES ciudad(id) ON DELETE RESTRICT
);

CREATE TABLE paquete (
    id              SERIAL PRIMARY KEY,
    codigo          VARCHAR(20) NOT NULL UNIQUE,
    descripcion     VARCHAR(200),
    peso_kg         NUMERIC(6,2) NOT NULL CHECK (peso_kg > 0),
    cliente_id      INTEGER NOT NULL REFERENCES cliente(id) ON DELETE CASCADE,
    fecha_creacion  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE envio (
    id              SERIAL PRIMARY KEY,
    paquete_id      INTEGER NOT NULL REFERENCES paquete(id) ON DELETE CASCADE,
    repartidor_id   INTEGER NOT NULL REFERENCES repartidor(id) ON DELETE RESTRICT,
    ciudad_origen_id  INTEGER NOT NULL REFERENCES ciudad(id),
    ciudad_destino_id INTEGER NOT NULL REFERENCES ciudad(id),
    fecha_despacho  TIMESTAMP NOT NULL DEFAULT NOW(),
    fecha_entrega   TIMESTAMP,
    estado          VARCHAR(20) NOT NULL CHECK (estado IN ('pendiente','en_transito','entregado','devuelto'))
);

-- Jefe de repartidor (relación reflexiva)
ALTER TABLE repartidor ADD COLUMN jefe_id INTEGER REFERENCES repartidor(id) ON DELETE SET NULL;
```

```sql
-- Datos
INSERT INTO ciudad (nombre) VALUES
    ('Bogotá'),('Medellín'),('Cali'),('Barranquilla'),('Bucaramanga');

INSERT INTO cliente (cedula, nombre, email, ciudad_id) VALUES
    ('1001','Ana Pérez',     'ana@x.com',     1),
    ('1002','Beto Gómez',     'beto@x.com',    1),
    ('1003','Camila Ruiz',    'cami@x.com',    2),
    ('1004','Daniel Castro',  'dani@x.com',    3),
    ('1005','Elena Mora',     'elena@x.com',   1),
    ('1006','Federico Cano',  NULL,            5);

INSERT INTO repartidor (cedula, nombre, vehiculo, ciudad_id, jefe_id) VALUES
    ('2001','Hugo Vélez',    'moto',      1, NULL),
    ('2002','Laura Sánchez', 'bicicleta', 1, 1),
    ('2003','Mario Pinto',   'carro',     2, NULL),
    ('2004','Nora Restrepo', 'moto',      2, 3),
    ('2005','Óscar Jaramillo','moto',     3, NULL);

INSERT INTO paquete (codigo, descripcion, peso_kg, cliente_id) VALUES
    ('PKG-001','Libro',          0.5, 1),
    ('PKG-002','Celular',        0.4, 1),
    ('PKG-003','Ropa',           2.0, 2),
    ('PKG-004','Electrodoméstico',8.5, 3),
    ('PKG-005','Medicamento',    0.3, 4),
    ('PKG-006','Documentos',     0.1, 5);

INSERT INTO envio (paquete_id, repartidor_id, ciudad_origen_id, ciudad_destino_id, fecha_despacho, fecha_entrega, estado) VALUES
    (1, 1, 1, 2, '2026-01-10 09:00', '2026-01-11 14:00', 'entregado'),
    (2, 2, 1, 1, '2026-01-12 10:00', '2026-01-12 18:00', 'entregado'),
    (3, 3, 2, 3, '2026-01-15 08:00', '2026-01-16 11:00', 'entregado'),
    (4, 4, 2, 4, '2026-01-20 12:00', NULL,                'en_transito'),
    (5, 5, 3, 5, '2026-01-22 15:00', NULL,                'pendiente'),
    (6, 1, 1, 1, '2026-01-25 09:00', NULL,                'en_transito');
```

---

## 2. INNER JOIN

Devuelve solo las filas que **tienen coincidencias** en ambas tablas.

```sql
SELECT c.nombre AS cliente, p.codigo, e.estado
FROM envio e
INNER JOIN paquete p   ON p.id = e.paquete_id
INNER JOIN cliente c   ON c.id = p.cliente_id;
```

> Si un cliente no tiene envíos, **no aparece** en el resultado.

### Sintaxis alternativa (implícita, no recomendada)

```sql
SELECT c.nombre, p.codigo
FROM cliente c, paquete p, envio e
WHERE c.id = p.cliente_id AND p.id = e.paquete_id;
```

---

## 3. LEFT JOIN

Devuelve **todas** las filas de la tabla izquierda y las coincidencias de la derecha. Si no hay coincidencia, las columnas de la derecha son NULL.

```sql
-- Todos los clientes, tengan o no envíos
SELECT c.nombre, p.codigo
FROM cliente c
LEFT JOIN paquete p ON p.cliente_id = c.id
LEFT JOIN envio e   ON e.paquete_id = p.id;
```

### Encontrar "los que NO tienen"

```sql
-- Clientes sin envíos
SELECT c.nombre
FROM cliente c
LEFT JOIN paquete p ON p.cliente_id = c.id
WHERE p.id IS NULL;
```

---

## 4. RIGHT JOIN

Igual que LEFT JOIN pero al revés: prioriza la tabla derecha.

```sql
-- Todos los repartidores, tengan o no envíos
SELECT r.nombre, e.estado
FROM envio e
RIGHT JOIN repartidor r ON r.id = e.repartidor_id;
```

> En la práctica, el `RIGHT JOIN` se puede reescribir como `LEFT JOIN` invirtiendo el orden, por eso es menos usado.

---

## 5. FULL OUTER JOIN

Devuelve todas las filas de ambas tablas, con NULL donde no haya coincidencia.

```sql
-- Todos los clientes y todos los repartidores (mezclados por ciudad)
SELECT c.nombre AS cliente, r.nombre AS repartidor, ci.nombre AS ciudad
FROM cliente c
FULL OUTER JOIN repartidor r ON r.ciudad_id = c.ciudad_id
FULL OUTER JOIN ciudad ci    ON ci.id = c.ciudad_id;
```

---

## 6. SELF JOIN

Une una tabla **consigo misma**. Útil para relaciones reflexivas (jefe-empleado, categorías con padre, etc.).

```sql
-- Cada repartidor con su jefe
SELECT r.nombre AS repartidor, j.nombre AS jefe
FROM repartidor r
LEFT JOIN repartidor j ON r.jefe_id = j.id;
```

---

## 7. CROSS JOIN

Producto cartesiano: cada fila de A con cada fila de B.

```sql
-- Todas las combinaciones posibles de origen-destino (sin envíos)
SELECT co.nombre AS origen, cd.nombre AS destino
FROM ciudad co
CROSS JOIN ciudad cd
WHERE co.id <> cd.id;
```

> Útil para generar calendarios, matrices o combinaciones. ¡Cuidado con tablas grandes!

---

## 8. Comparación visual

```
        A    B            A LEFT B   A RIGHT B   A FULL B
        ─    ─            ─ ──────   ─ ──────    ─ ──────
        1 ─── a           1 a        1 a         1 a
        2     b           2 b        2 b         2 b
        3                3 NULL      3 NULL      3 NULL
        4                    4 c     4 c         4 c
        5                            5 NULL      5 NULL
```

---

## Ejercicios (40 progresivos)

### Básicos (INNER)
1. Lista todos los envíos con el nombre del cliente.
2. Lista los envíos con el nombre del repartidor.
3. Muestra el código del paquete, la descripción, el estado del envío y la fecha de despacho.
4. Lista el nombre del cliente y la ciudad de origen del envío.

### LEFT/RIGHT
5. Lista todos los clientes y, si tienen envíos, muestra el código del paquete.
6. Lista los clientes que NO han hecho ningún envío.
7. Lista los repartidores y la cantidad de envíos realizados (incluyendo los que no tienen).
8. Lista los repartidores que no tienen ningún envío asignado.

### Múltiples JOINs
9. Muestra el código del paquete, el cliente, el repartidor, la ciudad de origen y la de destino.
10. Para cada envío, muestra la fecha de despacho y la diferencia en días hasta la entrega.
11. Lista los envíos con tiempo de entrega mayor a 24 horas.

### Self join
12. Lista los repartidores con su respectivo jefe.
13. Lista los repartidores que no tienen jefe asignado.
14. Lista los jefes y cuántos repartidores tienen a cargo.

### Cross join
15. Genera una matriz con todas las combinaciones de ciudades como origen-destino.
16. Genera una lista de repartidores con todos los estados posibles (para simular escenarios).

### FULL OUTER
17. Lista los clientes y repartidores que comparten ciudad.
18. Muestra todos los clientes y repartidores en una sola lista (con un campo que indique el tipo).

### Integración con filtros
19. Envíos de repartidores de moto.
20. Paquetes de clientes de Bogotá con peso mayor a 1 kg.
21. Envíos en tránsito a Medellín.
22. Repartidores con envíos pendientes en Bucaramanga.
23. Clientes de Cali o Barranquilla con paquetes no entregados.

### Desafíos
24. Cliente con más paquetes enviados.
25. Repartidor con más entregas exitosas.
26. Ciudad con más orígenes de envío.
27. Promedio de peso de paquetes por ciudad de origen.
28. Días promedio entre despacho y entrega, por repartidor.
29. Repartidor que más rápido entrega (en promedio).
30. Clientes con paquetes que pesan más que el promedio general.
31. Listado de clientes con su último envío (subconsulta o `LATERAL`).
32. Repartidor y jefe que comparten la misma ciudad.
33. Paquetes cuya ciudad de origen es la misma del repartidor.
34. Total facturable: $5000 por kilo, por cada envío entregado.
35. Envíos donde la fecha de entrega es anterior a la de despacho (error).
36. Ciudades que son origen pero nunca destino, y viceversa.
37. Parejas de repartidores que comparten vehículo.
38. Clientes con todos sus paquetes ya entregados.
39. Paquetes que han pasado por más de 3 repartidores.
40. Ranking de repartidores por entregas (top 3).

---

## Recursos

- https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN
- https://www.postgresqltutorial.com/postgresql-joins/
- https://sqlbolt.com/lesson/select_joins

---

**Próxima sesión:** Módulo 10 — Consultas Avanzadas.
