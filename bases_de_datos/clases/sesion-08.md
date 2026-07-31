# Sesión 8 — Módulo 6: CRUD (DML + SELECT)

**Semana:** 8
**Corte:** 2
**Duración estimada:** 2 horas
**Caso de estudio:** Veterinaria (continuación)

---

## 1. ¿Qué es CRUD?

CRUD = **C**reate, **R**ead, **U**pdate, **D**elete. Son las 4 operaciones básicas de persistencia.

| Operación | SQL |
|-----------|-----|
| Create | `INSERT` |
| Read | `SELECT` |
| Update | `UPDATE` |
| Delete | `DELETE` |

---

## 2. INSERT

### Inserción simple

```sql
INSERT INTO dueno (cedula, nombre, telefono, email)
VALUES ('1234567890', 'María Pérez', '3001234567', 'maria@mail.com');
```

> Si omites columnas, debes proporcionar valores para **todas** las columnas (en orden). Si especificas columnas, solo para esas.

### Inserción múltiple

```sql
INSERT INTO dueno (cedula, nombre, telefono) VALUES
    ('111', 'Juan López',     '3110000001'),
    ('222', 'Ana Gómez',      '3110000002'),
    ('333', 'Pedro Ramírez',  '3110000003');
```

### Inserción con retorno

```sql
INSERT INTO dueno (cedula, nombre) VALUES ('444', 'Lucía')
RETURNING id, nombre;
```

---

## 3. SELECT

```sql
SELECT * FROM dueno;
SELECT nombre, email FROM dueno;
```

### WHERE — filtros

```sql
SELECT * FROM dueno WHERE nombre = 'María Pérez';
SELECT * FROM mascota WHERE especie = 'perro' AND peso_kg > 10;
SELECT * FROM consulta WHERE fecha >= '2026-01-01';
```

### Operadores

| Operador | Significado |
|----------|-------------|
| `=` | Igual |
| `<>`, `!=` | Distinto |
| `<`, `>`, `<=`, `>=` | Comparación |
| `AND`, `OR`, `NOT` | Lógicos |
| `IN` | En una lista |
| `BETWEEN` | En un rango |
| `LIKE` | Patrón (con `%` y `_`) |
| `IS NULL` | Es nulo |

### LIKE

```sql
SELECT * FROM dueno WHERE nombre LIKE 'M%';      -- empieza por M
SELECT * FROM dueno WHERE nombre LIKE '%rez';     -- termina en rez
SELECT * FROM dueno WHERE nombre LIKE '%ar%';     -- contiene "ar"
SELECT * FROM dueno WHERE nombre LIKE '____';     -- exactamente 4 caracteres
```

### IN y BETWEEN

```sql
SELECT * FROM mascota WHERE especie IN ('perro','gato');
SELECT * FROM consulta WHERE fecha BETWEEN '2026-01-01' AND '2026-12-31';
```

### ORDER BY

```sql
SELECT * FROM mascota ORDER BY nombre ASC;       -- ascendente
SELECT * FROM consulta ORDER BY fecha DESC;      -- descendente
SELECT * FROM mascota ORDER BY especie, nombre;  -- múltiples columnas
```

### LIMIT y OFFSET (paginación)

```sql
SELECT * FROM consulta ORDER BY fecha DESC LIMIT 10;
SELECT * FROM consulta ORDER BY fecha DESC LIMIT 10 OFFSET 10;  -- página 2
```

### DISTINCT

```sql
SELECT DISTINCT especie FROM mascota;
```

### Alias (AS)

```sql
SELECT d.nombre AS "Dueño", m.nombre AS "Mascota"
FROM dueno d
JOIN mascota m ON m.dueno_id = d.id;
```

---

## 4. UPDATE

```sql
UPDATE mascota SET peso_kg = 12.5 WHERE id = 1;

-- Múltiples columnas
UPDATE dueno SET telefono = '3009998877', email = 'nuevo@mail.com'
WHERE cedula = '1234567890';
```

### ⚠ Cuidado con UPDATE sin WHERE

```sql
UPDATE mascota SET peso_kg = 0;   -- ¡Actualiza TODAS las filas!
```

### UPDATE con RETURNING

```sql
UPDATE consulta SET costo = costo * 1.10
WHERE fecha < '2026-01-01'
RETURNING id, costo;
```

---

## 5. DELETE

```sql
DELETE FROM mascota WHERE id = 5;
DELETE FROM consulta WHERE fecha < '2020-01-01';
```

### ⚠ Cuidado con DELETE sin WHERE

```sql
DELETE FROM mascota;   -- ¡Vacía la tabla!
```

> Diferencia con `TRUNCATE`: `DELETE` se puede deshacer dentro de una transacción y dispara triggers. `TRUNCATE` es más rápido pero no se puede filtrar.

