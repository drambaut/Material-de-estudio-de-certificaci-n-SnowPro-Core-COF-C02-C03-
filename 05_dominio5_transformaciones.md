# 🔄 Dominio 5 — Transformaciones de Datos

**Peso en el examen: 20%**
**Días del plan: 22–25 (12–15 de mayo)**

---

## 5.1 Trabajando con datos estándar

### 🎲 Funciones de estimación

Para cálculos aproximados (más rápidos que los exactos, útiles en BI):

| Función | Qué hace |
|---|---|
| `APPROX_COUNT_DISTINCT(col)` | Distintos aproximados (HyperLogLog) |
| `APPROX_TOP_K(col, k)` | Top-K aproximado |
| `APPROX_PERCENTILE(col, p)` | Percentil aproximado |
| `HLL(col)` | HyperLogLog directo, retorna estado que se puede mergear |

Ejemplo:
```sql
-- Exacto (lento en tablas grandes)
SELECT COUNT(DISTINCT user_id) FROM eventos;

-- Aproximado (rapidísimo, error <2%)
SELECT APPROX_COUNT_DISTINCT(user_id) FROM eventos;
```

### 🎰 Muestreo: SAMPLE / TABLESAMPLE

Son sinónimos. Dos métodos:

```sql
-- Basado en fracción (porcentaje)
SELECT * FROM ventas SAMPLE (10);           -- 10 % aleatorio
SELECT * FROM ventas SAMPLE BERNOULLI (10); -- Bernoulli: fila a fila
SELECT * FROM ventas SAMPLE SYSTEM (10);    -- System: bloques enteros (más rápido)

-- Tamaño fijo
SELECT * FROM ventas SAMPLE (100 ROWS);     -- Exactamente 100 filas
```

- **BERNOULLI / ROW:** más lento pero uniforme.
- **SYSTEM / BLOCK:** muy rápido, pero puede sesgar si los bloques tienen distribución no uniforme.

### 🧰 Tipos de funciones soportadas

| Tipo | Descripción | Ejemplo |
|---|---|---|
| **System functions** | Internas de Snowflake | `SYSTEM$CLUSTERING_INFORMATION()`, `CURRENT_VERSION()` |
| **Built-in (SQL)** | Estándar + extensiones Snowflake | `SUM`, `UPPER`, `DATEDIFF`, `LATERAL FLATTEN` |
| **UDF (User-Defined Functions)** | Las creas tú | Escalar |
| **UDTF (Table Functions)** | Retornan una tabla | `CREATE FUNCTION ... RETURNS TABLE` |
| **External Functions** | Llaman a APIs externas vía API Gateway | Enriquecer datos con servicios |

#### Ejemplos de UDF en SQL, JavaScript y Python:
```sql
-- SQL UDF
CREATE FUNCTION iva(monto FLOAT) RETURNS FLOAT
AS $$ monto * 0.19 $$;

-- JavaScript UDF
CREATE FUNCTION reverse_str(s STRING) RETURNS STRING
LANGUAGE JAVASCRIPT
AS $$ return S.split('').reverse().join(''); $$;

-- Python UDF
CREATE FUNCTION area_circulo(r FLOAT) RETURNS FLOAT
LANGUAGE PYTHON RUNTIME_VERSION='3.10'
HANDLER = 'calcula'
AS $$
import math
def calcula(r):
    return math.pi * r * r
$$;
```

### 📜 Procedimientos almacenados

Similar a UDFs pero pueden:
- Ejecutar múltiples queries
- Tener lógica de control de flujo (IF/WHILE/LOOP)
- Lenguajes: **SQL (Snowflake Scripting), JavaScript, Python, Java, Scala**

```sql
CREATE PROCEDURE truncar_y_recargar() RETURNS STRING
LANGUAGE SQL
AS $$
BEGIN
  TRUNCATE TABLE staging;
  INSERT INTO staging SELECT * FROM otra_tabla;
  RETURN 'OK';
END;
$$;

CALL truncar_y_recargar();
```

**UDF vs Procedure (clave para el examen):**

