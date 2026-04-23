# 📦 Dominio 4 — Carga y Descarga de Datos

**Peso en el examen: 10%**
**Día del plan: 6**

---

## 4.1 Conceptos y mejores prácticas para CARGAR datos

### 🗂️ Stages: el puente entre tu data y Snowflake

Un **stage** es una ubicación donde residen los archivos antes de ser cargados (o después de ser descargados). Pueden ser internos (dentro de Snowflake) o externos (S3/Azure/GCS).

| Tipo | Sintaxis | Alcance | Persistencia |
|---|---|---|---|
| **User stage** | `@~` | 1 por usuario | Permanente |
| **Table stage** | `@%mi_tabla` | 1 por tabla | Atado a la tabla |
| **Named internal stage** | `@mi_stage` | Reutilizable, con permisos | Objeto de BD |
| **Named external stage** | `@mi_stage_ext` (a S3/Azure/GCS) | Reutilizable, apunta afuera | Objeto de BD |

### 📏 Tamaño y formato de archivos

**Formatos soportados para carga:**
- **Estructurados:** CSV, TSV
- **Semiestructurados:** JSON, Avro, ORC, Parquet, XML

**Tamaño recomendado** (¡MUY preguntado!):
- **100–250 MB comprimidos por archivo**.
- Archivos más pequeños → muchos archivos → overhead de metadata.
- Archivos muy grandes → paralelismo limitado.

### 🗃️ Estructura de carpetas

Organiza tus stages para aprovechar el paralelismo de COPY:
```
@mi_stage/
  ├── 2025/
  │   ├── 01/
  │   │   ├── data_001.csv.gz
  │   │   ├── data_002.csv.gz
  └── ...
```

Y puedes cargar subsets con patrones:
```sql
COPY INTO mi_tabla FROM @mi_stage/2025/01/
  PATTERN = '.*data_.*\\.csv\\.gz';
```

### 🧪 Ad-hoc vs batch vs continuous

| Método | Latencia | Uso |
|---|---|---|
| **Ad-hoc / batch (COPY INTO)** | Minutos-horas | Cargas programadas, ETL nocturno |
| **Snowpipe** | Segundos-minutos | Casi real-time, archivos llegando constantemente |
| **Snowpipe Streaming** | Segundos | Streaming directo de filas (Kafka connector) |

### 🚰 Snowpipe

Ingesta continua automática:
- Usa **Cloud Services + serverless compute** (NO warehouse del usuario).
- Se factura por archivos procesados + cómputo.
- Dos modos de trigger:
  - **Auto-ingest:** notificaciones del bucket (SQS/Event Grid/Pub-Sub)
  - **REST API:** tu app llama al endpoint cuando llega archivo

```sql
CREATE PIPE mi_pipe
  AUTO_INGEST = TRUE
  AS
  COPY INTO mi_tabla
  FROM @mi_stage_externo
  FILE_FORMAT = (FORMAT_NAME = 'mi_csv');
```

### ✅ Mejores prácticas de carga
1. Archivos **100–250 MB comprimidos**
2. Usar compresión **gzip, bzip2, zstd** (Snowflake descomprime al cargar)
3. **FILE FORMAT reutilizable** (no repetirlo en cada COPY)
4. Usar `VALIDATION_MODE` para detectar errores antes de la carga real
5. Para JSON/Parquet usa un tamaño algo más grande (hasta 1 GB OK)

---

## 4.2 Comandos para cargar datos

### 🏗️ CREATE STAGE

```sql
-- Interno
CREATE STAGE mi_stage_int
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1);

-- Externo en S3
CREATE STAGE mi_stage_s3
  URL = 's3://mi-bucket/datos/'
  STORAGE_INTEGRATION = mi_integracion;
```

### 📐 CREATE FILE FORMAT

Objeto reutilizable con las opciones de parseo:
```sql
CREATE FILE FORMAT mi_csv
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  NULL_IF = ('NULL', '')
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  DATE_FORMAT = 'YYYY-MM-DD';

CREATE FILE FORMAT mi_json
  TYPE = JSON
  STRIP_OUTER_ARRAY = TRUE;
```

### 📤 PUT (de local → stage interno)

**Solo funciona en SnowSQL** (no Snowsight). Sube archivos de tu máquina al stage:
```bash
snowsql> PUT file:///path/datos.csv @mi_stage AUTO_COMPRESS=TRUE;
```
- `AUTO_COMPRESS = TRUE` (default) → comprime con gzip antes de subir.
- `OVERWRITE = TRUE` si el archivo ya existe.

### 📋 COPY INTO `<tabla>` (de stage → tabla)

