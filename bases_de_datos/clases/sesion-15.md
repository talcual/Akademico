# Sesión 15 — Módulo 13 (Vistas) + Módulo 14 (Procedimientos Almacenados)

**Semana:** 15
**Corte:** 3
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema de Recursos Humanos "RRHHPro"

---

## 1. Base de datos del sistema

```sql
CREATE DATABASE rrhhpro;
\c rrhhpro

CREATE TABLE departamento (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(80) NOT NULL UNIQUE
);

CREATE TABLE empleado (
    id           SERIAL PRIMARY KEY,
    cedula       VARCHAR(20) NOT NULL UNIQUE,
    nombre       VARCHAR(120) NOT NULL,
    email        VARCHAR(120) UNIQUE,
    salario      NUMERIC(12,2) NOT NULL CHECK (salario > 0),
    fecha_ingreso DATE NOT NULL,
    departamento_id INTEGER NOT NULL REFERENCES departamento(id) ON DELETE RESTRICT,
    jefe_id      INTEGER REFERENCES empleado(id) ON DELETE SET NULL
);

CREATE TABLE proyecto (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(150) NOT NULL,
    presupuesto NUMERIC(12,2) CHECK (presupuesto >= 0),
    fecha_inicio DATE,
    fecha_fin DATE
);

CREATE TABLE asignacion (
    id           SERIAL PRIMARY KEY,
    empleado_id  INTEGER NOT NULL REFERENCES empleado(id) ON DELETE CASCADE,
    proyecto_id  INTEGER NOT NULL REFERENCES proyecto(id) ON DELETE CASCADE,
    horas        INTEGER NOT NULL CHECK (horas > 0),
    rol          VARCHAR(80),
    UNIQUE (empleado_id, proyecto_id)
);

-- Datos
INSERT INTO departamento (nombre) VALUES
    ('Tecnología'),('Talento Humano'),('Ventas'),('Finanzas'),('Operaciones');

INSERT INTO empleado (cedula, nombre, email, salario, fecha_ingreso, departamento_id, jefe_id) VALUES
    ('1001','Laura CEO',     'laura@e.com',  25000000, '2018-01-10', 1, NULL),
    ('1002','Carlos CTO',    'carlos@e.com', 18000000, '2019-03-15', 1, 1),
    ('1003','Marta Dev',     'marta@e.com',  10000000, '2020-07-01', 1, 2),
    ('1004','Pedro Dev',     'pedro@e.com',  9500000,  '2021-02-20', 1, 2),
    ('1005','Sofía HR',      'sofia@e.com',  8000000,  '2020-05-10', 2, 1),
    ('1006','Andrés Ventas', 'andres@e.com', 7000000,  '2021-08-12', 3, 1),
    ('1007','Lucía Fin',     'lucia@e.com',  9000000,  '2019-11-05', 4, 1),
    ('1008','Tomás Ops',     'tomas@e.com',  6500000,  '2022-04-01', 5, 1);

INSERT INTO proyecto (nombre, presupuesto, fecha_inicio, fecha_fin) VALUES
    ('Migración Cloud',     50000000, '2025-01-15', '2026-06-30'),
    ('Nuevo Portal Web',    20000000, '2025-03-01', '2025-12-31'),
    ('App Móvil Clientes',  35000000, '2025-06-01', '2026-12-31'),
    ('Sistema de Nómina',   15000000, '2025-02-01', '2025-08-31');

INSERT INTO asignacion (empleado_id, proyecto_id, horas, rol) VALUES
    (2, 1, 200, 'Líder técnico'),
    (3, 1, 400, 'Desarrolladora'),
    (4, 1, 400, 'Desarrollador'),
    (3, 2, 150, 'Backend'),
    (4, 2, 150, 'Frontend'),
    (2, 3, 100, 'Arquitecto'),
    (5, 4, 300, 'Analista funcional');
```

---

## MÓDULO 13: VISTAS

## 2. ¿Qué es una vista?

Una vista es una **consulta almacenada** que se puede usar como si fuera una tabla. No almacena datos por sí misma (salvo las vistas materializadas).

### Ventajas

