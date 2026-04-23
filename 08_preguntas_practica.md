# 📝 Preguntas de Práctica Adicionales — SnowPro Core

Organizado en **3 bloques** para usar en los días 29, 31 y 22:
- **Bloque 1** (20 preguntas): Simulacro #1 — día 19 mayo
- **Bloque 2** (20 preguntas): Simulacro #2 — día 21 mayo
- **Bloque 3** (20 preguntas): Repaso final — día 22 mayo (víspera)

**Instrucciones:**
1. Responde sin mirar la guía.
2. Dale 1 minuto por pregunta (20 min por bloque).
3. Revisa las respuestas al final.
4. Si fallas 6+ en un bloque → revisa el dominio correspondiente.

---

## 🎯 BLOQUE 1 — Simulacro amplio (20 preguntas)

**1.** ¿Qué capa de Snowflake es responsable de la optimización de queries?
- A) Database Storage
- B) Query Processing
- C) Cloud Services
- D) Virtual Warehouse

**2.** Para que una tabla tenga 90 días de Time Travel se necesita:
- A) Snowflake Standard
- B) Snowflake Enterprise o superior
- C) Tabla transient
- D) Cualquier edición con configuración

**3.** ¿Cuáles de estos son esquemas del sistema? (Elige DOS)
- A) INFORMATION_SCHEMA
- B) PUBLIC
- C) ACCOUNT_USAGE (dentro de SNOWFLAKE)
- D) SYSTEM

**4.** ¿Qué comando se usa para subir archivos locales a un stage interno?
- A) UPLOAD
- B) PUT
- C) COPY INTO
- D) SEND

**5.** ¿Qué rol debería poseer las bases de datos de producción según best practices?
- A) ACCOUNTADMIN
- B) SECURITYADMIN
- C) SYSADMIN
- D) ORGADMIN

**6.** La política de escalamiento STANDARD de un multi-cluster:
- A) Intenta saturar clusters existentes primero
- B) Prioriza arrancar un cluster ante cualquier cola
- C) Solo escala 1 cluster por hora
- D) No existe, solo hay "Economy"

**7.** ¿Cuál es la retención mínima de Fail-safe para una tabla permanent?
- A) 0 días
- B) 1 día
- C) 7 días
- D) Configurable

**8.** ¿Qué privilegio necesitas para hacer SELECT en una tabla? (Elige TODOS los requeridos)
- A) USAGE en la BD
- B) USAGE en el esquema
- C) SELECT en la tabla
- D) USAGE en un warehouse

**9.** Un archivo JSON de 800 MB descomprimido para carga:
- A) Es óptimo
- B) Es demasiado grande, conviene dividir
- C) Snowflake lo rechaza
- D) Es demasiado pequeño

**10.** ¿Cuál de las siguientes es UNA función de agregación aproximada?
- A) SUM_APPROX
- B) APPROX_COUNT_DISTINCT
- C) COUNT_ESTIMATE
- D) HLL_SUM

**11.** ¿Cuánto puede durar la Result Cache si una query se sigue reejecutando?
- A) 24 horas fijas
- B) Hasta 31 días máximo
- C) Indefinidamente
- D) 7 días

**12.** Para cargar datos en tiempo casi real de un bucket S3 a una tabla Snowflake:
- A) CREATE EXTERNAL TABLE
- B) CREATE PIPE con AUTO_INGEST
- C) INSERT INTO con un bucle
- D) CREATE STREAM

**13.** ¿Qué operador extrae un campo de un VARIANT JSON?
- A) `->`
- B) `:`
- C) `.`
- D) `#`

**14.** Un stream "stale" significa:
- A) No hay datos nuevos
- B) Sobrepasó su retención sin ser consumido y quedó obsoleto
- C) La tabla origen fue dropeada
- D) Está pausado

**15.** Un zero-copy clone...
- A) Duplica todos los datos inmediatamente
- B) Apunta a las mismas micropartitions hasta que se modifique
- C) Solo funciona con tablas externas
- D) Requiere Business Critical

