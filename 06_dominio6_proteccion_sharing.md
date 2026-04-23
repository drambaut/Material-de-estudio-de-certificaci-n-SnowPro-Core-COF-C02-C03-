# 🛡️ Dominio 6 — Protección de Datos y Data Sharing

**Peso en el examen: 10%**
**Días del plan: 26–27 (16–17 de mayo)**

---

## 6.1 Protección continua de datos con Snowflake

### 🛡️ Snowflake's Continuous Data Protection (CDP)

Snowflake tiene varias capas de protección que juntas forman el CDP:

```
Objeto activo → Time Travel → Fail-safe → ❌ eliminado
  (modificable)  (accesible) (NO accesible)
```

### ⏰ Time Travel (¡MUY preguntado!)

Permite consultar o restaurar datos tal como estaban en un punto pasado.

**Duración según edición:**

| Edición | Default | Máximo |
|---|---|---|
| Standard | 1 día | **1 día** |
| Enterprise+ | 1 día (configurable) | **90 días** |

Se configura con `DATA_RETENTION_TIME_IN_DAYS` en cuenta, BD, esquema o tabla.

**Por tipo de tabla:**
| Tipo | Max Time Travel |
|---|---|
| Permanent | 1 o 90 días |
| Transient | 0 o 1 día |
| Temporary | 0 o 1 día |

**Consultar datos en el pasado:**
```sql
-- Por timestamp
SELECT * FROM ventas AT(TIMESTAMP => '2026-04-20 10:00:00'::TIMESTAMP);

-- Por offset en segundos (hace 5 minutos)
SELECT * FROM ventas AT(OFFSET => -60*5);

-- Antes de un statement específico
SELECT * FROM ventas BEFORE(STATEMENT => '<query_id>');
```

**Recuperar datos:**
```sql
-- UNDROP: recuperar tabla, esquema o BD eliminados (dentro del Time Travel)
UNDROP TABLE ventas;
UNDROP SCHEMA public;
UNDROP DATABASE mi_bd;

-- Recuperar datos: crear una tabla nueva a partir del pasado
CREATE TABLE ventas_recuperada CLONE ventas AT(OFFSET => -3600);
```

> 🎯 **Pregunta típica:** "Eliminaste accidentalmente una tabla hace 2 horas. ¿Qué haces?" → **`UNDROP TABLE`** (si el Time Travel está activo).

### 🧯 Fail-safe (¡MUY preguntado!)

**7 días FIJOS** (no configurable) de protección **adicional** después de que termina Time Travel.

**Puntos CLAVE:**
- ⚠️ NO es accesible por el usuario → solo Snowflake Support puede recuperar.
- ⚠️ Costo: **se factura el storage** (igual que cualquier otro dato).
- ⚠️ **Tablas TRANSIENT y TEMPORARY NO tienen Fail-safe.**

```
Time Travel (0–90 días) → Fail-safe (7 días fijos, solo soporte)
```

### 🔐 Encriptación (cifrado)

Snowflake encripta **todos los datos automáticamente** y no lo puedes desactivar:
- **At rest:** AES-256 sobre todos los archivos en S3/Azure/GCS.
- **In transit:** TLS 1.2+.
- **Key rotation:** automática cada 30 días.
- **Hierarchical key model:** cada cuenta/tabla tiene su jerarquía de llaves.

**Tri-Secret Secure** (solo **Business Critical+**):
- Combina la llave de Snowflake + una llave gestionada por ti en tu cloud (AWS KMS, Azure Key Vault) + código de acceso.
- Si retiras tu llave → ni Snowflake puede leer tus datos.

**Customer-Managed Keys (CMK):** tú gestionas la llave en tu KMS/Key Vault.

### 🧬 Cloning (zero-copy cloning)

Crear una copia **instantánea** de una tabla, esquema o BD **sin duplicar datos**.
- No consume storage inicialmente (apunta a las mismas micropartitions).
- Cuando modificas el clone, se escriben **nuevas** micropartitions solo para los cambios.
- Súper útil para: ambientes de dev/test, backups instantáneos, análisis what-if.

```sql
CREATE TABLE ventas_dev CLONE ventas;
CREATE SCHEMA dev CLONE prod;
CREATE DATABASE test_bd CLONE prod_bd;

-- Combinado con Time Travel: clonar un estado pasado
CREATE TABLE ventas_ayer CLONE ventas AT(OFFSET => -86400);
```

> 💡 Tip: los clones **NO incluyen el historial de Time Travel** de la tabla original. El clone empieza su propio Time Travel.

### 🔄 Replication

Copia de datos entre cuentas Snowflake (misma organización) para:
- **Disaster recovery**
- **Proximidad geográfica** (latencia baja a usuarios regionales)
- **Business continuity** con failover/failback (Business Critical+)

