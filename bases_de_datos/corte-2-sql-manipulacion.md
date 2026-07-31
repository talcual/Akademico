# Corte 2 — SQL y Manipulación de Datos

**Módulos:** 5, 6, 7, 8, 9 y 10
**Duración aprox.:** Semanas 7 a 12
**Foco:** DDL, DML, funciones, relaciones, JOINs y consultas avanzadas.

---

## Competencias a desarrollar

- Crear, modificar y eliminar objetos de base de datos con DDL.
- Realizar operaciones CRUD completas sobre una base de datos relacional.
- Aplicar funciones de agregación, escalares y de fecha.
- Implementar correctamente relaciones 1:1, 1:N y N:M con integridad referencial.
- Dominar todos los tipos de JOIN.
- Construir consultas avanzadas con subconsultas, `GROUP BY`, `HAVING` y `CASE`.

---

## Cronograma por semanas

### Semana 7 — Módulo 5: SQL Básico (DDL)
**Temas:**
- `CREATE DATABASE`
- `CREATE TABLE`
- `ALTER TABLE`
- `DROP`
- `TRUNCATE`
- Tipos de datos en PostgreSQL
- Restricciones (PK, FK, UNIQUE, NOT NULL, CHECK, DEFAULT)
- `SERIAL` / `AUTO_INCREMENT`

**Práctica:** Crear la base de datos del **Proyecto 2** (sistema de gestión de una clínica veterinaria, asignado por el docente).

---

### Semana 8 — Módulo 6: CRUD (DML)
**Temas:**
- `INSERT`
- `UPDATE`
- `DELETE`
- `SELECT`
- Filtros (`WHERE`, operadores lógicos y de comparación)
- `ORDER BY`
- `LIMIT` / `OFFSET`
- `DISTINCT`
- Alias (`AS`)

**Práctica:** Cargar datos de prueba y resolver 30 ejercicios CRUD sobre la base de la veterinaria.

---

### Semana 9 — Módulo 7: Funciones SQL
**Temas:**
- Agregación: `COUNT`, `SUM`, `AVG`, `MAX`, `MIN`
- Fecha/hora: `NOW`, `DATE`, `EXTRACT`, `AGE`
- Cadenas: `CONCAT`, `UPPER`, `LOWER`
- Nulas: `COALESCE`

**Práctica:** Consultas de reporte sobre la base de la veterinaria (mascotas atendidas por mes, total facturado por veterinario, etc.).

---

### Semana 10 — Módulo 8: Relaciones
**Temas:**
- Uno a Uno
- Uno a Muchos
- Muchos a Muchos y tablas puente
- Integridad referencial
- `ON DELETE` (CASCADE, RESTRICT, SET NULL, NO ACTION)
- `ON UPDATE`

**Práctica:** Modelar las relaciones completas del sistema veterinario (mascotas ↔ dueños, veterinarios ↔ consultas, consultas ↔ medicamentos).

---

### Semana 11 — Módulo 9: JOINs
**Temas:**
- `INNER JOIN`
- `LEFT JOIN`
- `RIGHT JOIN`
- `FULL OUTER JOIN`
- `SELF JOIN`
- `CROSS JOIN`

**Práctica:** Set de 40+ ejercicios de JOINs de dificultad progresiva sobre la base de la veterinaria.

**Tarea:** Resolver los mismos ejercicios equivalentes en MariaDB para evidenciar compatibilidad.

---

### Semana 12 — Módulo 10: Consultas Avanzadas y Cierre del Corte
**Temas:**
- `GROUP BY`
- `HAVING`
- Subconsultas (escalar, de lista, correlacionadas)
- `EXISTS`
- `IN`
- `ANY` / `ALL`
- `CASE WHEN`

**Práctica:** Consultas analíticas (top 5 clientes, productos sin ventas, categorización de pacientes según edad).

**Cierre del Corte 2:**
- **Parcial 2 (teórico-práctico):** 40%
- Entrega del **Proyecto 2** (script SQL completo: DDL + DML + 20 consultas significativas). 30%

---

## Recursos del corte

- PostgreSQL Tutorial — https://www.postgresqltutorial.com/
- SQLBolt — https://sqlbolt.com/
- HackerRank SQL Practice — https://www.hackerrank.com/domains/sql
- Mode Analytics SQL Tutorial — https://mode.com/sql-tutorial

---

## Entregables

| # | Entregable | Semana |
|---|------------|--------|
| 1 | Script DDL del Proyecto 2 | 7 |
| 2 | 30 ejercicios CRUD resueltos | 8 |
| 3 | Reportes con funciones de agregación | 9 |
| 4 | Modelo relacional con todas las relaciones | 10 |
| 5 | 40 ejercicios de JOINs | 11 |
| 6 | Consultas avanzadas (subconsultas y CASE) | 12 |
| 7 | **Proyecto 2** (script completo) | 12 |
| 8 | **Parcial 2** | 12 |
