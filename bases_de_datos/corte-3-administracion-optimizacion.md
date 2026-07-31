# Corte 3 — Administración y Optimización

**Módulos:** 11, 12, 13, 14, 15, 16, 17, 18, 19 y 20
**Duración aprox.:** Semanas 13 a 18
**Foco:** Normalización, índices, vistas, programación en BD, transacciones, concurrencia, seguridad y optimización.

---

## Competencias a desarrollar

- Aplicar las formas normales (1FN, 2FN, 3FN, BCNF) y justificar desnormalizaciones.
- Diseñar índices adecuados y leer planes de ejecución.
- Crear vistas, funciones, procedimientos almacenados y triggers.
- Garantizar propiedades ACID en transacciones y manejar concurrencia.
- Administrar usuarios, roles y permisos.
- Optimizar consultas usando `EXPLAIN` y `EXPLAIN ANALYZE`.

---

## Cronograma por semanas

### Semana 13 — Módulo 11: Normalización
**Temas:**
- 1FN, 2FN, 3FN
- BCNF
- Desnormalización: cuándo y por qué

**Práctica:** Partir de un esquema desnormalizado y normalizarlo paso a paso hasta 3FN/BCNF.

---

### Semana 14 — Módulo 12: Índices
**Temas:**
- ¿Qué es un índice y cómo funciona?
- B-Tree y Hash
- Índices compuestos
- Índices únicos
- Cuándo crear un índice y cuándo NO hacerlo
- Uso de `EXPLAIN`

**Práctica:** Medir tiempos de consulta antes y después de crear índices. Comparar planes de ejecución.

---

### Semana 15 — Módulo 13 (Vistas) y Módulo 14 (Procedimientos Almacenados)
**Temas — Vistas:**
- `CREATE VIEW`
- Vistas actualizables
- Casos de uso

**Temas — Procedimientos:**
- Parámetros (IN, OUT, INOUT)
- Variables
- `IF`, `CASE`
- `WHILE`, `FOR`

**Práctica:** Crear vistas de reporte y procedimientos para automatizar tareas del sistema veterinario.

---

### Semana 16 — Módulo 15 (Funciones) y Módulo 16 (Triggers)
**Temas — Funciones:**
- Funciones SQL
- Funciones PL/pgSQL
- Retorno de tablas

**Temas — Triggers:**
- `BEFORE INSERT`, `AFTER INSERT`
- `BEFORE UPDATE`
- `AFTER DELETE`
- Auditoría automática

**Práctica:** Implementar triggers para auditoría de cambios de precios y control de stock de medicamentos.

---

### Semana 17 — Módulo 17 (Transacciones), Módulo 18 (Concurrencia) y Módulo 19 (Seguridad)
**Temas — Transacciones:**
- Propiedades ACID
- `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`

**Temas — Concurrencia:**
- Bloqueos
- Deadlocks
- Niveles de aislamiento (READ COMMITTED, REPEATABLE READ, SERIALIZABLE)
- MVCC en PostgreSQL

**Temas — Seguridad:**
- Usuarios y roles
- Permisos
- `GRANT`, `REVOKE`

**Práctica:** Simular escenarios de concurrencia (dos transacciones compitiendo por el mismo registro) y resolver deadlocks. Crear roles con permisos diferenciados (recepcionista, veterinario, administrador).

---

### Semana 18 — Módulo 20: Optimización y Cierre del Corte
**Temas:**
- `EXPLAIN` y `EXPLAIN ANALYZE`
- Lectura del plan de ejecución
- Optimización de consultas
- Índices adecuados según el patrón de uso
- Estadísticas del optimizador (`ANALYZE`)

**Práctica:** Optimizar un set de 10 consultas lentas proporcionadas por el docente, midiendo tiempos antes/después.

**Cierre del Corte 3:**
- **Parcial 3 (teórico-práctico):** 40%
- Entrega del **Proyecto 3** (sistema completo, optimizado, con documentación). 30%

---

## Proyecto final del semestre (opcional, según criterio del docente)

Migrar el sistema veterinario a un entorno multiusuario con:
- Roles y permisos definidos.
- Triggers de auditoría.
- Procedimientos para procesos críticos.
- Consultas optimizadas con `EXPLAIN` documentado.

---

## Recursos del corte

- PostgreSQL Indexes — https://www.postgresql.org/docs/current/indexes.html
- Use The Index, Luke! — https://use-the-index-luke.com/
- PostgreSQL Performance — https://www.postgresql.org/docs/current/performance-tips.html
- pgAdmin Documentation — https://www.pgadmin.org/docs/

---

## Entregables

| # | Entregable | Semana |
|---|------------|--------|
| 1 | Ejercicios de normalización (1FN → BCNF) | 13 |
| 2 | Reporte comparativo de rendimiento con/sin índices | 14 |
| 3 | Vistas y procedimientos almacenados | 15 |
| 4 | Funciones y triggers (incluye auditoría) | 16 |
| 5 | Demo de transacciones, concurrencia y seguridad | 17 |
| 6 | 10 consultas optimizadas con `EXPLAIN ANALYZE` | 18 |
| 7 | **Proyecto 3** (sistema completo) | 18 |
| 8 | **Parcial 3** | 18 |
