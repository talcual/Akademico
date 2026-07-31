# Sesión 5 — Módulo 4: Álgebra Relacional (Conceptual)

**Semana:** 5
**Corte:** 1
**Duración estimada:** 2 horas
**Caso de estudio:** Sistema Universitario

---

## 1. ¿Qué es el álgebra relacional?

Es un **lenguaje formal** (no procedural) que describe operaciones sobre relaciones. Es la **base teórica de SQL**: cada consulta SQL puede expresarse en términos de operaciones del álgebra relacional.

---

## 2. Esquema de trabajo (Universidad)

```
ESTUDIANTE(id, nombre, email, programa_id)
PROGRAMA(id, nombre, facultad_id)
FACULTAD(id, nombre)
ASIGNATURA(id, nombre, creditos, profesor_id, programa_id)
PROFESOR(id, nombre, email)
MATRICULA(id, estudiante_id, asignatura_id, nota_final)
```

Datos de ejemplo:

**ESTUDIANTE**
| id | nombre | email | programa_id |
|----|--------|-------|-------------|
| 1 | Ana | ana@u.co | 1 |
| 2 | Beto | beto@u.co | 1 |
| 3 | Camila | camila@u.co | 2 |

**ASIGNATURA**
| id | nombre | creditos | profesor_id | programa_id |
|----|--------|----------|-------------|-------------|
| 10 | BD I | 3 | 100 | 1 |
| 11 | BD II | 3 | 100 | 1 |
| 12 | IA | 4 | 101 | 2 |

**MATRICULA**
| id | estudiante_id | asignatura_id | nota_final |
|----|---------------|---------------|------------|
| 1 | 1 | 10 | 4.5 |
| 2 | 1 | 11 | 3.8 |
| 3 | 2 | 10 | 2.5 |
| 4 | 3 | 12 | 4.8 |

---

## 3. Operaciones

### 3.1 Selección (σ)

Filtra filas que cumplen una condición. Equivale al `WHERE` de SQL.

**Símbolo:** σ<sub>condición</sub>(Tabla)

**Ejemplo:** Estudiantes del programa 1.

```
σ_programa_id=1(ESTUDIANTE)
```

**Equivalente SQL:**
```sql
SELECT * FROM estudiante WHERE programa_id = 1;
```

**Resultado:**
| id | nombre | email | programa_id |
|----|--------|-------|-------------|
| 1 | Ana | ana@u.co | 1 |
| 2 | Beto | beto@u.co | 1 |

---

### 3.2 Proyección (π)

Selecciona ciertas columnas. Equivale al `SELECT columna1, columna2` de SQL.

**Símbolo:** π<sub>atributos</sub>(Tabla)

**Ejemplo:** Solo nombre y email de los estudiantes.

```
π_nombre,email(ESTUDIANTE)
```

**Equivalente SQL:**
```sql
SELECT nombre, email FROM estudiante;
```

> Se pueden **combinar** selección y proyección:
> `π_nombre(σ_programa_id=1(ESTUDIANTE))`

---

### 3.3 Unión (∪)

Une dos relaciones **compatibles** (mismo esquema). Equivale a `UNION` en SQL.

```
π_nombre(ESTUDIANTE) ∪ π_nombre(PROFESOR)
```

Resultado: todos los nombres de personas (estudiantes o profesores).

```sql
SELECT nombre FROM estudiante
UNION
SELECT nombre FROM profesor;
```

---

### 3.4 Intersección (∩)

Devuelve las filas presentes en **ambas** relaciones.

```
A ∩ B
```

```sql
SELECT nombre FROM estudiante
INTERSECT
SELECT nombre FROM profesor;
```

---

### 3.5 Diferencia (−)

Devuelve filas de A que **no están** en B.

```
A − B
```

```sql
SELECT nombre FROM estudiante
EXCEPT
SELECT nombre FROM profesor;
```

---

### 3.6 Producto cartesiano (×)