```sql
-- En cuenta primaria
ALTER DATABASE prod_bd ENABLE REPLICATION TO ACCOUNTS myorg.secundaria;

-- En cuenta secundaria
CREATE DATABASE prod_bd AS REPLICA OF myorg.primaria.prod_bd;
ALTER DATABASE prod_bd REFRESH;
```

**Replication vs Cloning:**
- **Clone:** dentro de la misma cuenta (dev/test)
- **Replication:** entre cuentas (DR, geográfica)

---

## 6.2 Capacidades de Data Sharing

### 🏷️ Tipos de cuenta (para sharing)

| Cuenta | Descripción |
|---|---|
| **Provider** | Cuenta que crea y ofrece shares |
| **Consumer** | Cuenta que recibe y consume shares |
| **Reader account** | Cuenta "light" creada por un provider para consumers que NO tienen Snowflake. El provider paga el cómputo. |

### 🎁 ¿Qué es Secure Data Sharing?

Compartir datos entre cuentas de Snowflake **SIN copiar ni mover** los datos. El consumer ve los datos en tiempo real a medida que el provider los actualiza.

**Características:**
- ✅ Sin duplicación (cero storage adicional para el consumer).
- ✅ Sin ETL (cero latencia, cambios visibles en minutos).
- ✅ El consumer usa su **propio warehouse** para las queries.
- ⚠️ Solo funciona **dentro de la misma región y mismo cloud** (para cross-region hay **replication** de shares).

### 📦 ¿Qué se puede compartir?

Dentro de un share puedes incluir:
- Bases de datos, esquemas, tablas y vistas
- **Secure views, secure UDFs, secure materialized views**
- **Directory tables** y tablas externas

⚠️ No puedes compartir directamente: stages, tareas, pipes, streams.

### 🔧 Crear y administrar shares (DDL)

```sql
-- 1. Crear un share (solo ACCOUNTADMIN o rol con CREATE SHARE)
CREATE SHARE ventas_share;

-- 2. Conceder privilegios al share
GRANT USAGE ON DATABASE prod_bd TO SHARE ventas_share;
GRANT USAGE ON SCHEMA prod_bd.public TO SHARE ventas_share;
GRANT SELECT ON TABLE prod_bd.public.ventas TO SHARE ventas_share;

-- 3. Agregar consumidores
ALTER SHARE ventas_share ADD ACCOUNTS = cuenta_consumidor1, cuenta_consumidor2;

-- 4. (En el consumer) Crear BD desde el share
CREATE DATABASE ventas_compartidas FROM SHARE provider.ventas_share;

-- Ver shares
SHOW SHARES;
SHOW SHARES IN ACCOUNT;
DESC SHARE ventas_share;
```

**Privilegios necesarios:**
- `CREATE SHARE` (cuenta) para crear shares → por default solo ACCOUNTADMIN.
- `USAGE` y `SELECT` sobre objetos del share.

### 🛍️ Snowflake Marketplace

Plataforma pública donde **providers** ofrecen datasets y servicios a **cualquier cliente de Snowflake** globalmente.
- **Gratuitos** o **de pago** (facturado por Snowflake).
- Todo cliente de Snowflake puede explorarlo y suscribirse con 1 click.
- Ejemplos: Weather Source, S&P Global, Foursquare, Nasdaq.

### 🔁 Data Exchange (privado)

Versión **privada** del Marketplace: tu organización (ej. un ecosistema de partners) comparte datasets solo con miembros invitados.
- Tú controlas quién entra.
- Mismo modelo de shares por debajo.

### 👥 Listings (listados)

Nueva forma moderna de empaquetar shares para Marketplace y Data Exchange:
- Un **listing** es una "vitrina" con nombre, descripción, documentación, imagen y un share detrás.
- Reemplaza gradualmente el modelo directo de shares en el Marketplace.

### 📊 Resumen de opciones de sharing

| Opción | Audiencia | Pago |
|---|---|---|
| **Direct Share** | Cuentas específicas que tú conoces | Gratis (consumer paga su compute) |
| **Marketplace (Listings)** | Público global | Puede ser gratis o pago |
| **Data Exchange (Listings)** | Miembros invitados de tu organización | Interno |

### 🧪 ¿Cuántos consumers caben en un share?

**Ilimitado** ✅ (una de las preguntas ejemplo de Snowflake).

---

## 🧪 Ejercicios prácticos

### Día 26 — Time Travel, Fail-safe, Cloning

