# 📘 Dominio 1 — Características y Arquitectura de Snowflake AI Data Cloud

**Peso en el examen: 25% (el dominio más grande)**
**Días del plan: 1 y 2**

---

## 1.1 Características clave de Snowflake AI Data Cloud

### 🏛️ La arquitectura de 3 capas (¡MUY preguntado!)

Snowflake combina lo mejor de shared-disk y shared-nothing en una arquitectura **multi-cluster, shared-data** de 3 capas independientes:

```
┌─────────────────────────────────────────────────────┐
│       3. CLOUD SERVICES (Cerebro)                   │
│   Autenticación, metadata, optimizer, seguridad,    │
│   infraestructura, compilación, transacciones       │
├─────────────────────────────────────────────────────┤
│       2. QUERY PROCESSING / COMPUTE (Músculo)       │
│   Warehouses virtuales (clusters de MPP nodes)      │
│   Cada warehouse es INDEPENDIENTE del resto         │
├─────────────────────────────────────────────────────┤
│       1. DATABASE STORAGE (Almacén)                 │
│   Datos almacenados en formato columnar comprimido  │
│   en S3/Azure Blob/GCS (micropartitions)            │
└─────────────────────────────────────────────────────┘
```

**Punto CLAVE:** las 3 capas escalan de forma **independiente**. Puedes tener 10 warehouses corriendo contra los mismos datos sin conflictos.

### ⚡ Almacenamiento elástico y cómputo elástico

- **Storage elástico:** pagas solo por lo que almacenas (comprimido). Snowflake cobra por TB/mes.
- **Compute elástico:** warehouses que se pueden crear, redimensionar y pausar en segundos. Pagas por segundo de uso (mínimo 60 seg la primera vez que arranca).

### 🌩️ Partners de la nube (clouds soportadas)

Snowflake corre sobre 3 proveedores (NO es multi-cloud simultáneo, eliges uno por cuenta):
- **AWS** — S3 para storage
- **Microsoft Azure** — Blob Storage
- **Google Cloud (GCP)** — GCS

### 🏷️ Ediciones de Snowflake (¡memorízalas!)

| Edición | Features clave |
|---|---|
| **Standard** | Features básicos, 1 día de Time Travel, cifrado automático |
| **Enterprise** | Multi-cluster warehouses, hasta **90 días de Time Travel**, vistas materializadas, masking policies, search optimization |
| **Business Critical** | Todo lo anterior + HIPAA/PCI, **Tri-Secret Secure** (doble cifrado), failover/failback, private link |
| **Virtual Private Snowflake (VPS)** | Instancia aislada para los clientes más sensibles (gobierno, banca) |

> 🎯 **Pregunta típica:** "¿Qué edición mínima necesitas para vistas materializadas?" → **Enterprise**.

---

## 1.2 Herramientas e interfaces de usuario

### 🖥️ Snowsight
La **UI web moderna** de Snowflake. Incluye:
- Worksheets (editor SQL con autocompletado)
- Dashboards nativos
- Query Profile visual
- Marketplace integrado
- Gestión de usuarios, roles, warehouses

### 💻 SnowSQL
**Cliente de línea de comandos oficial**. Úsalo para:
- Scripts automatizados
- Comandos `PUT`/`GET` (estos NO funcionan en Snowsight)
- Ejecución batch

Se instala en tu máquina local (Windows/Mac/Linux).

### 🔌 Conectores vs Drivers

| Conectores | Drivers |
|---|---|
| Python, Spark, Kafka, Node.js, .NET, PHP | JDBC, ODBC |
| Lenguaje específico | Protocolos estándar (usados por BI tools como Tableau, Power BI) |

### 🐍 Snowpark
API para correr código en **Python, Java, Scala** directamente dentro de Snowflake (el cómputo se ejecuta en warehouses, no en tu máquina). Reemplaza muchos casos de Spark.

Incluye:
- **Snowpark DataFrames** — estilo Spark/pandas
- **UDFs y procedimientos almacenados en Python/Java/Scala**
- **Snowpark Container Services** (para workloads custom con GPUs, etc.)