---

## 6. Carga de datos de prueba (script sugerido)

```sql
-- Dueños
INSERT INTO dueno (cedula, nombre, telefono, email) VALUES
    ('1001', 'Carlos Vega',     '3001111111', 'carlos@mail.com'),
    ('1002', 'Diana Ortiz',     '3002222222', 'diana@mail.com'),
    ('1003', 'Eduardo Castro',  '3003333333', 'eduardo@mail.com'),
    ('1004', 'Fernanda Ruiz',   '3004444444', 'fernanda@mail.com'),
    ('1005', 'Gabriel Torres',  '3005555555', 'gabriel@mail.com');

-- Veterinarios
INSERT INTO veterinario (nombre, especialidad, email) VALUES
    ('Dr. Hugo Méndez',  'Cirugía',     'hugo@vete.com'),
    ('Dra. Laura Ríos',  'Dermatología','laura@vete.com'),
    ('Dr. Pablo Niño',   'Cardiología', 'pablo@vete.com');

-- Mascotas
INSERT INTO mascota (nombre, especie, raza, fecha_nac, peso_kg, dueno_id) VALUES
    ('Firulais',  'perro', 'Labrador',  '2020-03-15', 28.5, 1),
    ('Michi',     'gato',  'Persa',     '2021-07-22', 4.2,  1),
    ('Rex',       'perro', 'Husky',     '2019-11-05', 32.0, 2),
    ('Luna',      'gato',  'Siamés',    '2022-01-10', 3.8,  3),
    ('Toby',      'perro', 'Beagle',    '2018-05-20', 11.5, 4),
    ('Nala',      'gato',  'Común',     '2023-04-12', 3.0,  5),
    ('Max',       'perro', 'Golden',    '2020-08-30', 30.0, 2);

-- Consultas
INSERT INTO consulta (fecha, motivo, diagnostico, costo, mascota_id, veterinario_id) VALUES
    ('2026-01-15 10:00', 'Vacunación anual',  'Sano, aplicar refuerzo', 50000,  1, 1),
    ('2026-02-20 11:30', 'Pérdida de pelo',  'Dermatitis alérgica',    85000,  2, 2),
    ('2026-03-05 09:00', 'Control general',  'Peso adecuado',          45000,  3, 1),
    ('2026-03-15 16:00', 'Tos persistente',  'Infección respiratoria', 75000,  5, 1),
    ('2026-04-02 14:00', 'Chequeo corazón',  'Soplo leve',             120000, 7, 3);
```

---

## Ejercicios (30 propuestos)

### Inserción
1. Inserta 3 dueños nuevos.
2. Inserta 2 veterinarios adicionales.
3. Inserta 5 mascotas para esos dueños.

### SELECT básico
4. Muestra todos los dueños.
5. Muestra solo nombre y email de los dueños.
6. Muestra las mascotas de especie 'perro'.
7. Muestra las mascotas nacidas después de 2021-01-01.
8. Muestra los dueños con email que contenga "@mail.com".

### WHERE
9. Mascotas con peso entre 5 y 30 kg.
10. Consultas con costo mayor a 60000.
11. Mascotas cuyo nombre empieza por 'L' o termina en 'a'.
12. Veterinarios cuya especialidad NO es 'Cirugía'.

### ORDER BY, LIMIT, OFFSET
13. Las 3 consultas más recientes.
14. Las 5 mascotas más pesadas.
15. Mascotas ordenadas por especie y luego por nombre.
16. Página 2 del listado de mascotas (5 por página).

### DISTINCT y Alias
17. Lista de especies sin duplicados.
18. Lista de motivos de consulta únicos.
19. Reporte con alias "Veterinario" y "Especialidad".

### UPDATE
20. Cambia el peso de Firulais a 29.0.
21. Aumenta el costo de todas las consultas en un 10%.
22. Cambia el email de Carlos Vega a "nuevo_carlos@mail.com".

### DELETE
23. Elimina la mascota con id=10.
24. Elimina todos los dueños sin mascotas (cuidado con la FK).

### Combinados
25. Mascotas de Carlos Vega, ordenadas por nombre.
26. Consultas de los últimos 6 meses, solo las 5 más costosas.
27. Veterinarios cuyo email contenga 'vete' o cuyo nombre empiece por 'Dr.'.
28. Mascotas de especie 'gato' con peso < 5 kg.
29. Dueños sin mascotas (usando subconsulta `NOT IN`, vista después).
30. Consultas del año 2026 agrupadas por mes (se verá en módulos posteriores, intenta hacerlo).

---

## Recursos

- https://www.postgresql.org/docs/current/sql-select.html
- https://www.postgresqltutorial.com/postgresql-select/
- https://sqlbolt.com/

---

**Próxima sesión:** Módulo 7 — Funciones SQL.
