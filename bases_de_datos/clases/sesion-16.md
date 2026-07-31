# Sesión 16 — Módulo 15 (Funciones) + Módulo 16 (Triggers)

**Semana:** 16
**Corte:** 3
**Duración estimada:** 2 horas
**Caso de estudio:** Tienda en línea "ShopMaster"

---

## 1. Base de datos del sistema

```sql
CREATE DATABASE shopmaster;
\c shopmaster

CREATE TABLE categoria (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(80) NOT NULL UNIQUE
);

CREATE TABLE producto (
    id          SERIAL PRIMARY KEY,
    sku         VARCHAR(20) NOT NULL UNIQUE,
    nombre      VARCHAR(200) NOT NULL,
    precio      NUMERIC(10,2) NOT NULL CHECK (precio > 0),
    stock       INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    categoria_id INTEGER REFERENCES categoria(id) ON DELETE SET NULL
);

CREATE TABLE cliente (
    id      SERIAL PRIMARY KEY,
    email   VARCHAR(120) NOT NULL UNIQUE,
    nombre  VARCHAR(120) NOT NULL
);

CREATE TABLE pedido (
    id          SERIAL PRIMARY KEY,
    cliente_id  INTEGER NOT NULL REFERENCES cliente(id) ON DELETE RESTRICT,
    fecha       TIMESTAMP NOT NULL DEFAULT NOW(),
    total       NUMERIC(12,2) NOT NULL DEFAULT 0,
    estado      VARCHAR(20) NOT NULL DEFAULT 'pendiente'
        CHECK (estado IN ('pendiente','pagado','enviado','entregado','cancelado'))
);

CREATE TABLE linea_pedido (
    pedido_id   INTEGER REFERENCES pedido(id) ON DELETE CASCADE,
    producto_id INTEGER REFERENCES producto(id) ON DELETE RESTRICT,
    cantidad    INTEGER NOT NULL CHECK (cantidad > 0),
    precio_unit NUMERIC(10,2) NOT NULL,
    PRIMARY KEY (pedido_id, producto_id)
);

-- Tabla para auditoría
CREATE TABLE auditoria (
    id          SERIAL PRIMARY KEY,
    tabla       VARCHAR(50) NOT NULL,
    operacion   VARCHAR(10) NOT NULL,
    usuario     VARCHAR(80) NOT NULL,
    fecha       TIMESTAMP NOT NULL DEFAULT NOW(),
    datos_antes JSONB,
    datos_despues JSONB
);
```

```sql
-- Datos
INSERT INTO categoria (nombre) VALUES
    ('Electrónica'),('Ropa'),('Hogar'),('Deportes');

INSERT INTO producto (sku, nombre, precio, stock, categoria_id) VALUES
    ('SKU-001','Laptop Lenovo',   2500000, 10, 1),
    ('SKU-002','Mouse Logitech',   80000,  50, 1),
    ('SKU-003','Camiseta Nike',   120000, 100, 2),
    ('SKU-004','Sofá 3 puestos', 1800000,  5, 3),
    ('SKU-005','Bicicleta GW',   1500000,  8, 4);

INSERT INTO cliente (email, nombre) VALUES
    ('juan@x.com', 'Juan Pérez'),
    ('maria@x.com','María López');
```

---

## MÓDULO 15: FUNCIONES

## 2. Función SQL (pura)

```sql
-- Función que retorna un escalar
CREATE OR REPLACE FUNCTION fn_stock_disponible(p_producto_id INT)
RETURNS INTEGER AS $$
    SELECT stock FROM producto WHERE id = p_producto_id;
$$ LANGUAGE SQL;

SELECT fn_stock_disponible(1);
```

## 3. Función PL/pgSQL con lógica

