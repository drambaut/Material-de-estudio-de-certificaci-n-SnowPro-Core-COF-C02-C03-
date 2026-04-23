# 🧠 Cheatsheet SnowPro Core — Repaso Final

**Repasa esto el día 22 de mayo (víspera del examen) y justo antes de entrar.**

---

## 🏛️ Arquitectura (25%)

### 3 Capas (independientes y escalan solas)
1. **Database Storage** → datos columnar comprimidos en S3/Azure/GCS
2. **Query Processing (Compute)** → warehouses virtuales
3. **Cloud Services** → auth, metadata, optimizer, seguridad

### Ediciones (de menor a mayor)
| Edición | Time Travel max | Feature clave |
|---|---|---|
| Standard | 1 día | Base |
| Enterprise | 90 días | MV, multi-cluster, masking, SOS |
| Business Critical | 90 días | HIPAA/PCI, Tri-Secret Secure, private link, DR |
| VPS | 90 días | Instancia aislada |

### Tipos de tabla
| Tipo | Persistencia | Time Travel | Fail-safe |
|---|---|---|---|
| Permanent | Permanente | 0–90 días | **7 días** |
| Transient | Permanente | 0 o 1 día | **0** ❌ |
| Temporary | Sesión | 0 o 1 día | **0** ❌ |
| External | Externa | — | — |

### Stages
- `@~` = user stage
- `@%tabla` = table stage
- `@mi_stage` = named internal
- `@mi_stage_ext` (URL=S3/Azure/GCS) = named external

### Micropartitions
- **50–500 MB comprimidos**
- Inmutables, columnar
- Metadata por µ-partition → **pruning**
- **Clustering automático** + opcional `CLUSTER BY (col)`

### INFORMATION_SCHEMA vs ACCOUNT_USAGE
| | INFO_SCHEMA | ACCOUNT_USAGE |
|---|---|---|
| Retención | 7 días max | 1 año |
| Latencia | Real-time | 45 min – 3 h |
| Alcance | Por BD | Global (BD `SNOWFLAKE`) |

---

## 🔐 Seguridad (20%)

### Jerarquía de roles (¡MEMORIZAR!)
```
        ORGADMIN
           │
       ACCOUNTADMIN
         /     \
 SECURITYADMIN  SYSADMIN
       │
  USERADMIN
       │
    PUBLIC
```

### Qué hace cada uno
- **ORGADMIN:** organización (crear cuentas)
- **ACCOUNTADMIN:** todo dentro de la cuenta
- **SECURITYADMIN:** manage grants globalmente
- **USERADMIN:** crear/gestionar users y roles
- **SYSADMIN:** dueño de BDs y warehouses (recomendado)
- **PUBLIC:** todos los users lo tienen

### Métodos de autenticación
- Password
- **MFA** (obligatorio para ACCOUNTADMIN)
- SSO / federated (SAML 2.0)
- **Key-pair** (ideal para scripts/CI/CD)
- OAuth
- SCIM (provisioning)

### Privilegios para SELECT
Para leer una tabla necesitas:
1. `USAGE` en la BD
2. `USAGE` en el esquema
3. `USAGE` en el warehouse
4. `SELECT` en la tabla

### Gobierno
- **Secure view / UDF** → ocultan definición
- **Masking policy** → enmascara columnas
- **Row access policy** → filtra filas por rol
- **Tags** → etiquetas para clasificación
- **Access history** → `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY`

---

## ⚡ Rendimiento (15%)

### 3 tipos de caché
| Caché | Dónde | TTL | Costo |
|---|---|---|---|
| **Metadata** | Cloud Services | Mientras exista | Gratis |
| **Result** | Cloud Services | **24 h** (ext. 31 d) | Gratis (sin WH) |
| **Warehouse (SSD)** | Compute layer | Mientras WH activo | Parte del WH |

⚠️ Suspender el WH **pierde** la warehouse cache.

### Scaling
- **Scaling UP** (cambiar tamaño XS→S→M→L): queries **complejas/lentas individuales**
- **Scaling OUT** (multi-cluster): **concurrencia** alta (muchos usuarios)

### Tamaños (cada uno dobla créditos/h)
XS=1 · S=2 · M=4 · L=8 · XL=16 · 2XL=32 · 3XL=64 · 4XL=128 · 5XL=256 · 6XL=512

### Multi-cluster warehouses (Enterprise+)
- `MIN_CLUSTER_COUNT` / `MAX_CLUSTER_COUNT`
- Políticas: **STANDARD** (prioriza arrancar cluster) / **ECONOMY** (prioriza ahorro)

