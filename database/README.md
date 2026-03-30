# Database

Lineamientos para el diseño, modelado y uso de bases de datos relacionales y no relacionales.

---

## Guías disponibles

| Guía | Descripción | Estado |
|------|-------------|--------|
| [Convenciones de nomenclatura — SQL Server](./sqlserver-naming-conventions.md) | Nombres de tablas, columnas, índices, constraints y objetos de base de datos | `Stable` |
| [Templates DDL — SQL Server](./sqlserver-ddl-templates.md) | Scripts base reutilizables para creación de tablas, índices, constraints y estructuras comunes | `Stable` |
| [Configuración de Base de Datos — SQL Server](./sqlserver-database-setup.md) | Setup de base de datos: collation, isolation (RCSI), recovery model, seguridad y permisos | `Stable` |

---

## Estado de los documentos

| Documento | Versión | Audiencia | Contenido |
|-----------|--------|-----------|-----------|
| `sqlserver-naming-conventions.md` | 1.0 | Dev | Convenciones, tipos, índices, scripts, patrones |
| `sqlserver-ddl-templates.md` | 1.0 | Dev | Templates DDL ejecutables |
| `sqlserver-database-setup.md` | 1.0 | DBA / Infra | Configuración DB, permisos, seguridad de datos |

---

## Alcance

Estas guías cubren:

- Diseño consistente de esquemas y objetos
- Estandarización de scripts DDL
- Buenas prácticas de modelado relacional
- Configuración base de seguridad y performance

---

## Separación de responsabilidades

- **Dev**
  - Define estructura lógica (tablas, columnas, relaciones)
  - Aplica convenciones de nomenclatura
  - Usa templates DDL estandarizados

- **DBA / Infra**
  - Configura base de datos (collation, isolation levels, recovery model)
  - Gestiona seguridad (roles, permisos, TDE, RLS, masking)
  - Administra performance a nivel instancia/base

---

## Notas

- `sqlserver-database-setup.md` cubre exclusivamente configuración de base de datos:
  - Collation
  - Read Committed Snapshot Isolation (RCSI)
  - Recovery Model
  - Permisos por schema
  - Seguridad (TDE, Row-Level Security, Data Masking)

- No hay solapamiento entre documentos: cada guía tiene un propósito único y acotado.

---

## Principios

1. **Consistency over preference**  
   Las decisiones son estándar, no opinables por proyecto.

2. **Shift-left en calidad**  
   Las reglas se aplican desde el diseño, no en producción.

3. **Separación de concerns**  
   Modelado ≠ configuración ≠ operación.

4. **Reproducibilidad**  
   Todo cambio debe poder ejecutarse desde cero en una base limpia.

---