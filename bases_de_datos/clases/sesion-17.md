# Sesión 17 — Módulo 17 (Transacciones) + Módulo 18 (Concurrencia) + Módulo 19 (Seguridad)

**Semana:** 17
**Corte:** 3
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema Bancario "BancoSeguro"

---

## 1. Base de datos del sistema

```sql
CREATE DATABASE bancoseguro;
\c bancoseguro

CREATE TABLE cliente (
    id      SERIAL PRIMARY KEY,
    cedula  VARCHAR(20) NOT NULL UNIQUE,
    nombre  VARCHAR(120) NOT NULL
);

CREATE TABLE cuenta (
    id          SERIAL PRIMARY KEY,
    numero      VARCHAR(20) NOT NULL UNIQUE,
    cliente_id  INTEGER NOT NULL REFERENCES cliente(id),
    saldo       NUMERIC(15,2) NOT NULL DEFAULT 0 CHECK (saldo >= 0),
    tipo        VARCHAR(20) NOT NULL CHECK (tipo IN ('ahorros','corriente'))
);

CREATE TABLE transferencia (
    id              SERIAL PRIMARY KEY,
    cuenta_origen   INTEGER NOT NULL REFERENCES cuenta(id),
    cuenta_destino  INTEGER NOT NULL REFERENCES cuenta(id),
    monto           NUMERIC(15,2) NOT NULL CHECK (monto > 0),
    fecha           TIMESTAMP NOT NULL DEFAULT NOW(),
    descripcion     VARCHAR(200)
);

-- Datos
INSERT INTO cliente (cedula, nombre) VALUES
    ('1001','Ana Pérez'),
    ('1002','Beto Gómez');

INSERT INTO cuenta (numero, cliente_id, saldo, tipo) VALUES
    ('CTA-001', 1, 1000000, 'ahorros'),
    ('CTA-002', 1, 500000,  'corriente'),
    ('CTA-003', 2, 2000000, 'ahorros');
```

---

## MÓDULO 17: TRANSACCIONES

## 2. ¿Qué es una transacción?

Una transacción es una **unidad de trabajo** que agrupa varias operaciones SQL y se ejecuta de forma **atómica**: o se ejecutan todas, o ninguna.

### Propiedades ACID

| Propiedad | Significado |
|-----------|-------------|
| **A**tomicity | Todo o nada |
| **C**onsistency | La BD pasa de un estado válido a otro válido |
| **I**solation | Las transacciones concurrentes no se interfieren |
| **D**urability | Una vez confirmada, los cambios persisten |

---

## 3. BEGIN, COMMIT, ROLLBACK

```sql
-- Iniciar transacción
BEGIN;

UPDATE cuenta SET saldo = saldo - 100000 WHERE id = 1;
UPDATE cuenta SET saldo = saldo + 100000 WHERE id = 2;

-- Si todo está bien
COMMIT;

-- Si algo salió mal
ROLLBACK;
```

### Ejemplo real: transferencia bancaria

```sql
-- Transferir $200.000 de CTA-001 a CTA-002
BEGIN;

UPDATE cuenta SET saldo = saldo - 200000 WHERE numero = 'CTA-001';

-- Verificar saldo
DO $$
BEGIN
    IF (SELECT saldo FROM cuenta WHERE numero = 'CTA-001') < 0 THEN
        RAISE EXCEPTION 'Saldo insuficiente';
    END IF;
END $$;

UPDATE cuenta SET saldo = saldo + 200000 WHERE numero = 'CTA-002';
INSERT INTO transferencia (cuenta_origen, cuenta_destino, monto, descripcion)
VALUES (1, 2, 200000, 'Pago de servicios');

COMMIT;
```

## 4. SAVEPOINT

Permite hacer "rollback parcial" dentro de una transacción.

```sql
BEGIN;

UPDATE cuenta SET saldo = saldo - 100 WHERE id = 1;

SAVEPOINT sp1;

UPDATE cuenta SET saldo = saldo - 500 WHERE id = 1;
-- Si nos arrepentimos:
ROLLBACK TO sp1;

COMMIT;  -- Solo se aplica el primer UPDATE
```