```sql
COPY INTO ventas
  FROM @mi_stage/ventas.csv
  FILE_FORMAT = (FORMAT_NAME = 'mi_csv')
  ON_ERROR = 'CONTINUE';
```

**Opciones clave de ON_ERROR:**
- `CONTINUE` — ignora archivos con errores, continúa con el resto
- `SKIP_FILE` — salta el archivo con error
- `SKIP_FILE_N` — salta si hay N errores
- `ABORT_STATEMENT` (default) — aborta todo si hay cualquier error

**Load metadata:** Snowflake recuerda qué archivos cargó en cada tabla por **64 días**. Si corres COPY de nuevo, **NO vuelve a cargar los mismos archivos** (a menos que uses `FORCE = TRUE`).

### 📝 INSERT / INSERT OVERWRITE

```sql
INSERT INTO tabla VALUES (1,'a'), (2,'b');
INSERT INTO tabla SELECT * FROM otra_tabla;

-- INSERT OVERWRITE → trunca la tabla y luego inserta
INSERT OVERWRITE INTO tabla SELECT * FROM otra_tabla;
```

### 🌳 CREATE EXTERNAL TABLE

Consulta archivos en un stage externo **sin cargarlos** a una tabla Snowflake:
```sql
CREATE EXTERNAL TABLE ventas_ext
  LOCATION = @mi_stage_s3
  FILE_FORMAT = (TYPE = PARQUET)
  AUTO_REFRESH = TRUE;

SELECT VALUE:nombre::STRING FROM ventas_ext;
```

Los datos quedan en S3/Azure/GCS, Snowflake solo lee. Más lento pero útil para data lakes.

### ✓ VALIDATE

Función tabular que muestra los errores de un COPY previo:
```sql
-- Después de un COPY con errores
SELECT * FROM TABLE(VALIDATE(ventas, JOB_ID => '_last'));
```

O con `VALIDATION_MODE` en COPY, hace un "dry-run":
```sql
COPY INTO ventas FROM @mi_stage
  VALIDATION_MODE = 'RETURN_ERRORS';
```

---

## 4.3 Conceptos para DESCARGAR datos

### 📤 Tamaño y formatos de archivo

**Formatos soportados para descarga:** CSV, JSON, Parquet.

Por defecto Snowflake **divide la salida en múltiples archivos** (para paralelismo). Cada archivo ~16 MB comprimidos.

### 📦 Métodos de compresión

Snowflake comprime al descargar (default: **gzip**):
- CSV/JSON: gzip, bzip2, brotli, zstd, deflate, raw_deflate, none
- Parquet: snappy, lzo, gzip (internos al formato)

### ⚠️ Strings vacíos vs NULL

Por defecto al descargar CSV:
- NULL → cadena vacía
- Cadena vacía → `""` con quotes

Puedes cambiarlo con:
```sql
NULL_IF = ('NULL')           -- cómo tratar nulls
EMPTY_FIELD_AS_NULL = FALSE
```

### 📄 Descarga a archivo único
```sql
COPY INTO @mi_stage/export.csv FROM mi_tabla
  FILE_FORMAT = (TYPE = CSV)
  SINGLE = TRUE              -- fuerza un solo archivo
  MAX_FILE_SIZE = 5368709120; -- máx 5 GB
```

### 🔗 Descarga de tablas relacionales
- Snowflake descarga **filas, no la estructura** de JOIN.
- Si necesitas JOIN antes, haz un CTAS (CREATE TABLE AS SELECT) o descarga el resultado de un SELECT:
  ```sql
  COPY INTO @mi_stage/join_result.csv
    FROM (SELECT a.*, b.nombre FROM a JOIN b ON a.id = b.id);
  ```

---

## 4.4 Comandos para descargar datos

### 🎯 COPY INTO `<location>`

```sql
COPY INTO @mi_stage/export/
  FROM ventas
  FILE_FORMAT = (TYPE = CSV COMPRESSION = 'GZIP')
  HEADER = TRUE
  OVERWRITE = TRUE;
```

### 📋 LIST

Lista los archivos en un stage:
```sql
LIST @mi_stage;
LIST @mi_stage/2025/;
LIST @~;     -- user stage
LIST @%ventas;  -- table stage
```

### 📥 GET (de stage → local, solo SnowSQL)

```bash
snowsql> GET @mi_stage/export.csv.gz file:///home/user/descargas/;
```

### 🏗️ CREATE STAGE y CREATE FILE FORMAT
Igual que en carga — se usan para ambos flujos.

---

## 🧪 Ejercicios prácticos (Día 6)

