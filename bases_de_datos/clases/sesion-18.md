# Sesión 18 — Módulo 20: Optimización y Cierre del Corte 3

**Semana:** 18
**Corte:** 3
**Duración estimada:** 2 horas

---

## 1. El arte de optimizar

Optimizar una base de datos no es solo "hacer consultas más rápidas". Es encontrar el equilibrio entre:

- **Velocidad de lectura** vs. **velocidad de escritura**.
- **Espacio en disco** vs. **rendimiento**.
- **Complejidad** vs. **mantenibilidad**.

Regla de oro: **mide primero, optimiza después**. Nunca optimices basándote en suposiciones.

---

## 2. Recap: EXPLAIN y EXPLAIN ANALYZE

```sql
-- Solo plan estimado
EXPLAIN SELECT ...;

-- Plan estimado + ejecución real (mide tiempos)
EXPLAIN ANALYZE SELECT ...;

-- Más detalle
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING) SELECT ...;
```

### Opciones útiles

| Opción | Significado |
|--------|-------------|
| `ANALYZE` | Ejecuta la consulta y mide tiempos reales |
| `BUFFERS` | Muestra uso de caché/disco |
| `COSTS` | Muestra costos estimados (default) |
| `TIMING` | Tiempo por nodo (default con ANALYZE) |
| `FORMAT JSON` | Salida en JSON para herramientas |

---

## 3. Lectura del plan de ejecución

### Operaciones comunes (de mejor a peor en general)

| Operación | Significado |
|-----------|-------------|
| `Index Only Scan` | Lee solo del índice, sin tocar la tabla |
| `Index Scan` | Usa índice para encontrar filas |
| `Bitmap Index Scan` | Usa índice para múltiples filas |
| `Seq Scan` | Lee toda la tabla (¡ojo!) |
| `Nested Loop` | Join por fuerza bruta |
| `Hash Join` | Join con tabla hash en memoria |
| `Merge Join` | Join con ordenamiento previo |

### Cuándo preocuparse

- `Seq Scan` sobre tablas grandes.
- `actual time` o `actual rows` muy distintos del estimado → falta `ANALYZE`.
- Operaciones repetitivas en bucles anidados.

---

## 4. Ejemplo paso a paso: optimización

### Tabla inicial

```sql
CREATE TABLE ventas (
    id          SERIAL PRIMARY KEY,
    cliente_id  INTEGER NOT NULL,
    fecha       DATE NOT NULL,
    monto       NUMERIC(10,2) NOT NULL,
    estado      VARCHAR(20) NOT NULL
);

-- Generar 1 millón de filas
INSERT INTO ventas (cliente_id, fecha, monto, estado)
SELECT
    (random() * 10000)::INT,
    CURRENT_DATE - (random() * 730)::INT,
    (random() * 1000000)::NUMERIC(10,2),
    CASE WHEN random() < 0.6 THEN 'pagada'
         WHEN random() < 0.4 THEN 'pendiente'
         ELSE 'cancelada' END
FROM generate_series(1, 1000000);
```

### Consulta lenta: total de ventas del último mes por cliente

```sql
EXPLAIN ANALYZE
SELECT cliente_id, SUM(monto)
FROM ventas
WHERE fecha >= CURRENT_DATE - INTERVAL '30 days'
  AND estado = 'pagada'
GROUP BY cliente_id;
```

Probable resultado: **Seq Scan** y un plan poco eficiente.

---

### Optimización 1: índice por fecha

```sql
CREATE INDEX idx_ventas_fecha ON ventas(fecha);
ANALYZE ventas;

EXPLAIN ANALYZE
SELECT cliente_id, SUM(monto)
FROM ventas
WHERE fecha >= CURRENT_DATE - INTERVAL '30 days'
  AND estado = 'pagada'
GROUP BY cliente_id;
```

Mejora parcial: ahora puede acotar el rango por fecha, pero aún revisa todas las filas del último mes.

---

### Optimización 2: índice compuesto

```sql
DROP INDEX idx_ventas_fecha;
CREATE INDEX idx_ventas_fecha_estado ON ventas(fecha, estado);
ANALYZE;

EXPLAIN ANALYZE
SELECT cliente_id, SUM(monto)
FROM ventas
WHERE fecha >= CURRENT_DATE - INTERVAL '30 days'
  AND estado = 'pagada'
GROUP BY cliente_id;
```

Mejor: el índice se usa para ambos filtros.

---

### Optimización 3: índice parcial

```sql
DROP INDEX idx_ventas_fecha_estado;
CREATE INDEX idx_ventas_pagadas_recientes ON ventas(cliente_id, monto)
WHERE estado = 'pagada' AND fecha >= CURRENT_DATE - INTERVAL '90 days';
ANALYZE;

EXPLAIN ANALYZE
SELECT cliente_id, SUM(monto)
FROM ventas
WHERE fecha >= CURRENT_DATE - INTERVAL '30 days'
  AND estado = 'pagada'
GROUP BY cliente_id;
```

> Un índice parcial es más pequeño y rápido si el filtro es muy selectivo.

---

## 5. Optimización de consultas

### Reglas de oro

1. **Selecciona solo lo necesario**: evita `SELECT *`.
2. **Filtra lo antes posible**: usa `WHERE` efectivo.
3. **Limita resultados**: usa `LIMIT` cuando aplique.
4. **Usa EXISTS en lugar de IN** para subconsultas grandes.
5. **Evita funciones sobre columnas indexadas** en el `WHERE` (rompen el índice).
6. **Cuidado con `OR`**: a veces conviene reescribir como `UNION`.

### Ejemplos de reescritura