Combina **cada fila** de A con **cada fila** de B. Si A tiene *m* filas y B tiene *n*, el resultado tiene *m·n* filas.

```
ESTUDIANTE × ASIGNATURA
```

```sql
SELECT * FROM estudiante, asignatura;
```

> Rara vez se usa solo; casi siempre va seguido de una selección (que es justamente un JOIN).

---

### 3.7 Join (⨝)

Combina dos tablas por una condición. Es la operación más importante en la práctica.

**Theta join:** `A ⨝_condición B`

**Natural join:** une por columnas con el mismo nombre.

**Equi-join:** cuando la condición es una igualdad (caso típico).

**Ejemplo:** Nombre del estudiante y nombre de la asignatura matriculada.

```
ESTUDIANTE ⨝ (ESTUDIANTE.id = MATRICULA.estudiante_id) MATRICULA
⨝ (MATRICULA.asignatura_id = ASIGNATURA.id) ASIGNATURA
```

```sql
SELECT e.nombre, a.nombre
FROM estudiante e
JOIN matricula m ON e.id = m.estudiante_id
JOIN asignatura a ON m.asignatura_id = a.id;
```

**Resultado:**
| e.nombre | a.nombre |
|----------|----------|
| Ana | BD I |
| Ana | BD II |
| Beto | BD I |
| Camila | IA |

---

## 4. Ejercicios resueltos

### Ejercicio 1
Nombres de los estudiantes matriculados en "BD I".

```
π_ESTUDIANTE.nombre ( σ_ASIGNATURA.nombre='BD I' (
  ESTUDIANTE ⨝ MATRICULA ⨝ ASIGNATURA
))
```

### Ejercicio 2
Asignaturas con más de 3 créditos del programa 1.

```
π_nombre,creditos ( σ_creditos>3 ∧ programa_id=1 (ASIGNATURA) )
```

### Ejercicio 3
Estudiantes que **no** están matriculados en ninguna asignatura.

```
π_ESTUDIANTE.nombre (ESTUDIANTE) −
π_ESTUDIANTE.nombre (ESTUDIANTE ⨝ MATRICULA)
```

---

## Ejercicios propuestos

### Ejercicio 1
Escribe en álgebra relacional: "Obtener el nombre y email de los profesores que dictan asignaturas de más de 3 créditos".

### Ejercicio 2
Escribe en álgebra relacional: "Obtener los nombres de los estudiantes que han visto asignaturas dictadas por el profesor con id=100".

### Ejercicio 3
Traduce las siguientes expresiones a SQL:
1. `π_nombre(σ_creditos=3(ASIGNATURA))`
2. `π_nombre(FACULTAD) − π_nombre(σ_programa_id=1(PROGRAMA) ⨝ FACULTAD)`

### Ejercicio 4
Dado el siguiente resultado de un JOIN entre `ESTUDIANTE` y `MATRICULA`, indica qué operación adicional aplicarías para quedarte solo con los estudiantes que tienen nota_final ≥ 4.0:

| e.nombre | nota_final |
|----------|------------|
| Ana | 4.5 |
| Ana | 3.8 |
| Beto | 2.5 |

### Ejercicio 5
Explica con un ejemplo por qué el producto cartesiano "puro" rara vez es útil, pero combinado con una selección (es decir, un join) es la operación más poderosa.

### Ejercicio 6
Dada la siguiente consulta SQL, tradúcela a álgebra relacional:

```sql
SELECT DISTINCT e.nombre
FROM estudiante e, matricula m, asignatura a
WHERE e.id = m.estudiante_id
  AND m.asignatura_id = a.id
  AND a.creditos >= 3
  AND m.nota_final >= 4.0;
```

---

## Recursos

- Elmasri & Navathe — Capítulo 6.
- https://es.wikipedia.org/wiki/%C3%81lgebra_relacional
- https://www.geeksforgeeks.org/relational-algebra-in-dbms/

---

**Próxima sesión:** Cierre y evaluación del Corte 1.