---

## MÓDULO 18: CONCURRENCIA

## 5. Problemas de concurrencia

Sin control adecuado, dos transacciones pueden generar:

- **Lectura sucia (dirty read):** una transacción lee datos no confirmados de otra.
- **Lectura no repetible:** la misma consulta devuelve valores distintos dentro de una transacción.
- **Lectura fantasma (phantom):** nuevas filas aparecen en una segunda lectura.
- **Actualización perdida:** dos transacciones modifican la misma fila y se pierde una actualización.

---

## 6. Niveles de aislamiento en PostgreSQL

| Nivel | Dirty Read | Non-Repeatable | Phantom |
|-------|------------|----------------|---------|
| READ UNCOMMITTED | Posible | Posible | Posible |
| READ COMMITTED (default) | No | Posible | Posible |
| REPEATABLE READ | No | No | Posible |
| SERIALIZABLE | No | No | No |

```sql
-- Configurar nivel
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
-- ... operaciones ...
COMMIT;
```

---

## 7. Bloqueos (Locks)

PostgreSQL usa bloqueos automáticos. Explícitamente:

```sql
-- Bloquear fila específica
SELECT * FROM cuenta WHERE id = 1 FOR UPDATE;

-- Bloquear varias
SELECT * FROM cuenta WHERE id IN (1, 2) FOR UPDATE;
```

```sql
-- NOWAIT: no esperar, fallar si está bloqueado
SELECT * FROM cuenta WHERE id = 1 FOR UPDATE NOWAIT;
```

### Ejemplo de bloqueo intencional

```sql
BEGIN;
SELECT * FROM cuenta WHERE id = 1 FOR UPDATE;
-- Aquí otra transacción que intente FOR UPDATE sobre la fila esperará
UPDATE cuenta SET saldo = saldo - 100 WHERE id = 1;
COMMIT;
```

---

## 8. Deadlocks

Dos transacciones se bloquean mutuamente. PostgreSQL detecta y aborta una automáticamente con error `deadlock detected`.

```sql
-- Transacción A
BEGIN;
UPDATE cuenta SET saldo = saldo - 100 WHERE id = 1;
UPDATE cuenta SET saldo = saldo + 100 WHERE id = 2;
COMMIT;

-- Transacción B (concurrente)
BEGIN;
UPDATE cuenta SET saldo = saldo - 100 WHERE id = 2;  -- espera a A
UPDATE cuenta SET saldo = saldo + 100 WHERE id = 1;  -- deadlock con A
COMMIT;
```

### Cómo evitar deadlocks

- Adquirir bloqueos siempre en el mismo orden.
- Transacciones cortas.
- No mezclar lecturas largas con escrituras.

---

## 9. MVCC en PostgreSQL

**MVCC** (*Multi-Version Concurrency Control*) permite que cada transacción vea una "versión" consistente de los datos, sin bloquear lecturas.

- Los `UPDATE` no sobrescriben: crean una nueva versión de la fila.
- Los `DELETE` marcan la fila como "muerta" (no la borran físicamente).
- Un proceso en segundo plano (`VACUUM`) limpia las versiones antiguas.

```sql
-- Forzar limpieza
VACUUM;
VACUUM FULL;  -- más agresivo
ANALYZE;       -- actualiza estadísticas
```

---

## MÓDULO 19: SEGURIDAD

## 10. Usuarios y roles

```sql
-- Crear rol
CREATE ROLE vendedor NOLOGIN;
CREATE ROLE admin_bd LOGIN PASSWORD 'secreto123';

-- Crear usuario
CREATE USER recepcionista WITH PASSWORD 'pass456' VALID UNTIL '2026-12-31';
```

### Atributos del rol

```sql
ALTER ROLE vendedor LOGIN PASSWORD 'vend123';
ALTER ROLE admin_bd CREATEDB CREATEROLE;
```

### Listar roles

