# Sesión 9 — Módulo 7: Funciones SQL

**Semana:** 9
**Corte:** 2
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema de Streaming "CinePlus"

---

## 1. Sistema de ejemplo: CinePlus

```sql
-- Crear BD
CREATE DATABASE cineplus;
\c cineplus

-- Tablas
CREATE TABLE cliente (
    id          SERIAL PRIMARY KEY,
    email       VARCHAR(120) NOT NULL UNIQUE,
    nombre      VARCHAR(120) NOT NULL,
    fecha_alta  DATE NOT NULL DEFAULT CURRENT_DATE,
    pais        VARCHAR(60) DEFAULT 'Colombia'
);

CREATE TABLE pelicula (
    id           SERIAL PRIMARY KEY,
    titulo       VARCHAR(200) NOT NULL,
    genero       VARCHAR(50) NOT NULL,
    duracion_min INTEGER NOT NULL CHECK (duracion_min > 0),
    anio         INTEGER,
    calificacion NUMERIC(3,1) CHECK (calificacion BETWEEN 0 AND 10)
);

CREATE TABLE suscripcion (
    id          SERIAL PRIMARY KEY,
    cliente_id  INTEGER NOT NULL REFERENCES cliente(id) ON DELETE CASCADE,
    plan        VARCHAR(20) NOT NULL CHECK (plan IN ('Basico','Estandar','Premium')),
    fecha_inicio DATE NOT NULL DEFAULT CURRENT_DATE,
    costo_mensual NUMERIC(10,2) NOT NULL CHECK (costo_mensual > 0),
    activa      BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE visualizacion (
    id          SERIAL PRIMARY KEY,
    cliente_id  INTEGER NOT NULL REFERENCES cliente(id) ON DELETE CASCADE,
    pelicula_id INTEGER NOT NULL REFERENCES pelicula(id) ON DELETE RESTRICT,
    fecha_hora  TIMESTAMP NOT NULL DEFAULT NOW(),
    minutos_vistos INTEGER NOT NULL
);
```

```sql
-- Datos de prueba
INSERT INTO cliente (email, nombre, fecha_alta, pais) VALUES
    ('a@x.com', 'Andrea Salas',  '2023-03-15', 'Colombia'),
    ('b@x.com', 'Beto Lara',     '2024-01-10', 'México'),
    ('c@x.com', 'Camila Ríos',   '2025-08-22', 'Colombia'),
    ('d@x.com', 'Diego Pinto',   '2022-11-05', 'Argentina'),
    ('e@x.com', 'Elena Mora',    '2024-06-30', 'Colombia'),
    ('f@x.com', 'Federico Cano', '2025-12-01', 'Chile');

INSERT INTO pelicula (titulo, genero, duracion_min, anio, calificacion) VALUES
    ('El Origen',          'Ciencia Ficción', 148, 2010, 8.8),
    ('Interestelar',      'Ciencia Ficción', 169, 2014, 8.6),
    ('Coco',               'Animación',       105, 2017, 8.4),
    ('Parásitos',          'Thriller',        132, 2019, 8.5),
    ('El Padrino',         'Drama',           175, 1972, 9.2),
    ('Spider-Man NWH',     'Acción',          148, 2021, 8.2),
    ('Pulp Fiction',       'Crimen',          154, 1994, 8.9),
    ('La La Land',         'Musical',         128, 2016, 8.0),
    ('Duna',               'Ciencia Ficción', 155, 2021, 8.0),
    ('El Conjuro',         'Terror',          112, 2013, 7.5);

INSERT INTO suscripcion (cliente_id, plan, fecha_inicio, costo_mensual, activa) VALUES
    (1, 'Premium',  '2023-03-15', 45000, TRUE),
    (1, 'Estandar', '2024-01-01', 25000, FALSE),
    (2, 'Basico',   '2024-01-10', 15000, TRUE),
    (3, 'Premium',  '2025-08-22', 45000, TRUE),
    (4, 'Estandar', '2022-11-05', 25000, TRUE),
    (5, 'Basico',   '2024-06-30', 15000, FALSE),
    (6, 'Premium',  '2025-12-01', 45000, TRUE);

INSERT INTO visualizacion (cliente_id, pelicula_id, fecha_hora, minutos_vistos) VALUES
    (1, 1, '2026-01-10 20:00', 148),
    (1, 2, '2026-01-15 22:00', 169),
    (1, 3, '2026-02-01 18:00', 105),
    (2, 5, '2026-01-12 21:00', 175),
    (2, 7, '2026-02-05 23:00', 154),
    (3, 2, '2026-02-10 20:30', 169),
    (3, 9, '2026-02-20 21:00', 90),
    (4, 5, '2026-01-08 19:00', 175),
    (4, 1, '2026-02-14 22:00', 148),
    (5, 6, '2026-02-25 20:00', 100),
    (6, 4, '2026-03-01 22:00', 132),
    (6, 10,'2026-03-05 23:30', 112);
```