**16.** ¿Qué tipo de URL expira en 24 horas máximo?
- A) File URL
- B) Stage URL
- C) Scoped URL
- D) Pre-signed URL (depende)

**17.** ¿Qué edición soporta Multi-Cluster Warehouses?
- A) Standard
- B) Enterprise
- C) Business Critical
- D) B y C

**18.** Un warehouse suspendido:
- A) Sigue consumiendo créditos
- B) No consume créditos de compute
- C) Pierde todo el storage
- D) Se elimina después de 24 h

**19.** ¿Cómo consultas el uso de cómputo de los últimos 6 meses?
- A) INFORMATION_SCHEMA.QUERY_HISTORY
- B) SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
- C) SHOW WAREHOUSES HISTORY
- D) SELECT CURRENT_USAGE()

**20.** Para compartir una tabla entre dos cuentas Snowflake de la misma región:
- A) Replication
- B) Direct Share
- C) Exportar y subir
- D) Marketplace obligatorio

---

### ✅ Respuestas Bloque 1

| # | Resp. | Justificación breve |
|---|---|---|
| 1 | **C** | Cloud Services incluye optimizer, metadata, auth, seguridad |
| 2 | **B** | Enterprise da hasta 90 días de TT |
| 3 | **A, C** | Ambos son esquemas del sistema (INFO por BD, ACCOUNT_USAGE global) |
| 4 | **B** | PUT sube desde local al stage interno |
| 5 | **C** | SYSADMIN es recomendado como dueño de BDs |
| 6 | **B** | STANDARD prioriza arrancar cluster ante cola; ECONOMY prioriza saturar |
| 7 | **C** | 7 días fijos para permanent |
| 8 | **A, B, C, D** | Los 4 son necesarios |
| 9 | **A** | Archivos de 100 MB – 1 GB son OK para JSON/Parquet |
| 10 | **B** | HyperLogLog bajo el capó |
| 11 | **B** | 24 h base, hasta 31 días si se reutiliza |
| 12 | **B** | Snowpipe + AUTO_INGEST con eventos de S3 |
| 13 | **B** | `v:campo` |
| 14 | **B** | Stale = sobrepasó retención sin consumir |
| 15 | **B** | Zero-copy no duplica inicialmente |
| 16 | **C** | Scoped URL expira en 24 h |
| 17 | **D** | Enterprise y superiores |
| 18 | **B** | Compute no cobra mientras esté suspendido |
| 19 | **B** | ACCOUNT_USAGE tiene 1 año de historia |
| 20 | **B** | Direct Share es la vía más simple |

**Puntuación:** Si tienes 15+ correctas, vas muy bien. Si <12, repasa los dominios donde fallaste.

---

## 🎯 BLOQUE 2 — Segundo simulacro (20 preguntas)

**1.** ¿Cuánto dura la caché de metadata para una tabla?
- A) 24 horas
- B) 7 días
- C) Mientras la tabla exista con sus µ-partitions
- D) 1 hora

**2.** ¿Qué comando muestra los grants de un rol?
- A) `SHOW ROLES GRANTS`
- B) `SHOW GRANTS TO ROLE <rol>`
- C) `LIST GRANTS ROLE <rol>`
- D) `DESC ROLE <rol>`

**3.** Un ALTER WAREHOUSE ... SET WAREHOUSE_SIZE='XLARGE' es un ejemplo de:
- A) Scaling out
- B) Scaling up
- C) Query acceleration
- D) Auto-resume

**4.** Si un resource monitor dispara "SUSPEND_IMMEDIATE":
- A) Espera a que las queries actuales terminen
- B) Solo envía email
- C) Mata las queries en ejecución inmediatamente
- D) Escala al siguiente tier

**5.** ¿Qué tipo de función permite retornar múltiples filas?
- A) Scalar UDF
- B) Stored Procedure
- C) UDTF (Table Function)
- D) External Function

**6.** LATERAL FLATTEN se usa típicamente para:
- A) Aplanar arrays/objetos JSON en filas
- B) Particionar tablas
- C) Crear joins implícitos
- D) Desfragmentar micropartitions

