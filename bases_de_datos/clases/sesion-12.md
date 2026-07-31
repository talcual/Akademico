# Sesión 12 — Módulo 10: Consultas Avanzadas y Cierre del Corte 2

**Semana:** 12
**Corte:** 2
**Duración estimada:** 2 horas
**Caso de estudio:** Red Social de Viajeros "TripConnect"

---

## 1. Sistema de ejemplo: TripConnect

```sql
CREATE DATABASE tripconnect;
\c tripconnect

CREATE TABLE usuario (
    id          SERIAL PRIMARY KEY,
    username    VARCHAR(40) NOT NULL UNIQUE,
    nombre      VARCHAR(120) NOT NULL,
    pais        VARCHAR(60),
    fecha_registro DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE viaje (
    id          SERIAL PRIMARY KEY,
    usuario_id  INTEGER NOT NULL REFERENCES usuario(id) ON DELETE CASCADE,
    titulo      VARCHAR(150) NOT NULL,
    destino     VARCHAR(100) NOT NULL,
    fecha_inicio DATE NOT NULL,
    fecha_fin    DATE NOT NULL,
    presupuesto  NUMERIC(10,2) CHECK (presupuesto >= 0),
    calificacion INTEGER CHECK (calificacion BETWEEN 1 AND 5)
);

CREATE TABLE publicacion (
    id          SERIAL PRIMARY KEY,
    usuario_id  INTEGER NOT NULL REFERENCES usuario(id) ON DELETE CASCADE,
    contenido   TEXT NOT NULL,
    fecha       TIMESTAMP NOT NULL DEFAULT NOW(),
    likes       INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE etiqueta (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(40) NOT NULL UNIQUE
);

CREATE TABLE publicacion_etiqueta (
    publicacion_id INTEGER REFERENCES publicacion(id) ON DELETE CASCADE,
    etiqueta_id    INTEGER REFERENCES etiqueta(id) ON DELETE CASCADE,
    PRIMARY KEY (publicacion_id, etiqueta_id)
);
```

```sql
-- Datos
INSERT INTO usuario (username, nombre, pais, fecha_registro) VALUES
    ('@ana',   'Ana',     'Colombia',    '2023-01-15'),
    ('@beto',  'Beto',    'México',      '2023-03-20'),
    ('@cami',  'Camila',  'Argentina',   '2024-01-10'),
    ('@diego', 'Diego',   'Colombia',    '2024-05-22'),
    ('@elena', 'Elena',   'Chile',       '2025-02-08');

INSERT INTO viaje (usuario_id, titulo, destino, fecha_inicio, fecha_fin, presupuesto, calificacion) VALUES
    (1, 'Aventura en Cartagena', 'Cartagena', '2025-12-01', '2025-12-07', 2500000, 5),
    (1, 'Tour por el Eje Cafetero', 'Eje Cafetero', '2024-06-01', '2024-06-05', 1800000, 4),
    (2, 'Cancún todo incluido', 'Cancún', '2025-07-10', '2025-07-17', 5500000, 5),
    (2, 'CDMX cultural', 'Ciudad de México', '2024-09-01', '2024-09-04', 1200000, 3),
    (3, 'Buenos Aires y tango', 'Buenos Aires', '2025-11-15', '2025-11-22', 3200000, 4),
    (4, 'Bogotá gastronómica', 'Bogotá', '2026-01-10', '2026-01-12', 600000, NULL),
    (5, 'Santiago y viñedos', 'Santiago', '2025-03-05', '2025-03-10', 2800000, 4);

INSERT INTO publicacion (usuario_id, contenido, fecha, likes) VALUES
    (1, 'Atardecer en Cartagena, increíble!',           '2025-12-05 19:00', 120),
    (1, 'El café de Salento es otro nivel',             '2024-06-03 16:00',  85),
    (2, 'Las pirámides de Teotihuacán, una experiencia única', '2024-09-02 12:00', 200),
    (3, 'El barrio de La Boca, lleno de color',         '2025-11-18 17:00',  95),
    (4, 'El mejor ajiaco de Bogotá',                    '2026-01-11 13:00',  60),
    (4, 'Recomendaciones de Monserrate',                '2026-01-12 09:00',  45);

INSERT INTO etiqueta (nombre) VALUES
    ('playa'),('aventura'),('gastronomía'),('cultura'),('naturaleza');

INSERT INTO publicacion_etiqueta (publicacion_id, etiqueta_id) VALUES
    (1, 1), (1, 2),
    (2, 3), (2, 5),
    (3, 4),
    (4, 4), (4, 1),
    (5, 3), (6, 4);
```