- Simplificar consultas complejas.
- Reutilizar lógica.
- Encapsular la estructura (seguridad / abstracción).
- Controlar acceso (dar acceso a una vista, no a la tabla).

---

## 3. CREATE VIEW

```sql
-- Vista simple
CREATE VIEW v_empleados_basicos AS
SELECT id, nombre, email, departamento_id
FROM empleado;
```

```sql
-- Vista con JOIN
CREATE VIEW v_empleado_detalle AS
SELECT
    e.id,
    e.nombre,
    e.email,
    e.salario,
    d.nombre AS departamento,
    j.nombre AS jefe
FROM empleado e
JOIN departamento d ON d.id = e.departamento_id
LEFT JOIN empleado j ON j.id = e.jefe_id;
```

```sql
-- Vista con cálculos
CREATE VIEW v_salario_departamento AS
SELECT
    d.nombre AS departamento,
    COUNT(e.id)            AS num_empleados,
    SUM(e.salario)         AS total_salarios,
    ROUND(AVG(e.salario),2) AS promedio_salario,
    MAX(e.salario)         AS salario_max
FROM departamento d
LEFT JOIN empleado e ON e.departamento_id = d.id
GROUP BY d.nombre;
```

```sql
-- Vista con subconsulta
CREATE VIEW v_empleados_proyectos AS
SELECT
    e.nombre AS empleado,
    COUNT(a.id) AS num_proyectos,
    SUM(a.horas) AS total_horas
FROM empleado e
LEFT JOIN asignacion a ON a.empleado_id = e.id
GROUP BY e.nombre;
```

### Usar la vista

```sql
SELECT * FROM v_empleado_detalle WHERE departamento = 'Tecnología';
```

### Listar / eliminar vistas

```sql
\dv          -- En psql
DROP VIEW v_empleados_basicos;
```

---

## 4. Vistas actualizables

Una vista es **actualizable** si:
- Se basa en una sola tabla.
- No usa `DISTINCT`, `GROUP BY`, `HAVING`, `UNION`, funciones de agregación.
- No usa subconsultas ni `JOIN`.

```sql
CREATE VIEW v_devs AS
SELECT id, nombre, email, salario
FROM empleado
WHERE departamento_id = 1;

-- Se puede hacer:
UPDATE v_devs SET salario = 11000000 WHERE nombre = 'Marta Dev';
```

```sql
-- No se puede actualizar (tiene JOIN/agrupación):
UPDATE v_empleado_detalle SET salario = 0 WHERE id = 1;
-- ERROR:  cannot update view ...
```

---

## 5. Vistas materializadas (introducción)

Almacenan el **resultado** físicamente. Hay que refrescarlas.

```sql
CREATE MATERIALIZED VIEW mv_resumen_proyectos AS
SELECT
    p.nombre AS proyecto,
    COUNT(a.id) AS num_empleados,
    SUM(a.horas) AS total_horas
FROM proyecto p
LEFT JOIN asignacion a ON a.proyecto_id = p.id
GROUP BY p.nombre;

-- Refrescar
REFRESH MATERIALIZED VIEW mv_resumen_proyectos;
```

> Útiles para reportes costosos que se consultan frecuentemente y cambian poco.

---

## MÓDULO 14: PROCEDIMIENTOS ALMACENADOS

## 6. ¿Qué es un procedimiento almacenado?

Un bloque de código SQL/PL que se almacena en la base de datos y se ejecuta con `CALL`. Permite lógica compleja, validaciones y reutilización.

### Ventajas

- Centralizar lógica de negocio.
- Reducir tráfico de red.
- Encapsular transacciones complejas.

---

## 7. Sintaxis PL/pgSQL

### Estructura básica

```sql
CREATE OR REPLACE FUNCTION fn_sumar(a INT, b INT)
RETURNS INT AS $$
BEGIN
    RETURN a + b;
END;
$$ LANGUAGE plpgsql;

-- Uso
SELECT fn_sumar(3, 4);
```

### Procedimiento (PostgreSQL 11+)

