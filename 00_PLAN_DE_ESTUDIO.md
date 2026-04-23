# 🎯 Plan de Estudio SnowPro Core COF-C02 — 32 días

**Fecha inicio:** Martes 21 de abril de 2026
**Fecha del examen:** Sábado 23 de mayo de 2026
**Horas recomendadas/día:** 1.5 a 2 horas (45 min teoría + 45–60 min práctica)
**Ritmo:** Sostenible, con 1 día de descanso por semana

---

## 📊 Distribución por dominio (según peso oficial del examen)

| Dominio | Peso | Días de estudio |
|---|---|---|
| 1. Arquitectura y características | **25%** | 5 días |
| 2. Acceso y seguridad | **20%** | 4 días |
| 3. Rendimiento | **15%** | 3 días |
| 4. Carga y descarga de datos | **10%** | 2 días |
| 5. Transformaciones de datos | **20%** | 4 días |
| 6. Protección de datos y Data Sharing | **10%** | 2 días |
| Quizzes semanales (repasos) | — | 4 días |
| Simulacros finales + repaso | — | 5 días |
| Descansos | — | 3 días |
| **TOTAL** | | **32 días** |

---

## 🗓️ SEMANA 1 — Fundamentos y Arquitectura (Dominio 1)

### Mar 21 abr · Día 1 — Arquitectura de 3 capas
- Leer `01_dominio1_arquitectura.md` **sección 1.1**
- Crear la cuenta de prueba de 30 días de Snowflake
- Hands-on: Entrar a Snowsight, explorar la UI
- **Concepto clave:** Storage / Compute / Cloud Services escalan independientes

### Mié 22 abr · Día 2 — Herramientas e interfaces
- Leer `01_dominio1_arquitectura.md` **sección 1.2**
- Instalar **SnowSQL** en tu máquina
- Instalar la extensión de Snowflake para VS Code (opcional)
- Hands-on: Conectarte por SnowSQL, correr `SELECT CURRENT_VERSION();`

### Jue 23 abr · Día 3 — Objetos de Snowflake (parte 1)
- Leer `01_dominio1_arquitectura.md` **sección 1.3** hasta "Tipos de stages"
- Hands-on: Crear BD, esquema, los 4 tipos de tabla, ver con `SHOW TABLES`
- **Concepto clave:** Permanent vs Transient vs Temporary vs External

### Vie 24 abr · Día 4 — Objetos de Snowflake (parte 2)
- Leer `01_dominio1_arquitectura.md` **sección 1.3** (resto: UDFs, streams, tasks, pipes)
- Hands-on: Crear un UDF sencillo, una secuencia, ver `SHOW STREAMS`, `SHOW TASKS`
- **Concepto clave:** Diferencia stream / task / pipe

### Sáb 25 abr · Día 5 — Almacenamiento (micropartitions)
- Leer `01_dominio1_arquitectura.md` **sección 1.4**
- Hands-on: `SYSTEM$CLUSTERING_INFORMATION`, consultar `SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS`
- **Concepto clave:** 50–500 MB comprimidos, pruning, clustering

### Dom 26 abr · Día 6 — 🛋️ DESCANSO ACTIVO
- Sin estudio formal. Si quieres: ver un video corto de 30 min sobre Snowflake.
- Descansar la mente ayuda a consolidar lo aprendido.

### Lun 27 abr · Día 7 — 📝 QUIZ Dominio 1
- Responder las preguntas de práctica al final de `01_dominio1_arquitectura.md`
- Revisar el checklist final del documento
- **Si tienes 4+ errores:** re-lee la sección correspondiente antes de seguir

---

## 🗓️ SEMANA 2 — Acceso y Seguridad (Dominio 2)

### Mar 28 abr · Día 8 — Autenticación y seguridad de red
- Leer `02_dominio2_seguridad.md` **sección 2.1**
- Hands-on: Crear una NETWORK POLICY de prueba
- **Concepto clave:** MFA, key-pair auth, SSO con SAML

### Mié 29 abr · Día 9 — Control de acceso (RBAC)
- Leer `02_dominio2_seguridad.md` **sección 2.2** hasta "Roles del sistema"
- Hands-on: Dibujar la jerarquía de roles con papel y lápiz (¡sí, funciona!)
- **Concepto clave:** DAC + RBAC, los 5 roles del sistema

### Jue 30 abr · Día 10 — Grants y herencia de roles
- Leer `02_dominio2_seguridad.md` **sección 2.2** (resto)
- Hands-on: Crear rol, usuario, hacer GRANT, probar `USE ROLE`, anidar roles
- **Concepto clave:** Para SELECT se necesita USAGE en BD + SCHEMA + USAGE warehouse + SELECT tabla

### Vie 1 may · Día 11 — 🇨🇴 DÍA DEL TRABAJO (festivo)
- Descanso opcional o práctica libre
- Si quieres estudiar algo ligero: explorar el Snowflake Marketplace