| UDF | Stored Procedure |
|---|---|
| Se usa en `SELECT col, mi_udf(x)` | Se llama con `CALL mi_proc()` |
| Retorna valor (escalar o tabla) | Retorna valor pero no se usa en SELECT |
| NO puede ejecutar DDL ni DML | SÍ puede ejecutar DDL y DML |
| Debe ser determinista (o marcarla) | Puede tener side effects |

### 🌊 Streams (Change Data Capture)

Un stream captura los **cambios** (INSERT / UPDATE / DELETE) en una tabla desde el último "consumption point".

**Tipos de stream:**

| Tipo | Captura |
|---|---|
| **Standard** | INSERTs + UPDATEs + DELETEs (net changes) |
| **Append-only** | Solo INSERTs (más barato) |
| **Insert-only** | Solo INSERTs (para external tables) |

```sql
CREATE STREAM mi_stream ON TABLE ventas;

-- Ver cambios
SELECT * FROM mi_stream;
-- Columnas extras del stream:
-- METADATA$ACTION, METADATA$ISUPDATE, METADATA$ROW_ID
```

Cuando consumes el stream con INSERT/CTAS/MERGE en una transacción, el stream avanza su "offset":
```sql
INSERT INTO histórico SELECT * FROM mi_stream;
-- Ahora el stream está "vacío" hasta que haya nuevos cambios
```

**Tiempo de retención del stream** = periodo de Time Travel de la tabla (por defecto 1 día; hasta 14 días si la tabla tiene `DATA_RETENTION_TIME_IN_DAYS=14`).

> ⚠️ Si no consumes el stream durante más tiempo que su retención → el stream queda **stale** (obsoleto) y debes recrearlo.

### ⏰ Tasks (tareas programadas)

```sql
CREATE TASK tarea_consolida
  WAREHOUSE = compute_wh
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('mi_stream')  -- Solo corre si hay datos nuevos
AS
  INSERT INTO histórico SELECT * FROM mi_stream;

-- Hay que ACTIVARLA explícitamente
ALTER TASK tarea_consolida RESUME;
```

**Schedule** puede ser:
- `'5 MINUTE'`, `'15 MINUTE'`, `'1 HOUR'`, etc.
- Expresiones cron: `USING CRON 0 9 * * MON America/Bogota`

**Tasks encadenadas** con `AFTER` forman un **DAG**:
```sql
CREATE TASK tarea_2 AFTER tarea_consolida AS ...;
```

**Serverless tasks:** Si no especificas `WAREHOUSE`, Snowflake usa compute serverless (más eficiente para tasks pequeñas y frecuentes). Se factura por segundo de ejecución.

### 🔁 Patrón CDC típico: Stream + Task

```
Tabla fuente → STREAM captura cambios → TASK consume el stream cada X min → Tabla consolidada
```

Este patrón es **pregunta casi fija** en el examen.

---

## 5.2 Trabajando con datos semiestructurados

### 📋 Formatos soportados
- **JSON** (el más común)
- **Avro**
- **ORC**
- **Parquet**
- **XML**

Todos se cargan en una columna de tipo **VARIANT** (o se "shred" a columnas estructuradas).

### 💾 La columna VARIANT

- Puede contener **cualquier tipo**: objeto, array, string, número, null.
- **Hasta 16 MB comprimidos por valor**.
- Snowflake **extrae columnas virtuales** de los VARIANT (automatic schema detection) y las almacena optimizadas → consultas rápidas sobre JSON sin parsear manualmente.

### 🔤 Notación para acceder a datos VARIANT

```sql
-- Supón: v = {"nombre":"Ana","dir":{"ciudad":"Bogotá"},"tags":["admin","vip"]}

SELECT
  v:nombre::STRING AS nombre,          -- ":nombre" extrae, "::STRING" castea
  v:dir.ciudad::STRING AS ciudad,
  v:tags[0]::STRING AS primer_tag,
  v:tags[1]::STRING AS segundo_tag
FROM tabla_variant;
```

