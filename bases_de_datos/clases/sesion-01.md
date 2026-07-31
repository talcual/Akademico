# Sesión 1 — Módulo 1: Introducción a las Bases de Datos

**Semana:** 1
**Corte:** 1
**Duración estimada:** 2 horas

---

## 1. ¿Qué es una base de datos?

Una **base de datos (BD)** es una colección organizada de datos relacionados entre sí, que se almacena de forma persistente para ser consultada, actualizada y gestionada de manera eficiente.

**Analogía:** Una base de datos es como una biblioteca bien organizada — tiene estantes (tablas), libros (registros), fichas catalográficas (índices) y un bibliotecario (DBMS) que te ayuda a encontrar lo que buscas.

### Características deseables

- **Persistencia:** los datos sobreviven a la ejecución del programa.
- **Estructura:** los datos siguen un esquema definido.
- **Integridad:** los datos son correctos y consistentes.
- **Concurrencia:** múltiples usuarios pueden acceder simultáneamente.
- **Seguridad:** control de accesos y permisos.

### Ejemplo de la vida real

Un sistema de gestión de un hospital almacena: pacientes, médicos, citas, historiales clínicos, recetas, facturas. Todo eso conforma una base de datos.

---

## 2. Sistemas gestores de bases de datos (DBMS)

Un **DBMS** (*Database Management System*) es el software que permite crear, mantener y manipular una base de datos.

### Funciones del DBMS

| Función | Descripción |
|---------|-------------|
| Almacenamiento | Guardar los datos en disco |
| Recuperación | Consultar datos de forma eficiente |
| Seguridad | Controlar quién accede y a qué |
| Integridad | Hacer cumplir reglas sobre los datos |
| Concurrencia | Coordinar accesos simultáneos |
| Respaldo | Permitir copias de seguridad y recuperación |

### DBMS populares

- **PostgreSQL** — open source, robusto, ideal para aprender y producción.
- **MariaDB / MySQL** — open source, muy popular en aplicaciones web.
- **Oracle Database** — comercial, líder empresarial.
- **Microsoft SQL Server** — comercial, ecosistema Microsoft.
- **SQLite** — embebido, sin servidor, ideal para apps móviles.

---

## 3. Tipos de bases de datos

### Por modelo de datos

| Tipo | Ejemplo | Uso típico |
|------|---------|-----------|
| Relacional | PostgreSQL, MySQL | Datos estructurados, transacciones |
| Documental | MongoDB, CouchDB | JSON/BSON, contenido flexible |
| Clave-valor | Redis, DynamoDB | Caché, sesiones, carros de compra |
| Columnar | Cassandra, HBase | Big Data, analítica |
| Grafos | Neo4j | Redes sociales, recomendaciones |
| Vectorial | Pinecone, Milvus | Embeddings, IA |

---

## 4. SQL vs NoSQL

| Aspecto | SQL (Relacional) | NoSQL |
|---------|------------------|-------|
| Esquema | Rígido (definido antes) | Flexible o sin esquema |
| Escalabilidad | Vertical (más CPU/RAM) | Horizontal (más servidores) |
| Transacciones ACID | Nativas | A veces limitado |
| Consistencia | Fuerte | Eventual (a menudo) |
| Lenguaje | SQL | Varía (MongoDB Query, etc.) |
| Ejemplo | PostgreSQL | MongoDB |

### ¿Cuándo usar cada uno?

- **SQL:** cuando los datos son altamente estructurados y la consistencia es crítica (banca, ERP, facturación).
- **NoSQL:** cuando el volumen es masivo, el esquema varía o se necesita escala horizontal (redes sociales, IoT, logs).

> La realidad: muchas arquitecturas modernas usan **ambos** (persistencia políglota).

---

## 5. Arquitectura Cliente-Servidor

```
┌──────────┐     red (TCP/IP)     ┌──────────────┐
│ Cliente  │ ◀──────────────────▶ │ Servidor BD  │
│ (DBeaver)│    SQL por sockets   │ (PostgreSQL) │
└──────────┘                      └──────┬───────┘
                                          │
                                          ▼
                                  ┌──────────────┐
                                  │   Disco /    │
                                  │   Archivos   │
                                  └──────────────┘
```

- El **cliente** envía sentencias SQL a través de la red.
- El **servidor** las procesa, accede al almacenamiento y devuelve resultados.
- PostgreSQL escucha por defecto en el **puerto 5432**, MySQL/MariaDB en el **3306**.

### Conexión típica

```
Host:     localhost  (o IP del servidor)
Puerto:   5432
Usuario:  postgres
Contraseña: ********
Base de datos: nombre_bd
```

---

## 6. Instalación de PostgreSQL y MariaDB

### PostgreSQL (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### PostgreSQL (macOS con Homebrew)

```bash
brew install postgresql@16
brew services start postgresql@16
```

### MariaDB (Ubuntu/Debian)

```bash
sudo apt install mariadb-server
sudo systemctl start mariadb
sudo mysql_secure_installation
```

### Verificar instalación

```bash
# PostgreSQL
psql --version
sudo -u postgres psql -c "SELECT version();"

# MariaDB
mysql --version
sudo mysql -u root -e "SELECT VERSION();"
```

---

## 7. Herramientas de administración

### DBeaver
- Cliente universal (multi-DBMS), gratuito y open source.
- Conexión visual, autocompletado, exportación de datos.
- Descarga: https://dbeaver.io/

### pgAdmin
- Herramienta oficial para PostgreSQL.
- Interfaz web, ideal para administrar servidores remotos.
- Descarga: https://www.pgadmin.org/

### psql (CLI)
- Cliente de línea de comandos de PostgreSQL.
- Ideal para scripting y administración remota por SSH.

```bash
psql -h localhost -U postgres -d mibd
```

---

## Ejercicios

### Ejercicio 1 — Conceptual
Explica con tus propias palabras la diferencia entre una **base de datos** y un **DBMS**. Da un ejemplo de cada uno.

### Ejercicio 2 — Investigación
Busca en la documentación oficial:
1. ¿En qué año se creó PostgreSQL y por qué?
2. ¿Qué significa el nombre "PostgreSQL"?

### Ejercicio 3 — Práctico
1. Instala PostgreSQL en tu equipo.
2. Conéctate con `psql` y ejecuta: `SELECT version();`
3. Conéctate desde DBeaver a la misma instancia.
4. Toma una captura de cada conexión.

### Ejercicio 4 — Comparación
Elabora una tabla comparativa de 3 DBMS relacionales y 3 NoSQL, con al menos: nombre, tipo, licencia, caso de uso principal.

### Ejercicio 5 — Reflexión
Para cada uno de estos sistemas, indica si usarías SQL o NoSQL y justifica:
1. Sistema de facturación de una empresa
2. Almacenamiento de publicaciones de una red social
3. Registro de transacciones bancarias
4. Catálogo de productos de un e-commerce
5. Sistema de recomendaciones de películas

---

## Recursos

- https://www.postgresql.org/docs/current/tutorial-start.html
- https://mariadb.com/kb/en/getting-started/
- https://db-engines.com/en/ranking

---

**Próxima sesión:** Módulo 2 (parte 1) — Modelado de Datos.