### Warehouse params
- `AUTO_SUSPEND = 60` (seg min)
- `AUTO_RESUME = TRUE`
- Arranque primera vez: mínimo 60 seg facturado

### Query Profile — señales clave
- **Bytes spilled to REMOTE storage** → aumentar tamaño del WH ⬆️
- **Partitions scanned 95/100** → poco pruning → clustering key o filtros distintos
- **QUEUED_OVERLOAD_TIME** → falta cómputo → multi-cluster o WH más grande

### Optimización
- **Vistas materializadas** (Enterprise+): precomputadas, se refrescan solas, con costo de mantenimiento. NO soportan JOIN ni UDFs no-deterministas.
- **Search Optimization Service** (Enterprise+): acelera lookups `WHERE col = X` en tablas grandes. Costo serverless.
- **Clustering key**: `ALTER TABLE t CLUSTER BY (col)` para tablas grandes con filtros frecuentes.
- **Query Acceleration Service** (Enterprise+): cómputo extra temporal.

### Resource Monitor
- Solo ACCOUNTADMIN a nivel cuenta
- Triggers: `DO NOTIFY / SUSPEND / SUSPEND_IMMEDIATE`

---

## 📦 Carga/Descarga (10%)

### Tamaño óptimo de archivo
- **100–250 MB comprimidos** (gzip/bzip2/zstd)
- JSON/Parquet pueden ser algo más grandes (hasta 1 GB)

### Flujo básico
```
Local → PUT → Stage interno → COPY INTO → Tabla
```

- **PUT** solo en SnowSQL (no en Snowsight)
- `AUTO_COMPRESS = TRUE` (default)

### Load metadata
Snowflake recuerda **64 días** qué archivos cargó (para no duplicar). Override con `FORCE = TRUE`.

### ON_ERROR
- `CONTINUE` (ignora errores)
- `SKIP_FILE` / `SKIP_FILE_N`
- `ABORT_STATEMENT` (default)

### Snowpipe
- Ingesta continua
- **Serverless** (no usa warehouse del usuario)
- Facturación: archivos procesados + cómputo serverless
- Auto-ingest con eventos de S3/Azure/GCS o REST API

### External tables
- Lee archivos en stage externo **sin copiar**
- `AUTO_REFRESH = TRUE` refresca metadata con eventos

### Descarga
- `COPY INTO @stage FROM tabla`
- `SINGLE = TRUE` → un solo archivo
- `GET` solo en SnowSQL

---

## 🔄 Transformaciones (20%)

### VARIANT
- Hasta **16 MB comprimidos**
- Notación: `v:campo.subcampo[0]::TIPO`

### FLATTEN / LATERAL FLATTEN
```sql
SELECT f.value:sku FROM pedidos, LATERAL FLATTEN(input => raw:items) f;
```
Opciones: `outer`, `recursive`, `path`

### Funciones semi-estructurados
- `PARSE_JSON`, `TO_JSON`, `CHECK_JSON`
- `OBJECT_CONSTRUCT`, `ARRAY_CONSTRUCT`
- `IS_OBJECT`, `IS_ARRAY`, `TYPEOF`

### Streams
| Tipo | Captura |
|---|---|
| Standard | INSERT + UPDATE + DELETE |
| Append-only | Solo INSERT |
| Insert-only | Solo INSERT (external tables) |

Avanza cuando se consume con INSERT/CTAS/MERGE en transacción.

### Tasks
- `SCHEDULE = '5 MINUTE'` o cron
- `WHEN SYSTEM$STREAM_HAS_DATA('stream')`
- Deben `ALTER TASK ... RESUME`
- Sin WAREHOUSE = serverless
- Se encadenan con `AFTER tarea_padre` (DAG)

### UDF vs Procedure
| UDF | Procedure |
|---|---|
| Usado en SELECT | `CALL mi_proc()` |
| Retorna valor | Retorna valor pero no en SELECT |
| NO DML/DDL | SÍ DML/DDL |

### Datos no estructurados
- **Directory tables**: `SELECT * FROM DIRECTORY(@stage);`
- **3 tipos de URL:**
  - **Scoped URL** → temporal, 24 h, permisos validados
  - **File URL** → permanente, requiere permisos
  - **Pre-signed URL** → temporal, sin auth Snowflake

