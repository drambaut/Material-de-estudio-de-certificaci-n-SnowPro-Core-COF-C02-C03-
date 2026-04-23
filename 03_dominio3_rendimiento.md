# ⚡ Dominio 3 — Conceptos de Rendimiento

**Peso en el examen: 15%**
**Día del plan: 5**

---

## 3.1 Query Profile

### 🔍 ¿Dónde se encuentra?
En Snowsight → History → click en cualquier query → pestaña **Query Profile**.

### 📊 Qué información da
- **Plan de ejecución visual** (árbol de operadores)
- **Tiempo por paso** (Scan, Join, Aggregate, Sort, etc.)
- **Bytes scanned, bytes spilled, % pruning**
- **Micropartitions totales vs leídas**

### 💾 Spilling (desbordamiento)

Cuando la memoria del warehouse no alcanza, los datos "derraman" a disco:

| Tipo | Impacto | Qué hacer |
|---|---|---|
| **Spilling to local storage** | Pequeño, al SSD del warehouse | Aceptable, pero alerta |
| **Spilling to remote storage** | **Grande**, va al storage de la cloud | ⚠️ **Aumenta el warehouse** o reescribe la query |

Si ves mucho spilling remoto → **tu warehouse es demasiado pequeño para la query**.

### 🧱 Pruning (poda de microparticiones)

Indicador clave: **"Partitions scanned" / "Partitions total"**.
- Ratio bajo (ej. 5/1000) = **buena poda** ✅
- Ratio alto (ej. 950/1000) = **poca poda**, probablemente filtrando por columna no clusterizada

### 📜 Query History
- Snowsight → Activity → Query History
- O con SQL:
  ```sql
  -- Últimas queries (rápido, reciente)
  SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
    ORDER BY start_time DESC LIMIT 20;

  -- Histórico completo (lento pero extenso)
  SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE start_time > DATEADD(day, -30, CURRENT_TIMESTAMP);
  ```

---

## 3.2 Configuración de Virtual Warehouses

### 📏 Tamaños de warehouse

| Tamaño | Créditos/hora | Nodos relativos |
|---|---|---|
| X-Small | 1 | 1 |
| Small | 2 | 2 |
| Medium | 4 | 4 |
| Large | 8 | 8 |
| X-Large | 16 | 16 |
| 2X-Large | 32 | 32 |
| 3X-Large | 64 | 64 |
| 4X-Large | 128 | 128 |
| 5X-Large | 256 | 256 |
| 6X-Large | 512 | 512 |

**Regla:** cada tamaño **dobla** los créditos/h y (aproximadamente) **reduce a la mitad** el tiempo de queries que saturan el warehouse.

### ⚙️ Tipos de warehouse

- **Standard:** el warehouse clásico, para cargas de trabajo generales.
- **Snowpark-optimized:** hasta **16x más memoria por nodo** — para ML, Java/Scala UDFs pesadas, Snowpark.

### 🔀 Multi-cluster warehouses (SOLO Enterprise+)

Un warehouse puede tener **varios clusters del mismo tamaño** corriendo en paralelo para manejar **concurrencia**, NO queries más rápidas.

```sql
CREATE WAREHOUSE wh_bi WITH
  WAREHOUSE_SIZE = 'MEDIUM'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 5
  SCALING_POLICY = 'STANDARD';
```

#### Modos de escalamiento (MIN/MAX)
- **Maximized mode:** MIN = MAX = N → siempre tengo N clusters corriendo
- **Auto-scale mode:** MIN < MAX → escala según demanda

#### Políticas de escalamiento
- **STANDARD** (default) → prioriza arrancar cluster ante cualquier query en cola
- **ECONOMY** → intenta saturar clusters existentes antes de arrancar uno nuevo (ahorra créditos)

### 🎛️ Parámetros clave

```sql
CREATE WAREHOUSE mi_wh
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 60        -- Segundos de inactividad antes de pausar
  AUTO_RESUME = TRUE       -- Reanuda automático al llegar una query
  INITIALLY_SUSPENDED = TRUE;
```

- **AUTO_SUSPEND** — mínimo 60 segundos. Evita consumir créditos inactivo.
- **AUTO_RESUME** — arranca solo cuando llega una query.
- **Primera vez que arranca un WH:** se factura **mínimo 60 seg**, después por segundo.

> 🎯 **Trampa:** un warehouse parado **no consume créditos de compute**, pero sí de storage (si fuera a contar, que no cuenta).

---

## 3.3 Herramientas de rendimiento

