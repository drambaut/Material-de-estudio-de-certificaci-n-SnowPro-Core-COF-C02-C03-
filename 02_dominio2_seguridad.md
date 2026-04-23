# 🔐 Dominio 2 — Acceso y Seguridad de Cuentas

**Peso en el examen: 20%**
**Días del plan: 3 y 4**

---

## 2.1 Principios de seguridad

### 🌐 Network policies

Controlan qué IPs pueden conectarse a tu cuenta de Snowflake.
- Se crean con `CREATE NETWORK POLICY` y listas de IPs permitidas/bloqueadas.
- Se aplican a nivel de **cuenta o usuario**.
- Ejemplo:
  ```sql
  CREATE NETWORK POLICY mi_politica
    ALLOWED_IP_LIST = ('192.168.1.0/24','10.0.0.1')
    BLOCKED_IP_LIST = ('1.2.3.4');

  ALTER ACCOUNT SET NETWORK_POLICY = mi_politica;
  ```

### 🔑 Métodos de autenticación

| Método | Descripción | Uso típico |
|---|---|---|
| **Usuario + Contraseña** | Básico | Dev/testing (no recomendado solo) |
| **MFA (Multi-Factor Auth)** | Contraseña + Duo Mobile | **Obligatorio para ACCOUNTADMIN** |
| **Federated auth (SSO)** | SAML 2.0 con IdP externo (Okta, Azure AD, ADFS) | Empresas con IdP |
| **Key-pair authentication** | RSA 2048-bit pública/privada | Integraciones programáticas, CI/CD |
| **OAuth** | Token OAuth 2.0 | Apps web/móviles |
| **SCIM** | Aprovisionamiento automático | Sincronización de usuarios desde IdP |

> 🎯 **Pregunta típica:** "¿Qué método se recomienda para conexiones programáticas/scripts?" → **Key-pair authentication**.

### ⚠️ Importante
- Snowflake **aplica MFA automáticamente** a usuarios con ACCOUNTADMIN/ORGADMIN.
- Contraseñas + MFA NO son lo mismo que SSO. MFA es un segundo factor; SSO redirige toda la auth al IdP.

---

## 2.2 Entidades y roles: el control de acceso

### 🎭 Snowflake combina DAC + RBAC

- **DAC (Discretionary Access Control):** cada objeto tiene un **owner**. El owner puede conceder privilegios sobre su objeto.
- **RBAC (Role-Based Access Control):** los privilegios NO se asignan a usuarios directamente, se asignan a **roles**, y los roles se asignan a usuarios.

```
Privilegio  →  Rol  →  Usuario
```

Un usuario puede tener varios roles pero solo **1 activo por sesión** (cambias con `USE ROLE`).

### 👑 Jerarquía de roles del sistema (¡MEMORIZAR!)

```
                  ORGADMIN
                     │
                ACCOUNTADMIN
               /      │      \
      SECURITYADMIN   │    SYSADMIN
            │         │        │
       USERADMIN      │   (roles personalizados,
            │         │    dueños de BDs)
            │         │
            └────── PUBLIC ◄─── todos los usuarios
```

### 📋 Qué hace cada rol del sistema

| Rol | Función principal |
|---|---|
| **ORGADMIN** | Administra la **organización** (crear cuentas, listar, regiones) |
| **ACCOUNTADMIN** | **Rol más alto** dentro de una cuenta. Todo-poderoso. Pocos usuarios deben tenerlo. |
| **SECURITYADMIN** | Administra **roles y concesiones** de toda la cuenta (puede hacer `GRANT`/`REVOKE` globalmente) |
| **USERADMIN** | Crea y administra **usuarios y roles** (pero no concesiones a nivel cuenta) |
| **SYSADMIN** | Recomendado como **dueño de bases de datos y warehouses**. Crea BDs, warehouses, etc. |
| **PUBLIC** | Rol automático que tienen TODOS los usuarios. Para objetos accesibles por todos. |

### ⚠️ Puntos que confunden en el examen
- **USERADMIN** crea y gestiona usuarios y roles, pero NO concede privilegios de objetos.
- **SECURITYADMIN** hereda USERADMIN + puede gestionar grants globalmente.
- **SYSADMIN** debería ser el dueño de las BDs, NO ACCOUNTADMIN.
- Los objetos creados con ACCOUNTADMIN solo los ven los que tienen ACCOUNTADMIN → **mala práctica**.

### 🎁 Privilegios y su concesión