**Sintaxis:**
- `:campo` → accede a campo de objeto
- `.campo` → campo anidado
- `[N]` → índice de array
- `::TIPO` → cast explícito (**muy recomendado**)

### 🧩 FLATTEN y LATERAL FLATTEN

Para "desanidar" arrays u objetos y expandir cada elemento como una fila:

```sql
-- Dado: {"pedido":1, "items":[{"sku":"A"},{"sku":"B"}]}

SELECT
  v:pedido::INT,
  f.value:sku::STRING
FROM pedidos,
LATERAL FLATTEN(input => v:items) f;
```

**Resultado:**
```
1, A
1, B
```

**LATERAL FLATTEN** se usa en la cláusula FROM, es una **table function**.

Argumentos útiles:
- `input` — el objeto/array a aplanar
- `path` — ruta dentro del input
- `outer => TRUE` — mantener filas aun si el array está vacío (estilo LEFT JOIN)
- `recursive => TRUE` — aplanar recursivamente

### 🛠️ Funciones para datos semiestructurados

| Categoría | Funciones clave |
|---|---|
| **Crear ARRAY** | `ARRAY_CONSTRUCT()`, `ARRAY_APPEND()`, `ARRAY_CAT()` |
| **Crear OBJECT** | `OBJECT_CONSTRUCT()`, `OBJECT_INSERT()`, `OBJECT_DELETE()` |
| **Extraer** | `GET()`, `GET_PATH()`, operadores `:` y `[]` |
| **Parsear / convertir** | `PARSE_JSON()`, `TO_JSON()`, `CHECK_JSON()`, `TO_VARIANT()` |
| **Predicados de tipo** | `IS_OBJECT()`, `IS_ARRAY()`, `IS_NULL_VALUE()`, `IS_INTEGER()`, `TYPEOF()` |

Ejemplos:
```sql
SELECT OBJECT_CONSTRUCT('a', 1, 'b', 2);    -- {"a":1,"b":2}
SELECT ARRAY_CONSTRUCT(1, 2, 3);             -- [1,2,3]
SELECT PARSE_JSON('{"x":1}'):x;              -- 1
SELECT IS_ARRAY(PARSE_JSON('[1,2]'));        -- TRUE
SELECT TYPEOF(PARSE_JSON('"hola"'));         -- VARCHAR
```

### 🏷️ Cargar JSON

```sql
CREATE FILE FORMAT ff_json TYPE = JSON STRIP_OUTER_ARRAY = TRUE;

CREATE TABLE eventos (raw VARIANT);

COPY INTO eventos FROM @mi_stage FILE_FORMAT = ff_json;

-- Consultar
SELECT raw:user_id::INT, raw:event::STRING FROM eventos;
```

---

## 5.3 Trabajando con datos no estructurados

### 📂 Directory tables

Tabla especial que expone metadata de archivos en un stage:

```sql
CREATE STAGE imagenes
  DIRECTORY = (ENABLE = TRUE, AUTO_REFRESH = TRUE);

-- Consultar metadata de archivos
SELECT * FROM DIRECTORY(@imagenes);
-- Columnas: relative_path, size, last_modified, file_url, md5, etag
```

Úsalo para:
- Procesamiento de imágenes, PDFs, audios
- Pipelines de IA con Cortex / Document AI
- Inventarios de archivos en data lakes

### 🔗 Tipos de URL de archivos (¡MUY preguntado!)

| URL | Duración | Uso |
|---|---|---|
| **Scoped URL** | **Temporal (hasta 24 h)** | Descarga vía web con permisos validados. `BUILD_SCOPED_FILE_URL(@stage, 'archivo.jpg')` |
| **File URL** | Permanente | Requiere permisos cada vez que se usa. `BUILD_STAGE_FILE_URL(@stage, 'archivo.jpg')` |
| **Pre-signed URL** | Temporal configurable | URL directa a S3/Azure sin auth de Snowflake. `GET_PRESIGNED_URL(@stage, 'archivo.jpg', 3600)` |

### 🧱 Funciones SQL de archivos