```sql
-- 1. Crear y manipular
CREATE DATABASE lab_ct;
USE DATABASE lab_ct;
CREATE TABLE ventas (id INT, monto FLOAT);
INSERT INTO ventas VALUES (1,100),(2,200),(3,300);

-- 2. Time Travel
SELECT * FROM ventas AT(OFFSET => -10);  -- hace 10 seg

-- 3. DROP + UNDROP
DROP TABLE ventas;
SHOW TABLES HISTORY LIKE 'ventas';   -- Ver tablas dropeadas
UNDROP TABLE ventas;
SELECT * FROM ventas;  -- ¡restaurada!

-- 4. Ver Time Travel actual
SELECT DATA_RETENTION_TIME_IN_DAYS
FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME='VENTAS';

-- 5. Configurar Time Travel (si tienes Enterprise)
ALTER TABLE ventas SET DATA_RETENTION_TIME_IN_DAYS = 7;

-- 6. CLONE zero-copy
CREATE TABLE ventas_dev CLONE ventas;
INSERT INTO ventas_dev VALUES (4,400);
SELECT * FROM ventas;       -- Original intacta
SELECT * FROM ventas_dev;   -- Con la fila nueva

-- 7. Clonar un estado pasado
CREATE TABLE ventas_antes_drop CLONE ventas AT(OFFSET => -300);
```

### Día 27 — Data Sharing

```sql
-- (Necesitas 2 cuentas de prueba para ver el flujo completo; si solo tienes una,
--  practica los comandos DDL.)

USE ROLE ACCOUNTADMIN;

-- 1. Crear share
CREATE OR REPLACE SHARE compartido_ventas;

-- 2. Conceder privilegios
GRANT USAGE ON DATABASE lab_ct TO SHARE compartido_ventas;
GRANT USAGE ON SCHEMA lab_ct.public TO SHARE compartido_ventas;
GRANT SELECT ON TABLE lab_ct.public.ventas TO SHARE compartido_ventas;

-- 3. Ver el share
SHOW SHARES;
DESC SHARE compartido_ventas;

-- 4. Agregar consumidor (necesitas otra cuenta)
-- ALTER SHARE compartido_ventas ADD ACCOUNTS = myorg.consumidor_cuenta;

-- 5. (En el CONSUMER) crear BD desde el share
-- USE ROLE ACCOUNTADMIN;
-- CREATE DATABASE ventas_shared FROM SHARE provider_cuenta.compartido_ventas;
-- SELECT * FROM ventas_shared.public.ventas;

-- 6. Revocar acceso
ALTER SHARE compartido_ventas REMOVE ACCOUNTS = myorg.consumidor_cuenta;

-- 7. Ver el Marketplace (desde Snowsight: Data Products → Marketplace)
```

---

## ❓ Preguntas de práctica del Dominio 6

**1.** ¿Cuánto dura Fail-safe en Snowflake?
- A) 1 día, configurable
- B) 7 días fijos ✅
- C) 14 días
- D) Depende de la edición

**2.** ¿Qué tipo de tabla NO tiene Fail-safe?
- A) Permanent
- B) Transient ✅
- C) External
- D) A y B

**3.** ¿Cuál es el Time Travel máximo en Snowflake Enterprise?
- A) 1 día
- B) 7 días
- C) 30 días
- D) 90 días ✅

**4.** El comando UNDROP recupera...
- A) Solo tablas
- B) Tablas, esquemas y bases de datos ✅
- C) Tablas y usuarios
- D) Cualquier objeto dropeado

**5.** Zero-copy cloning consume storage al crearse:
- A) Sí, duplica los datos
- B) No, solo apunta a las mismas micropartitions ✅
- C) Solo si la tabla origen es mayor a 1 GB
- D) Solo en Enterprise

**6.** ¿Cuántas cuentas consumer puede tener un SHARE?
- A) 10
- B) 100
- C) 1000
- D) Ilimitado ✅

**7.** ¿Qué objeto NO se puede incluir en un share?
- A) Tablas
- B) Vistas seguras
- C) Streams ✅
- D) External tables

**8.** Para compartir datos con alguien que NO tiene cuenta Snowflake:
- A) Exportar a CSV
- B) Crear una Reader Account ✅
- C) No se puede
- D) Exponer una API REST

**9.** ¿Qué diferencia hay entre cloning y replication?
- A) Clone es entre cuentas, replication dentro de la misma
- B) Clone es dentro de la misma cuenta, replication entre cuentas ✅
- C) Son lo mismo
- D) Replication duplica storage; clone no

**10.** Tri-Secret Secure requiere la edición:
- A) Standard
- B) Enterprise
- C) Business Critical ✅
- D) VPS

---

## 🎯 Checklist antes de pasar a la semana final

- [ ] Sé de memoria: Time Travel (1–90 d) + Fail-safe (7 d fijo, no accesible)
- [ ] Entiendo que Transient/Temporary NO tienen Fail-safe
- [ ] Puedo hacer UNDROP y consultar con AT(OFFSET/TIMESTAMP)
- [ ] Entiendo que clone es zero-copy y no duplica storage
- [ ] Diferencio Direct Share / Marketplace / Data Exchange
- [ ] Sé qué objetos se pueden compartir y cuáles no
- [ ] Conozco Reader Accounts para consumers sin Snowflake

¡Listo para los simulacros! 🎯