```sql
CREATE OR REPLACE FUNCTION fn_total_pedido(p_pedido_id INT)
RETURNS NUMERIC AS $$
DECLARE
    v_total NUMERIC(12,2);
BEGIN
    SELECT SUM(cantidad * precio_unit) INTO v_total
    FROM linea_pedido
    WHERE pedido_id = p_pedido_id;

    RETURN COALESCE(v_total, 0);
END;
$$ LANGUAGE plpgsql;

SELECT fn_total_pedido(1);
```

## 4. Función que retorna una tabla

```sql
-- Opción 1: RETURNS TABLE
CREATE OR REPLACE FUNCTION fn_productos_bajo_stock(p_minimo INT DEFAULT 10)
RETURNS TABLE (
    sku VARCHAR,
    nombre VARCHAR,
    stock INTEGER
) AS $$
BEGIN
    RETURN QUERY
    SELECT p.sku, p.nombre, p.stock
    FROM producto p
    WHERE p.stock < p_minimo
    ORDER BY p.stock ASC;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM fn_productos_bajo_stock(20);
```

```sql
-- Opción 2: SETOF
CREATE OR REPLACE FUNCTION fn_top_clientes(p_limite INT)
RETURNS SETOF cliente AS $$
    SELECT * FROM cliente ORDER BY id LIMIT p_limite;
$$ LANGUAGE SQL;

SELECT * FROM fn_top_clientes(5);
```

## 5. Funciones vs procedimientos

| Aspecto | Función | Procedimiento |
|---------|---------|--------------|
| Retorna valor | Sí, obligatorio | No |
| Se invoca con | `SELECT` | `CALL` |
| Puede hacer transacciones | Limitado | Sí |
| Uso típico | Cálculos, validaciones | Procesos batch |

---

## MÓDULO 16: TRIGGERS

## 6. ¿Qué es un trigger?

Un trigger es un **procedimiento que se ejecuta automáticamente** cuando ocurre un evento (`INSERT`, `UPDATE`, `DELETE`) en una tabla.

### Casos de uso

- Auditoría (registrar cambios).
- Validaciones complejas.
- Mantener campos calculados.
- Replicación de datos.

---

## 7. Anatomía de un trigger

```sql
-- 1. Crear la función del trigger
CREATE OR REPLACE FUNCTION trg_fn_auditoria()
RETURNS TRIGGER AS $$
BEGIN
    -- Lógica
    RETURN NEW;  -- o NULL para cancelar
END;
$$ LANGUAGE plpgsql;

-- 2. Crear el trigger
CREATE TRIGGER tr_auditoria_producto
AFTER INSERT OR UPDATE OR DELETE ON producto
FOR EACH ROW EXECUTE FUNCTION trg_fn_auditoria();
```

---

## 8. Tipos de trigger

| Cuándo | Nivel | Uso |
|--------|-------|-----|
| `BEFORE` | `FOR EACH ROW` | Validar/modificar antes de guardar |
| `AFTER` | `FOR EACH ROW` | Auditar, propagar a otras tablas |
| `INSTEAD OF` | `FOR EACH ROW` | Vistas actualizables |

---

## 9. Ejemplo 1: Auditoría de cambios de precio

```sql
CREATE OR REPLACE FUNCTION trg_fn_auditar_precio()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND OLD.precio <> NEW.precio THEN
        INSERT INTO auditoria (tabla, operacion, usuario, datos_antes, datos_despues)
        VALUES (
            'producto',
            'UPDATE',
            current_user,
            jsonb_build_object('id', OLD.id, 'precio', OLD.precio),
            jsonb_build_object('id', NEW.id, 'precio', NEW.precio)
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_auditar_precio
BEFORE UPDATE ON producto
FOR EACH ROW EXECUTE FUNCTION trg_fn_auditar_precio();
```

```sql
-- Probar
UPDATE producto SET precio = 2600000 WHERE id = 1;
SELECT * FROM auditoria;
```

---

## 10. Ejemplo 2: Mantener stock y total del pedido

