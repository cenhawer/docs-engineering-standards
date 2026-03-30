# Templates DDL — SQL Server

> Templates ejecutables de referencia para [Convenciones de Nomenclatura — SQL Server](sqlserver-naming-conventions.md).
> Reemplazar `{tabla}`, `{schema}`, `{columna}` y similares con los valores del proyecto.
> Para el contexto y decisiones detrás de cada template, consultar la guía principal.

**Versión:** 1.0

---

## Contenido

- [A.0 Configuración de Base de Datos](#a0-configuración-de-base-de-datos)
- [A.1 Tablas](#a1-tablas)
- [A.2 Campos — Tipos y Constraints](#a2-campos--tipos-y-constraints)
- [A.3 Índices y Constraints](#a3-índices-y-constraints)
- [A.4 Particionamiento](#a4-particionamiento)
- [A.5 Stored Procedures](#a5-stored-procedures)
- [A.6 Vistas](#a6-vistas)
- [A.7 Funciones](#a7-funciones)
- [A.8 Triggers](#a8-triggers)
- [A.9 Sequences](#a9-sequences)
- [A.10 Tablas Temporales en SPs](#a10-tablas-temporales-en-sps)
- [A.11 Idempotencia DDL](#a11-idempotencia-ddl)
- [A.12 Seeds DML](#a12-seeds-dml)
- [A.13 Rollback Scripts](#a13-rollback-scripts)
- [A.14 Permisos](#a14-permisos)
- [A.15 Row-Level Security](#a15-row-level-security)

---

## A.0 Configuración de Base de Datos

> Contexto: [sqlserver-database-setup.md — 1 Configuración de la Base de Datos](sqlserver-database-setup.md#1-configuración-de-la-base-de-datos)
>
> **Responsabilidad de DBA/Infra.** Este script lo ejecuta el equipo de infraestructura al crear la base de datos. El equipo de desarrollo no ejecuta este script.

```sql
-- ============================================================
-- Script de creación de base de datos
-- Ejecutar como primer paso, antes de cualquier DDL
-- ============================================================

-- 1. Crear con collation explícito
IF NOT EXISTS (SELECT 1 FROM sys.databases WHERE name = 'NombreProyecto')
BEGIN
    CREATE DATABASE [NombreProyecto]
        COLLATE Latin1_General_CI_AS
END
GO

-- 2. Recovery model según ambiente
-- FULL para producción / SIMPLE para desarrollo y QA
ALTER DATABASE [NombreProyecto] SET RECOVERY SIMPLE
GO

USE [NombreProyecto]
GO

-- 3. Activar RCSI — lecturas sin bloqueos, elimina la necesidad de WITH (NOLOCK)
ALTER DATABASE [NombreProyecto] SET READ_COMMITTED_SNAPSHOT ON
GO

-- Verificar configuración
SELECT
    name,
    collation_name,
    is_read_committed_snapshot_on,
    recovery_model_desc
FROM sys.databases
WHERE name = DB_NAME()
```

---

## A.1 Tablas

> Contexto: [3 Tipos de Tablas](sqlserver-naming-conventions.md#3-tipos-de-tablas)

### A.1.1 Tabla principal — sin soft delete

```sql
CREATE TABLE {schema}.{tabla} (
    id_{tabla}      UNIQUEIDENTIFIER    NOT NULL CONSTRAINT df_{tabla}_id_{tabla}  DEFAULT NEWSEQUENTIALID(),
    -- columnas de negocio aquí
    created_at      DATETIME2(3)        NOT NULL CONSTRAINT df_{tabla}_created_at  DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2(3)        NULL,
    created_by      NVARCHAR(100)       NOT NULL,
    updated_by      NVARCHAR(100)       NULL,
    CONSTRAINT pk_{tabla} PRIMARY KEY (id_{tabla})
)
```

### A.1.2 Tabla principal — con soft delete

```sql
CREATE TABLE {schema}.{tabla} (
    id_{tabla}      UNIQUEIDENTIFIER    NOT NULL CONSTRAINT df_{tabla}_id_{tabla}   DEFAULT NEWSEQUENTIALID(),
    -- columnas de negocio aquí
    created_at      DATETIME2(3)        NOT NULL CONSTRAINT df_{tabla}_created_at   DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2(3)        NULL,
    created_by      NVARCHAR(100)       NOT NULL,
    updated_by      NVARCHAR(100)       NULL,
    is_deleted      BIT                 NOT NULL CONSTRAINT df_{tabla}_is_deleted   DEFAULT 0,
    deleted_at      DATETIME2(3)        NULL,
    deleted_by      NVARCHAR(100)       NULL,
    CONSTRAINT pk_{tabla} PRIMARY KEY (id_{tabla})
)
```

### A.1.3 Tabla de relación (muchos a muchos)

```sql
-- Nombre = ambas entidades en orden alfabético (ver 3.2 de la guía)
CREATE TABLE {schema}.{entidad_a}_{entidad_b} (
    {entidad_a}_id  UNIQUEIDENTIFIER    NOT NULL,
    {entidad_b}_id  UNIQUEIDENTIFIER    NOT NULL,
    assigned_at     DATETIME2(3)        NOT NULL CONSTRAINT df_{entidad_a}_{entidad_b}_assigned_at DEFAULT SYSUTCDATETIME(),
    created_by      NVARCHAR(100)       NOT NULL,
    CONSTRAINT pk_{entidad_a}_{entidad_b} PRIMARY KEY ({entidad_a}_id, {entidad_b}_id)
)

-- FKs en scripts separados (ver A.11.4)
ALTER TABLE {schema}.{entidad_a}_{entidad_b}
    ADD CONSTRAINT fk_{entidad_a}_{entidad_b}_{entidad_a}_id
        FOREIGN KEY ({entidad_a}_id) REFERENCES {schema}.{entidad_a} (id_{entidad_a})

ALTER TABLE {schema}.{entidad_a}_{entidad_b}
    ADD CONSTRAINT fk_{entidad_a}_{entidad_b}_{entidad_b}_id
        FOREIGN KEY ({entidad_b}_id) REFERENCES {schema}.{entidad_b} (id_{entidad_b})

-- Índice para la consulta inversa (la PK ya cubre la consulta directa)
CREATE INDEX idx_{entidad_a}_{entidad_b}_{entidad_b}_id
    ON {schema}.{entidad_a}_{entidad_b} ({entidad_b}_id)
```

### A.1.4 Tabla de historial (hist\_) — manual

```sql
-- Creación de la tabla
CREATE TABLE {schema}.hist_{tabla} (
    id_hist_{tabla}     UNIQUEIDENTIFIER    NOT NULL CONSTRAINT df_hist_{tabla}_id_hist_{tabla} DEFAULT NEWSEQUENTIALID(),
    {tabla}_id          UNIQUEIDENTIFIER    NOT NULL,
    -- columnas de negocio a auditar (espejo de la tabla principal, sin campos de auditoría)
    recorded_at         DATETIME2(3)        NOT NULL CONSTRAINT df_hist_{tabla}_recorded_at    DEFAULT SYSUTCDATETIME(),
    recorded_by         NVARCHAR(100)       NOT NULL,
    CONSTRAINT pk_hist_{tabla} PRIMARY KEY (id_hist_{tabla})
)

-- FK en script separado
ALTER TABLE {schema}.hist_{tabla}
    ADD CONSTRAINT fk_hist_{tabla}_{tabla}_{tabla}_id
        FOREIGN KEY ({tabla}_id) REFERENCES {schema}.{tabla} (id_{tabla})
```

### A.1.5 Tabla con Temporal Tables (SYSTEM\_VERSIONING)

```sql
-- SQL Server 2016+ — el motor gestiona hist_{tabla} automáticamente
-- valid_from / valid_to usan DATETIME2(7) — es un requisito del motor, no se puede cambiar
CREATE TABLE {schema}.{tabla} (
    id_{tabla}          UNIQUEIDENTIFIER    NOT NULL,
    -- columnas de negocio aquí
    created_at          DATETIME2(3)        NOT NULL CONSTRAINT df_{tabla}_created_at DEFAULT SYSUTCDATETIME(),
    created_by          NVARCHAR(100)       NOT NULL,
    valid_from          DATETIME2(7)        GENERATED ALWAYS AS ROW START NOT NULL,
    valid_to            DATETIME2(7)        GENERATED ALWAYS AS ROW END   NOT NULL,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to),
    CONSTRAINT pk_{tabla} PRIMARY KEY (id_{tabla})
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = {schema}.hist_{tabla}))

-- Consultar estado en un momento específico
SELECT * FROM {schema}.{tabla}
FOR SYSTEM_TIME AS OF '2026-01-15'

-- Consultar cambios en un rango de fechas
SELECT * FROM {schema}.{tabla}
FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-03-01'
```

### A.1.6 Tabla de log (log\_)

```sql
-- Sin FK — los IDs son referencias informativas, nunca restricciones referenciales
CREATE TABLE {schema}.log_{evento} (
    id_log_{evento}     UNIQUEIDENTIFIER    NOT NULL CONSTRAINT df_log_{evento}_id_log_{evento} DEFAULT NEWSEQUENTIALID(),
    event_name          NVARCHAR(200)       NOT NULL,
    event_detail        NVARCHAR(MAX)       NULL,
    entity_id           UNIQUEIDENTIFIER    NULL,   -- referencia informativa, sin FK
    occurred_at         DATETIME2(3)        NOT NULL CONSTRAINT df_log_{evento}_occurred_at    DEFAULT SYSUTCDATETIME(),
    triggered_by        NVARCHAR(100)       NOT NULL,
    CONSTRAINT pk_log_{evento} PRIMARY KEY (id_log_{evento})
)
```

### A.1.7 Catálogo del sistema (sin auditoría)

```sql
-- Valores fijos, no editables por el usuario
CREATE TABLE {schema}.cat_{nombre} (
    id_cat_{nombre}     UNIQUEIDENTIFIER    NOT NULL CONSTRAINT df_cat_{nombre}_id_cat_{nombre} DEFAULT NEWSEQUENTIALID(),
    code                NVARCHAR(10)        NOT NULL,
    description         NVARCHAR(200)       NOT NULL,
    is_active           BIT                 NOT NULL CONSTRAINT df_cat_{nombre}_is_active       DEFAULT 1,
    CONSTRAINT pk_cat_{nombre}      PRIMARY KEY (id_cat_{nombre}),
    CONSTRAINT uk_cat_{nombre}_code UNIQUE (code)
)
```

### A.1.8 Catálogo gestionado por usuario (con auditoría)

```sql
-- Valores configurables desde la UI por usuarios administradores
CREATE TABLE {schema}.cat_{nombre} (
    id_cat_{nombre}     UNIQUEIDENTIFIER    NOT NULL CONSTRAINT df_cat_{nombre}_id_cat_{nombre}  DEFAULT NEWSEQUENTIALID(),
    code                NVARCHAR(50)        NOT NULL,
    description         NVARCHAR(200)       NOT NULL,
    is_active           BIT                 NOT NULL CONSTRAINT df_cat_{nombre}_is_active         DEFAULT 1,
    created_at          DATETIME2(3)        NOT NULL CONSTRAINT df_cat_{nombre}_created_at        DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2(3)        NULL,
    created_by          NVARCHAR(100)       NOT NULL,
    updated_by          NVARCHAR(100)       NULL,
    CONSTRAINT pk_cat_{nombre}      PRIMARY KEY (id_cat_{nombre}),
    CONSTRAINT uk_cat_{nombre}_code UNIQUE (code)
)
```

---

## A.2 Campos — Tipos y Constraints

> Contexto: [4 Campos](sqlserver-naming-conventions.md#4-campos)

### A.2.1 Campos de auditoría

```sql
created_at  DATETIME2(3)    NOT NULL CONSTRAINT df_{tabla}_created_at DEFAULT SYSUTCDATETIME(),
updated_at  DATETIME2(3)    NULL,
created_by  NVARCHAR(100)   NOT NULL,
updated_by  NVARCHAR(100)   NULL
```

### A.2.2 Clave primaria

```sql
-- UUID secuencial (estándar de esta guía)
id_{tabla}  UNIQUEIDENTIFIER  NOT NULL  CONSTRAINT df_{tabla}_id_{tabla} DEFAULT NEWSEQUENTIALID()

-- ID numérico — solo cuando el estándar externo o la regulación lo requiere
id_folio    BIGINT  NOT NULL  CONSTRAINT df_{tabla}_id_folio DEFAULT NEXT VALUE FOR {schema}.seq_{nombre}
```

### A.2.3 Claves foráneas

```sql
-- FK NOT NULL — relación obligatoria
{entidad}_id    UNIQUEIDENTIFIER    NOT NULL

-- FK NULL — relación opcional
{entidad}_id    UNIQUEIDENTIFIER    NULL
```

### A.2.4 Soft delete — mínimo obligatorio

```sql
is_deleted  BIT  NOT NULL CONSTRAINT df_{tabla}_is_deleted DEFAULT 0
```

### A.2.5 Soft delete — con trazabilidad completa

```sql
-- Si se agrega uno de los dos campos opcionales, se agregan los dos
is_deleted  BIT           NOT NULL CONSTRAINT df_{tabla}_is_deleted DEFAULT 0,
deleted_at  DATETIME2(3)  NULL,
deleted_by  NVARCHAR(100) NULL
```

### A.2.6 Campo code (slug kebab-case)

```sql
code  NVARCHAR(50)  NOT NULL,

-- CHECK con COLLATE explícito para garantizar validación de minúsculas
-- independientemente del collation de la base de datos
CONSTRAINT ck_{tabla}_code_format
    CHECK (code COLLATE Latin1_General_CS_AS LIKE '%[a-z0-9]%'
       AND code COLLATE Latin1_General_CS_AS NOT LIKE '%[^a-z0-9-]%'
       AND code NOT LIKE '-%'
       AND code NOT LIKE '%-'
       AND code NOT LIKE '%--%'),
CONSTRAINT uk_{tabla}_code UNIQUE (code)
```

### A.2.7 Columna JSON con validación

```sql
-- Objeto único {} — nullable
ALTER TABLE {schema}.{tabla}
    ADD CONSTRAINT ck_{tabla}_{columna}_isjson
        CHECK ({columna} IS NULL OR ISJSON({columna}) = 1)
ALTER TABLE {schema}.{tabla}
    ADD CONSTRAINT ck_{tabla}_{columna}_is_object
        CHECK ({columna} IS NULL OR LEFT(LTRIM({columna}), 1) = '{')

-- Array de objetos [{}] — nullable
ALTER TABLE {schema}.{tabla}
    ADD CONSTRAINT ck_{tabla}_{columna}_isjson
        CHECK ({columna} IS NULL OR ISJSON({columna}) = 1)
ALTER TABLE {schema}.{tabla}
    ADD CONSTRAINT ck_{tabla}_{columna}_is_array
        CHECK ({columna} IS NULL OR LEFT(LTRIM({columna}), 1) = '[')

-- Deshabilitar durante bulk load, re-habilitar con verificación completa al terminar
ALTER TABLE {schema}.{tabla} NOCHECK CONSTRAINT ck_{tabla}_{columna}_isjson
-- ... BULK INSERT o carga masiva ...
ALTER TABLE {schema}.{tabla} WITH CHECK CHECK CONSTRAINT ck_{tabla}_{columna}_isjson
```

### A.2.8 Tipos numéricos — referencia rápida

```sql
amount          DECIMAL(18, 2)  NOT NULL   -- montos monetarios — precisión exacta
tax_rate        DECIMAL(5, 4)   NOT NULL   -- porcentajes como fracción: 0.1600 = 16%
exchange_rate   DECIMAL(10, 6)  NOT NULL   -- tipos de cambio con mayor escala
quantity        INT             NOT NULL   -- cantidades enteras estándar
year            SMALLINT        NOT NULL   -- años y rangos cortos (2 bytes)
external_id     BIGINT          NOT NULL   -- IDs externos, contadores masivos
is_active       BIT             NOT NULL   -- flags y booleanos — nunca TINYINT
latitude        FLOAT           NOT NULL   -- coordenadas geográficas (imprecisión aceptable)

-- Prohibidos para montos
-- amount  FLOAT        -- imprecisión binaria: 0.1 + 0.2 ≠ 0.3
-- amount  MONEY        -- tipo propietario, imprecisión en multiplicación/división
```

---

## A.3 Índices y Constraints

> Contexto: [5 Índices y Constraints](sqlserver-naming-conventions.md#5-índices-y-constraints)

### A.3.1 Índice común

```sql
CREATE INDEX idx_{tabla}_{columna}
    ON {schema}.{tabla} ({columna})
```

### A.3.2 Índice único como constraint (unicidad simple)

```sql
-- Dentro del CREATE TABLE o vía ALTER TABLE
CONSTRAINT uk_{tabla}_{columna} UNIQUE ({columna})
```

### A.3.3 Índice único con filtro (filtered unique index)

```sql
-- Cuando la unicidad aplica solo sobre registros activos
CREATE UNIQUE INDEX uk_{tabla}_{columna}
    ON {schema}.{tabla} ({columna})
    WHERE is_deleted = 0
```

### A.3.4 Índice compuesto

```sql
-- No único
CREATE INDEX idx_{tabla}_{col1}_{col2}
    ON {schema}.{tabla} ({col1}, {col2})   -- más selectivo primero

-- Único
CREATE UNIQUE INDEX uk_{tabla}_{col1}_{col2}
    ON {schema}.{tabla} ({col1}, {col2})
```

### A.3.5 Covering index (INCLUDE)

```sql
-- Nombre basado en columna(s) de búsqueda, no en las columnas INCLUDE
CREATE INDEX idx_{tabla}_{columna_busqueda}
    ON {schema}.{tabla} ({columna_busqueda})
    INCLUDE ({col_proyectada_1}, {col_proyectada_2})
    WHERE is_deleted = 0   -- filtro opcional
```

### A.3.6 Columnstore index (analytics)

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX ccsi_{tabla}_analytics
    ON {schema}.{tabla} ({col1}, {col2}, {col3})
```

### A.3.7 Constraints nombrados

```sql
CONSTRAINT pk_{tabla}                           PRIMARY KEY (id_{tabla})
CONSTRAINT uk_{tabla}_{columna}                 UNIQUE ({columna})
CONSTRAINT ck_{tabla}_{columna_o_descripcion}   CHECK ({condicion})
CONSTRAINT df_{tabla}_{columna}                 DEFAULT {valor} [FOR {columna}]
CONSTRAINT fk_{origen}_{destino}_{columna}
    FOREIGN KEY ({columna}) REFERENCES {schema}.{destino} (id_{destino})
```

### A.3.8 Índice explícito en columna FK

```sql
-- SQL Server NO crea automáticamente índices en columnas FK
CREATE INDEX idx_{tabla}_{entidad}_id
    ON {schema}.{tabla} ({entidad}_id)
```

---

## A.4 Particionamiento

> Contexto: [5.8 Particionamiento — Naming](sqlserver-naming-conventions.md#58-particionamiento--naming)

```sql
-- Partition Function — pf_{tabla}_{criterio}
CREATE PARTITION FUNCTION pf_{tabla}_monthly (DATETIME2(3))
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01',
    '2026-09-01', '2026-10-01', '2026-11-01', '2026-12-01'
)

-- Partition Scheme — ps_{tabla}_{criterio}
CREATE PARTITION SCHEME ps_{tabla}_monthly
AS PARTITION pf_{tabla}_monthly
ALL TO ([PRIMARY])

-- Tabla particionada
-- La columna de partición debe estar en el clustered index para partition elimination
CREATE TABLE {schema}.{tabla} (
    id_{tabla}      UNIQUEIDENTIFIER    NOT NULL CONSTRAINT df_{tabla}_id_{tabla} DEFAULT NEWSEQUENTIALID(),
    occurred_at     DATETIME2(3)        NOT NULL CONSTRAINT df_{tabla}_occurred_at DEFAULT SYSUTCDATETIME(),
    -- columnas adicionales
    CONSTRAINT pk_{tabla} PRIMARY KEY NONCLUSTERED (id_{tabla}),
    INDEX cidx_{tabla}_occurred_at CLUSTERED (occurred_at)
) ON ps_{tabla}_monthly (occurred_at)
```

---

## A.5 Stored Procedures

> Contexto: [6.2 Stored Procedures](sqlserver-naming-conventions.md#62-stored-procedures)

### A.5.1 SP con DML (escritura)

```sql
CREATE OR ALTER PROCEDURE {schema}.usp_{accion}_{entidad}
    @{param1}   {tipo},
    @{param2}   {tipo}
AS
BEGIN
    SET NOCOUNT ON      -- evita mensajes "N rows affected" que rompen algunos clientes
    SET XACT_ABORT ON   -- rollback automático ante cualquier error de runtime

    BEGIN TRY
        BEGIN TRANSACTION

        -- lógica DML aquí

        COMMIT TRANSACTION
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION
        THROW   -- re-lanza el error original al caller
    END CATCH
END
GO
```

### A.5.2 SP de solo lectura

```sql
CREATE OR ALTER PROCEDURE {schema}.usp_get_{entidad}_by_{criterio}
    @{param}    {tipo}
AS
BEGIN
    SET NOCOUNT ON

    BEGIN TRY
        -- Listar columnas explícitamente — nunca SELECT *
        SELECT
            {col1},
            {col2}
        FROM {schema}.{tabla}
        WHERE {criterio} = @{param}
          AND is_deleted = 0
    END TRY
    BEGIN CATCH
        THROW
    END CATCH
END
GO
```

---

## A.6 Vistas

> Contexto: [6.1 Vistas](sqlserver-naming-conventions.md#61-vistas)

```sql
CREATE OR ALTER VIEW {schema}.v_{descripcion} AS
    -- Listar columnas explícitamente — nunca SELECT *
    SELECT
        {col1},
        {col2}
    FROM {schema}.{tabla}
    WHERE is_deleted = 0
GO
```

---

## A.7 Funciones

> Contexto: [6.3 Funciones](sqlserver-naming-conventions.md#63-funciones)

```sql
-- Escalar — fn_{descripcion}
-- Advertencia: se ejecuta una vez por fila. Preferir TVF inline cuando sea posible.
CREATE OR ALTER FUNCTION {schema}.fn_{descripcion} (@{param} {tipo})
RETURNS {tipo_retorno}
AS
BEGIN
    DECLARE @result {tipo_retorno}
    -- lógica
    RETURN @result
END
GO

-- Table-valued inline — tvf_{descripcion}
-- Preferir sobre escalar para lógica que opera sobre conjuntos de datos
CREATE OR ALTER FUNCTION {schema}.tvf_{descripcion} (@{param} {tipo})
RETURNS TABLE
AS
RETURN (
    SELECT
        {col1},
        {col2}
    FROM {schema}.{tabla}
    WHERE -- condición
)
GO
```

---

## A.8 Triggers

> Contexto: [6.4 Triggers](sqlserver-naming-conventions.md#64-triggers)

```sql
-- El schema se declara en el nombre del trigger
-- Documentar en el cuerpo por qué no pudo resolverse con lógica de aplicación
CREATE OR ALTER TRIGGER {schema}.trg_{tabla}_{evento}
ON {schema}.{tabla} AFTER {INSERT|UPDATE|DELETE}
AS
BEGIN
    SET NOCOUNT ON
    -- lógica del trigger
END
GO
```

---

## A.9 Sequences

> Contexto: [6.5 Sequences](sqlserver-naming-conventions.md#65-sequences)

```sql
-- seq_{descripcion}
CREATE SEQUENCE {schema}.seq_{descripcion}
    AS BIGINT
    START WITH 1
    INCREMENT BY 1
    NO CYCLE
    CACHE 50

-- Como DEFAULT en columna
id_folio    BIGINT  NOT NULL CONSTRAINT df_{tabla}_id_folio DEFAULT NEXT VALUE FOR {schema}.seq_{descripcion}

-- Obtener el valor antes del INSERT
DECLARE @folio BIGINT = NEXT VALUE FOR {schema}.seq_{descripcion}
-- ... lógica de negocio con @folio ...
INSERT INTO {schema}.{tabla} (id_folio) VALUES (@folio)
```

---

## A.10 Tablas Temporales en SPs

> Contexto: [6.6 Tablas Temporales en SPs](sqlserver-naming-conventions.md#66-tablas-temporales-en-sps)

```sql
-- Tabla temporal local — #tmp_{descripcion}
-- Para conjuntos > 100 filas o cuando se hacen JOINs subsiguientes
CREATE TABLE #tmp_{descripcion} (
    {col1}  {tipo}  NOT NULL,
    {col2}  {tipo}  NULL
)

-- Se puede indexar para mejorar JOINs posteriores
CREATE INDEX idx_tmp_{descripcion}_{col}
    ON #tmp_{descripcion} ({col})

-- Variable de tabla — para conjuntos pequeños (< 100 filas, sin JOINs complejos)
DECLARE @{nombre} TABLE (
    {col1}  {tipo}  NOT NULL
)
```

---

## A.11 Idempotencia DDL

> Contexto: [8 Scripts DDL y DML](sqlserver-naming-conventions.md#8-scripts-ddl-y-dml)

### A.11.1 Schema

```sql
-- CREATE SCHEMA requiere estar en un batch propio — se envuelve en EXEC
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = '{schema}')
    EXEC('CREATE SCHEMA {schema}')
GO
```

### A.11.2 Tabla

```sql
IF NOT EXISTS (
    SELECT 1 FROM sys.tables
    WHERE name = '{tabla}' AND schema_id = SCHEMA_ID('{schema}')
)
BEGIN
    CREATE TABLE {schema}.{tabla} ( ... )
END
```

### A.11.3 Índice

```sql
-- Filtrar por object_id: dos schemas pueden tener un índice con el mismo nombre
IF NOT EXISTS (
    SELECT 1 FROM sys.indexes
    WHERE name = 'idx_{tabla}_{columna}'
    AND object_id = OBJECT_ID('{schema}.{tabla}')
)
BEGIN
    CREATE INDEX idx_{tabla}_{columna} ON {schema}.{tabla} ({columna})
END
```

### A.11.4 Foreign Key

```sql
IF NOT EXISTS (
    SELECT 1 FROM sys.foreign_keys
    WHERE name = 'fk_{origen}_{destino}_{columna}'
    AND parent_object_id = OBJECT_ID('{schema}.{origen}')
)
BEGIN
    ALTER TABLE {schema}.{origen}
    ADD CONSTRAINT fk_{origen}_{destino}_{columna}
        FOREIGN KEY ({columna}) REFERENCES {schema}.{destino} (id_{destino})
END
```

### A.11.5 Otros constraints

```sql
-- PRIMARY KEY
IF NOT EXISTS (
    SELECT 1 FROM sys.key_constraints
    WHERE name = 'pk_{tabla}' AND type = 'PK'
    AND parent_object_id = OBJECT_ID('{schema}.{tabla}')
)
BEGIN
    ALTER TABLE {schema}.{tabla} ADD CONSTRAINT pk_{tabla} PRIMARY KEY (id_{tabla})
END

-- UNIQUE
IF NOT EXISTS (
    SELECT 1 FROM sys.key_constraints
    WHERE name = 'uk_{tabla}_{columna}' AND type = 'UQ'
    AND parent_object_id = OBJECT_ID('{schema}.{tabla}')
)
BEGIN
    ALTER TABLE {schema}.{tabla} ADD CONSTRAINT uk_{tabla}_{columna} UNIQUE ({columna})
END

-- DEFAULT
IF NOT EXISTS (
    SELECT 1 FROM sys.default_constraints
    WHERE name = 'df_{tabla}_{columna}'
    AND parent_object_id = OBJECT_ID('{schema}.{tabla}')
)
BEGIN
    ALTER TABLE {schema}.{tabla}
    ADD CONSTRAINT df_{tabla}_{columna} DEFAULT {valor} FOR {columna}
END

-- CHECK
IF NOT EXISTS (
    SELECT 1 FROM sys.check_constraints
    WHERE name = 'ck_{tabla}_{columna}'
    AND parent_object_id = OBJECT_ID('{schema}.{tabla}')
)
BEGIN
    ALTER TABLE {schema}.{tabla}
    ADD CONSTRAINT ck_{tabla}_{columna} CHECK ({condicion})
END
```

### A.11.6 SP, Vista, Función, Trigger

```sql
-- CREATE OR ALTER disponible desde SQL Server 2016 SP1
-- Crea si no existe, reemplaza si existe — sin DROP previo necesario
CREATE OR ALTER PROCEDURE {schema}.usp_{accion}_{entidad} ...
CREATE OR ALTER VIEW {schema}.v_{descripcion} ...
CREATE OR ALTER FUNCTION {schema}.fn_{descripcion} ...
CREATE OR ALTER TRIGGER {schema}.trg_{tabla}_{evento} ...
```

---

## A.12 Seeds DML

> Contexto: [8 Scripts DDL y DML — Idempotencia DML](sqlserver-naming-conventions.md#idempotencia-dml--seeds)

```sql
-- MERGE idempotente — usar cuando los valores pueden cambiar entre versiones
-- HOLDLOCK previene race conditions bajo concurrencia
-- El campo ON debe ser el código natural (code), no el UUID
MERGE INTO {schema}.cat_{nombre} WITH (HOLDLOCK) AS target
USING (VALUES
    (N'{code1}', N'{desc1}', 1),
    (N'{code2}', N'{desc2}', 1)
) AS source (code, description, is_active)
ON target.code = source.code
WHEN NOT MATCHED BY TARGET THEN
    INSERT (code, description, is_active)
    VALUES (source.code, source.description, source.is_active)
WHEN MATCHED THEN
    UPDATE SET
        description = source.description,
        is_active   = source.is_active;

-- Guard simple — solo cuando los datos son estrictamente inmutables
IF NOT EXISTS (SELECT 1 FROM {schema}.cat_{nombre})
BEGIN
    INSERT INTO {schema}.cat_{nombre} (code, description, is_active)
    VALUES
        (N'{code1}', N'{desc1}', 1),
        (N'{code2}', N'{desc2}', 1)
END
```

---

## A.13 Rollback Scripts

> Contexto: [8 Scripts DDL y DML — Rollback scripts](sqlserver-naming-conventions.md#rollback-scripts)

```sql
-- Rollback de schema (solo si está completamente vacío)
IF EXISTS (SELECT 1 FROM sys.schemas WHERE name = '{schema}')
    EXEC('DROP SCHEMA {schema}')

-- Rollback de tabla
IF EXISTS (
    SELECT 1 FROM sys.tables
    WHERE name = '{tabla}' AND schema_id = SCHEMA_ID('{schema}')
)
    DROP TABLE {schema}.{tabla}

-- Rollback de FK
IF EXISTS (
    SELECT 1 FROM sys.foreign_keys
    WHERE name = 'fk_{origen}_{destino}_{columna}'
    AND parent_object_id = OBJECT_ID('{schema}.{origen}')
)
    ALTER TABLE {schema}.{origen} DROP CONSTRAINT fk_{origen}_{destino}_{columna}

-- Rollback de índice
IF EXISTS (
    SELECT 1 FROM sys.indexes
    WHERE name = 'idx_{tabla}_{columna}'
    AND object_id = OBJECT_ID('{schema}.{tabla}')
)
    DROP INDEX idx_{tabla}_{columna} ON {schema}.{tabla}

-- Rollback de constraint (pk_, uk_, ck_, df_)
IF EXISTS (
    SELECT 1 FROM sys.key_constraints  -- o sys.check_constraints / sys.default_constraints
    WHERE name = '{nombre_constraint}'
    AND parent_object_id = OBJECT_ID('{schema}.{tabla}')
)
    ALTER TABLE {schema}.{tabla} DROP CONSTRAINT {nombre_constraint}

-- Rollback de trigger
IF EXISTS (
    SELECT 1 FROM sys.triggers
    WHERE name = 'trg_{tabla}_{evento}'
    AND parent_id = OBJECT_ID('{schema}.{tabla}')
)
    DROP TRIGGER {schema}.trg_{tabla}_{evento}
```

---

## A.14 Permisos

> Contexto: [sqlserver-database-setup.md — 2 Permisos / 3 Seguridad](sqlserver-database-setup.md#2-permisos-por-schema)
>
> **Responsabilidad de DBA/Infra.** Este script lo ejecuta el equipo de infraestructura. El equipo de desarrollo debe conocer el modelo conceptual (ver Apéndice C de la guía principal), no ejecutar estos scripts.

```sql
-- Usuario de servicio por schema — sin login (autenticación vía managed identity o app)
CREATE USER {schema}_service WITHOUT LOGIN
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::{schema} TO {schema}_service

-- Rol de solo lectura (BI, reportes, soporte)
CREATE ROLE {schema}_reader
GRANT SELECT ON SCHEMA::{schema} TO {schema}_reader
ALTER ROLE {schema}_reader ADD MEMBER {nombre_usuario}

-- Rol de escritura para la aplicación
CREATE ROLE {schema}_writer
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::{schema} TO {schema}_writer

-- Permiso de ejecución sobre SP (sin acceso directo a tablas)
GRANT EXECUTE ON {schema}.usp_{nombre} TO {schema}_reader

-- Verificar permisos existentes sobre un schema
SELECT
    dp.name             AS principal_name,
    dp.type_desc        AS principal_type,
    p.permission_name,
    p.state_desc
FROM sys.database_permissions p
JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
JOIN sys.schemas s ON p.major_id = s.schema_id
WHERE s.name = '{schema}'
ORDER BY dp.name, p.permission_name
```

---

## A.15 Row-Level Security

> Contexto: [sqlserver-database-setup.md — 2 Permisos / 3 Seguridad](sqlserver-database-setup.md#2-permisos-por-schema)

```sql
-- Función predicado — fn_rls_{descripcion}
-- Retorna 1 (acceso permitido) o 0 (acceso denegado)
CREATE FUNCTION {schema}.fn_rls_{tabla}_filter (@created_by NVARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS result
    WHERE @created_by = USER_NAME()
       OR IS_MEMBER('{schema}_admin') = 1

-- Política de seguridad — pol_{tabla}
CREATE SECURITY POLICY {schema}.pol_{tabla}
    ADD FILTER PREDICATE {schema}.fn_rls_{tabla}_filter(created_by) ON {schema}.{tabla},
    ADD BLOCK  PREDICATE {schema}.fn_rls_{tabla}_filter(created_by) ON {schema}.{tabla} AFTER INSERT
WITH (STATE = ON)
```