### Sáb 2 may · Día 12 — Gobierno de datos
- Leer `02_dominio2_seguridad.md` **sección 2.3**
- Hands-on: Crear SECURE VIEW, masking policy, row access policy, consultar ACCESS_HISTORY
- **Concepto clave:** INFORMATION_SCHEMA (real-time, 7 d) vs ACCOUNT_USAGE (latencia, 1 año)

### Dom 3 may · Día 13 — 🛋️ DESCANSO

### Lun 4 may · Día 14 — 📝 QUIZ Dominio 2
- Responder preguntas de práctica del documento
- Re-practicar grants en la cuenta de prueba
- Si tienes dudas, repasar sección específica

---

## 🗓️ SEMANA 3 — Rendimiento y Carga (Dominios 3 y 4)

### Mar 5 may · Día 15 — Query profile y caches
- Leer `03_dominio3_rendimiento.md` **secciones 3.1 y 3.4** (caches)
- Hands-on: Ejecutar queries en sample data, ver Query Profile, probar result cache
- **Concepto clave:** 3 tipos de caché y su TTL

### Mié 6 may · Día 16 — Warehouses y escalado
- Leer `03_dominio3_rendimiento.md` **secciones 3.2 y 3.3**
- Hands-on: Redimensionar warehouse, crear resource monitor
- **Concepto clave:** Scaling UP (query lenta) vs Scaling OUT (concurrencia)

### Jue 7 may · Día 17 — Optimización de queries
- Leer `03_dominio3_rendimiento.md` **sección 3.4** (resto: vistas materializadas, SOS, clustering)
- Hands-on: Probar `SAMPLE`, explorar clustering keys
- **Concepto clave:** Vistas materializadas vs SOS vs clustering (cuándo usar cada uno)

### Vie 8 may · Día 18 — Carga de datos
- Leer `04_dominio4_carga_descarga.md` **secciones 4.1 y 4.2**
- Hands-on: Crear un stage, un file format, cargar un CSV con PUT + COPY INTO
- **Concepto clave:** Archivos de 100–250 MB comprimidos

### Sáb 9 may · Día 19 — Descarga de datos
- Leer `04_dominio4_carga_descarga.md` **secciones 4.3 y 4.4**
- Hands-on: Descargar datos con COPY INTO @stage, bajar con GET, crear un PIPE
- **Concepto clave:** Diferencia Snowpipe vs COPY, external tables

### Dom 10 may · Día 20 — 🛋️ DESCANSO

### Lun 11 may · Día 21 — 📝 QUIZ Dominios 3 y 4
- Responder preguntas de práctica de ambos documentos
- Re-hacer el flujo PUT → COPY → GET de memoria, sin mirar

---

## 🗓️ SEMANA 4 — Transformaciones y Protección (Dominios 5 y 6)

### Mar 12 may · Día 22 — Datos estándar y UDFs
- Leer `05_dominio5_transformaciones.md` **sección 5.1** (hasta procedimientos)
- Hands-on: Crear UDF en SQL y en Python, probar SAMPLE
- **Concepto clave:** UDF vs UDTF vs External Function vs Stored Procedure

### Mié 13 may · Día 23 — Streams y Tasks (CDC)
- Leer `05_dominio5_transformaciones.md` **sección 5.1** (streams y tasks)
- Hands-on: Crear stream, insertar datos, crear task con schedule, consumir el stream
- **Concepto clave:** Patrón Stream + Task para CDC, modo serverless vs warehouse

### Jue 14 may · Día 24 — Datos semiestructurados
- Leer `05_dominio5_transformaciones.md` **sección 5.2**
- Hands-on: Cargar JSON en VARIANT, usar `:` y `[]`, FLATTEN, LATERAL FLATTEN
- **Concepto clave:** VARIANT hasta 16 MB, predicados de tipo, funciones de objetos/arrays

### Vie 15 may · Día 25 — Datos no estructurados
- Leer `05_dominio5_transformaciones.md` **sección 5.3**
- Hands-on: Crear stage con DIRECTORY, consultar `DIRECTORY(@stage)`, generar pre-signed URL
- **Concepto clave:** Directory tables, tipos de URL (scoped, file, pre-signed)

### Sáb 16 may · Día 26 — Time Travel, Fail-safe, Clonación
- Leer `06_dominio6_proteccion_sharing.md` **sección 6.1**
- Hands-on: DROP + UNDROP, query con `AT(OFFSET => -60)`, clonar una BD completa
- **Concepto clave:** Time Travel (0–90 días) + Fail-safe (7 días fijo, no accesible)

### Dom 17 may · Día 27 — Data Sharing
- Leer `06_dominio6_proteccion_sharing.md` **sección 6.2**
- Hands-on: Crear un SHARE, agregarle una tabla, simular cuenta consumidora
- **Concepto clave:** Direct Share vs Marketplace vs Data Exchange, tipos de cuenta

### Lun 18 may · Día 28 — 📝 QUIZ Dominios 5 y 6
- Responder preguntas de práctica de ambos documentos
- Empezar a revisar `07_cheatsheet.md`