```sql
-- Función: actualiza stock y total del pedido al insertar línea
CREATE OR REPLACE FUNCTION trg_fn_actualizar_stock()
RETURNS TRIGGER AS $$
BEGIN
    -- Restar stock
    UPDATE producto
    SET stock = stock - NEW.cantidad
    WHERE id = NEW.producto_id;

    -- Si no hay suficiente stock, lanzar excepción
    IF (SELECT stock FROM producto WHERE id = NEW.producto_id) < 0 THEN
        RAISE EXCEPTION 'Stock insuficiente para el producto %', NEW.producto_id;
    END IF;

    -- Recalcular total del pedido
    UPDATE pedido
    SET total = (SELECT fn_total_pedido(NEW.pedido_id))
    WHERE id = NEW.pedido_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_actualizar_stock
AFTER INSERT ON linea_pedido
FOR EACH ROW EXECUTE FUNCTION trg_fn_actualizar_stock();
```

```sql
-- Probar
INSERT INTO pedido (cliente_id) VALUES (1) RETURNING id;  -- devuelve id=1
INSERT INTO linea_pedido (pedido_id, producto_id, cantidad, precio_unit)
VALUES (1, 1, 2, 2500000);

SELECT * FROM pedido;  -- El total debería estar actualizado
SELECT stock FROM producto WHERE id = 1;  -- El stock debería haber bajado
```

---

## 11. Ejemplo 3: Auditoría de DELETE (borrado lógico)

```sql
CREATE OR REPLACE FUNCTION trg_fn_auditar_delete()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO auditoria (tabla, operacion, usuario, datos_antes)
    VALUES ('producto', 'DELETE', current_user,
            jsonb_build_object('id', OLD.id, 'sku', OLD.sku, 'nombre', OLD.nombre));
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_auditar_delete
AFTER DELETE ON producto
FOR EACH ROW EXECUTE FUNCTION trg_fn_auditar_delete();
```

---

## 12. Eliminar triggers y funciones

```sql
DROP TRIGGER IF EXISTS tr_auditar_precio ON producto;
DROP FUNCTION IF EXISTS trg_fn_auditar_precio();
```

---

## Ejercicios

### Funciones
1. Crea una función `fn_precio_con_iva(p_precio NUMERIC)` que devuelva el precio con el 19% de IVA.
2. Crea una función `fn_cliente_top()` que retorne el cliente con más pedidos.
3. Crea una función que retorne una tabla con los productos de una categoría específica.
4. Crea una función `fn_promedio_pedidos(p_cliente_id)` que retorne el promedio de gasto del cliente.

### Triggers
5. Crea un trigger que registre en `auditoria` cualquier cambio de stock en `producto`.
6. Crea un trigger `BEFORE INSERT` sobre `linea_pedido` que valide que la `cantidad` no supere el stock disponible; de lo contrario, lance una excepción.
7. Crea un trigger que al eliminar un cliente, en lugar de eliminar sus pedidos, los marque como 'cancelado' (soft delete con nueva columna).
8. Crea un trigger que al actualizar el `precio` de un producto, registre también el cambio en una tabla `historial_precios`.
9. Crea un trigger que al insertar un pedido, le asigne automáticamente la fecha actual si viene NULL.
10. Crea un trigger que después de actualizar `linea_pedido` también recalcule el total del pedido.

### Reflexión
11. ¿Por qué crees que los triggers pueden ser peligrosos si no se documentan bien?
12. ¿Cuándo usarías un trigger y cuándo una vista o un procedimiento?

---

## Recursos

- https://www.postgresql.org/docs/current/sql-createfunction.html
- https://www.postgresql.org/docs/current/sql-createtrigger.html
- https://www.postgresql.org/docs/current/plpgsql-trigger.html

---

**Próxima sesión:** Módulo 17 (Transacciones), Módulo 18 (Concurrencia) y Módulo 19 (Seguridad).