### 🔍 SnowCD (Snowflake Connectivity Diagnostic Tool)
Herramienta para **diagnosticar problemas de red/conectividad** cuando no puedes conectarte a tu cuenta. Verifica DNS, firewalls, proxies.

---

## 1.3 Catálogo y objetos de Snowflake

### 🗂️ Jerarquía de contenedores

```
ORGANIZATION
    └── ACCOUNT (1 por región/cloud)
            └── DATABASE
                    └── SCHEMA
                            ├── TABLE, VIEW
                            ├── STAGE, STREAM, TASK, PIPE
                            ├── FUNCTION (UDF/UDTF)
                            ├── PROCEDURE
                            └── SEQUENCE
```

### 📊 Tipos de tablas (¡MUY preguntado!)

| Tipo | Persistencia | Time Travel | Fail-safe | Uso típico |
|---|---|---|---|---|
| **Permanent** | Permanente | 0–90 días (edición) | 7 días | Producción |
| **Transient** | Permanente | 0 o 1 día | **NO (0 días)** | Datos recreables, ahorro de costos |
| **Temporary** | Solo sesión | 0 o 1 día | NO | Staging intra-sesión |
| **External** | Apunta a stage externo | NO aplica | NO | Data lake, sin cargar |

> 🎯 **Trampa común:** "Transient" no significa "temporal". Una tabla transient **persiste entre sesiones**, pero no tiene Fail-safe.

### 👁️ Tipos de vistas

- **Regular views** — lógica SQL guardada, se ejecuta cada vez que se consulta.
- **Materialized views** — datos precomputados y almacenados. Solo en **Enterprise+**. Se actualizan automáticamente. No soportan JOINs complejos.
- **Secure views** — ocultan la definición de la vista a usuarios sin privilegios. No usan optimizaciones que podrían filtrar datos.

### 🏢 Tipos de esquemas

- **User-defined** — los que tú creas
- **INFORMATION_SCHEMA** — esquema de metadatos por BD (ANSI SQL, **1 hora a 7 días de retención**)
- **ACCOUNT_USAGE** — esquema global en la BD `SNOWFLAKE` (**hasta 1 año** de historia, **45 min–3 h de latencia**)

### 📦 Tipos de stages

| Stage | Sintaxis | Uso |
|---|---|---|
| **User stage** | `@~` | Personal de cada usuario, no se puede alterar |
| **Table stage** | `@%nombre_tabla` | Asociado a una tabla, no se puede alterar |
| **Named stage (internal)** | `@mi_stage` | Creado por usuario, se puede granular permisos |
| **Named stage (external)** | `@mi_stage_ext` (apunta a S3/Azure/GCS) | Apunta a tu bucket externo |

### 🧮 Tipos de datos clave

- **Numéricos:** `NUMBER`, `INT`, `FLOAT`, `DECIMAL`
- **String:** `VARCHAR`, `STRING`, `TEXT` (todos son alias, max ~16 MB)
- **Fechas:** `DATE`, `TIME`, `TIMESTAMP_NTZ/LTZ/TZ`
- **Semiestructurados:** `VARIANT` (hasta 16 MB), `OBJECT`, `ARRAY`
- **Geoespaciales:** `GEOGRAPHY`, `GEOMETRY`
- **Binarios:** `BINARY`, `VARBINARY`

### 🛠️ UDFs y UDTFs

- **UDF (User Defined Function):** retorna un valor escalar
- **UDTF (User Defined Table Function):** retorna una tabla
- Se pueden escribir en **SQL, JavaScript, Python, Java, Scala**

### 🔄 Streams, Tasks, Pipes (¡clave!)

- **Stream** → captura cambios (CDC) de una tabla
- **Task** → programa la ejecución de SQL (cron style)
- **Pipe** → ingesta continua con Snowpipe (auto-ingest desde S3/Azure)
- **Sequence** → generador de números únicos

### 🤝 Shares
Objetos que exponen datos (BDs, esquemas, tablas, vistas) a otras cuentas Snowflake **sin copiar datos**.

---

## 1.4 Conceptos de almacenamiento

### 🔬 Micropartitions (¡PREGUNTA FIJA!)