**7.** ¿Cuál es la diferencia entre BERNOULLI y SYSTEM sampling?
- A) BERNOULLI es por filas, SYSTEM por bloques
- B) BERNOULLI es por bloques, SYSTEM por filas
- C) Son idénticos
- D) SYSTEM solo funciona en Enterprise

**8.** Un esquema creado puede tener Time Travel:
- A) Sí, heredado de la BD
- B) No, solo las tablas
- C) Solo en Business Critical
- D) Siempre 0 días

**9.** Una tabla externa:
- A) Copia los datos a Snowflake
- B) Consulta archivos en stage externo sin copiar
- C) Solo soporta CSV
- D) No se puede consultar con SQL

**10.** ¿Qué PARÁMETRO configura el tiempo de Time Travel de una tabla?
- A) TIME_TRAVEL_DAYS
- B) RETENTION_PERIOD
- C) DATA_RETENTION_TIME_IN_DAYS
- D) TT_DAYS

**11.** Snowpipe se factura por:
- A) Tiempo de warehouse
- B) Archivos procesados + compute serverless
- C) Tamaño de storage
- D) Número de consultas

**12.** ¿Cuál método de autenticación se recomienda para pipelines automatizados de CI/CD?
- A) Contraseña + MFA
- B) Key-pair authentication
- C) SAML SSO
- D) OAuth popup

**13.** Un `CREATE TABLE ... CLONE ... AT(OFFSET => -3600)` hace:
- A) Clona la tabla en su estado actual
- B) Clona la tabla en su estado de hace 1 hora
- C) Clona la tabla en su estado de hace 1 día
- D) Error de sintaxis

**14.** Para buscar rápidamente `WHERE email = 'x@y.com'` en una tabla grande no clusterizada:
- A) Vista materializada
- B) Search Optimization Service
- C) Clustering key por email
- D) QAS

**15.** ¿Qué información NO está en el Query Profile?
- A) Bytes scanned
- B) Plan de ejecución visual
- C) Contraseñas de usuarios
- D) Bytes spilled

**16.** Una sesión de SnowSQL permite:
- A) Ejecutar PUT y GET
- B) Abrir múltiples worksheets visuales
- C) Editar dashboards
- D) Ver imágenes embebidas

**17.** ¿Qué rol es necesario para CREAR un resource monitor a nivel de cuenta?
- A) SYSADMIN
- B) SECURITYADMIN
- C) ACCOUNTADMIN
- D) USERADMIN

**18.** Las tablas materializadas:
- A) No soportan JOINs
- B) Soportan cualquier query
- C) No se actualizan solas
- D) Están disponibles en Standard

**19.** Un share puede tener como consumidor:
- A) Máximo 100 cuentas
- B) Cuentas ilimitadas
- C) Solo cuentas en la misma región
- D) Solo la cuenta propietaria

**20.** ¿Qué archivo de configuración suele usarse para conectar SnowSQL?
- A) `~/.snowsql/config`
- B) `/etc/snowflake/settings.json`
- C) `~/.snowflake/.credentials`
- D) `snowflake.yaml`

---

### ✅ Respuestas Bloque 2

| # | Resp. | Justificación |
|---|---|---|
| 1 | **C** | Metadata cache dura mientras existan las µ-partitions |
| 2 | **B** | `SHOW GRANTS TO ROLE` muestra privilegios del rol |
| 3 | **B** | Cambiar tamaño = scaling up |
| 4 | **C** | SUSPEND_IMMEDIATE mata queries en ejecución |
| 5 | **C** | UDTF retorna tabla |
| 6 | **A** | Aplana arrays/objetos JSON |
| 7 | **A** | BERNOULLI fila a fila, SYSTEM por bloques |
| 8 | **A** | Los esquemas heredan TT de la BD |
| 9 | **B** | External tables leen sin copiar |
| 10 | **C** | DATA_RETENTION_TIME_IN_DAYS |
| 11 | **B** | Archivos + serverless compute |
| 12 | **B** | Key-pair para automatización |
| 13 | **B** | Estado de hace 1 hora (3600 segundos) |
| 14 | **B** | SOS acelera lookups de equality |
| 15 | **C** | Por seguridad, no expone creds |
| 16 | **A** | PUT y GET solo en SnowSQL |
| 17 | **C** | Solo ACCOUNTADMIN a nivel cuenta |
| 18 | **A** | MV no soporta JOINs |
| 19 | **B** | Ilimitadas |
| 20 | **A** | `~/.snowsql/config` |

