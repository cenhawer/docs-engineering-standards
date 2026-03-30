# Convenciones de Nomenclatura — SQL Server

> **Audiencia:** equipo de desarrollo. Esta guía no cubre gestión de infraestructura, deployment ni administración de seguridad de base de datos — esas responsabilidades corresponden al equipo de DBA/Infra.
>
> **Cómo usar esta guía:** para diseñar un modelo nuevo, lee las secciones 1 → 3 → 4 → 5 en orden. Para verificar un nombre puntual, ve directo a la [tabla de prefijos](#55-tabla-de-prefijos-y-constraints--referencia-rápida). Los templates DDL ejecutables están en [sqlserver-ddl-templates.md](sqlserver-ddl-templates.md).

**Versión:** 1.0

---

## Contenido

1. [Reglas Generales](#1-reglas-generales)
2. [Schemas vs Prefijos de Dominio](#2-schemas-vs-prefijos-de-dominio)
3. [Tipos de Tablas](#3-tipos-de-tablas)
   - [3.1 Tablas Principales](#31-tablas-principales)
   - [3.2 Tablas de Relación](#32-tablas-de-relación-muchos-a-muchos)
   - [3.3 Tablas de Historial](#33-tablas-de-historial-hist_)
     - [3.3.1 Alternativa nativa: Temporal Tables](#331-alternativa-nativa-temporal-tables-system_versioning)
   - [3.4 Tablas de Log](#34-tablas-de-log-log_)
   - [3.5 Tablas de Catálogo](#35-tablas-de-catálogo-cat_)
4. [Campos](#4-campos)
   - [4.1 Auditoría](#41-auditoría)
   - [4.2 Clave Primaria](#42-clave-primaria)
   - [4.3 Clave Foránea](#43-clave-foránea)
   - [4.4 Soft Delete](#44-soft-delete-is_deleted)
   - [4.5 NVARCHAR vs VARCHAR](#45-tipos-de-texto-nvarchar-vs-varchar)
   - [4.6 Longitudes de NVARCHAR](#46-longitudes-de-nvarchar)
   - [4.7 Campo code](#47-campo-code-slug-legible)
   - [4.8 Tipos Numéricos](#48-tipos-numéricos)
   - [4.9 Estrategia NULL / NOT NULL](#49-estrategia-null--not-null)
5. [Índices y Constraints](#5-índices-y-constraints)
   - [5.1 Índice Común](#51-índice-común)
   - [5.2 Índice Único](#52-índice-único)
   - [5.3 Índice Compuesto](#53-índice-compuesto)
   - [5.4 Covering Index (INCLUDE)](#54-covering-index-include)
   - [5.5 Tabla de Prefijos y Constraints — Referencia Rápida](#55-tabla-de-prefijos-y-constraints--referencia-rápida)
   - [5.6 Índices en columnas FK](#56-índices-en-columnas-fk)
   - [5.7 Columnstore Index (analytics)](#57-columnstore-index-analytics)
   - [5.8 Particionamiento — Naming](#58-particionamiento--naming)
6. [Otros Objetos](#6-otros-objetos)
   - [6.1 Vistas](#61-vistas)
   - [6.2 Stored Procedures](#62-stored-procedures)
   - [6.3 Funciones](#63-funciones)
   - [6.4 Triggers](#64-triggers)
   - [6.5 Sequences](#65-sequences)
   - [6.6 Tablas Temporales en SPs](#66-tablas-temporales-en-sps)
7. [Palabras Reservadas](#7-palabras-reservadas-en-sql-server)
8. [Scripts DDL y DML](#8-scripts-ddl-y-dml)
9. [Anti-patterns](#9-anti-patterns)
10. [Referencias](#10-referencias)

**Apéndice** *(lectura recomendada según tipo de proyecto)*
- [Apéndice A — Arquitectura de Integración](#apéndice-a--arquitectura-de-integración)

---

## Contexto

Un esquema nombrado de forma inconsistente genera deuda técnica silenciosa: queries confusos, errores por suposiciones incorrectas y fricción al incorporar nuevos integrantes al equipo. Esta guía establece un único criterio para todos los proyectos que usen SQL Server, con decisiones explícitas y justificadas.

---

## 1. Reglas Generales

### 1.1 Formato y capitalización

| Regla | Valor |
|-------|-------|
| Formato | `snake_case` |
| Capitalización | minúsculas |
| Idioma | decisión del equipo técnico — un solo idioma en todo el modelo |
| Número | singular |
| Prefijos húngaros | prohibidos (`tbl_`, `vw_`) |
| Prefijo `sp_` | prohibido — razón técnica, no de estilo (ver [Stored Procedures](#62-stored-procedures)) |

> **Prefijos húngaros (`tbl_`, `vw_`):** añaden información redundante al nombre. El tipo de objeto ya se conoce por contexto. Se prohíben porque generan ruido sin valor.
>
> **El prefijo `sp_`** está en una categoría distinta: es un **prefijo reservado por SQL Server** para procedimientos del sistema en `master`. Usarlo en procedimientos propios obliga al motor a buscar primero en `master`, añadiendo overhead y riesgo de colisión. El prefijo correcto para procedimientos propios es `usp_`.

> **Nota sobre el idioma:** puede ser inglés o español, según el contexto del negocio y el equipo. Lo que no es negociable es la consistencia: **un solo idioma en todo el modelo**. No se permiten modelos mixtos. Los ejemplos de esta guía usan inglés como referencia.

**Por qué `snake_case`:** es más legible en queries largos y portable entre motores (PostgreSQL, MySQL, SQLite). Si el proyecto usa un ORM, asegurarse de configurarlo para respetar `snake_case` — de lo contrario el ORM generará nombres con su convención por defecto (normalmente PascalCase), divergiendo del modelo. Ver nota en [Scripts DDL y DML](#8-scripts-ddl-y-dml).

```
-- Correcto
person
ofac_list
transaction_id

-- Incorrecto
tbl_People          -- prefijo húngaro + plural
ListaOFAC           -- PascalCase + acrónimo sin separación
transactionId       -- camelCase
sp_get_person       -- prefijo reservado por SQL Server
```

### 1.2 Convenciones Semánticas

Reglas de nomenclatura que aplican a toda columna y tabla del modelo.

| Regla | Correcto | Incorrecto |
|-------|----------|------------|
| Sin abreviaciones ambiguas | `document_number` | `doc_num`, `nro_doc` |
| Sin nombres genéricos | `sale_price`, `product_code` | `value`, `data`, `info` |
| Nombres del dominio del negocio | `purchase_order`, `block_list` | `table1`, `misc_data` |
| Booleanos con prefijo semántico | `is_active`, `has_discount` | `active`, `discount`, `flag` |
| Fechas con sufijo de acción | `created_at`, `expiration_date` | `date1`, `fecha`, `date` |
| Montos en DECIMAL | `DECIMAL(18, 2)` | `FLOAT`, `REAL`, `MONEY` |
| NOT NULL por defecto | `full_name NVARCHAR(200) NOT NULL` | `full_name NVARCHAR(200) NULL` |
| Porcentajes como fracción | `tax_rate DECIMAL(5,4)` = `0.1600` | `tax_rate INT` = `16` sin documentar la unidad |

---

## 2. Schemas vs Prefijos de Dominio

Esta es una decisión de arquitectura que debe tomarse desde el inicio del proyecto y mantenerse consistente.

### Escenario A — Schema dedicado (recomendado en microservicios / DDD)

Cuando cada servicio o dominio tiene su propio schema, el schema es el identificador de contexto. **Los prefijos de dominio en el nombre de la tabla son innecesarios y redundantes.**

```
pld.person      pld.ofac_list
crm.person      crm.contract
```

> En microservicios y arquitecturas orientadas a DDD, **esta es la opción recomendada**. Cada servicio es dueño exclusivo de su schema. Los schemas deben registrarse en un diccionario oficial del proyecto para evitar abreviaciones improvisadas.

### Escenario B — Schema compartido (`dbo`)

Cuando múltiples dominios coexisten en el mismo schema, el prefijo de dominio es **obligatorio** para evitar colisiones y dar contexto.

```
dbo.pld_person      dbo.crm_person      dbo.gd_document
```

> Los prefijos deben estar registrados en un diccionario oficial. No se permiten abreviaciones improvisadas (`pl`, `pd`, `plds`). El diccionario es la única fuente de verdad.

> Las reglas de FKs entre dominios según el tipo de proyecto (monolito, monolito con schemas, microservicios) están en el [Apéndice A — Arquitectura de Integración](#apéndice-a--arquitectura-de-integración).

---

## 3. Tipos de Tablas

### 3.1 Tablas Principales

Representan entidades del negocio. Siempre incluyen campos de auditoría. Los campos de soft delete (`is_deleted`, `deleted_at`, `deleted_by`) son opcionales — se agregan únicamente cuando la entidad requiere eliminación lógica (ver criterios en [Soft Delete](#44-soft-delete-is_deleted)).

> Templates: [A.1.1 sin soft delete](sqlserver-ddl-templates.md#a11-tabla-principal--sin-soft-delete) · [A.1.2 con soft delete](sqlserver-ddl-templates.md#a12-tabla-principal--con-soft-delete)

### 3.2 Tablas de Relación (muchos a muchos)

Se nombran con ambas entidades en **orden alfabético por nombre completo**, incluyendo prefijos (`cat_`, `hist_`, etc.). Usan llave primaria compuesta.

```
person + role           → person_role
cat_country + person    → cat_country_person   (cat_country < person alfabéticamente)
```

> **Orden alfabético con prefijos:** el ordenamiento usa el nombre completo, no el nombre sin prefijo. Si `person` se relaciona con `cat_country`, el resultado es `cat_country_person` — `c` < `p`.

> **Estabilidad del nombre:** el nombre se fija al momento de crear la tabla y no cambia aunque una de las entidades sea renombrada en el futuro. Renombrarlo implicaría una migración que puede romper FKs y referencias existentes.

Las tablas de relación **no incluyen `updated_at`/`updated_by`** porque una relación no se modifica — se crea o se elimina. Un UPDATE sobre una tabla de relación no tiene semántica válida; si ocurre, es un error de diseño.

> Template: [A.1.3](sqlserver-ddl-templates.md#a13-tabla-de-relación-muchos-a-muchos)

### 3.3 Tablas de Historial (`hist_`)

Guardan el **estado anterior** de un registro de negocio ante cambios relevantes. Responden a: *"¿cómo estaba este registro antes de ser modificado?"*

- Espeja las columnas de negocio relevantes de la tabla principal — no los campos de auditoría
- No requieren campos de auditoría adicionales — la fecha del cambio es explícita (`recorded_at`)
- Pueden tener FK a la tabla principal
- Las escribe la lógica de negocio en cada UPDATE relevante

> Template: [A.1.4](sqlserver-ddl-templates.md#a14-tabla-de-historial-hist_--manual)

#### 3.3.1 Alternativa nativa: Temporal Tables (`SYSTEM_VERSIONING`)

SQL Server 2016+ incluye **Temporal Tables** como funcionalidad nativa. En lugar de escribir lógica de aplicación que inserta en `hist_`, el motor gestiona el historial automáticamente — sin triggers, sin lógica adicional, con soporte de consultas `AS OF` nativas.

**Comparación: `hist_` manual vs Temporal Tables**

| | `hist_` manual | Temporal Tables |
|---|---|---|
| **Soporte de motor** | Lógica de aplicación | El motor escribe el historial |
| **Sintaxis de consulta** | `JOIN` manual a `hist_` | `FOR SYSTEM_TIME AS OF` nativo |
| **Compatibilidad** | SQL Server 2008+ | SQL Server 2016+ |
| **Control del esquema de `hist_`** | Total | Limitado — el motor lo gestiona |
| **Purga de historial** | Manual con DELETE | `sp_cleanup_temporal_history` |

> **Cuándo usar Temporal Tables:** proyectos nuevos sobre SQL Server 2016+, cuando se necesita auditoría automática con mínimo código de aplicación. **Cuándo mantener `hist_` manual:** SQL Server 2014 o anterior, cuando el historial requiere columnas adicionales, o cuando el proyecto ya usa `hist_` de forma consolidada. El prefijo `hist_` aplica en ambos casos.

> **Incompatibilidad con Soft Delete:** Temporal Tables y soft delete son mutuamente excluyentes. Con `SYSTEM_VERSIONING` activo, usar `is_active` en lugar de `is_deleted` — el motor no permite eliminar filas mientras el versioning está activo.

> Template: [A.1.5](sqlserver-ddl-templates.md#a15-tabla-con-temporal-tables-system_versioning)

### 3.4 Tablas de Log (`log_`)

Registran **eventos o acciones** que ocurrieron en el sistema. Responden a: *"¿qué acciones se ejecutaron y cuándo?"*

- No espeja ninguna tabla — su estructura es propia del evento
- **Nunca llevan FK** — deben ser independientes del estado actual del negocio y no verse afectadas por eliminaciones
- Los IDs de otras entidades son referencias informativas, no restricciones referenciales
- Las escribe el sistema ante eventos técnicos o de dominio

**`hist_` vs `log_` — diferencias clave**

| | `hist_` | `log_` |
|---|---------|--------|
| **Pregunta que responde** | ¿Cómo era este dato antes? | ¿Qué acción ocurrió? |
| **Qué almacena** | Estado anterior de la entidad | Evento o acción del sistema |
| **Lo escribe** | Lógica de negocio ante un UPDATE | El sistema ante cualquier evento |
| **FK** | Permitidas | Prohibidas |
| **Ejemplo** | Estado de `person` antes de un update | "El usuario X bloqueó la entidad Y a las Z" |

**`log_` en base de datos vs logs del sistema**

| | `log_` (tabla SQL) | Logs del sistema |
|---|---|---|
| **Formato** | Estructurado (columnas definidas) | Texto / JSON semi-estructurado |
| **Para quién** | Negocio, reportes, auditoría | Desarrolladores, operaciones, alertas |
| **Se consultan vía** | SQL, reportes, APIs | Serilog, NLog, Datadog |

> Las tablas `log_` crecen indefinidamente. Planificar desde el inicio una estrategia de archivado por fecha o particionamiento — ver [Particionamiento](#58-particionamiento--naming).

> Template: [A.1.6](sqlserver-ddl-templates.md#a16-tabla-de-log-log_)

### 3.5 Tablas de Catálogo (`cat_`)

Contienen valores referenciales que parametrizan el comportamiento del sistema. Se dividen en dos subtipos:

**Catálogo del sistema:** valores fijos definidos por el equipo técnico. Se siembran vía scripts DML y no son editables por el usuario. No requieren campos de auditoría.
Ejemplos: `cat_country`, `cat_currency`, `cat_document_type`

**Catálogo gestionado por usuario:** valores configurables desde la UI por usuarios con rol administrativo. Sí requieren campos de auditoría.
Ejemplos: `cat_risk_level`, `cat_alert_category`

**`is_active` vs `is_deleted` — cuándo usar cada uno**

| | `is_active` | `is_deleted` |
|---|---|---|
| **Semántica** | Registro habilitado / deshabilitado operacionalmente | Registro eliminado lógicamente |
| **¿Reversible?** | Sí — se activa y desactiva libremente | Sí técnicamente, pero representa una acción irreversible de negocio |
| **Contexto típico** | Catálogos, configuraciones | Entidades de negocio con ciclo de vida que incluye eliminación |

**Regla:** los catálogos usan `is_active` — no se eliminan, se deshabilitan. Las entidades de negocio con eliminación lógica usan `is_deleted`. Una tabla puede tener ambos si el dominio lo requiere.

> Templates: [A.1.7 catálogo del sistema](sqlserver-ddl-templates.md#a17-catálogo-del-sistema-sin-auditoría) · [A.1.8 catálogo de usuario](sqlserver-ddl-templates.md#a18-catálogo-gestionado-por-usuario-con-auditoría)

---

## 4. Campos

### 4.1 Auditoría

Presentes en tablas principales, de relación y catálogos editables.

| Campo | Tipo | Obligatorio | Quién lo escribe |
|-------|------|-------------|------------------|
| `created_at` | `DATETIME2(3)` | Sí | SQL Server (DEFAULT) |
| `updated_at` | `DATETIME2(3)` | No (NULL inicial) | La aplicación en cada UPDATE |
| `created_by` | `NVARCHAR(100)` | Sí | La aplicación (username del usuario autenticado) |
| `updated_by` | `NVARCHAR(100)` | No (NULL inicial) | La aplicación en cada UPDATE |

> **`DATETIME2(3)` y no `DATETIME`:** `DATETIME` es un tipo legacy. `DATETIME2` ofrece mayor precisión y cumple con ISO 8601. Siempre usar `DATETIME2`. El `(3)` indica milisegundos — suficiente para auditoría de negocio.

> **`SYSUTCDATETIME()` y no `SYSDATETIME()`:** `SYSDATETIME()` retorna la hora local del servidor, que varía según zona horaria. `SYSUTCDATETIME()` retorna UTC — invariante y comparable entre servicios distribuidos. Siempre usar UTC en auditoría.

> **Los campos `*_by` almacenan username, no email.** Ver longitudes en [Longitudes de NVARCHAR](#46-longitudes-de-nvarchar).

> Template: [A.2.1](sqlserver-ddl-templates.md#a21-campos-de-auditoría)

### 4.2 Clave Primaria

Formato: `id_{entidad}` · Tipo: `UNIQUEIDENTIFIER` con `NEWSEQUENTIALID()` como default.

**Por qué UUID y no BIGINT:**
- Los `BIGINT IDENTITY` son predecibles y secuenciales, lo que expone el volumen de datos y facilita la enumeración desde APIs
- En arquitecturas DDD y microservicios, el ID viaja entre servicios — un UUID es globalmente único sin coordinación central

**Por qué `NEWSEQUENTIALID()` y no `NEWID()`:**
- `NEWID()` genera UUIDs completamente aleatorios, causando fragmentación severa en el índice clustered
- `NEWSEQUENTIALID()` genera UUIDs incrementales, eliminando la fragmentación sin sacrificar la unicidad global
- Restricción: solo puede usarse como `DEFAULT` en la columna. Si la aplicación necesita conocer el ID antes del INSERT, generar UUID v7 (secuencial por timestamp) en la aplicación

```
id_person   UNIQUEIDENTIFIER   -- Correcto
id_person   BIGINT IDENTITY    -- Incorrecto — predecible, no apto para microservicios
```

> **ORMs y `NEWSEQUENTIALID()`:** la mayoría de ORMs generan el UUID en la aplicación por defecto en lugar de delegarlo al motor. Esto produce UUIDs aleatorios — exactamente el anti-pattern que se busca evitar. Configurar el ORM para delegar la generación de `id_{entidad}` al servidor, no generarla en la aplicación.

**Cuándo sí aplica un ID numérico:**
- Integraciones con sistemas legacy que definen el tipo
- Cuando la regulación impone un identificador numérico secuencial (folios regulatorios)

Para estos casos, usar `SEQUENCE` sobre `IDENTITY` — ver [Sequences](#65-sequences).

> Template: [A.2.2](sqlserver-ddl-templates.md#a22-clave-primaria)

### 4.3 Clave Foránea

Formato: `{entidad}_id`

```
-- Correcto
country_id
person_id

-- Incorrecto
id_country      -- mismo formato que PK, genera confusión
countryID       -- camelCase
fk_country      -- prefijo húngaro
```

> Template: [A.2.3](sqlserver-ddl-templates.md#a23-claves-foráneas)

### 4.4 Soft Delete (`is_deleted`)

El soft delete marca un registro como eliminado sin borrarlo físicamente. Permite recuperación, auditoría y trazabilidad.

**Campos estándar:**
- `is_deleted` es el único campo obligatorio del patrón
- `deleted_at` y `deleted_by` son opcionales, pero rigen la regla de todo o nada: si se agrega uno, se agregan los dos

> **¿Por qué `deleted_at`/`deleted_by` si ya existen `updated_at`/`updated_by`?** El soft delete es técnicamente un UPDATE, por lo que `updated_at`/`updated_by` se escriben. Sin embargo, si después del delete se realiza otra modificación, `updated_at`/`updated_by` se sobreescriben y se pierde la trazabilidad de quién eliminó. `deleted_at`/`deleted_by` nunca se actualizan una vez asignados — son el registro definitivo del evento de eliminación.

> **Incompatibilidad con Temporal Tables:** si la tabla usa `SYSTEM_VERSIONING`, no usar `is_deleted` — usar `is_active` en su lugar. Ver [Temporal Tables](#331-alternativa-nativa-temporal-tables-system_versioning).

**Cuándo usarlo:**
- Entidades de negocio que requieren trazabilidad o posibilidad de recuperación
- Registros que otras tablas referencian con FK (un DELETE físico rompería la integridad)
- Cuando la regulación o auditoría exige retención del dato

**Cuándo NO usarlo:**
- Tablas de log (`log_`) — los logs nunca se eliminan, el patrón no aplica
- Tablas de historial (`hist_`) — son registros de solo inserción por naturaleza
- Catálogos del sistema — se controla su vigencia con `is_active`
- Tablas con `SYSTEM_VERSIONING` — incompatible

**Convención de consultas:** todo query sobre tablas con soft delete debe filtrar `WHERE is_deleted = 0`. Omitirlo es un error silencioso frecuente.

> Si el ORM soporta filtros de consulta globales, configurar `is_deleted = 0` a nivel del ORM para que se aplique automáticamente. Importante: los filtros globales del ORM **no aplican a SQL crudo** — en ese caso incluir `WHERE is_deleted = 0` explícitamente.

> **Filtered index para soft delete:** toda tabla con `is_deleted` recibirá queries `WHERE is_deleted = 0` permanentemente. Usar un filtered index reduce el tamaño del índice excluyendo registros eliminados y mejora el performance de las consultas activas. Ver template en [A.3.5](sqlserver-ddl-templates.md#a35-covering-index-include).

> Templates: [A.2.4 mínimo](sqlserver-ddl-templates.md#a24-soft-delete--mínimo-obligatorio) · [A.2.5 con trazabilidad](sqlserver-ddl-templates.md#a25-soft-delete--con-trazabilidad-completa)

### 4.5 Tipos de texto: `NVARCHAR` vs `VARCHAR`

**Regla general: usar siempre `NVARCHAR`.**

`NVARCHAR` almacena Unicode (UTF-16), soportando cualquier idioma, símbolo o carácter especial. `VARCHAR` almacena solo ASCII extendido y depende del collation del servidor, generando problemas silenciosos con acentos, ñ o caracteres no-latinos.

| Tipo | Cuándo usar |
|------|-------------|
| `NVARCHAR` | Todo campo de texto — regla general sin excepción |
| `VARCHAR` | Solo en integraciones con sistemas legacy que requieren ASCII estricto, o en columnas de alto volumen donde el espacio físico es una restricción documentada y medida |

### 4.6 Longitudes de `NVARCHAR`

Las longitudes no deben ser arbitrarias. Usar criterios semánticos predecibles reduce inconsistencias entre tablas y facilita las integraciones.

| Campo | Longitud | Justificación |
|-------|----------|---------------|
| Código interno (`code`, `sku`) | `NVARCHAR(50)` | Identificadores cortos, controlados |
| Código externo / estándar (ISO, SWIFT) | `NVARCHAR(10)` | Longitud definida por el estándar |
| Email | `NVARCHAR(254)` | Límite RFC 5321 |
| Nombre de persona | `NVARCHAR(200)` | Cubre nombres compuestos internacionales |
| Descripción corta / etiqueta | `NVARCHAR(200)` | Textos de una línea para UI |
| Descripción larga / observación | `NVARCHAR(500)` | Párrafo corto |
| Username / identificador de sistema | `NVARCHAR(100)` | Usernames de sistema, UPNs de AD. Los campos `*_by` almacenan username — no email |
| URL | `NVARCHAR(2048)` | Límite práctico en navegadores |
| Texto libre / notas | `NVARCHAR(MAX)` | Sin límite predefinido |
| Metadata JSON | `NVARCHAR(MAX)` | Ver criterios en [A.2.7](sqlserver-ddl-templates.md#a27-columna-json-con-validación) |

> **`NVARCHAR(MAX)` y performance:** SQL Server puede almacenar el valor fuera de la fila cuando supera 8000 bytes, añadiendo una lectura adicional. Usar solo cuando el contenido genuinamente no tiene límite predecible.

> **Columnas JSON:** SQL Server no tiene un tipo JSON nativo que rechace contenido inválido. Usar CHECK constraints con `ISJSON()` para garantizar integridad. La validación de schema completo (propiedades requeridas, tipos específicos) debe hacerse en la capa de aplicación. Ver template en [A.2.7](sqlserver-ddl-templates.md#a27-columna-json-con-validación).

### 4.7 Campo `code` (slug legible)

En entidades con UUID como PK que deben ser referenciables por usuarios finales, se complementa con un campo `code`: legible, corto y amigable para la UI, reportes o URLs.

**Cuándo usarlo:**
- La entidad es referenciada frecuentemente por personas (formularios, reportes, URLs)
- Se necesita un identificador memorable o imprimible

**Cuándo NO usarlo:**
- Tablas de log, historial o relación pura
- Entidades que nunca son expuestas directamente al usuario

**Formato: kebab-case** — palabras en minúsculas separadas por guiones (`-`). Es el estándar para slugs en sistemas modernos (GitHub, Stripe, Notion) por ser URL-friendly y legible.

```
intake-request       ← correcto
vip-client-type      ← correcto

intake_request       ← incorrecto — guion bajo no es URL-friendly
intakeRequest        ← incorrecto — camelCase
INTAKE-REQUEST       ← incorrecto — mayúsculas
```

> SQL Server no soporta regex completo en CHECK constraints. El patrón con `LIKE` es la aproximación más cercana; complementar con validación en la capa de aplicación para cobertura completa. El CHECK constraint con `COLLATE Latin1_General_CS_AS` garantiza la validación de minúsculas independientemente del collation de la base.

> Template: [A.2.6](sqlserver-ddl-templates.md#a26-campo-code-slug-kebab-case)

### 4.8 Tipos Numéricos

| Categoría | Tipo recomendado | Por qué |
|-----------|-----------------|---------|
| Montos monetarios | `DECIMAL(18, 2)` | Precisión exacta — `FLOAT` tiene representación binaria imprecisa |
| Porcentajes | `DECIMAL(5, 4)` | Almacenar como fracción (`0.1525` = 15.25%). Usar el sufijo `_pct` en el nombre si se almacena como entero (ej: `15`). Para tasas > 999.99%, usar `DECIMAL(7, 4)` |
| Cantidades enteras estándar | `INT` | Suficiente para la mayoría de los casos de negocio |
| Año / rangos cortos | `SMALLINT` | Ahorra espacio — 2 bytes vs 4 de `INT` |
| Contadores masivos / IDs externos | `BIGINT` | Para valores que pueden superar 2 mil millones |
| Flags / booleanos | `BIT` | Nunca `TINYINT` ni `CHAR(1)` para booleanos |
| Coordenadas geográficas | `FLOAT` | La imprecisión de centésimas de grado es irrelevante |

**Prohibidos para valores financieros:**
- `FLOAT` / `REAL` — imprecisión binaria: `0.1 + 0.2` puede resultar en `0.30000000000000004`
- `MONEY` / `SMALLMONEY` — tipos propietarios de SQL Server, imprecisión en multiplicación/división, no portables

> Template con ejemplos: [A.2.8](sqlserver-ddl-templates.md#a28-tipos-numéricos--referencia-rápida)

### 4.9 Estrategia NULL / NOT NULL

**Regla: `NOT NULL` por defecto. `NULL` solo cuando el valor es genuinamente opcional por diseño del negocio.**

Poner `NULL` en todo "para evitar errores de inserción" corrompe el modelo: los `NULL` en columnas numéricas afectan `SUM` y `AVG` de formas no obvias; `COUNT(columna)` ignora `NULL` mientras `COUNT(*)` no.

```
-- NOT NULL: la columna siempre tiene un valor en toda circunstancia de negocio
full_name   NVARCHAR(200)   NOT NULL
is_deleted  BIT             NOT NULL

-- NULL: el valor es conocido solo en ciertos estados del ciclo de vida
updated_at  DATETIME2(3)    NULL   -- NULL hasta el primer UPDATE
deleted_at  DATETIME2(3)    NULL   -- NULL mientras no esté eliminado
middle_name NVARCHAR(200)   NULL   -- no todas las personas tienen segundo nombre
```

**FK nullable:** una FK nullable expresa que la relación es opcional. Es semánticamente válido pero debe ser intencional.

```
country_id  UNIQUEIDENTIFIER  NOT NULL   -- relación obligatoria
manager_id  UNIQUEIDENTIFIER  NULL       -- relación opcional
```

---

## 5. Índices y Constraints

> **Clustered vs Non-Clustered:** SQL Server crea el índice `PRIMARY KEY` como **clustered** por defecto — las filas se almacenan físicamente en el orden de la PK. Con `NEWSEQUENTIALID()` esto funciona bien: los registros se insertan secuencialmente. La excepción son las tablas `log_` con altísimo volumen donde el patrón de acceso dominante es por rango de fecha — en ese caso puede convenir un clustered index en `occurred_at`. Decidir basándose en el patrón de acceso real, no en una regla automática.

### 5.1 Índice Común

Formato: `idx_{tabla}_{columna}`

```
idx_person_email
idx_log_transaction_entity_id_occurred_at   -- compuesto
```

> Template: [A.3.1](sqlserver-ddl-templates.md#a31-índice-común)

### 5.2 Índice Único

Formato: `uk_{tabla}_{columna}`

Hay dos formas de declarar unicidad en SQL Server con diferencias prácticas:

| | `CONSTRAINT ... UNIQUE` | `CREATE UNIQUE INDEX` |
|---|---|---|
| **Soporta filtered index** | No | Sí (`WHERE` clause) |
| **Soporta `INCLUDE` columns** | No | Sí |
| **Cuándo usarlo** | Unicidad simple, sin condiciones adicionales | Cuando se necesita filtro o columnas incluidas |

> Templates: [A.3.2 constraint](sqlserver-ddl-templates.md#a32-índice-único-como-constraint-unicidad-simple) · [A.3.3 con filtro](sqlserver-ddl-templates.md#a33-índice-único-con-filtro-filtered-unique-index)

### 5.3 Índice Compuesto

Un índice compuesto puede ser único o no único. Las columnas van en orden de selectividad — la más selectiva primero.

```
uk_{tabla}_{col1}_{col2}    -- compuesto único
idx_{tabla}_{col1}_{col2}   -- compuesto no único
```

> Template: [A.3.4](sqlserver-ddl-templates.md#a34-índice-compuesto)

### 5.4 Covering Index (`INCLUDE`)

Un covering index incluye columnas adicionales que no son parte de la clave de búsqueda. El objetivo es que el índice cubra completamente el query — sin necesidad de un key lookup adicional a la tabla.

**El problema que resuelve:** cuando SQL Server usa un índice para resolver un query y el índice no contiene todas las columnas del `SELECT`, el motor hace un **key lookup** por cada fila — costoso en tablas grandes.

**El nombre se basa en la columna de búsqueda, no en las columnas `INCLUDE`:** las columnas incluidas son un detalle de implementación del índice.

```
idx_person_country_id   -- basado en la columna de búsqueda
                        -- las columnas INCLUDE no aparecen en el nombre
```

**Cuándo usar `INCLUDE`:**
- El query frecuente proyecta pocas columnas además de la clave de búsqueda
- El key lookup aparece como operador costoso en el plan de ejecución
- Evaluar el plan de ejecución antes de agregar — no optimizar prematuramente

> Template: [A.3.5](sqlserver-ddl-templates.md#a35-covering-index-include)

### 5.5 Tabla de Prefijos y Constraints — Referencia Rápida

#### Objetos de base de datos

| Tipo de objeto | Prefijo | Ejemplo |
|---|---|---|
| Tabla principal / relación / catálogo / log / historial | *(sin prefijo)* | `person`, `cat_country`, `log_transaction` |
| Vista | `v_` | `v_active_person` |
| Stored Procedure | `usp_` | `usp_get_person_by_id` |
| Función escalar | `fn_` | `fn_calculate_risk_score` |
| Función table-valued | `tvf_` | `tvf_get_transactions_by_period` |
| Trigger | `trg_` | `trg_person_after_update` |
| Sequence | `seq_` | `seq_folio` |
| Índice B-tree (no único) | `idx_` | `idx_person_email` |
| Índice único | `uk_` | `uk_person_email` |
| Columnstore index | `ccsi_` | `ccsi_log_transaction_analytics` |
| Primary Key constraint | `pk_` | `pk_person` |
| Foreign Key constraint | `fk_` | `fk_person_cat_country_country_id` |
| Unique constraint | `uk_` | `uk_person_email` |
| Check constraint | `ck_` | `ck_person_age` |
| Default constraint | `df_` | `df_person_is_deleted` |
| Partition Function | `pf_` | `pf_log_transaction_monthly` |
| Partition Scheme | `ps_` | `ps_log_transaction_monthly` |
| RLS predicate function | `fn_rls_` | `fn_rls_person_filter` |
| RLS security policy | `pol_` | `pol_person` |
| Tabla temporal local en SP | `#tmp_` | `#tmp_person_results` |

> `uk_` aplica tanto para `CREATE UNIQUE INDEX` como para `CONSTRAINT ... UNIQUE` — en ambos casos representa unicidad. La distinción entre los dos mecanismos está en [Índice Único](#52-índice-único).

#### Constraints — Formato de nombre completo

| Tipo | Prefijo | Formato |
|------|---------|---------|
| Primary Key | `pk_` | `pk_{tabla}` |
| Foreign Key | `fk_` | `fk_{tabla_origen}_{tabla_destino}_{columna}` |
| Unique | `uk_` | `uk_{tabla}_{columna}` |
| Check | `ck_` | `ck_{tabla}_{columna_o_descripcion}` |
| Default | `df_` | `df_{tabla}_{columna}` |

```
pk_person
uk_person_email
ck_person_age
fk_person_cat_country_country_id
df_person_is_deleted
```

**Límite de 128 caracteres (`sysname`):** con tablas de nombres compuestos, el formato de FK puede acercarse o superar ese límite. Cuando el nombre generado supera los **100 caracteres**, se permite abreviar el nombre de la tabla destino usando la abreviación oficial del diccionario de schemas. La abreviación debe documentarse como comentario en el script DDL y registrarse en el diccionario del proyecto.

> Template con sintaxis de DEFAULT en CREATE TABLE vs ALTER TABLE: [A.3.7](sqlserver-ddl-templates.md#a37-constraints-nombrados)

### 5.6 Índices en columnas FK

**SQL Server no crea automáticamente índices en columnas de clave foránea.** Una FK sin índice produce un table scan completo en cada JOIN o lookup por esa columna.

**Regla: toda columna FK debe tener un índice explícito.**

Para tablas de relación con PK compuesta, el índice de la PK ya cubre la primera columna. La segunda columna necesita su propio índice si se consulta de forma independiente (consulta inversa).

> **Sin FK entre schemas de distintos servicios:** las foreign keys entre schemas de distintos servicios crean acoplamiento estructural. La consistencia entre servicios se gestiona a nivel de aplicación o mediante eventos de dominio. Ver contexto completo en [Apéndice A — Arquitectura de Integración](#apéndice-a--arquitectura-de-integración).

> Template: [A.3.8](sqlserver-ddl-templates.md#a38-índice-explícito-en-columna-fk)

### 5.7 Columnstore Index (analytics)

Para tablas `log_` y `hist_` que alimentan reportes o analytics, los **columnstore indexes** son la herramienta más eficiente para ese patrón de acceso. A diferencia de los B-tree, almacenan datos por columna con compresión vectorial — pueden reducir el tiempo de queries analíticos entre 10x y 100x.

**Cuándo usarlo:**
- Queries que agregan grandes volúmenes de filas (`GROUP BY`, `SUM`, `COUNT`, `AVG`) sobre columnas específicas
- Tablas `log_` o `hist_` que alimentan dashboards o reportes
- A partir de ~1 millón de filas — debajo de ese umbral un B-tree convencional es suficiente

Formato: `ccsi_{tabla}_{descripcion}`

> Template: [A.3.6](sqlserver-ddl-templates.md#a36-columnstore-index-analytics)

### 5.8 Particionamiento — Naming

Cuando las tablas `log_` o `hist_` alcanzan volúmenes que hacen inviable el archivado por DELETE masivo, el particionamiento permite mover o eliminar particiones completas con una operación de metadatos instantánea (`SWITCH PARTITION`).

El particionamiento requiere dos objetos con naming predecible:

| Objeto | Prefijo | Formato |
|--------|---------|---------|
| Partition Function | `pf_` | `pf_{tabla}_{criterio}` |
| Partition Scheme | `ps_` | `ps_{tabla}_{criterio}` |

El criterio describe la granularidad: `monthly`, `yearly`.

**Cuándo usar particionamiento:**
- Tablas con decenas de millones de filas y patrón de acceso por rango de fecha
- Cuando el archivado por partición es la estrategia de retención
- Nunca como optimización prematura

> Template: [A.4](sqlserver-ddl-templates.md#a4-particionamiento)

---

## 6. Otros Objetos

### 6.1 Vistas

Formato: `v_{descripcion}`

```
v_active_person
v_transaction_summary

-- Incorrecto
vw_person    -- prefijo húngaro
person2      -- número sin semántica
```

> **¿Por qué `v_` si la guía prohíbe prefijos húngaros?** En queries con múltiples JOINs entre tablas y vistas, saber visualmente cuál es vista tiene valor operacional — su comportamiento (no materializada, puede tener filtros implícitos, puede o no ser actualizable) es distinto al de una tabla base. Si el equipo decide eliminar el prefijo completamente, aplicarlo de forma consistente en todo el proyecto.

> Template: [A.6](sqlserver-ddl-templates.md#a6-vistas)

### 6.2 Stored Procedures

Formato: `usp_{accion}_{entidad}`

Acciones estándar: `get`, `create`, `update`, `delete`, `search`

```
usp_get_person_by_id
usp_create_person
usp_search_person_by_document

-- Incorrecto
sp_get_person   -- sp_ está reservado por SQL Server para procedimientos del sistema
GetPerson       -- PascalCase
```

> El prefijo `sp_` está reservado. SQL Server busca primero en `master` cualquier procedimiento con ese prefijo — overhead innecesario y posible colisión con procedimientos del sistema.

**Todo SP con DML debe incluir:**
- `SET NOCOUNT ON` — evita mensajes "N rows affected" que rompen algunos clientes ADO.NET
- `SET XACT_ABORT ON` — si ocurre un error de runtime, SQL Server hace rollback automático sin depender del CATCH. Sin esto, un timeout o conexión cortada puede dejar una transacción abierta indefinidamente con locks colgados
- `BEGIN TRY / BEGIN CATCH` con `THROW` para propagar errores al caller

> Templates: [A.5.1 SP con DML](sqlserver-ddl-templates.md#a51-sp-con-dml-escritura) · [A.5.2 SP de lectura](sqlserver-ddl-templates.md#a52-sp-de-solo-lectura)

### 6.3 Funciones

- Escalares: `fn_{descripcion}` — retornan un valor único
- Table-valued: `tvf_{descripcion}` — retornan un conjunto de filas

> **Advertencia — funciones escalares y performance:** se ejecutan **una vez por fila**, impiden planes de ejecución paralelos. En tablas con miles de filas esto puede producir degradación severa. SQL Server 2019 introdujo Scalar UDF Inlining que puede resolverlo automáticamente si la función cumple ciertas condiciones. Para lógica frecuente sobre conjuntos grandes, preferir una función table-valued inline o lógica directa en el query.

> Template: [A.7](sqlserver-ddl-templates.md#a7-funciones)

### 6.4 Triggers

Formato: `trg_{tabla}_{evento}`

| Evento | Cuándo se dispara |
|--------|-------------------|
| `after_insert` | Después de un INSERT |
| `after_update` | Después de un UPDATE |
| `after_delete` | Después de un DELETE |
| `instead_of_insert` | En lugar del INSERT |
| `instead_of_update` | En lugar del UPDATE |
| `instead_of_delete` | En lugar del DELETE |

> El schema se declara en el nombre del trigger. Sin schema explícito, el trigger se crea en `dbo` independientemente del schema de la tabla.

> **Los triggers deben evitarse en la mayoría de los casos.** Ejecutan lógica invisible para el consumidor del modelo, dificultan el debugging y generan comportamiento implícito que complica las migraciones. Preferir lógica explícita en la capa de aplicación o SPs. Si un trigger es inevitable, documentar en el cuerpo del trigger por qué no pudo resolverse de otra forma.

> Template: [A.8](sqlserver-ddl-templates.md#a8-triggers)

### 6.5 Sequences

Formato: `seq_{descripcion}`

SQL Server soporta `SEQUENCE` (desde 2012) como alternativa a `IDENTITY`. A diferencia de `IDENTITY`, un `SEQUENCE` es independiente de la tabla — su valor puede obtenerse antes del INSERT con `NEXT VALUE FOR` y puede compartirse entre múltiples tablas.

| | `IDENTITY` | `SEQUENCE` |
|---|---|---|
| **Scope** | Ligado a una tabla | Independiente — compartible |
| **Valor antes del INSERT** | No | Sí — `NEXT VALUE FOR` |
| **Cuándo usar** | Surrogate key numérica simple | Folios regulatorios, numeración compartida |

> Esta guía usa `UNIQUEIDENTIFIER` como estándar para PKs. Los `SEQUENCE` aplican cuando el negocio o la regulación requiere un identificador numérico secuencial — no como alternativa general a UUID.

> Template: [A.9](sqlserver-ddl-templates.md#a9-sequences)

### 6.6 Tablas Temporales en SPs

SQL Server ofrece tres mecanismos para almacenamiento intermedio dentro de un SP:

| Mecanismo | Nombre | Scope | Estadísticas | Cuándo usar |
|-----------|--------|-------|--------------|-------------|
| Variable de tabla | `@{descripcion}` | Batch actual | No | Conjuntos pequeños (< 100 filas), sin JOINs complejos |
| Tabla temporal local | `#tmp_{descripcion}` | Sesión actual | Sí | Conjuntos medianos/grandes, cuando el optimizador necesita estadísticas |
| Tabla temporal global | `##tmp_{descripcion}` | Todas las sesiones | Sí | **Prohibido en SPs de producción** — colisiones entre sesiones |

**Regla de decisión:**
- ¿Más de 100 filas o JOINs subsiguientes? → usar `#tmp_`
- ¿Menos de 100 filas, sin JOINs? → usar `@table_variable`
- ¿Necesita ser visible en otras sesiones? → nunca en producción

> Template: [A.10](sqlserver-ddl-templates.md#a10-tablas-temporales-en-sps)

---

## 7. Palabras Reservadas en SQL Server

SQL Server tiene palabras reservadas que, combinadas con snake_case y nombres de dominio comunes, generan colisiones frecuentes.

| Palabra reservada | Alternativa sugerida |
|-------------------|----------------------|
| `order` | `purchase_order`, `work_order`, `sort_order` |
| `user` | `system_user`, `app_user`, `account` |
| `table` | `data_table`, `reference_table` |
| `group` | `user_group`, `category_group` |
| `key` | `access_key`, `api_key`, `record_key` |
| `value` | `amount`, `quantity`, `setting_value` |
| `type` | `record_type`, `document_type`, `event_type` |
| `name` | `full_name`, `display_name`, `entity_name` |
| `level` | `access_level`, `risk_level` |
| `status` | `record_status`, `transaction_status` |

**Regla 1 — Preferir renombrar semánticamente (siempre):** un nombre más específico es más legible y evita la colisión sin fricción adicional.

**Regla 2 — Si el nombre es inevitable, escapar con corchetes `[order]`:** solo como medida de último recurso. Genera fricción en ORMs y hace los scripts más verbosos. No usar corchetes como solución permanente o sistemática.

---

## 8. Scripts DDL y DML

### Estructura de carpetas

```
/migrations
  /ddl
    01_create_schema_pld.sql
    02_create_table_person.sql
    03_create_table_cat_country.sql
    04_create_table_person_role.sql
    05_create_fk_person_cat_country.sql       ← FKs en scripts separados, al final
    06_create_fk_person_role_person.sql
    07_create_indexes_person.sql
  /dml
    01_seed_cat_country.sql
    02_seed_cat_risk_level.sql
  /rollback
    /ddl
      01_rollback_create_schema_pld.sql
      02_rollback_create_table_person.sql
      ...
```

### Orden de ejecución

1. `/ddl` en orden numérico: primero schemas → tablas (sin FKs) → FKs → índices
2. `/dml` en orden numérico

### Numeración de scripts

El esquema `01_, 02_, ...` es simple pero tiene fricción: insertar un script entre `02_` y `03_` obliga a renumerar los posteriores, generando conflictos en ramas paralelas. Dos alternativas:

- **Numeración con gaps** (`010_, 020_, 030_`) — deja espacio para insertar intermedios
- **Timestamp como prefijo** (`20260329_001_create_table_person.sql`) — evita colisiones entre ramas

### Por qué las FKs van en scripts separados

Una FK referencia una tabla que debe existir previamente. En modelos con referencias circulares, intentar crear FKs inline puede generar errores de orden irresolubles. Con scripts separados: primero todas las tablas, luego todas las FKs.

### Idempotencia DDL

Un script que falla en la segunda ejecución no es un script seguro. Dos patrones:

- **`IF NOT EXISTS ... CREATE`** — para objetos que se crean una vez (tablas, índices, constraints)
- **`CREATE OR ALTER`** — para objetos cuya definición cambia con el tiempo (SPs, vistas, funciones, triggers)

> Templates de idempotencia para cada tipo de objeto: [A.11](sqlserver-ddl-templates.md#a11-idempotencia-ddl)

### Idempotencia DML — seeds

Los scripts de seed también deben ser re-ejecutables. Usar `MERGE` en lugar de `INSERT` simple.

> `MERGE` sin `WITH (HOLDLOCK)` puede insertar filas duplicadas bajo alta concurrencia. Para seeds ejecutados durante mantenimiento sin carga concurrente no es un problema. Para upserts en producción, agregar `WITH (HOLDLOCK)`.

> Templates de seeds y cuándo usar MERGE vs guard simple: [A.12](sqlserver-ddl-templates.md#a12-seeds-dml)

### Rollback scripts

Cada script DDL tiene un script de reversa en `/rollback/ddl`. El rollback se escribe **junto con el DDL**, no después. Si un deploy falla a mitad de camino, los rollback scripts permiten revertir sin intervención manual bajo presión.

> Templates de rollback para cada tipo de objeto: [A.13](sqlserver-ddl-templates.md#a13-rollback-scripts)

### ORM Migrations

Si el proyecto usa un ORM con gestión de migraciones (EF Core, Hibernate, etc.), el ORM gestiona el orden de ejecución internamente. **No mezclar** scripts DDL manuales con migraciones del ORM sobre el mismo schema — fuente frecuente de inconsistencias de estado.

> **Compatibilidad con `snake_case`:** la mayoría de ORMs usan su propia convención de nombres por defecto. Configurar el ORM para respetar `snake_case` y los schemas correctos antes de generar la primera migración — un error de configuración en este punto puede crear objetos en el schema o con el casing incorrecto sin generar errores visibles. Consultar la guía de integración del ORM correspondiente al proyecto.

> **`NEWSEQUENTIALID()` y ORMs:** muchos ORMs generan el UUID en la aplicación (client-side) en lugar de delegar al motor. Esto produce UUIDs aleatorios en lugar de secuenciales — el anti-pattern que `NEWSEQUENTIALID()` busca evitar. Verificar que el ORM esté configurado para dejar que SQL Server genere el valor del campo `id_{entidad}`.

---

## 9. Anti-patterns

| Anti-pattern | Síntoma | Consecuencia | Solución en esta guía |
|---|---|---|---|
| `SELECT * FROM person` sin filtro de soft delete | Query retorna registros eliminados sin advertirlo | Datos incorrectos en UI y reportes | Siempre `WHERE is_deleted = 0` — ver [Soft Delete](#44-soft-delete-is_deleted) |
| `FLOAT` o `MONEY` en columnas de montos | `0.1 + 0.2 = 0.30000000000000004` / imprecisión en división | Errores de redondeo en operaciones financieras | `DECIMAL(18, 2)` — ver [Tipos Numéricos](#48-tipos-numéricos) |
| `NULL` en columnas que siempre deben tener valor | `SUM(amount)` subestima el total | Resultados incorrectos en agregaciones | `NOT NULL` por defecto — ver [NULL / NOT NULL](#49-estrategia-null--not-null) |
| `sp_get_person` | SQL Server busca primero en `master` | Overhead innecesario, posible colisión | Usar `usp_` — ver [Stored Procedures](#62-stored-procedures) |
| FK sin índice | Table scan en cada JOIN por esa columna | Degradación de performance en queries frecuentes | Índice explícito en toda FK — ver [Índices en columnas FK](#56-índices-en-columnas-fk) |
| `VARCHAR` en campos de texto | Pérdida de acentos, ñ y caracteres no-ASCII | Datos corruptos silenciosos | `NVARCHAR` siempre — ver [NVARCHAR vs VARCHAR](#45-tipos-de-texto-nvarchar-vs-varchar) |
| `DATETIME` en lugar de `DATETIME2` | Menor precisión, tipo legacy | Incompatibilidad con ISO 8601 | `DATETIME2(3)` — ver [Auditoría](#41-auditoría) |
| `NEWID()` como default de PK | Fragmentación severa del índice clustered | Degradación progresiva de INSERTs | `NEWSEQUENTIALID()` — ver [Clave Primaria](#42-clave-primaria) |
| Triggers para lógica de negocio | Lógica invisible al consumidor del modelo | Bugs difíciles de reproducir y debuggear | Lógica explícita en SPs o aplicación — ver [Triggers](#64-triggers) |
| Seeds con `INSERT` simple | El script falla en la segunda ejecución | Deployments no repetibles | `MERGE` idempotente — ver [Scripts DDL y DML](#8-scripts-ddl-y-dml) |
| Collation mixto entre servidor y base de datos | Errores de comparación de strings entre contextos | Bugs intermitentes difíciles de rastrear | Collation explícito en `CREATE DATABASE` — ver [sqlserver-database-setup.md](sqlserver-database-setup.md) |
| Función escalar en `SELECT` masivo | Ejecución una vez por fila, bloquea paralelismo | Query 100x más lento que el equivalente en SQL directo | Table-valued inline o lógica en query — ver [Funciones](#63-funciones) |
| `SELECT *` en SPs y vistas | Agregar o reordenar una columna en la tabla rompe el SP silenciosamente | Resultados cambian sin error visible | Listar siempre las columnas explícitamente |
| `WITH (NOLOCK)` como atajo de performance | Lee datos no confirmados, filas a medio escribir | Dirty reads, resultados inconsistentes | Resolver con índices covering o activar RCSI — ver [sqlserver-database-setup.md](sqlserver-database-setup.md) |
| Consulta SQL cruda con filtro de soft delete omitido | Los filtros globales del ORM no aplican a SQL crudo | Registros eliminados en resultados sin advertencia | Incluir `WHERE is_deleted = 0` explícitamente en todo SQL crudo |
| `##tmp_` en SPs de producción | Scope global — visible en todas las sesiones | Colisiones entre sesiones concurrentes | Usar `#tmp_` (local) — ver [Tablas Temporales en SPs](#66-tablas-temporales-en-sps) |

---

## 10. Referencias

- [DATETIME2 — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetime2-transact-sql)
- [DECIMAL y NUMERIC — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/data-types/decimal-and-numeric-transact-sql)
- [MONEY y SMALLMONEY — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/data-types/money-and-smallmoney-transact-sql)
- [NEWSEQUENTIALID — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/functions/newsequentialid-transact-sql)
- [ISJSON, JSON_VALUE, OPENJSON — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/functions/isjson-transact-sql)
- [Scalar UDF Inlining — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/scalar-udf-inlining)
- [Filtered Indexes — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes)
- [Covering Indexes (INCLUDE) — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-indexes-with-included-columns)
- [Columnstore Indexes — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview)
- [Reserved Keywords — SQL Server](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/reserved-keywords-transact-sql)
- [CREATE OR ALTER — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-procedure-transact-sql)
- [Collations and Unicode Support — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support)
- [READ_COMMITTED_SNAPSHOT — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-set-options)
- [Database Permissions — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/security/permissions-database-engine)
- [Temporal Tables — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)
- [Table Partitioning — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes)
- [Partition Functions — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-partition-function-transact-sql)
- [Row-Level Security — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security)
- [EF Core Global Query Filters — Microsoft Docs](https://learn.microsoft.com/en-us/ef/core/querying/filters)
- [EFCore.NamingConventions — GitHub](https://github.com/efcore/EFCore.NamingConventions)
- [CREATE SEQUENCE — Microsoft Docs](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-sequence-transact-sql)
- [Always Encrypted — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine)
- [Transparent Data Encryption (TDE) — Microsoft Docs](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/transparent-data-encryption)

---

## Apéndice A — Arquitectura de Integración

Esta sección cubre las reglas de integración entre dominios o servicios que comparten la misma base de datos SQL Server. Las reglas aplican de forma distinta según el tipo de proyecto.

### A.1 Monolito — Schema compartido (`dbo`)

En un monolito con schema único, todos los dominios conviven en el mismo schema. Las restricciones de integridad referencial entre dominios son válidas y deseables.

**Reglas:**
- Las FKs entre tablas de distintos dominios están permitidas — el motor garantiza la integridad
- El prefijo de dominio en el nombre de la tabla es obligatorio para evitar colisiones — ver [Schemas vs Prefijos de Dominio](#2-schemas-vs-prefijos-de-dominio)
- No hay restricción de acceso por dominio a nivel de schema

```sql
-- FK entre dominios en monolito — válido
dbo.crm_contract.person_id → dbo.pld_person.id_pld_person
```

### A.2 Monolito con schemas separados por dominio

Un monolito puede usar schemas distintos por dominio sin ser microservicios. En este caso el schema elimina el prefijo de dominio en el nombre de la tabla, pero las FKs entre schemas del mismo monolito son técnicamente válidas.

**Reglas:**
- FKs entre schemas del mismo monolito: permitidas si el equipo decide asumir el acoplamiento
- Si se anticipa una futura extracción de servicio, tratar desde el inicio como si fueran schemas de servicios distintos (ver A.3)

```sql
-- FK entre schemas en el mismo monolito — válido pero genera acoplamiento
crm.contract.person_id → pld.person.id_person
```

### A.3 Microservicios / DDD — Schema por servicio

Cuando cada servicio es dueño exclusivo de su schema, las FKs entre schemas crean acoplamiento estructural que rompe la autonomía del modelo.

**Reglas:**
- **No hay FKs entre schemas de distintos servicios** — la integridad entre servicios se gestiona a nivel de aplicación o mediante eventos de dominio
- Cada servicio accede únicamente a su propio schema
- Los IDs de otras entidades se almacenan como referencias informativas — sin FK
- La consistencia entre servicios se logra mediante eventos de dominio o llamadas API, no por restricciones de base de datos

```sql
-- Prohibido — FK entre schemas de distintos servicios
FOREIGN KEY (person_id) REFERENCES crm.person (id_person)

-- Correcto — referencia informativa sin FK
person_id  UNIQUEIDENTIFIER  NOT NULL  -- ID del CRM, sin FK, consistencia por evento
```

> Esta regla aplica independientemente de la ubicación física — misma base de datos, bases distintas del mismo servidor, o servidores distintos. La autonomía del modelo es una decisión de arquitectura, no de infraestructura.

**Duplicación de campos entre servicios:** algunos campos (`full_name`, `external_code`) pueden duplicarse entre servicios para desacoplar la persistencia. Es una decisión consciente de diseño, no un error — válida cuando el dato es de solo lectura para el receptor y la consistencia eventual es suficiente.