---

## 2. GROUP BY

Agrupa filas que comparten valores en una o más columnas, permitiendo aplicar funciones de agregación.

```sql
-- Cantidad de viajes por usuario
SELECT u.username, COUNT(v.id) AS total_viajes
FROM usuario u
LEFT JOIN viaje v ON v.usuario_id = u.id
GROUP BY u.username;
```

```sql
-- Total gastado por usuario
SELECT u.username, SUM(v.presupuesto) AS total_gastado
FROM usuario u
JOIN viaje v ON v.usuario_id = u.id
GROUP BY u.username
ORDER BY total_gastado DESC;
```

```sql
-- Viajes por país
SELECT u.pais, COUNT(v.id) AS total
FROM usuario u
JOIN viaje v ON v.usuario_id = u.id
GROUP BY u.pais;
```

---

## 3. HAVING

Filtra **después** de agrupar (a diferencia de `WHERE` que filtra antes).

```sql
-- Usuarios con más de 1 viaje
SELECT u.username, COUNT(v.id) AS total
FROM usuario u
JOIN viaje v ON v.usuario_id = u.id
GROUP BY u.username
HAVING COUNT(v.id) > 1;
```

```sql
-- Etiquetas con más de 1 publicación
SELECT e.nombre, COUNT(pe.publicacion_id) AS total
FROM etiqueta e
JOIN publicacion_etiqueta pe ON pe.etiqueta_id = e.id
GROUP BY e.nombre
HAVING COUNT(pe.publicacion_id) > 1;
```

### WHERE vs HAVING

| Aspecto | WHERE | HAVING |
|---------|-------|--------|
| Filtra | Filas individuales | Grupos |
| Antes de | `GROUP BY` | `GROUP BY` |
| Permite agregaciones | No | Sí |

```sql
-- Combinados
SELECT u.username, AVG(v.calificacion) AS promedio
FROM usuario u
JOIN viaje v ON v.usuario_id = u.id
WHERE v.fecha_inicio >= '2025-01-01'   -- filtra filas
GROUP BY u.username
HAVING AVG(v.calificacion) >= 4;        -- filtra grupos
```

---

## 4. Subconsultas

Una subconsulta es un `SELECT` dentro de otro `SELECT`.

### Subconsulta escalar (devuelve un valor)

```sql
-- Viajes con presupuesto mayor al promedio
SELECT * FROM viaje
WHERE presupuesto > (SELECT AVG(presupuesto) FROM viaje);
```

### Subconsulta de lista (con `IN`)

```sql
-- Usuarios que han viajado a Cartagena
SELECT * FROM usuario
WHERE id IN (SELECT usuario_id FROM viaje WHERE destino = 'Cartagena');
```

### Subconsulta correlacionada

```sql
-- Publicaciones con más likes que el promedio de su propio usuario
SELECT p1.*
FROM publicacion p1
WHERE p1.likes > (
    SELECT AVG(p2.likes)
    FROM publicacion p2
    WHERE p2.usuario_id = p1.usuario_id
);
```

---

## 5. EXISTS

Verifica si una subconsulta devuelve **al menos una fila**.

```sql
-- Usuarios que tienen al menos una publicación
SELECT * FROM usuario u
WHERE EXISTS (
    SELECT 1 FROM publicacion p WHERE p.usuario_id = u.id
);
```

> `EXISTS` es más eficiente que `IN` en muchos casos.

---

## 6. IN, ANY, ALL

### IN

```sql
-- Usuarios de Colombia o México
SELECT * FROM usuario WHERE pais IN ('Colombia', 'México');
```

### ANY (o SOME)

```sql
-- Viajes cuyo presupuesto es mayor que AL MENOS UNO de los viajes a Cancún
SELECT * FROM viaje
WHERE presupuesto > ANY (SELECT presupuesto FROM viaje WHERE destino = 'Cancún');
```

### ALL

```sql
-- Viajes cuyo presupuesto es mayor que TODOS los de Buenos Aires
SELECT * FROM viaje
WHERE presupuesto > ALL (SELECT presupuesto FROM viaje WHERE destino = 'Buenos Aires');
```

---

## 7. CASE WHEN

Permite lógica condicional en SQL (como un `if/else`).

