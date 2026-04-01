# Engineering Standards

> Lineamientos de ingeniería de software para equipos y proyectos. Una fuente de verdad para decisiones técnicas consistentes.

---

## Navegación

| Sección | Descripción | Estado |
|---------|-------------|--------|
| [Database](./database/README.md) | SQL Server, modelado relacional, convenciones de nomenclatura, DDL templates, setup | `Stable` |
| [Git](./git/README.md) | Conventional commits, GitFlow, nomenclatura de ramas, configuración de repos | `Stable` |
| [REST](./rest/README.md) | Diseño de APIs REST, OpenAPI-first, convenciones HTTP, ejemplos y configuración | `Stable` |
| Backend | C#, .NET Core, diseño de APIs, manejo de errores | `Próximamente` |
| Frontend | Angular, estructura de proyectos, gestión de estado | `Próximamente` |
| DevOps | CI/CD, Docker, pipelines, infraestructura | `Próximamente` |
| Architecture | Clean Architecture, microservicios, patrones | `Próximamente` |

---

## Estado de las secciones

| Sección | Estado | Cobertura |
|---------|--------|-----------|
| Database | `Stable` | Naming conventions, DDL templates, database setup |
| Git | `Stable` | Conventional commits, GitFlow por tipo de repo, branch naming, setup de repos |
| REST | `Stable` | Convenciones REST, OpenAPI-first, verbos HTTP, response pattern, ejemplos, setup |
| Backend | `Draft` | Pendiente definición inicial |
| Frontend | `Draft` | Pendiente definición inicial |
| DevOps | `Draft` | Pendiente definición inicial |
| Architecture | `Draft` | Pendiente definición inicial |

---

## Roadmap

### Database
- [x] Convenciones de nomenclatura — SQL Server
- [x] Templates DDL — SQL Server
- [x] Configuración de base de datos — SQL Server

### Git
- [x] Convenciones de Git (Conventional Commits, GitFlow, nomenclatura de ramas)
- [x] Templates y Ejemplos (commits, PRs, flujos por tipo de repo)
- [x] Configuración de Repositorios (branch protection, hooks, CI/CD)

### REST
- [x] Convenciones REST (principios RESTful, verbos HTTP, URLs, headers, response pattern, OpenAPI-first)
- [x] Ejemplos y Templates (requests HTTP, responses, snippets OpenAPI)
- [x] Configuración e Infraestructura (OpenAPI tooling, CORS, rate limiting, SSL/TLS, contract testing)

### Backend
- [ ] Convenciones de C# y .NET Core
- [ ] Diseño de APIs REST
- [ ] Manejo de errores y excepciones
- [ ] Logging y observabilidad

### Frontend
- [ ] Estructura de proyectos Angular
- [ ] Gestión de estado
- [ ] Convenciones de componentes

### DevOps
- [ ] Pipelines CI/CD
- [ ] Estándares Docker
- [ ] Gestión de ambientes

### Architecture
- [ ] Clean Architecture
- [ ] Microservicios
- [ ] Decisiones de diseño (ADRs)

---

## Alcance

Este repositorio define:

- Estándares de desarrollo por capa (DB, Backend, Frontend, Infra)
- Convenciones obligatorias para equipos
- Buenas prácticas alineadas a escalabilidad y mantenibilidad
- Decisiones técnicas reutilizables entre proyectos

---

## Principios

1. **Consistency over preference**  
   Las decisiones son estándar, no opinables por proyecto.

2. **Pragmatism over purity**  
   Se prioriza impacto real sobre dogma técnico.

3. **Shift-left en calidad**  
   Las reglas se aplican desde diseño, no en producción.

4. **Separation of concerns**  
   Cada capa tiene responsabilidades claras.

5. **Reproducibilidad**  
   Todo debe poder levantarse desde cero en un entorno limpio.

---

## Cómo usar este repositorio

- Cada lineamiento es **independiente y autocontenible**
- Navega por sección o accede directo al documento requerido
- Aplica los estándares desde el inicio del desarrollo

Estados posibles:
- `Draft`: En construcción
- `Stable`: Listo para uso en producción
- `Deprecated`: No usar

---

## Cómo contribuir

1. Crear un issue describiendo el lineamiento
2. Validar alcance (evitar solapamientos)
3. Usar el template estándar
4. Nombrar el archivo: `{tecnologia}-{tema}.md`
5. Actualizar el `README.md` correspondiente
6. Someter a revisión técnica

---

## Changelog

| Fecha | Lineamiento |
|-------|------------|
| 2026-03-29 | [Convenciones de nomenclatura — SQL Server](./database/sqlserver-naming-conventions.md) |
| 2026-03-30 | [Templates DDL — SQL Server](./database/sqlserver-ddl-templates.md) |
| 2026-03-30 | [Configuración de base de datos — SQL Server](./database/sqlserver-database-setup.md) |
| 2026-03-30 | [Convenciones de Git](./git/git-conventions.md) |
| 2026-03-30 | [Templates y Ejemplos — Git](./git/git-templates.md) |
| 2026-03-30 | [Configuración de Repositorios — Git](./git/git-setup.md) |
| 2026-03-31 | [Convenciones REST](./rest/rest-conventions.md) |
| 2026-03-31 | [Ejemplos y Templates — REST](./rest/rest-examples.md) |
| 2026-03-31 | [Configuración e Infraestructura — REST](./rest/rest-setup.md) |