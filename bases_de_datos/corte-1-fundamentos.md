# Corte 1 — Fundamentos y Modelado de Datos

**Módulos:** 1, 2, 3 y 4
**Duración aprox.:** Semanas 1 a 6
**Foco:** Conceptos básicos, DBMS, modelado ER, modelo relacional y álgebra relacional conceptual.

---

## Competencias a desarrollar

- Comprender qué es una base de datos, sus tipos y el rol de un SGBD.
- Diferenciar bases de datos relacionales y NoSQL, y sus casos de uso.
- Modelar problemas del mundo real mediante diagramas Entidad-Relación.
- Transformar un modelo ER a un modelo relacional.
- Interpretar operaciones del álgebra relacional como base teórica de SQL.

---

## Cronograma por semanas

### Semana 1 — Módulo 1: Introducción a las Bases de Datos
**Temas:**
- ¿Qué es una base de datos?
- Sistemas gestores de bases de datos (DBMS)
- Tipos de bases de datos (relacionales, documentales, clave-valor, grafos, columnares)
- SQL vs NoSQL
- Arquitectura Cliente-Servidor

**Práctica:**
- Instalación de PostgreSQL y MariaDB.
- Tour por DBeaver y pgAdmin.
- Conexión a un servidor local y ejecución de `SELECT version();`

**Tarea:** Documentar la instalación con capturas de pantalla y crear una conexión desde DBeaver.

---

### Semana 2 — Módulo 2 (parte 1): Modelado de Datos
**Temas:**
- Análisis de requerimientos
- Entidades
- Atributos
- Relaciones
- Cardinalidad (1:1, 1:N, N:M)

**Práctica:** Modelar un sistema de biblioteca identificando entidades, atributos y relaciones.

---

### Semana 3 — Módulo 2 (parte 2): Diagramas ER
**Temas:**
- Diagramas Entidad-Relación (notación Chen y notación pata de cuervo)
- Casos de estudio

**Práctica (taller en clase):**
- Diseñar una biblioteca
- Diseñar una tienda virtual
- Diseñar un sistema universitario (gestión de estudiantes, cursos, profesores, matrículas)

**Entregable:** Diagramas ER de los tres casos (herramienta: draw.io, dbdiagram.io o Lucidchart).

---

### Semana 4 — Módulo 3: Modelo Relacional
**Temas:**
- Tablas, filas, columnas
- Tipos de datos (enteros, texto, fecha/hora, booleanos, numéricos, serial, etc.)
- Claves primarias (PK)
- Claves foráneas (FK)
- Restricciones (NOT NULL, UNIQUE, CHECK, DEFAULT)
- Índices (introducción conceptual)

**Práctica:** Transformar el diagrama ER de la tienda virtual al modelo relacional (tablas con PK, FK y restricciones).

**Tarea:** Hacer lo mismo con el sistema universitario.

---

### Semana 5 — Módulo 4: Álgebra Relacional (Conceptual)
**Temas:**
- Selección (σ)
- Proyección (π)
- Unión (∪)
- Intersección (∩)
- Diferencia (−)
- Producto cartesiano (×)
- Join (⨝)

**Práctica:** Resolver expresiones de álgebra relacional sobre los esquemas de los proyectos anteriores y traducirlas mentalmente a SQL.

---

### Semana 6 — Cierre y evaluación del Corte 1
**Actividades:**
- Repaso general de los módulos 1 a 4.
- **Parcial 1 (teórico-práctico):** 40%
- Entrega del **Proyecto 1** (modelo ER + modelo relacional de un caso asignado). 30%
- Quiz sorpresa sobre SQL vs NoSQL y arquitectura cliente-servidor.
- Cierre y retroalimentación.

---

## Recursos del corte

- Documentación PostgreSQL — https://www.postgresql.org/docs/
- dbdiagram.io — https://dbdiagram.io/
- Elmasri & Navathe — Capítulos 1 a 7.

---

## Entregables

| # | Entregable | Semana |
|---|------------|--------|
| 1 | Reporte de instalación y conexión | 1 |
| 2 | Diagramas ER (3 casos) | 3 |
| 3 | Modelo relacional de la tienda virtual | 4 |
| 4 | Modelo relacional del sistema universitario | 4 |
| 5 | Ejercicios de álgebra relacional | 5 |
| 6 | **Proyecto 1:** Modelo completo de un caso | 6 |
| 7 | **Parcial 1** | 6 |