### 📈 Monitoreo de cargas de warehouse
- Snowsight → Admin → Warehouses → click en WH → **Load monitoring chart**
- Dos líneas:
  - **Running** (queries activas) — si toca el techo, queries haciendo cola
  - **Queued** (en cola) — si aparece, necesitas más recursos

### ⬆️ Scaling Up vs ⬌ Scaling Out

| Estrategia | Qué hace | Resuelve |
|---|---|---|
| **Scaling UP (vertical)** | Cambiar tamaño del WH (S→M→L) | Queries **complejas/lentas** individuales |
| **Scaling OUT (horizontal)** | Multi-cluster, más clusters del mismo tamaño | **Concurrencia** alta (muchos usuarios a la vez) |

> 🎯 **Pregunta típica:** "Muchos usuarios se quejan de lentitud al mismo tiempo" → scaling **OUT**. "Una query ETL tarda 2 horas" → scaling **UP**.

### 💰 Resource monitors

Controlan el gasto de créditos de la cuenta o de WH específicos:

```sql
CREATE RESOURCE MONITOR mi_monitor WITH
  CREDIT_QUOTA = 1000
  FREQUENCY = 'MONTHLY'
  START_TIMESTAMP = 'IMMEDIATELY'
  TRIGGERS
    ON 75 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND
    ON 110 PERCENT DO SUSPEND_IMMEDIATE;
```

- **NOTIFY** → email a admins
- **SUSPEND** → deja que las queries corriendo terminen, no acepta nuevas
- **SUSPEND_IMMEDIATE** → mata queries corriendo 💀

Solo **ACCOUNTADMIN** puede crear monitors a nivel cuenta.

### 🚀 Query Acceleration Service (QAS)

- Acelera queries aisladas que requieren mucho cómputo agregando cómputo elástico **temporal**.
- No es para cargas sostenidas.
- Requiere **Enterprise**.
- Activación: `ALTER WAREHOUSE wh SET ENABLE_QUERY_ACCELERATION = TRUE;`

---

## 3.4 Optimizar el rendimiento de las consultas

### 📐 Vistas materializadas (Enterprise+)

- Datos **precomputados y persistidos**.
- Se **refrescan automáticamente** cuando cambia la tabla base.
- Tienen **costo de mantenimiento** (créditos de un serverless service).
- Útiles para:
  - Agregaciones pesadas y frecuentes
  - Subsets filtrados muy consultados
- **Limitaciones importantes:**
  - No soportan JOINs
  - No soportan UDFs ni funciones no-deterministas
  - No soportan window functions complejas

### 🎯 SELECT específicos
Siempre evita `SELECT *`. Como Snowflake es **columnar**, leer menos columnas = menos bytes escaneados = más barato y rápido.

### 🧬 Clustering keys

Para **tablas muy grandes** (>1 TB) donde un WHERE siempre filtra por la misma columna:

```sql
ALTER TABLE ventas CLUSTER BY (fecha);
SYSTEM$CLUSTERING_DEPTH('ventas', '(fecha)');  -- mide la salud del clustering
```

- Snowflake tiene **Automatic Clustering** (servicio en background que reorganiza micropartitions, tiene costo).
- Activar/desactivar:
  ```sql
  ALTER TABLE ventas SUSPEND RECLUSTER;
  ALTER TABLE ventas RESUME RECLUSTER;
  ```

### 🔍 Search Optimization Service (SOS)

- Acelera queries con **filtros muy selectivos** (equality, IN, LIKE) sobre columnas NO clusterizadas.
- Ideal para lookups tipo `WHERE email = 'foo@bar.com'` en tablas muy grandes.
- Tiene **costo de mantenimiento** (serverless).
- Se activa por tabla/columna:
  ```sql
  ALTER TABLE clientes ADD SEARCH OPTIMIZATION;
  ALTER TABLE clientes ADD SEARCH OPTIMIZATION ON EQUALITY(email);
  ```

### 🧠 Resultados persistidos

Snowflake **guarda el resultado de toda query exitosa por 24 horas**. Si alguien ejecuta **exactamente la misma query** (bit por bit, misma sesión o no), devuelve el resultado **sin usar warehouse** (gratis).

Si la query se reusa, el tiempo se extiende hasta **31 días máximo**.

Se invalida si:
- Cambian los datos subyacentes
- Se usan funciones no-deterministas (`CURRENT_TIMESTAMP()`, `RANDOM()`)
- Cambia el rol con privilegios diferentes

### 🧩 Los 3 tipos de caché (¡MUY preguntado!)