**Puntuación:** Si tienes 16+ correctas, estás listo. Si 12–15, afina detalles. Si <12, repasa el cheatsheet con calma.

---

## 🎯 BLOQUE 3 — Repaso final (20 preguntas con más trampas)

**1.** Un usuario ejecuta una query con un rol que tiene SELECT en la tabla, pero obtiene error. ¿Qué puede faltar?
- A) USAGE en el warehouse
- B) USAGE en la BD o esquema
- C) MFA activado
- D) A o B

**2.** ¿Cuándo se pierde la caché del warehouse (SSD)?
- A) Al suspender el warehouse
- B) Cada 24 horas
- C) Al cambiar el default_role
- D) Nunca, es persistente

**3.** Tienes una tabla de 10 TB. Las queries siempre filtran por `fecha`. ¿Qué haces?
- A) Crear vista materializada
- B) Definir clustering key por `fecha`
- C) Activar SOS
- D) Aumentar el warehouse

**4.** Una tabla TRANSIENT al dropearse:
- A) Se puede recuperar con UNDROP (si dentro del TT)
- B) Va a Fail-safe 7 días
- C) Se borra inmediatamente sin recuperación
- D) Se convierte en PERMANENT

**5.** ¿Qué declaración es CIERTA sobre Snowpipe?
- A) Necesita un warehouse asignado
- B) Es siempre más barato que COPY INTO
- C) Usa cómputo serverless gestionado por Snowflake
- D) Solo funciona con CSV

**6.** Tienes que cargar 10 millones de filas desde S3 cada noche. ¿La mejor opción?
- A) Snowpipe con auto-ingest
- B) COPY INTO con un task programado
- C) Stream + Task
- D) INSERT fila por fila

**7.** Un UDF determinista...
- A) Siempre retorna el mismo valor para el mismo input
- B) Puede incluir llamadas a APIs externas
- C) No puede escribirse en Python
- D) Solo funciona con datos estructurados

**8.** Al crear una task, ¿qué pasa inmediatamente después?
- A) Se ejecuta una vez y luego espera al schedule
- B) Queda suspendida hasta que hagas RESUME
- C) Arranca automáticamente
- D) Espera al siguiente reinicio del warehouse

**9.** Una organización con 3 cuentas en 3 regiones diferentes quiere compartir un dataset. ¿Qué combina?
- A) Direct Share entre cuentas
- B) Replication + Sharing
- C) Copy manual por SNS
- D) Marketplace público

**10.** ¿Qué NO se incluye al clonar una tabla?
- A) Los privilegios (grants) del origen
- B) Los datos al momento del clone
- C) La clustering key
- D) A y C

**11.** ¿Qué edición ofrece failover/failback automático entre regiones?
- A) Standard
- B) Enterprise
- C) Business Critical
- D) Solo VPS

**12.** Tienes un archivo CSV malformado y quieres ver qué filas fallarían ANTES de cargar:
- A) `COPY INTO ... VALIDATION_MODE = 'RETURN_ERRORS'`
- B) `LIST @stage`
- C) `DESCRIBE STAGE`
- D) `SELECT * FROM @stage`

**13.** ¿Qué afirmación es VERDADERA?
- A) Una network policy puede aplicarse a un usuario específico
- B) Las masking policies solo funcionan con STRING
- C) Los shares permiten INSERT al consumer
- D) Fail-safe es configurable

**14.** ¿Cuál es la latencia típica de ACCOUNT_USAGE.QUERY_HISTORY?
- A) Tiempo real
- B) 45 min – 3 horas
- C) 24 horas
- D) 7 días

