# Configuración y Administración — SQL Server

> **Audiencia:** DBA / Infra. Este documento cubre la configuración inicial de la base de datos, gestión de permisos y seguridad de datos.
>
> Para convenciones de nomenclatura y diseño de objetos, ver [sqlserver-naming-conventions.md](sqlserver-naming-conventions.md). Los templates ejecutables están en [sqlserver-ddl-templates.md](sqlserver-ddl-templates.md).

**Versión:** 1.0

---

## Contenido

1. [Configuración de la Base de Datos](#1-configuración-de-la-base-de-datos)
   - [1.1 Collation](#11-collation)
   - [1.2 Read Committed Snapshot Isolation (RCSI)](#12-read-committed-snapshot-isolation-rcsi)
   - [1.3 Recovery Model](#13-recovery-model)
2. [Permisos por Schema](#2-permisos-por-schema)
3. [Seguridad de Datos](#3-seguridad-de-datos)

---

## 1. Configuración de la Base de Datos

Estas decisiones se toman al crear la base de datos y afectan el comportamiento del motor para todos los desarrolladores del proyecto. Documentar el collation y el estado de RCSI en el repositorio.

> Template ejecutable: [A.0](sqlserver-ddl-templates.md#a0-configuración-de-base-de-datos)

### 1.1 Collation

El collation define cómo SQL Server compara y ordena texto. Afecta directamente las búsquedas, los filtros y los índices sobre columnas de texto.

**Collation recomendado para proyectos nuevos:** `Latin1_General_CI_AS`

| Collation | Comportamiento | Cuándo usar |
|-----------|----------------|-------------|
| `Latin1_General_CI_AS` | `'A' = 'a'`, `'e' ≠ 'é'` | **Default recomendado** |
| `SQL_Latin1_General_CP1_CI_AS` | Igual al anterior | Legacy — evitar en proyectos nuevos |
| `Latin1_General_CS_AS` | `'A' ≠ 'a'`, `'e' ≠ 'é'` | Cuando el negocio requiere distinguir mayúsculas |
| `Latin1_General_CI_AI` | `'A' = 'a'`, `'e' = 'é'` | Búsquedas que ignoran acentos |

> `CI` = Case Insensitive · `CS` = Case Sensitive · `AS` = Accent Sensitive · `AI` = Accent Insensitive

Toda la guía de nomenclatura asume `Latin1_General_CI_AS` salvo que se indique lo contrario.

### 1.2 Read Committed Snapshot Isolation (RCSI)

RCSI define cómo SQL Server gestiona la concurrencia entre lectores y escritores.

**Con RCSI activo:**
- Los readers leen la última versión confirmada del dato sin bloquear a writers
- Los writers no bloquean a readers
- Desaparece la mayoría de los deadlocks por contención reader/writer
- `WITH (NOLOCK)` pierde su justificación — comunicar al equipo de desarrollo que no lo usen

> Una vez activado RCSI, comunicar al equipo de desarrollo que está activo. El uso de `WITH (NOLOCK)` como solución a bloqueos es un anti-pattern con consecuencias graves de consistencia (dirty reads, filas a medio escribir).

### 1.3 Recovery Model

| Ambiente | Recovery Model | Razón |
|----------|---------------|-------|
| Producción | `FULL` | Permite point-in-time recovery ante fallo |
| Desarrollo / QA | `SIMPLE` | Sin acumulación de transaction log |

---

## 2. Permisos por Schema

Los permisos siguen el principio de **mínimo privilegio**: cada usuario o servicio accede solo a lo que necesita.

**Usuario de servicio por schema:** cada servicio tiene su propio usuario sin login (autenticación vía Managed Identity o app credential). El servicio no tiene acceso a otros schemas por defecto.

**Roles estándar:**
- `{schema}_reader` — solo lectura (BI, reportes, soporte)
- `{schema}_writer` — lectura y escritura (la aplicación)

**Ownership chaining:** el SP y las tablas que usa pertenecen al mismo schema y propietario, lo que permite ejecutar el SP sin otorgar acceso directo a las tablas. Otorgar solo `EXECUTE` sobre el SP específico — no `SELECT` sobre las tablas.

> En Azure SQL y SQL Server con Windows Auth, preferir **Managed Identities** o **Windows Groups** sobre usuarios con contraseña para eliminar la gestión manual de credenciales.

> Template ejecutable: [A.14](sqlserver-ddl-templates.md#a14-permisos)

---

## 3. Seguridad de Datos

Al trabajar con datos financieros, PII o datos sujetos a regulación (PLD, GDPR, PCI-DSS), considerar las siguientes tecnologías:

| Tecnología | Protege | Cuándo evaluar |
|---|---|---|
| **Transparent Data Encryption (TDE)** | Datos en reposo — archivos `.mdf`, `.ldf`, backups | Producción con datos financieros o PII. Transparente para la aplicación — no requiere cambios en queries |
| **Always Encrypted** | Datos en tránsito y en reposo a nivel de columna | Cuando columnas específicas deben estar cifradas incluso para administradores del servidor |
| **Row-Level Security (RLS)** | Acceso a filas según el contexto del usuario | Cuando distintos usuarios deben ver subconjuntos distintos de los mismos datos |
| **Dynamic Data Masking** | Enmascaramiento de datos en queries | Entornos de desarrollo/QA donde los developers no deben ver datos reales |

### Convención de nombres para Row-Level Security

Los objetos de RLS siguen los prefijos definidos en la [tabla de prefijos de la guía de nomenclatura](sqlserver-naming-conventions.md#55-tabla-de-prefijos-y-constraints--referencia-rápida):

- Función predicado: `fn_rls_{tabla}_filter` — distingue predicados de RLS de funciones de negocio
- Política de seguridad: `pol_{tabla}`

> Documentar cada política de RLS indicando: tabla afectada, predicado utilizado, roles con acceso irrestricto, y cómo se prueba. Una política RLS mal configurada puede denegar acceso silenciosamente o exponer filas que no deberían ser visibles.

> Template ejecutable: [A.15](sqlserver-ddl-templates.md#a15-row-level-security)