### Sampling
- `SAMPLE BERNOULLI (10)` → fila a fila, lento pero uniforme
- `SAMPLE SYSTEM (10)` → bloques, rápido, puede sesgar
- `SAMPLE (100 ROWS)` → tamaño fijo

---

## 🛡️ Protección y Sharing (10%)

### Time Travel
- **0–1 día** (Standard), **0–90 días** (Enterprise+)
- `DATA_RETENTION_TIME_IN_DAYS`
- `AT(OFFSET => -60)`, `AT(TIMESTAMP => '...')`, `BEFORE(STATEMENT => '...')`
- `UNDROP TABLE / SCHEMA / DATABASE`

### Fail-safe
- **7 días FIJOS** (no configurable)
- **NO accesible** al usuario (solo Snowflake Support)
- **Transient y Temporary: 0 días de Fail-safe**

### Clone (zero-copy)
- **Instantáneo, no duplica storage**
- Modificaciones crean nuevas µ-partitions
- Clone puede combinarse con Time Travel
- Clone NO hereda el historial TT del origen

### Replication
- Entre cuentas (DR, geográfico)
- Failover/failback: Business Critical+

### Encriptación
- AES-256 at rest (automático, inmutable)
- TLS 1.2+ in transit
- Key rotation cada 30 días
- **Tri-Secret Secure**: BC+, combina llave Snowflake + llave del cliente

### Data Sharing
- **SIN copiar datos**, tiempo real
- Mismo region + cloud (para cross-region → replicación de shares)
- Consumer usa SU propio warehouse
- **Ilimitadas** cuentas consumer por share
- NO se puede compartir: stages, tasks, pipes, streams

### Tipos de share
| Tipo | Audiencia |
|---|---|
| **Direct Share** | Cuentas específicas |
| **Marketplace** | Global, público |
| **Data Exchange** | Privado, invitados |
| **Reader Account** | Consumers sin Snowflake (provider paga compute) |

---

## 🎯 Top 15 números que DEBES recordar

1. Micropartitions: **50–500 MB** comprimidos
2. Result cache: **24 h** (extensible 31 días)
3. AUTO_SUSPEND mínimo: **60 segundos**
4. Fail-safe: **7 días fijos**
5. Time Travel Standard: **1 día máx**
6. Time Travel Enterprise+: **90 días máx**
7. VARIANT máx: **16 MB comprimidos**
8. Archivo óptimo para carga: **100–250 MB comprimidos**
9. COPY INTO metadata: recuerda archivos por **64 días**
10. Scoped URL: válido **24 h**
11. Examen: **115 minutos, ~100 preguntas, aprobar con 750/1000**
12. Info_schema: retención **7 días**
13. Account_usage: latencia **45 min – 3 h**, retención **1 año**
14. INFO_SCHEMA.QUERY_HISTORY retención: **7 días**
15. Mínimo facturación primera vez WH arranca: **60 seg**

---

## ⚠️ Top 10 trampas del examen

1. **Fail-safe NO es accesible** → solo Snowflake Support puede recuperar.
2. **Transient y Temporary NO tienen Fail-safe.**
3. **USERADMIN crea usuarios, NO SYSADMIN.**
4. **SYSADMIN debe ser el dueño de BDs** (no ACCOUNTADMIN).
5. **Scaling UP ≠ Scaling OUT**. Up = tamaño; Out = clusters.
6. **COPY INTO no recarga mismos archivos** por default (64 días memoria).
7. **PUT/GET solo funcionan en SnowSQL**, no en Snowsight.
8. **MV no soporta JOINs ni UDFs no-deterministas.**
9. **Clone no duplica storage inicialmente**, pero sí después si modificas.
10. **Snowpipe factura por archivo + serverless**, no por tiempo de warehouse.

---

## 💡 Consejos finales para el examen

1. **Lee cada pregunta 2 veces.** Snowflake escribe preguntas con trampas de wording.
2. **"Which of the following" vs "Which TWO/THREE"** — fíjate en cuántas respuestas pide.
3. Si una pregunta tiene **"MINIMUM" o "BEST"**, elimina primero las opciones claramente incorrectas.
4. Si dudas entre 2 respuestas, elige la más **específica** y **simple** (Snowflake ama la simplicidad).
5. **Marca y sigue.** No te traben preguntas difíciles. Al final revisa las marcadas.
6. Tiempo: **~1 min/pregunta**. Haz pases rápidos y deja 15 min al final para revisar.
7. **Respira.** Estás preparado. 💪

¡A por la certificación! 🎓