---

## 2. Funciones de agregación

| Función | Descripción |
|---------|-------------|
| `COUNT(*)` | Cuenta filas |
| `COUNT(col)` | Cuenta valores no nulos de la columna |
| `COUNT(DISTINCT col)` | Cuenta valores distintos |
| `SUM(col)` | Suma |
| `AVG(col)` | Promedio |
| `MIN(col)` | Mínimo |
| `MAX(col)` | Máximo |

```sql
-- Total de clientes
SELECT COUNT(*) FROM cliente;

-- Total de películas con calificación
SELECT COUNT(calificacion) FROM pelicula;

-- Géneros distintos
SELECT COUNT(DISTINCT genero) FROM pelicula;

-- Duración total y promedio
SELECT SUM(duracion_min) AS total_minutos,
       AVG(duracion_min) AS promedio_min
FROM pelicula;

-- Película más larga y más corta
SELECT MAX(duracion_min), MIN(duracion_min) FROM pelicula;

-- Costo total mensual de suscripciones activas
SELECT SUM(costo_mensual) FROM suscripcion WHERE activa = TRUE;
```

---

## 3. Funciones de fecha y hora

| Función | Descripción |
|---------|-------------|
| `NOW()` | Timestamp actual |
| `CURRENT_DATE` | Fecha actual |
| `CURRENT_TIME` | Hora actual |
| `EXTRACT(campo FROM fecha)` | Extrae parte de una fecha |
| `AGE(fecha1, fecha2)` | Diferencia entre fechas |
| `DATE(fecha)` | Convierte a DATE |
| `DATE_TRUNC('campo', fecha)` | Trunca a un campo |

```sql
-- Clientes dados de alta este año
SELECT * FROM cliente
WHERE EXTRACT(YEAR FROM fecha_alta) = 2026;

-- Antigüedad en años
SELECT nombre,
       EXTRACT(YEAR FROM AGE(NOW(), fecha_alta)) AS anios_cliente
FROM cliente;

-- Visualizaciones del mes actual
SELECT * FROM visualizacion
WHERE EXTRACT(MONTH FROM fecha_hora) = EXTRACT(MONTH FROM NOW())
  AND EXTRACT(YEAR FROM fecha_hora) = EXTRACT(YEAR FROM NOW());
```

---

## 4. Funciones de cadena

| Función | Descripción |
|---------|-------------|
| `CONCAT(a, b, ...)` | Concatena |
| `\|\|` | Operador de concatenación |
| `UPPER(s)` | Mayúsculas |
| `LOWER(s)` | Minúsculas |
| `LENGTH(s)` | Longitud |
| `SUBSTRING(s, pos, len)` | Subcadena |
| `TRIM(s)` | Quita espacios extremos |
| `REPLACE(s, from, to)` | Reemplaza texto |