Tipos de privilegios:
- **Global (cuenta):** `CREATE WAREHOUSE`, `CREATE DATABASE`, `MANAGE GRANTS`, `MONITOR`
- **De objeto:** `USAGE`, `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `REFERENCES`, `OWNERSHIP`, `MODIFY`, `OPERATE`

Comandos clave:
```sql
-- Conceder
GRANT SELECT ON TABLE ventas TO ROLE analista;
GRANT USAGE ON DATABASE lab TO ROLE analista;
GRANT USAGE ON SCHEMA lab.public TO ROLE analista;

-- Con cascada (todas las tablas existentes)
GRANT SELECT ON ALL TABLES IN SCHEMA lab.public TO ROLE analista;

-- Aplicado también a objetos futuros
GRANT SELECT ON FUTURE TABLES IN SCHEMA lab.public TO ROLE analista;

-- Revocar
REVOKE SELECT ON TABLE ventas FROM ROLE analista;

-- Transferir ownership
GRANT OWNERSHIP ON TABLE ventas TO ROLE nuevo_dueño REVOKE CURRENT GRANTS;
```

### 🧬 Herencia de roles

Los roles pueden anidarse. Al hacer `GRANT ROLE A TO ROLE B`, el rol B hereda todos los privilegios de A.

```sql
GRANT ROLE analista TO ROLE manager;
-- Ahora manager tiene todo lo que tiene analista, más lo suyo
```

> 🎯 **Pregunta típica:** "Un usuario tiene el rol X que no puede ver la tabla Y. ¿Qué comando lo soluciona?"
> → Probablemente falte el `GRANT USAGE` en la BD/esquema, no solo el `SELECT` en la tabla.

### 🔍 Ver concesiones

```sql
SHOW GRANTS TO ROLE analista;       -- Qué tiene el rol
SHOW GRANTS TO USER pepe;           -- Qué roles tiene el user
SHOW GRANTS OF ROLE analista;       -- A quién está asignado
SHOW GRANTS ON TABLE ventas;        -- Quién tiene acceso a la tabla
```

---

## 2.3 Gobierno de datos en Snowflake

### 🏢 Cuentas y organizaciones

- **Organización:** agrupa varias cuentas de Snowflake (incluso en distintas clouds/regiones).
- **Cuenta:** unidad de facturación y configuración. 1 URL única, 1 región, 1 cloud.

### 🛡️ Vistas seguras (Secure Views)

```sql
CREATE SECURE VIEW ventas_publicas AS
  SELECT fecha, producto, SUM(monto) FROM ventas GROUP BY 1,2;
```

- **Ocultan la definición** del SQL a quien no tenga el privilegio.
- El optimizador de Snowflake **NO aplica ciertas optimizaciones** que podrían exponer datos subyacentes.
- Un poco más lentas, pero seguras.

### 🔒 Funciones seguras (Secure UDFs)
Similar a vistas seguras pero para UDFs.
```sql
CREATE SECURE FUNCTION ...
```

### 📚 Information Schema vs Account Usage (¡PREGUNTA FIJA!)

| Aspecto | INFORMATION_SCHEMA | ACCOUNT_USAGE |
|---|---|---|
| Ubicación | Por BD | BD `SNOWFLAKE` (global) |
| Retención | **7 días** (max) | **Hasta 1 año** |
| Latencia | **Tiempo real** | **45 min – 3 horas** |
| Filas borradas | No incluye | Incluye (marcadas) |
| Uso | Operativo, sesión actual | Auditoría, análisis histórico |

### 📜 Access History

Vista `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` que registra:
- **Lecturas (read):** qué objetos leyó cada query (incluyendo columnas).
- **Escrituras (write):** qué objetos fueron modificados.
- Útil para **auditoría, compliance, linaje de datos**.

### 🎭 Seguridad a nivel fila/columna

#### Masking policies (column-level)
Enmascaran valores de una columna según el rol que consulta.
```sql
CREATE MASKING POLICY ocultar_ssn AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('HR_ROLE') THEN val
    ELSE '***-**-****'
  END;

ALTER TABLE empleados MODIFY COLUMN ssn SET MASKING POLICY ocultar_ssn;
```

#### Row access policies (row-level)
Filtran qué filas ve cada rol.
```sql
CREATE ROW ACCESS POLICY ver_mi_region AS (region STRING) RETURNS BOOLEAN ->
  CURRENT_ROLE() = 'ADMIN' OR region = CURRENT_REGION();