```sql
-- URL firmada válida 1 hora
SELECT GET_PRESIGNED_URL(@imagenes, 'gato.jpg', 3600);

-- URL scoped (usa permisos del rol actual)
SELECT BUILD_SCOPED_FILE_URL(@imagenes, 'gato.jpg');

-- URL permanente (pero requiere permisos cada vez)
SELECT BUILD_STAGE_FILE_URL(@imagenes, 'gato.jpg');
```

### 🧠 UDFs para datos no estructurados

Puedes crear UDFs en Python/Java/Scala que **reciben archivos** de un stage como input:

```sql
CREATE FUNCTION img_size(file STRING) RETURNS STRING
LANGUAGE PYTHON RUNTIME_VERSION='3.10'
HANDLER = 'main'
PACKAGES = ('pillow')
AS $$
from PIL import Image
def main(file_path):
    img = Image.open(file_path)
    return f"{img.width}x{img.height}"
$$;
```

**Usos típicos:**
- Procesamiento de imágenes, PDFs, audios
- Extracción de texto con ML
- Clasificación de documentos
- Aplicaciones con Document AI o Cortex AI

---

## 🧪 Ejercicios prácticos

### Día 22 — UDFs y procedimientos
```sql
-- UDF simple
CREATE OR REPLACE FUNCTION aplicar_iva(monto FLOAT) RETURNS FLOAT
AS $$ monto * 1.19 $$;

SELECT aplicar_iva(100);  -- 119

-- Stored procedure
CREATE OR REPLACE PROCEDURE limpiar_staging() RETURNS STRING LANGUAGE SQL
AS $$
BEGIN
  DELETE FROM staging WHERE fecha < DATEADD(day, -30, CURRENT_DATE);
  RETURN 'Limpieza OK';
END;
$$;
CALL limpiar_staging();

-- SAMPLE
SELECT * FROM snowflake_sample_data.tpch_sf1.customer SAMPLE (1);  -- 1 %
SELECT * FROM snowflake_sample_data.tpch_sf1.customer SAMPLE (100 ROWS);
```

### Día 23 — Streams y Tasks
```sql
-- 1. Tabla + stream
CREATE OR REPLACE TABLE ordenes (id INT, monto FLOAT, fecha TIMESTAMP);
CREATE OR REPLACE STREAM ordenes_stream ON TABLE ordenes;

-- 2. Insertar y ver el stream
INSERT INTO ordenes VALUES (1, 100, CURRENT_TIMESTAMP), (2, 200, CURRENT_TIMESTAMP);
SELECT * FROM ordenes_stream;  -- Ves 2 filas con METADATA$ACTION = INSERT

-- 3. Tabla destino
CREATE OR REPLACE TABLE ordenes_historico LIKE ordenes;

-- 4. Task que consume el stream
CREATE OR REPLACE TASK mi_task
  WAREHOUSE = compute_wh
  SCHEDULE = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('ordenes_stream')
AS
  INSERT INTO ordenes_historico SELECT id, monto, fecha FROM ordenes_stream;

-- 5. Activar
ALTER TASK mi_task RESUME;

-- 6. Ver estado
SHOW TASKS;
SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY()) ORDER BY scheduled_time DESC LIMIT 5;
```

### Día 24 — Semiestructurados (JSON)
```sql
-- Crear tabla con VARIANT
CREATE OR REPLACE TABLE eventos (raw VARIANT);

-- Insertar JSON
INSERT INTO eventos
SELECT PARSE_JSON('{"user":"ana","events":[{"type":"login","ts":"2025-01-01"},{"type":"click","ts":"2025-01-02"}]}');

-- Extraer
SELECT raw:user::STRING, raw:events[0].type::STRING FROM eventos;

-- Aplanar
SELECT raw:user::STRING AS user, f.value:type::STRING AS tipo, f.value:ts::DATE AS fecha
FROM eventos, LATERAL FLATTEN(input => raw:events) f;

-- Construir objetos
SELECT OBJECT_CONSTRUCT('nombre','x','edad',25);
SELECT ARRAY_CONSTRUCT(1,2,3,4);

-- Predicados de tipo
SELECT TYPEOF(PARSE_JSON('[1,2]'));  -- ARRAY
SELECT IS_OBJECT(PARSE_JSON('{"a":1}'));  -- TRUE
```