| Caché | Dónde está | Qué guarda | TTL | Costo |
|---|---|---|---|---|
| **Metadata cache** | Cloud Services | Min/max/count por micropartition | Mientras exista la partition | Gratis |
| **Result cache** | Cloud Services | Resultado completo de queries | **24 h** (extendible a 31 d) | Gratis (no usa WH) |
| **Warehouse cache (local disk)** | SSD del warehouse | Micropartitions leídas recientemente | Mientras el WH esté activo | Ya pagas el WH |

> 🎯 **Pregunta típica:** "Corriste `SELECT COUNT(*) FROM tabla` sin warehouse activo. ¿Funciona?" → **Sí, usa Metadata Cache**.

> 🎯 **Otra trampa:** Suspender y reanudar el warehouse **pierde** la warehouse cache (SSD). Por eso multi-cluster comparte mejor la caché.

---

## 🧪 Ejercicios prácticos (Día 5)

```sql
-- Ejercicio 1: Probar caché de resultados
SELECT COUNT(*) FROM snowflake_sample_data.tpch_sf1.orders;
-- Ahora suspende el WH
ALTER WAREHOUSE compute_wh SUSPEND;
-- Ejecuta la misma query → ¡funciona! (result cache)
SELECT COUNT(*) FROM snowflake_sample_data.tpch_sf1.orders;

-- Ejercicio 2: Metadata cache
-- COUNT(*) sobre tabla completa sin filtros → usa metadata
SELECT COUNT(*) FROM snowflake_sample_data.tpch_sf1.lineitem;

-- Ejercicio 3: Query profile
ALTER WAREHOUSE compute_wh RESUME;
SELECT l_shipdate, SUM(l_quantity)
FROM snowflake_sample_data.tpch_sf1.lineitem
WHERE l_shipdate BETWEEN '1994-01-01' AND '1994-12-31'
GROUP BY 1;
-- Ve a History → click en la query → Query Profile
-- Observa: Bytes scanned, Partitions scanned/total, Tiempo por paso

-- Ejercicio 4: Resource Monitor
USE ROLE ACCOUNTADMIN;
CREATE OR REPLACE RESOURCE MONITOR monitor_lab WITH
  CREDIT_QUOTA = 10
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 50 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND;

-- Ejercicio 5: Scaling
ALTER WAREHOUSE compute_wh SET WAREHOUSE_SIZE = 'SMALL';  -- Scaling UP
-- (Para multi-cluster necesitas Enterprise, solo estudia la sintaxis)
```

---

## ❓ Preguntas de práctica del Dominio 3

**1.** Un warehouse XSMALL consume X créditos/hora; uno LARGE consume...
- A) 4
- B) 8 ✅
- C) 16
- D) 32

**2.** Ves `Bytes spilled to remote storage: 45 GB` en Query Profile. ¿Qué haces?
- A) Agregar más clusters (scaling out)
- B) Aumentar el tamaño del warehouse (scaling up) ✅
- C) Crear un resource monitor
- D) Activar caché de resultados

**3.** La Result Cache dura ____ pero se puede extender hasta ____ si la query se reejecuta.
- A) 24 h / 31 días ✅
- B) 1 h / 7 días
- C) 12 h / 24 h
- D) Para siempre

**4.** Para muchos usuarios concurrentes ejecutando queries cortas, ¿qué configuración es mejor?
- A) Un warehouse X-Large
- B) Multi-cluster con scaling out ✅
- C) Query Acceleration Service
- D) Search Optimization

**5.** ¿Cuál NO es un tipo de caché en Snowflake?
- A) Metadata cache
- B) Result cache
- C) Warehouse (local disk) cache
- D) Storage cache ✅ (no existe)

**6.** Un resource monitor con trigger `ON 100 PERCENT DO SUSPEND`...
- A) Mata queries corriendo inmediatamente
- B) Deja que las queries actuales terminen y no acepta nuevas ✅
- C) Solo envía notificaciones
- D) Escala el warehouse

**7.** Para acelerar lookups tipo `WHERE email = 'x@y.com'` en tablas grandes:
- A) Clustering key por email
- B) Vista materializada
- C) Search Optimization Service ✅
- D) QAS

**8.** El AUTO_SUSPEND mínimo es:
- A) 30 segundos
- B) 60 segundos ✅
- C) 5 minutos
- D) 1 hora

---

## 🎯 Checklist antes de pasar al Día 6

- [ ] Diferencio scaling up vs scaling out y sus casos de uso
- [ ] Conozco los 3 tipos de caché y su TTL
- [ ] Sé leer un Query Profile y qué es spilling
- [ ] Conozco las 2 políticas de escalamiento (Standard vs Economy)
- [ ] Entiendo cuándo usar vistas materializadas vs SOS vs clustering
- [ ] Puedo crear un resource monitor con triggers

¡Siguiente: cargar datos! 🔽