---

## 🏁 SEMANA FINAL — Simulacros y repaso (19–23 may)

### Mar 19 may · Día 29 — 🎯 SIMULACRO #1
- Hacer el examen de práctica oficial de Snowflake (gratis desde el portal de certificaciones)
- Apuntar los temas donde fallaste
- Revisar `08_preguntas_practica.md` — primer bloque

### Mié 20 may · Día 30 — 🔧 Repaso de flaquezas
- Re-leer solo las secciones donde tuviste errores en el simulacro
- Hacer los ejercicios hands-on de esas secciones otra vez
- No estudies cosas nuevas hoy, solo consolida

### Jue 21 may · Día 31 — 🎯 SIMULACRO #2
- Hacer el segundo bloque de `08_preguntas_practica.md`
- Repasar el `07_cheatsheet.md` completo dos veces
- Apuntar los 5–10 temas que todavía sientes flojos

### Vie 22 may · Día 32 — 💆 Repaso ligero y preparación
- Repasar SOLO el cheatsheet `07_cheatsheet.md`
- Revisar logística del examen (hora, enlace, documento de identidad)
- Preparar espacio físico (webcam, cuarto limpio, internet estable)
- **Dormir temprano** (NO estudiar en la noche)

### Sáb 23 may · Día 33 — 🎓 DÍA DEL EXAMEN
- Desayuna bien
- Llega al examen 15–30 min antes (o conéctate temprano si es online)
- Respira hondo, lee cada pregunta dos veces
- Si una pregunta te traba, **márcala y sigue**, vuelve al final
- **Tienes 115 min para ~100 preguntas** → ~1 min/pregunta
- Objetivo: 750/1000 (75%)

---

## 🧠 Método de estudio recomendado (cada día)

1. **Lectura activa (30–45 min):** Lee el tema, subraya, toma notas a mano (mejora retención).
2. **Hands-on (45–60 min):** Reproduce los ejercicios. **Esto es lo más importante**. El examen tiene preguntas muy prácticas.
3. **Retro (5–10 min):** Cierra la guía y explícate el tema a ti mismo. Si no puedes → relee.
4. **Anota dudas:** lleva un documento con "cosas que no entiendo" para revisar en los quizzes semanales.

---

## ⚠️ Errores comunes a evitar en el examen

- ❌ Confundir **Fail-safe (7 días, no accesible)** con **Time Travel (accesible por el usuario)**.
- ❌ Creer que **SYSADMIN administra usuarios** → eso lo hace **USERADMIN** (o SECURITYADMIN/ACCOUNTADMIN).
- ❌ Olvidar que **las tablas temporales NO tienen Fail-safe** y solo existen en la sesión.
- ❌ Confundir **scaling up (cambiar tamaño)** con **scaling out (multi-cluster)**.
- ❌ Pensar que la **caché de resultados** dura eternamente → son **24 h** (extensible hasta 31 días si se reusa).
- ❌ Olvidar que **Snowpipe factura por archivos procesados + cómputo serverless**, NO por tiempo de warehouse.
- ❌ Pensar que las **tablas TRANSIENT son temporales** → persisten, solo que sin Fail-safe.

---

## 📝 Formato del examen

- **Duración:** 115 minutos
- **Preguntas:** ~100 (opción múltiple y múltiple respuesta)
- **Puntaje para aprobar:** 750/1000 (75%)
- **Idioma:** Inglés (recomendado) o japonés
- **Modalidad:** Online proctored o centro de pruebas
- **Costo:** USD 175
- **Validez:** 2 años desde la fecha de emisión

---

## 📚 Orden de los archivos en este paquete

1. `00_PLAN_DE_ESTUDIO.md` — Este archivo (hoja de ruta)
2. `01_dominio1_arquitectura.md` — Arquitectura (25%)
3. `02_dominio2_seguridad.md` — Seguridad (20%)
4. `03_dominio3_rendimiento.md` — Rendimiento (15%)
5. `04_dominio4_carga_descarga.md` — Carga/Descarga (10%)
6. `05_dominio5_transformaciones.md` — Transformaciones (20%)
7. `06_dominio6_proteccion_sharing.md` — Protección y Sharing (10%)
8. `07_cheatsheet.md` — Resumen final ultra-condensado
9. `08_preguntas_practica.md` — Preguntas de práctica adicionales

---

## ✅ Recursos oficiales complementarios

1. **Snowflake University** (gratis, con tu cuenta de Snowflake Community)
   - Curso "Getting Started with Snowflake" (módulos 1–10)
   - Snowflake University Level Up series
2. **Documentación oficial:** https://docs.snowflake.com
3. **Cuenta de prueba de 30 días** (gratis, con $400 en créditos): https://signup.snowflake.com
4. **Examen de práctica oficial** (gratis): desde el portal de certificaciones SnowPro

---

**¡Tienes 4 semanas completas para dominar esto, vamos con paso firme! 🚀**

> "La constancia supera al talento cuando el talento no es constante."