```sql
-- ❌ Malo: función sobre columna indexada
SELECT * FROM cliente WHERE LOWER(email) = 'juan@x.com';

-- ✅ Bueno: índice sobre la expresión
CREATE INDEX idx_email_lower ON cliente(LOWER(email));
SELECT * FROM cliente WHERE LOWER(email) = 'juan@x.com';
```

```sql
-- ❌ A veces lento
SELECT * FROM pedido WHERE cliente_id = 5 OR fecha >= '2026-01-01';

-- ✅ A veces más rápido
SELECT * FROM pedido WHERE cliente_id = 5
UNION
SELECT * FROM pedido WHERE fecha >= '2026-01-01' AND cliente_id <> 5;
```

---

## 6. Estadísticas y ANALYZE

El optimizador de PostgreSQL usa estadísticas para decidir el plan. Si están desactualizadas, elige mal.

```sql
-- Actualizar estadísticas de una tabla
ANALYZE ventas;

-- Actualizar todas las tablas
ANALYZE;

-- Configurar auto-analyze (en postgresql.conf)
-- autovacuum_analyze_scale_factor = 0.05
-- autovacuum_naptime = 60s
```

---

## 7. Configuración del servidor

Parámetros clave en `postgresql.conf`:

| Parámetro | Efecto |
|-----------|--------|
| `shared_buffers` | Memoria para caché (25% de RAM recomendado) |
| `work_mem` | Memoria para ordenamientos y joins |
| `effective_cache_size` | Estimación de caché del sistema |
| `maintenance_work_mem` | Memoria para VACUUM, CREATE INDEX |
| `random_page_cost` | Costo de lectura aleatoria (1.1 para SSD) |

```sql
-- Ver configuración actual
SHOW shared_buffers;
SHOW work_mem;

-- Cambiar en la sesión
SET work_mem = '64MB';
```

---

## 8. VACUUM y mantenimiento

```sql
-- Vacuum regular
VACUUM ventas;

-- Vacuum full (bloquea la tabla, recupera espacio)
VACUUM FULL ventas;

-- Reindex
REINDEX INDEX idx_ventas_fecha;
REINDEX TABLE ventas;
```

> Configurar `autovacuum` siempre activo. En sistemas con muchas escrituras, ejecutarlo en horarios de bajo tráfico.

---

## 9. Buenas prácticas generales

1. **Diseña bien el esquema** desde el inicio.
2. **Normaliza** y luego desnormaliza selectivamente.
3. **Indexa lo necesario**, no de más.
4. **Escribe queries claras** (los nombres de alias importan).
5. **Usa parámetros preparados** para evitar SQL injection.
6. **Monitorea** con `pg_stat_statements`.
7. **Haz respaldos** regularmente y prueba restauraciones.
8. **Documenta** los triggers, procedimientos y políticas de seguridad.

---

## 10. Cierre del Corte 3

### Resumen de optimización

| Problema | Solución |
|----------|----------|
| Consulta lenta | EXPLAIN ANALYZE para diagnosticar |
| Seq Scan en tabla grande | Crear índice adecuado |
| Índice no se usa | Verificar selectividad, ANALYZE |
| Escrituras lentas | Reducir número de índices, revisar triggers |
| Disco lleno | VACUUM FULL, particionamiento |
| Memoria insuficiente | Ajustar shared_buffers y work_mem |

---

## Ejercicios finales

### Diagnóstico
1. Toma una tabla de al menos 1 millón de filas (puedes generarla con `generate_series`).
2. Escribe 3 consultas "lentas" de forma intencional (sin índices).
3. Para cada una, captura el plan con `EXPLAIN ANALYZE`.

### Optimización
4. Crea los índices apropiados para cada consulta.
5. Mide de nuevo con `EXPLAIN ANALYZE` y compara.
6. Documenta los resultados en una tabla: tiempo antes, tiempo después, plan usado.

### Reescritura
7. Toma las siguientes consultas y reescríbelas de forma más eficiente:

```sql
-- A
SELECT * FROM cliente WHERE LOWER(nombre) = 'juan';

-- B
SELECT * FROM pedido
WHERE fecha >= '2025-01-01' AND estado IN ('pagado','enviado');

-- C
SELECT c.* FROM cliente c
WHERE c.id IN (SELECT cliente_id FROM pedido WHERE monto > 1000000);
```

### Proyecto 3 — Especificación

#### Tema
Optimización del sistema CinePlus (sesión 9) o EduOnline (sesión 10), a elección.

#### Entregables
1. Script SQL completo del sistema (con datos de prueba).
2. Documento de análisis con:
   - 5 consultas optimizadas (antes/después con EXPLAIN ANALYZE).
   - Justificación de cada índice creado.
   - Al menos 2 vistas.
   - Al menos 2 procedimientos almacenados.
   - Al menos 1 trigger de auditoría.
   - Roles y permisos definidos.
3. README con instrucciones de instalación y uso.

#### Criterios de evaluación

| Criterio | Porcentaje |
|----------|------------|
| Funcionalidad | 25% |
| Optimización demostrada (EXPLAIN antes/después) | 25% |
| Vistas, procedimientos, triggers | 20% |
| Seguridad (roles y permisos) | 15% |
| Documentación | 15% |

---

## Recursos finales

- https://www.postgresql.org/docs/current/performance-tips.html
- https://use-the-index-luke.com/
- https://explain.dalibo.com/
- https://www.postgresql.org/docs/current/monitoring-stats.html
- https://www.pgconfig.org/ (configurador interactivo)

---

**Fin del plan de clases. ¡Éxitos en el semestre!**
