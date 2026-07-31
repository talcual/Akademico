# Sesión 6 — Cierre y Evaluación del Corte 1

**Semana:** 6
**Corte:** 1
**Duración estimada:** 2 horas

---

## 1. Repaso general de los Módulos 1 a 4

### Mapa conceptual

```
Bases de Datos
├── Conceptos fundamentales
│   ├── BD vs DBMS
│   ├── Tipos de BD
│   └── SQL vs NoSQL
├── Modelado de Datos
│   ├── Análisis de requerimientos
│   ├── Entidades, Atributos, Relaciones
│   ├── Cardinalidad
│   └── Diagramas ER
├── Modelo Relacional
│   ├── Tablas, filas, columnas
│   ├── Tipos de datos
│   ├── Claves (PK, FK)
│   ├── Restricciones
│   └── Índices
└── Álgebra Relacional
    ├── σ (selección)
    ├── π (proyección)
    ├── ∪ ∩ − (conjuntistas)
    ├── × (producto)
    └── ⨝ (join)
```

---

## 2. Checklist de estudio

Marca lo que ya dominas:

- [ ] Explicar la diferencia entre BD y DBMS.
- [ ] Instalar PostgreSQL y conectarme desde DBeaver o `psql`.
- [ ] Identificar entidades, atributos y relaciones en un caso real.
- [ ] Dibujar un diagrama ER completo (Chen o pata de cuervo).
- [ ] Diferenciar cardinalidad 1:1, 1:N y N:M.
- [ ] Transformar un modelo ER a un modelo relacional.
- [ ] Crear tablas con tipos de datos apropiados.
- [ ] Definir PKs, FKs y restricciones (CHECK, UNIQUE, NOT NULL).
- [ ] Escribir expresiones de álgebra relacional para consultas.
- [ ] Traducir álgebra relacional a SQL básico.

---

## 3. Preguntas tipo parcial

### Pregunta 1 — Conceptual (10%)
Define con tus palabras:
a) Base de datos
b) DBMS
c) Entidad
d) Relación 1:N

### Pregunta 2 — SQL vs NoSQL (10%)
Para cada caso, justifica si usarías SQL o NoSQL:
a) Sistema de matrícula de una universidad
b) Almacenamiento de logs de un servidor web
c) Red social con grafos de amistades
d) Sistema de facturación electrónica

### Pregunta 3 — Modelado ER (20%)
Modela el siguiente caso en un diagrama ER:

> "Una empresa de transporte tiene buses, conductores y rutas. Cada bus tiene placa, modelo y capacidad. Cada conductor tiene cédula, nombre y licencia. Cada ruta tiene código, origen, destino y distancia. Un bus realiza varias rutas en diferentes horarios, y cada ruta es asignada a un conductor en una fecha específica."

### Pregunta 4 — Transformación a modelo relacional (20%)
Transforma el diagrama anterior a tablas SQL (DDL).

### Pregunta 5 — Tipos de datos y restricciones (15%)
Dada la siguiente tabla, identifica errores o mejoras:

```sql
CREATE TABLE empleado (
    id          INT,
    cedula      VARCHAR(20),
    nombre      CHAR(100),
    salario     DECIMAL,
    fecha_ing   DATE,
    departamento VARCHAR
);
```

### Pregunta 6 — Álgebra relacional (15%)
Escribe una expresión de álgebra relacional que obtenga: "Los nombres de los conductores que han realizado la ruta con código 'R10'".

Esquema: `CONDUCTOR(id, nombre)`, `BUS(placa, modelo, capacidad)`, `RUTA(codigo, origen, destino, distancia)`, `ASIGNACION(id, fecha, placa_bus, id_conductor, codigo_ruta)`.

### Pregunta 7 — SQL básico (10%)
Escribe el SQL equivalente a: "Obtener nombre y salario de los empleados con salario mayor a 2'000.000 del departamento 'Sistemas'".

---

## 4. Proyecto 1 — Especificación

### Tema
**Sistema a elegir por el estudiante** (previa aprobación del docente), ejemplos:
- Sistema de gestión de una biblioteca
- Sistema de inventario de una ferretería
- Sistema de reservas de un hotel
- Sistema de seguimiento de pacientes en una clínica

### Entregables
1. Documento de análisis de requerimientos (1-2 páginas).
2. Diagrama ER completo (notación Chen o pata de cuervo).
3. Modelo relacional (tablas con tipos de datos, PK, FK, restricciones).
4. Script `.sql` con todos los `CREATE TABLE`.
5. Captura de la ejecución exitosa del script en PostgreSQL.

### Criterios de evaluación (30% del corte)

| Criterio | Porcentaje |
|----------|------------|
| Análisis de requerimientos | 15% |
| Correctitud del diagrama ER | 25% |
| Transformación a relacional | 25% |
| Calidad del SQL (tipos, restricciones) | 25% |
| Presentación y documentación | 10% |

### Fecha de entrega
Al final de la sesión 6.

---

## 5. Recursos de refuerzo

- Repasa los videos oficiales de PostgreSQL: https://www.youtube.com/c/PostgreSQLOfficial
- Practica modelado en dbdiagram.io: https://dbdiagram.io/
- Repasa los ejemplos de cada sesión anterior.

---

**Próxima sesión:** Inicio del Corte 2 — Módulo 5: SQL Básico (DDL).