```sql
-- 1. Crear un stage interno y un file format
CREATE DATABASE IF NOT EXISTS carga_lab;
USE DATABASE carga_lab;
CREATE SCHEMA ejercicios;

CREATE FILE FORMAT ff_csv TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY='"';
CREATE STAGE stage_interno FILE_FORMAT = ff_csv;

-- 2. Crear tabla destino
CREATE TABLE clientes (id INT, nombre STRING, email STRING, pais STRING);

-- 3. Desde SnowSQL (local):
--    snowsql> PUT file:///tmp/clientes.csv @stage_interno AUTO_COMPRESS=TRUE;

-- 4. Ver archivos en el stage
LIST @stage_interno;

-- 5. Cargar
COPY INTO clientes FROM @stage_interno
  FILE_FORMAT = (FORMAT_NAME='ff_csv')
  ON_ERROR = 'CONTINUE';

-- 6. Ver estado de la carga
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME=>'CLIENTES',
  START_TIME=> DATEADD(hour,-1,CURRENT_TIMESTAMP)));

-- 7. Descargar filtrado
COPY INTO @stage_interno/export/clientes_mx.csv
  FROM (SELECT * FROM clientes WHERE pais='MX')
  FILE_FORMAT = (FORMAT_NAME='ff_csv' COMPRESSION='GZIP')
  HEADER = TRUE
  OVERWRITE = TRUE
  SINGLE = TRUE;

-- 8. Validar un COPY con errores simulados
COPY INTO clientes FROM @stage_interno
  VALIDATION_MODE = 'RETURN_ERRORS';

-- 9. External table (si tienes bucket S3)
-- CREATE EXTERNAL TABLE clientes_ext
--   LOCATION = @stage_s3 FILE_FORMAT = ff_csv AUTO_REFRESH = TRUE;

-- 10. Crear un PIPE (Snowpipe)
CREATE PIPE mi_pipe AS
  COPY INTO clientes FROM @stage_interno FILE_FORMAT = (FORMAT_NAME='ff_csv');
SHOW PIPES;
```

---

## ❓ Preguntas de práctica del Dominio 4

**1.** ¿Cuál es el tamaño recomendado de archivo comprimido para carga óptima?
- A) 10–50 MB
- B) 100–250 MB ✅
- C) 500 MB – 1 GB
- D) 1–5 GB

**2.** El comando PUT se usa para:
- A) Descargar un archivo desde un stage
- B) Subir un archivo local a un stage interno ✅
- C) Conceder permisos sobre un stage
- D) Poner datos en una tabla externa

**3.** ¿Cuántos días recuerda Snowflake qué archivos ya cargó (para evitar duplicados)?
- A) 7
- B) 30
- C) 64 ✅
- D) 365

**4.** ¿Cuál es la diferencia entre Snowpipe y COPY INTO tradicional?
- A) Snowpipe solo carga Parquet
- B) Snowpipe usa serverless compute y se enfoca en ingesta continua ✅
- C) COPY INTO es más rápido
- D) Snowpipe es solo para descarga

**5.** ¿Qué valor de ON_ERROR aborta la carga si hay un solo error?
- A) CONTINUE
- B) SKIP_FILE
- C) ABORT_STATEMENT ✅ (default)
- D) SKIP_FILE_N

**6.** Una external table:
- A) Copia los datos al storage de Snowflake
- B) Lee los archivos desde el stage externo sin copiarlos ✅
- C) Solo soporta CSV
- D) No soporta AUTO_REFRESH

**7.** El comando GET:
- A) Solo funciona en Snowsight
- B) Solo funciona en SnowSQL (cliente CLI) ✅
- C) Carga datos a una tabla
- D) Crea un stage

**8.** Para descargar una tabla a UN SOLO archivo CSV:
- A) `COPY INTO @stage FROM tabla ONE_FILE=TRUE`
- B) `COPY INTO @stage FROM tabla SINGLE=TRUE` ✅
- C) `COPY INTO @stage FROM tabla MAX_FILES=1`
- D) No se puede

---

## 🎯 Checklist antes de pasar al Día 7

- [ ] Conozco los 4 tipos de stage y su sintaxis
- [ ] Sé el tamaño recomendado de archivo (100–250 MB)
- [ ] Diferencio Snowpipe vs COPY INTO
- [ ] Puedo hacer un flujo completo PUT → COPY → GET
- [ ] Entiendo las opciones de ON_ERROR
- [ ] Sé qué es una external table y cuándo usarla
- [ ] Conozco los comandos para descarga y el uso de SINGLE=TRUE

¡Vamos a las transformaciones! 🔄
