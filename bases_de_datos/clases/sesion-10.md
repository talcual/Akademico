# Sesión 10 — Módulo 8: Relaciones e Integridad Referencial

**Semana:** 10
**Corte:** 2
**Duración estimada:** 2 horas
**Caso de estudio:** Plataforma de Cursos "EduOnline"

---

## 1. Los tres tipos de relaciones

| Tipo | Notación | Implementación |
|------|----------|----------------|
| 1:1 | `||——||` | FK en cualquiera de las dos tablas con `UNIQUE` |
| 1:N | `||——o{` | FK en la tabla del lado "muchos" |
| N:M | `}o——o{` | Tabla puente con dos FKs |

---

## 2. Sistema de ejemplo: EduOnline

```sql
CREATE DATABASE eduonline;
\c eduonline

-- Un usuario puede ser un estudiante, un instructor, o ambos
CREATE TABLE usuario (
    id          SERIAL PRIMARY KEY,
    email       VARCHAR(120) NOT NULL UNIQUE,
    nombre      VARCHAR(120) NOT NULL,
    fecha_registro TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Datos extendidos del usuario (1:1 con usuario)
CREATE TABLE perfil_usuario (
    usuario_id  INTEGER PRIMARY KEY REFERENCES usuario(id) ON DELETE CASCADE,
    biografia   TEXT,
    pais        VARCHAR(60),
    foto_url    VARCHAR(255),
    sitio_web   VARCHAR(255)
);

CREATE TABLE categoria (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(80) NOT NULL UNIQUE
);

CREATE TABLE curso (
    id              SERIAL PRIMARY KEY,
    titulo          VARCHAR(200) NOT NULL,
    descripcion     TEXT,
    precio          NUMERIC(10,2) NOT NULL CHECK (precio >= 0),
    instructor_id   INTEGER NOT NULL REFERENCES usuario(id) ON DELETE RESTRICT,
    categoria_id    INTEGER NOT NULL REFERENCES categoria(id) ON DELETE RESTRICT,
    publicado       BOOLEAN NOT NULL DEFAULT FALSE,
    fecha_creacion  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- N:M: un estudiante puede inscribirse en muchos cursos
-- y un curso puede tener muchos estudiantes
CREATE TABLE inscripcion (
    id          SERIAL PRIMARY KEY,
    estudiante_id   INTEGER NOT NULL REFERENCES usuario(id) ON DELETE CASCADE,
    curso_id        INTEGER NOT NULL REFERENCES curso(id) ON DELETE CASCADE,
    fecha_inscripcion TIMESTAMP NOT NULL DEFAULT NOW(),
    progreso_porcentaje NUMERIC(5,2) DEFAULT 0 CHECK (progreso_porcentaje BETWEEN 0 AND 100),
    CONSTRAINT uq_inscripcion UNIQUE (estudiante_id, curso_id)
);

-- N:M: cursos y sus etiquetas
CREATE TABLE etiqueta (
    id      SERIAL PRIMARY KEY,
    nombre  VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE curso_etiqueta (
    curso_id     INTEGER REFERENCES curso(id) ON DELETE CASCADE,
    etiqueta_id  INTEGER REFERENCES etiqueta(id) ON DELETE CASCADE,
    PRIMARY KEY (curso_id, etiqueta_id)
);
```

---

## 3. Relación 1:1

Una fila de A se relaciona con **una y solo una** fila de B (y viceversa). Se implementa con una FK con `UNIQUE`.

```sql
-- Un usuario tiene un único perfil
perfil_usuario.usuario_id INTEGER PRIMARY KEY REFERENCES usuario(id)
```

> El truco: la FK es a la vez PK, garantizando que solo exista un perfil por usuario.

### Cuándo se usa

- Cuando una entidad fuerte tiene información opcional que solo aplica a algunos casos (perfil extendido, configuración).
- Para subdividir tablas muy grandes con campos específicos.

### Inserción

```sql
INSERT INTO usuario (email, nombre) VALUES ('juan@e.com', 'Juan Pérez');
INSERT INTO perfil_usuario (usuario_id, biografia, pais) VALUES (1, 'Apasionado por la tecnología', 'Colombia');
```

---

## 4. Relación 1:N

Una fila de A se relaciona con **muchas** filas de B, pero cada fila de B se relaciona con **una sola** de A.

```sql
-- Un instructor dicta muchos cursos
curso.instructor_id INTEGER REFERENCES usuario(id)
```

### Ejemplos típicos

- Cliente → Pedidos.
- Categoría → Productos.
- Departamento → Empleados.
- Autor → Libros.

### Inserción