```sql
SELECT UPPER(nombre), LOWER(email) FROM cliente;

-- Emails con dominio
SELECT nombre, CONCAT(email, ' | ', pais) AS info FROM cliente;

-- Películas cuyo título tiene más de 10 caracteres
SELECT titulo FROM pelicula WHERE LENGTH(titulo) > 10;

-- Reemplazar palabra en títulos
SELECT REPLACE(titulo, 'El ', 'The ') FROM pelicula;
```

---

## 5. COALESCE y NULLIF

`COALESCE` devuelve el primer valor no nulo. `NULLIF` devuelve NULL si los dos argumentos son iguales.

```sql
-- Si pais es null, mostrar 'Desconocido'
SELECT nombre, COALESCE(pais, 'Desconocido') FROM cliente;

-- Si la calificación es 0, tratarla como null
SELECT titulo, COALESCE(NULLIF(calificacion, 0), 0) FROM pelicula;
```

---

## 6. Reportes de ejemplo

### Reporte 1: Total facturado por mes

```sql
SELECT
    EXTRACT(YEAR FROM fecha_inicio)  AS anio,
    EXTRACT(MONTH FROM fecha_inicio) AS mes,
    SUM(costo_mensual)               AS total
FROM suscripcion
GROUP BY anio, mes
ORDER BY anio, mes;
```

### Reporte 2: Promedio de calificación por género

```sql
SELECT genero,
       COUNT(*)        AS total_peliculas,
       ROUND(AVG(calificacion), 2) AS promedio_calif
FROM pelicula
GROUP BY genero
ORDER BY promedio_calif DESC;
```

### Reporte 3: Cliente más activo

```sql
SELECT c.nombre,
       COUNT(v.id)                AS visualizaciones,
       SUM(v.minutos_vistos)      AS total_minutos
FROM cliente c
JOIN visualizacion v ON v.cliente_id = c.id
GROUP BY c.nombre
ORDER BY visualizaciones DESC;
```

---

## Ejercicios

### Ejercicios básicos
1. Muestra el total de películas.
2. Calcula el promedio de calificación de todas las películas.
3. Muestra la fecha y hora actual con `NOW()`.
4. Convierte el nombre de todos los clientes a mayúsculas.
5. Muestra el email y la longitud de cada email.

### Ejercicios de fecha
6. ¿Cuántos clientes se registraron en 2024?
7. Lista los clientes dados de alta en enero de cualquier año.
8. Muestra hace cuántos años se registró cada cliente (usando `AGE`).
9. ¿Cuántas visualizaciones hubo en febrero de 2026?
10. Lista las suscripciones que llevan más de 1 año activas.

### Ejercicios de agregación
11. Total de minutos visualizados por cada cliente.
12. Cantidad de clientes por país.
13. Promedio de costo mensual por plan.
14. Calificación máxima y mínima por género.
15. Total de clientes con suscripción activa vs. inactiva.

### Ejercicios con cadenas
16. Lista los clientes cuyo email empieza con vocal.
17. Reemplaza en los títulos la palabra "El" por "The".
18. Muestra los primeros 5 caracteres del título de cada película.

### Ejercicios con COALESCE
19. Lista las películas mostrando "Sin calificar" si la calificación es NULL.
20. Muestra la fecha_alta de los clientes, o "2024-01-01" si es NULL.

### Ejercicios de reporte
21. Reporte: total facturado por plan, ordenado de mayor a menor.
22. Reporte: minutos visualizados por género de película.
23. Reporte: clientes con más de 100 minutos totales visualizados.

---

## Recursos

- https://www.postgresql.org/docs/current/functions-aggregate.html
- https://www.postgresql.org/docs/current/functions-datetime.html
- https://www.postgresql.org/docs/current/functions-string.html

---

**Próxima sesión:** Módulo 8 — Relaciones (1:1, 1:N, N:M).