```sql
\du
SELECT * FROM pg_roles;
```

---

## 11. Permisos con GRANT

```sql
-- Otorgar permiso de lectura
GRANT SELECT ON cuenta TO vendedor;

-- Otorgar varios permisos
GRANT SELECT, INSERT, UPDATE ON pedido TO vendedor;

-- Otorgar todos los permisos
GRANT ALL PRIVILEGES ON cliente, cuenta TO admin_bd;

-- Otorgar sobre todas las tablas
GRANT SELECT ON ALL TABLES IN SCHEMA public TO vendedor;

-- Para tablas futuras
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO vendedor;
```

### Roles con membresía

```sql
CREATE ROLE ventas;
GRANT ventas TO vendedor;

-- O al usuario directamente
GRANT ventas TO recepcionista;
```

---

## 12. REVOKE

```sql
REVOKE INSERT ON pedido FROM vendedor;
REVOKE ALL PRIVILEGES ON cliente FROM admin_bd;
```

---

## 13. Ejemplo completo: roles del banco

```sql
-- Crear roles por tipo de usuario
CREATE ROLE cajero LOGIN PASSWORD 'cajero123';
CREATE ROLE auditor LOGIN PASSWORD 'auditor123';
CREATE ROLE administrador LOGIN PASSWORD 'admin123' CREATEDB;

-- Permisos
GRANT CONNECT ON DATABASE bancoseguro TO cajero, auditor, administrador;
GRANT USAGE ON SCHEMA public TO cajero, auditor, administrador;

-- Cajero: puede hacer transferencias
GRANT SELECT, UPDATE ON cuenta TO cajero;
GRANT SELECT, INSERT ON transferencia TO cajero;
GRANT SELECT ON cliente TO cajero;

-- Auditor: solo lectura
GRANT SELECT ON ALL TABLES IN SCHEMA public TO auditor;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO auditor;

-- Administrador: todo
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO administrador;
```

```sql
-- Verificar permisos
\dp cuenta
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'cuenta';
```

---

## 14. SCHEMAS y búsqueda

```sql
-- Crear esquema
CREATE SCHEMA contabilidad;

-- Mover tabla a un esquema
ALTER TABLE cuenta SET SCHEMA contabilidad;

-- Buscar en un esquema específico
SET search_path TO contabilidad, public;
```

---

## Ejercicios

### Transacciones
1. Escribe una transacción que transfiera $500.000 entre dos cuentas, validando saldo y registrando en `transferencia`.
2. Simula un error a mitad de la transacción y verifica que con `ROLLBACK` los cambios se deshagan.
3. Usa un `SAVEPOINT` para simular una transacción con "checkpoint" en una operación riesgosa.

### Concurrencia
4. En dos sesiones de `psql` (o dos conexiones de DBeaver), intenta el escenario de deadlock visto en clase. Captura el error.
5. Experimenta con `FOR UPDATE NOWAIT`.
6. Cambia el nivel de aislamiento a `SERIALIZABLE` y observa el comportamiento bajo concurrencia.

### Seguridad
7. Crea los roles `cajero`, `supervisor` y `auditor` con permisos diferenciados.
8. Crea un usuario `analista1` que solo pueda consultar la tabla `transferencia` y `cliente`.
9. Quita el permiso de `DELETE` a un rol.
10. Investiga: ¿qué hace `SET search_path` y por qué es importante en seguridad?

### Integrador
11. Crea un procedimiento `sp_realizar_transferencia(origen, destino, monto)` que use todo lo visto: transacciones, validaciones, registro de auditoría.
12. Implementa una política de seguridad donde solo el dueño pueda ver sus cuentas (con vistas y permisos).

---

## Recursos

- https://www.postgresql.org/docs/current/tutorial-transactions.html
- https://www.postgresql.org/docs/current/transaction-iso.html
- https://www.postgresql.org/docs/current/user-manag.html
- https://www.postgresql.org/docs/current/sql-grant.html

---

**Próxima sesión:** Módulo 20 — Optimización y cierre del Corte 3.