ALTER TABLE ventas ADD ROW ACCESS POLICY ver_mi_region ON (region);
```

Ambas requieren **Enterprise edition**.

### 🏷️ Object tags

Etiquetas (key-value) aplicadas a objetos para clasificación y gobierno.
```sql
CREATE TAG clasificacion;
ALTER TABLE clientes SET TAG clasificacion = 'PII';
```
Útiles para:
- Rastrear datos sensibles (PII, PHI)
- Aplicar masking policies automáticamente con **tag-based masking**
- Cost tracking por proyecto/equipo

---

## 🧪 Ejercicios prácticos (Día 3–4)

```sql
-- Ejercicio 1: Crear un rol, usuario y grants
USE ROLE USERADMIN;
CREATE ROLE analista;
CREATE USER ana PASSWORD='Temp1234!' DEFAULT_ROLE=analista MUST_CHANGE_PASSWORD=TRUE;
GRANT ROLE analista TO USER ana;

USE ROLE SECURITYADMIN;
GRANT USAGE ON DATABASE snowpro_lab TO ROLE analista;
GRANT USAGE ON SCHEMA snowpro_lab.practica TO ROLE analista;
GRANT SELECT ON ALL TABLES IN SCHEMA snowpro_lab.practica TO ROLE analista;

-- Ejercicio 2: Ver la jerarquía
SHOW GRANTS TO ROLE analista;
SHOW GRANTS OF ROLE analista;

-- Ejercicio 3: Crear vista segura
USE ROLE SYSADMIN;
CREATE SECURE VIEW snowpro_lab.practica.vista_segura AS
  SELECT id, nombre FROM snowpro_lab.practica.permanente;
GRANT SELECT ON VIEW snowpro_lab.practica.vista_segura TO ROLE analista;

-- Ejercicio 4: Consultar access history
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE query_start_time > DATEADD(day, -1, CURRENT_TIMESTAMP)
LIMIT 10;
```

---

## ❓ Preguntas de práctica del Dominio 2

**1.** ¿Qué rol del sistema es necesario para crear un nuevo usuario?
- A) SYSADMIN
- B) USERADMIN ✅
- C) PUBLIC
- D) ORGADMIN

**2.** ¿Cuál es la latencia típica de una consulta a `ACCOUNT_USAGE`?
- A) Tiempo real
- B) 1 minuto
- C) 45 min – 3 horas ✅
- D) 24 horas

**3.** ¿Qué tipo de autenticación se recomienda para integraciones programáticas?
- A) Contraseña + MFA
- B) Key-pair authentication ✅
- C) SSO con SAML
- D) OAuth con popup

**4.** ¿Cuál de las siguientes NO es una edición de Snowflake? (distractor)
- A) Standard
- B) Enterprise
- C) Business Essential ✅
- D) Business Critical

**5.** Para que un rol pueda hacer SELECT en una tabla, requiere privilegios en... (marca todas)
- A) La base de datos (USAGE) ✅
- B) El esquema (USAGE) ✅
- C) La tabla (SELECT) ✅
- D) El warehouse (USAGE) ✅ — lo necesita para ejecutar

**6.** ¿En qué esquema consultarías los accesos a columnas sensibles de los últimos 6 meses?
- A) INFORMATION_SCHEMA.ACCESS_HISTORY
- B) SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ✅
- C) PUBLIC.QUERY_HISTORY
- D) ACCOUNTADMIN.AUDIT

**7.** Una vista segura (Secure View) se distingue porque:
- A) Está cifrada adicionalmente
- B) Oculta su definición SQL y deshabilita ciertas optimizaciones ✅
- C) Solo se puede consultar con MFA
- D) Requiere VPS

**8.** ¿Cuál rol es el más adecuado para ser dueño de bases de datos productivas?
- A) ACCOUNTADMIN
- B) SECURITYADMIN
- C) SYSADMIN ✅
- D) PUBLIC

---

## 🎯 Checklist antes de pasar al Día 5

- [ ] Puedo dibujar la jerarquía de roles del sistema
- [ ] Sé la diferencia entre USERADMIN y SECURITYADMIN
- [ ] Conozco los 6 métodos de autenticación
- [ ] Entiendo la diferencia entre INFORMATION_SCHEMA y ACCOUNT_USAGE
- [ ] Puedo crear un rol, usuario y conceder privilegios
- [ ] Diferencio masking policies de row access policies

¡Vamos con el rendimiento! 🚀