```sql
INSERT INTO usuario (email, nombre) VALUES ('ana@e.com', 'Ana Profesora');
INSERT INTO categoria (nombre) VALUES ('Programación');
INSERT INTO curso (titulo, precio, instructor_id, categoria_id) VALUES ('SQL desde cero', 99000, 2, 1);
```

### Consultar 1:N

```sql
-- Cursos por instructor
SELECT u.nombre AS instructor, c.titulo
FROM usuario u
JOIN curso c ON c.instructor_id = u.id;
```

---

## 5. Relación N:M

Una fila de A se relaciona con muchas de B, y viceversa. Se implementa con una **tabla puente** con dos FKs.

```sql
CREATE TABLE inscripcion (
    estudiante_id INTEGER REFERENCES usuario(id),
    curso_id      INTEGER REFERENCES curso(id),
    PRIMARY KEY (estudiante_id, curso_id)
);
```

### Inserción

```sql
INSERT INTO inscripcion (estudiante_id, curso_id) VALUES (1, 1);
INSERT INTO inscripcion (estudiante_id, curso_id) VALUES (1, 2);
INSERT INTO inscripcion (estudiante_id, curso_id) VALUES (3, 1);
```

### Atributos de la relación

La tabla puente puede tener **atributos propios** (no solo las dos FKs):
- `fecha_inscripcion`
- `progreso_porcentaje`
- `calificacion`
- `fecha_completado`

---

## 6. Integridad referencial — ON DELETE y ON UPDATE

Acciones que se ejecutan cuando se elimina o actualiza la fila referenciada:

| Opción | Comportamiento |
|--------|----------------|
| `NO ACTION` | Por defecto. Error si hay filas referenciando. |
| `RESTRICT` | Igual que NO ACTION (error inmediato). |
| `CASCADE` | Propaga la operación a las filas dependientes. |
| `SET NULL` | Pone NULL en la FK. |
| `SET DEFAULT` | Pone el valor por defecto. |

### Ejemplos

```sql
-- Si se borra el curso, se borran sus inscripciones
curso_id INTEGER REFERENCES curso(id) ON DELETE CASCADE

-- Si se borra el instructor, no se puede (porque hay cursos)
instructor_id INTEGER REFERENCES usuario(id) ON DELETE RESTRICT

-- Si se borra la categoría, los cursos quedan con categoria_id NULL
categoria_id INTEGER REFERENCES categoria(id) ON DELETE SET NULL
```

### ¿Cuándo usar cada uno?

- **CASCADE** — cuando la fila hija no tiene sentido sin la padre (línea de pedido sin pedido).
- **RESTRICT** — cuando quieres proteger la integridad (no borrar instructor con cursos).
- **SET NULL** — cuando la relación es opcional.
- **NO ACTION** — por defecto, similar a RESTRICT pero se valida al final de la transacción.

---

## Ejercicios

### Modelado
1. Agrega a `EduOnline` una relación 1:1 entre `usuario` y `configuracion_usuario` (notificaciones_email BOOLEAN, tema VARCHAR).

2. Crea una tabla `leccion` con relación 1:N desde `curso`:
   - Campos: id, curso_id, titulo, contenido, duracion_min, orden.
   - Si se borra el curso, se borran sus lecciones.

3. Agrega una relación N:M entre `leccion` y `usuario` (progreso_porcentaje), llamada `progreso_leccion`. PK compuesta.

4. Modela una relación 1:1 entre `curso` y `certificado` (un curso aprobado genera un certificado único por estudiante). Crea la tabla `certificado` con campos: id, curso_id, estudiante_id, fecha_emision, codigo_unico, y restricciones apropiadas.

### Inserciones
5. Inserta 5 usuarios, 3 categorías, 4 cursos, 6 etiquetas.
6. Inscribe estudiantes en cursos (al menos 8 inscripciones).
7. Asocia etiquetas a cursos (mínimo 6 asociaciones).

### Consultas básicas
8. Lista los cursos con el nombre de su instructor.
9. Lista los cursos con su categoría.
10. ¿Cuántos estudiantes tiene cada curso?
11. ¿En cuántos cursos está inscrita cada persona?

### Integridad referencial
12. Intenta borrar un instructor que tiene cursos. ¿Qué pasa?
13. Borra un curso. ¿Qué pasa con sus inscripciones y lecciones?
14. Borra un usuario que solo era estudiante. ¿Qué pasa con sus inscripciones?

### Decisiones de diseño
15. Para cada FK del modelo, justifica qué `ON DELETE` usarías y por qué.
16. Investiga: ¿qué pasa con la opción `SET DEFAULT` si la columna FK no tiene `DEFAULT` definido?

---

## Recursos

- https://www.postgresql.org/docs/current/ddl-constraints.html
- https://www.postgresql.org/docs/current/sql-createtable.html

---

**Próxima sesión:** Módulo 9 — JOINs.