```sql
CREATE OR REPLACE PROCEDURE sp_aumentar_salario(
    p_departamento_id INT,
    p_porcentaje NUMERIC
)
LANGUAGE plpgsql AS $$
BEGIN
    UPDATE empleado
    SET salario = salario * (1 + p_porcentaje / 100)
    WHERE departamento_id = p_departamento_id;

    RAISE NOTICE 'Salarios actualizados para el departamento %', p_departamento_id;
END;
$$;

CALL sp_aumentar_salario(1, 10);
```

---

## 8. Variables y parámetros

```sql
CREATE OR REPLACE PROCEDURE sp_asignar_horas(
    p_empleado_id INT,
    p_proyecto_id INT,
    p_horas INT
)
LANGUAGE plpgsql AS $$
DECLARE
    v_existe BOOLEAN;
    v_rol VARCHAR(80) := 'Sin rol';
BEGIN
    -- Validar
    SELECT EXISTS (
        SELECT 1 FROM empleado WHERE id = p_empleado_id
    ) INTO v_existe;

    IF NOT v_existe THEN
        RAISE EXCEPTION 'Empleado % no existe', p_empleado_id;
    END IF;

    IF p_horas <= 0 THEN
        RAISE EXCEPTION 'Las horas deben ser positivas';
    END IF;

    -- Insertar o actualizar
    INSERT INTO asignacion (empleado_id, proyecto_id, horas, rol)
    VALUES (p_empleado_id, p_proyecto_id, p_horas, v_rol)
    ON CONFLICT (empleado_id, proyecto_id)
    DO UPDATE SET horas = asignacion.horas + EXCLUDED.horas;
END;
$$;

CALL sp_asignar_horas(3, 2, 50);
```

---

## 9. Estructuras de control

### IF / ELSIF / ELSE

```sql
IF salario > 15000000 THEN
    categoria := 'Alto';
ELSIF salario > 8000000 THEN
    categoria := 'Medio';
ELSE
    categoria := 'Bajo';
END IF;
```

### CASE

```sql
CASE departamento_id
    WHEN 1 THEN categoria := 'TI';
    WHEN 2 THEN categoria := 'RRHH';
    ELSE categoria := 'Otro';
END CASE;
```

### WHILE

```sql
WHILE v_contador < 10 LOOP
    v_contador := v_contador + 1;
    -- ...
END LOOP;
```

### FOR (sobre rango)

```sql
FOR i IN 1..10 LOOP
    -- i toma valores 1, 2, ..., 10
END LOOP;
```

### FOR (sobre query)

```sql
FOR v_emp IN
    SELECT nombre, salario FROM empleado WHERE departamento_id = 1
LOOP
    RAISE NOTICE '% gana %', v_emp.nombre, v_emp.salario;
END LOOP;
```

---

## Ejercicios

### Vistas
1. Crea una vista `v_empleados_jefes` que muestre cada empleado con su jefe y el departamento.
2. Crea una vista `v_presupuesto_proyectos` con total de horas asignadas y costo estimado (asume $50.000 por hora).
3. Crea una vista `v_antiguedad` con los años de antigüedad de cada empleado.
4. Crea una vista materializada con el top 5 de empleados con más horas asignadas.
5. ¿Cuándo usarías una vista materializada en lugar de una vista normal?

### Procedimientos
6. Crea un procedimiento `sp_transferir_empleado(empleado_id, nuevo_dep_id)` que cambie al empleado de departamento validando que el destino exista.
7. Crea un procedimiento que aplique un bono del 5% al salario de empleados con más de 5 años de antigüedad.
8. Crea un procedimiento que genere un reporte (RAISE NOTICE) listando los empleados de un departamento, con su antigüedad y salario.

### Variables y control
9. Crea un procedimiento que reciba un monto y muestre cuántos empleados ganan más y cuántos ganan menos que ese monto.
10. Crea un procedimiento que use un `FOR ... IN SELECT` para imprimir todos los proyectos con su presupuesto total asignado.

---

## Recursos

- https://www.postgresql.org/docs/current/sql-createview.html
- https://www.postgresql.org/docs/current/sql-createprocedure.html
- https://www.postgresql.org/docs/current/plpgsql.html

---

**Próxima sesión:** Módulo 15 (Funciones) + Módulo 16 (Triggers).