- Unidades contiguas de almacenamiento de **50 MB a 500 MB comprimidos** (≈16 MB descomprimidos).
- **Inmutables**: si actualizas una fila, Snowflake crea nueva micropartition.
- Almacenamiento **columnar** dentro de cada micropartition.
- Metadata por micropartition: min/max, distinct, null count → permite **pruning**.

### ✂️ Pruning (poda)

Cuando haces `WHERE fecha = '2025-01-01'`, Snowflake usa la metadata de cada micropartition para **saltarse las que no contienen esa fecha**. Menos micropartitions leídas = consulta más rápida = menos costo.

### 🧬 Clustering

- Snowflake asigna micropartitions automáticamente → **natural clustering**.
- Si tu tabla es muy grande y un query filtra siempre por una columna específica, puedes definir una **clustering key**:
  ```sql
  ALTER TABLE ventas CLUSTER BY (fecha);
  ```
- Snowflake tiene **Automatic Clustering** que reorganiza micropartitions en background (tiene costo).
- No es lo mismo que un índice — no hay estructura B-tree.

### 📈 Monitoreo del almacenamiento

Consultas útiles:
```sql
-- Uso de storage por tabla
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS;

-- Info de clustering
SELECT SYSTEM$CLUSTERING_INFORMATION('mi_tabla');

-- Storage total de la cuenta
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE;
```

---

## 🧪 Ejercicios prácticos (Día 1–2)

```sql
-- Ejercicio 1: Crea una BD y los 3 tipos de tabla
CREATE DATABASE snowpro_lab;
USE DATABASE snowpro_lab;
CREATE SCHEMA practica;

CREATE TABLE permanente (id INT, nombre STRING);
CREATE TRANSIENT TABLE transitoria (id INT, nombre STRING);
CREATE TEMPORARY TABLE temporal (id INT, nombre STRING);

SHOW TABLES;  -- Observa la columna "kind"

-- Ejercicio 2: Insertar datos y ver info de clustering
INSERT INTO permanente VALUES (1,'a'),(2,'b'),(3,'c');
SELECT SYSTEM$CLUSTERING_INFORMATION('permanente');

-- Ejercicio 3: Ver las 3 capas en acción
SELECT CURRENT_VERSION();        -- Services
SELECT CURRENT_WAREHOUSE();      -- Compute
SELECT CURRENT_DATABASE();       -- Storage context
```

---

## ❓ Preguntas de práctica del Dominio 1

**1.** ¿Cuál es el tamaño típico de una micropartition (comprimida)?
- A) 1–10 MB
- B) 50–500 MB ✅
- C) 1–5 GB
- D) Variable sin límite

**2.** ¿Qué edición de Snowflake se requiere para usar vistas materializadas?
- A) Standard
- B) Enterprise ✅
- C) Business Critical
- D) VPS

**3.** ¿Cuál de estos NO es parte de la capa de Cloud Services?
- A) Optimizer
- B) Metadata
- C) Ejecución de queries ✅
- D) Autenticación

**4.** Una tabla `TRANSIENT` tiene Fail-safe de:
- A) 7 días
- B) 1 día
- C) 0 días ✅
- D) Configurable

**5.** ¿Qué herramienta usas para diagnosticar problemas de conectividad de red con Snowflake?
- A) Snowsight
- B) SnowSQL
- C) SnowCD ✅
- D) Snowpark

**6.** En la jerarquía de contenedores de Snowflake, ¿qué está DEBAJO de una base de datos?
- A) Organización
- B) Cuenta
- C) Esquema ✅
- D) Tabla

**7.** ¿Qué tipo de stage es `@~`?
- A) Named stage
- B) User stage ✅
- C) Table stage
- D) External stage

---

## 🎯 Checklist antes de pasar al Día 3

- [ ] Puedo dibujar las 3 capas de Snowflake
- [ ] Sé las 4 ediciones y qué feature distingue a cada una
- [ ] Diferencio permanent / transient / temporary / external tables
- [ ] Conozco los 4 tipos de stage y su sintaxis
- [ ] Entiendo qué es una micropartition y qué es pruning
- [ ] Sé la diferencia entre INFORMATION_SCHEMA y ACCOUNT_USAGE

Si tienes algún ❌, **vuelve atrás** antes de avanzar. 👊