### Día 25 — No estructurados
```sql
-- Crear stage con directory
CREATE OR REPLACE STAGE imagenes DIRECTORY = (ENABLE = TRUE);

-- Simular: subir archivos (PUT desde SnowSQL)
-- snowsql> PUT file:///home/user/imagenes/*.jpg @imagenes;

-- Refrescar directory
ALTER STAGE imagenes REFRESH;

-- Consultar metadata
SELECT * FROM DIRECTORY(@imagenes);

-- Obtener URLs
SELECT relative_path,
       GET_PRESIGNED_URL(@imagenes, relative_path, 3600) AS url_1h,
       BUILD_SCOPED_FILE_URL(@imagenes, relative_path) AS scoped_url
FROM DIRECTORY(@imagenes);
```

---

## ❓ Preguntas de práctica del Dominio 5

**1.** ¿Qué función retorna un conteo aproximado de valores únicos con error <2%?
- A) `COUNT_UNIQUE`
- B) `APPROX_COUNT_DISTINCT` ✅
- C) `HLL_ESTIMATE`
- D) `DISTINCT_APPROX`

**2.** ¿Qué comando usarías para aplanar un array JSON y tener una fila por elemento?
- A) `EXPLODE`
- B) `UNPIVOT`
- C) `LATERAL FLATTEN` ✅
- D) `UNFOLD`

**3.** Un STREAM de tipo append-only captura:
- A) Solo INSERTs ✅
- B) INSERTs y UPDATEs
- C) INSERTs, UPDATEs y DELETEs
- D) Solo DELETEs

**4.** Una UDF, a diferencia de un stored procedure, NO puede:
- A) Retornar valores
- B) Escribirse en JavaScript
- C) Ejecutar DML (INSERT/UPDATE/DELETE) ✅
- D) Usarse dentro de un SELECT

**5.** El tamaño máximo de un valor VARIANT en Snowflake es:
- A) 1 MB
- B) 16 MB comprimidos ✅
- C) 100 MB
- D) Sin límite

**6.** ¿Qué tipo de URL debe usarse para exponer temporalmente un archivo a una aplicación web externa?
- A) File URL
- B) Stage URL
- C) Scoped URL ✅
- D) Account URL

**7.** Para que una tarea (TASK) se ejecute SOLO si hay cambios en un stream:
- A) `WHEN STREAM_ACTIVE(stream)`
- B) `WHEN SYSTEM$STREAM_HAS_DATA('stream')` ✅
- C) `IF HAS_CHANGES(stream)`
- D) Es automático, no se configura

**8.** ¿Qué comando sirve para extraer `{"a":{"b":5}}` y obtener `5`?
- A) `SELECT v.a.b`
- B) `SELECT v:a:b::INT`
- C) `SELECT v:a.b::INT` ✅
- D) `SELECT v->a->b`

**9.** ¿Qué método de SAMPLE es más rápido pero puede sesgar?
- A) BERNOULLI
- B) ROW
- C) SYSTEM / BLOCK ✅
- D) STRATIFIED

**10.** Una tarea SIN WAREHOUSE asignado usa:
- A) El warehouse del usuario
- B) Compute serverless de Snowflake ✅
- C) No se puede crear sin warehouse
- D) El warehouse por defecto de la BD

---

## 🎯 Checklist antes de pasar al Día 26

- [ ] Puedo escribir una UDF y un stored procedure en SQL y Python
- [ ] Sé la diferencia entre UDF y procedure (qué puede hacer cada uno)
- [ ] Puedo crear un patrón Stream + Task de CDC de memoria
- [ ] Domino la notación `:`, `.`, `[]`, `::` para VARIANT
- [ ] Sé usar LATERAL FLATTEN con sus argumentos
- [ ] Conozco los 3 tipos de URL (scoped / file / pre-signed) y cuándo usar cada uno
- [ ] Entiendo las directory tables

¡Último dominio! 💪