```sql
-- Categorizar viajes según presupuesto
SELECT titulo, destino, presupuesto,
    CASE
        WHEN presupuesto < 1000000 THEN 'Económico'
        WHEN presupuesto < 3000000 THEN 'Medio'
        ELSE 'Premium'
    END AS categoria
FROM viaje;
```

```sql
-- Estado de viajes según fecha
SELECT titulo, fecha_fin,
    CASE
        WHEN fecha_fin >= CURRENT_DATE THEN 'En curso o futuro'
        WHEN fecha_fin >= CURRENT_DATE - INTERVAL '1 year' THEN 'Reciente'
        ELSE 'Antiguo'
    END AS antiguedad
FROM viaje;
```

```sql
-- CASE en agregaciones
SELECT u.username,
    COUNT(v.id) AS total_viajes,
    SUM(CASE WHEN v.calificacion >= 4 THEN 1 ELSE 0 END) AS viajes_buenos
FROM usuario u
LEFT JOIN viaje v ON v.usuario_id = u.id
GROUP BY u.username;
```

---

## 8. Cierre del Corte 2

### Resumen de consultas que debes dominar

```sql
-- 1. GROUP BY + HAVING
SELECT genero, AVG(calificacion)
FROM pelicula
GROUP BY genero
HAVING AVG(calificacion) > 7;

-- 2. Subconsulta
SELECT nombre FROM usuario
WHERE id NOT IN (SELECT usuario_id FROM viaje);

-- 3. EXISTS
SELECT * FROM viaje v
WHERE EXISTS (
    SELECT 1 FROM publicacion p
    WHERE p.usuario_id = v.usuario_id
    AND p.likes > 100
);

-- 4. CASE
SELECT titulo, calificacion,
    CASE WHEN calificacion >= 4 THEN 'Recomendada' ELSE 'Normal' END
FROM viaje;

-- 5. ANY/ALL
SELECT * FROM viaje
WHERE presupuesto > ALL (SELECT presupuesto FROM viaje WHERE destino = 'Bogotá');
```

---

## Ejercicios

### GROUP BY
1. Cantidad de publicaciones por usuario.
2. Promedio de likes por usuario.
3. Total de likes por etiqueta.
4. Número de usuarios por país.
5. Cantidad de viajes por destino.

### HAVING
6. Usuarios con más de 2 publicaciones.
7. Etiquetas con al menos 2 publicaciones.
8. Destinos con presupuesto promedio mayor a 2'000.000.
9. Usuarios cuyo promedio de calificación de viajes es mayor a 4.

### Subconsultas
10. Usuarios que NO han publicado nada.
11. Viajes con presupuesto máximo.
12. Etiquetas que NO se usan en ninguna publicación.
13. Usuarios con más likes que el promedio general.
14. Publicaciones con likes mayores al promedio de su autor.

### EXISTS / IN
15. Usuarios que tengan viajes con calificación 5.
16. Usuarios que NO tengan viajes a 'Bogotá'.
17. Etiquetas que aparecen en publicaciones del usuario '@ana'.
18. Viajes a destinos donde también ha viajado otro usuario.

### ANY / ALL
19. Viajes con presupuesto mayor al de TODOS los viajes de '@beto'.
20. Publicaciones con likes mayor que AL MENOS UNA publicación de '@cami'.

### CASE
21. Clasifica usuarios: nuevos (este año), regulares (1-3 años), antiguos (>3 años).
22. Clasifica viajes según duración (días).
23. Etiqueta cada publicación como "viral" (>100 likes), "popular" (50-100), "normal" (resto).
24. Crea una columna que muestre "Positivo", "Neutro" o "Negativo" según la calificación del viaje.

### Integradores
25. Top 3 usuarios con más likes totales en sus publicaciones.
26. Promedio de presupuesto por destino, solo destinos con más de 1 viaje.
27. Publicación más popular por usuario.
28. Parejas de usuarios que viajaron al mismo destino.
29. Etiqueta más popular por país de origen del usuario.
30. Reporte mensual: viajes iniciados por mes en 2025, ordenado de mayor a menor.

---

## Recursos

- https://www.postgresql.org/docs/current/queries-group.html
- https://www.postgresql.org/docs/current/functions-subquery.html
- https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-EXPRESSIONS-CASE

---

**Próxima sesión:** Inicio del Corte 3 — Módulo 11: Normalización.