**15.** Si un stream de 1 día de retención no se consume por 48 horas:
- A) Se consume automáticamente
- B) Queda stale y hay que recrearlo
- C) Extiende su retención
- D) Bloquea la tabla origen

**16.** Para descargar una tabla como un solo archivo Parquet gigante:
- A) `COPY INTO @stage FROM tabla FILE_FORMAT=(TYPE=PARQUET) SINGLE=TRUE MAX_FILE_SIZE=5368709120`
- B) `GET @stage/parquet -single`
- C) `EXPORT TABLE ...`
- D) No se puede en un solo archivo

**17.** Un RESOURCE MONITOR se puede crear a nivel:
- A) Cuenta y warehouse
- B) Solo cuenta
- C) Solo warehouse
- D) BD, esquema o tabla

**18.** ¿Cuántos clusters puede tener un multi-cluster warehouse?
- A) 1–5
- B) 1–10
- C) Depende del plan, hasta decenas
- D) Solo 1

**19.** ¿Cómo ves el historial COMPLETO de queries del último año?
- A) `INFORMATION_SCHEMA.QUERY_HISTORY`
- B) `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`
- C) `SHOW QUERY HISTORY`
- D) UI solamente

**20.** El comando `UNDROP DATABASE mi_bd`:
- A) Solo funciona con ACCOUNTADMIN
- B) Funciona si el DROP está dentro del Time Travel
- C) Restaura la BD completa con sus objetos
- D) B y C

---

### ✅ Respuestas Bloque 3

| # | Resp. | Justificación |
|---|---|---|
| 1 | **D** | Faltar USAGE en BD/esquema/WH son todas causas comunes |
| 2 | **A** | Suspender pierde la cache SSD |
| 3 | **B** | Clustering key para tablas grandes con filtros recurrentes |
| 4 | **A** | Se recupera con UNDROP si está dentro del TT (no tiene Fail-safe) |
| 5 | **C** | Snowpipe es serverless |
| 6 | **B** | Para batch nocturno COPY INTO + task es lo ideal; Snowpipe es para continuo |
| 7 | **A** | Determinista = mismo input, mismo output |
| 8 | **B** | Las tasks nacen SUSPENDIDAS |
| 9 | **B** | Replication entre regiones + Sharing |
| 10 | **A** | Los grants NO se clonan |
| 11 | **C** | Business Critical para failover/failback |
| 12 | **A** | VALIDATION_MODE detecta errores sin cargar |
| 13 | **A** | Network policies pueden ser a nivel cuenta o usuario |
| 14 | **B** | 45 min a 3 horas |
| 15 | **B** | Stale stream hay que recrear |
| 16 | **A** | SINGLE=TRUE + MAX_FILE_SIZE |
| 17 | **A** | Cuenta y warehouse |
| 18 | **C** | Hasta 10 por default en Enterprise, pero puede pedirse más |
| 19 | **B** | ACCOUNT_USAGE retiene 1 año |
| 20 | **D** | Funciona si está en TT y restaura todo |

**Puntuación:** Si tienes 16+ correctas en este bloque (que tiene más trampas), estás **muy bien preparado** para el examen real.

---

## 📊 Tabla de autoevaluación

| Bloque | # correctas | Estado |
|---|---|---|
| Bloque 1 | __/20 | □ Listo (≥16) □ Afinar (12–15) □ Repasar (<12) |
| Bloque 2 | __/20 | □ Listo (≥16) □ Afinar (12–15) □ Repasar (<12) |
| Bloque 3 | __/20 | □ Listo (≥16) □ Afinar (12–15) □ Repasar (<12) |

---

## 🎓 Último mensaje antes del examen

Si llegaste hasta acá y estás haciendo estos simulacros:
- **Ya hiciste el trabajo duro.** Has estudiado, practicado y revisado.
- En el examen, **confía en tu preparación**. La primera respuesta que te viene suele ser la correcta.
- Si te trabas en 2–3 preguntas, **no entres en pánico** — puedes equivocarte en 25% y aún así aprobar.
- Recuerda: puedes **marcar y volver** a preguntas difíciles.

¡Ánimo! 🚀
