# Convenciones de Git

> **Audiencia:** equipo de desarrollo. Esta guía no cubre configuración de repositorios, branch protection ni integración CI/CD — esas responsabilidades corresponden al equipo de Infra/DevOps ([git-setup.md](git-setup.md)).
>
> **Cómo usar esta guía:** para un proyecto nuevo, lee las secciones 1 → 2 → 3 y luego ve a la sección 4 que corresponde al tipo de repositorio. Para consulta puntual, ve directamente a la sección temática correspondiente.

**Versión:** 1.0

---

## Contenido

1. [Reglas Generales](#1-reglas-generales)
2. [Conventional Commits](#2-conventional-commits)
   - [2.1 Estructura](#21-estructura)
   - [2.2 Tipos](#22-tipos)
   - [2.3 Scope](#23-scope)
   - [2.4 Breaking Changes](#24-breaking-changes)
3. [Nomenclatura de Ramas](#3-nomenclatura-de-ramas)
   - [3.1 Formato](#31-formato)
   - [3.2 Tipos de Rama](#32-tipos-de-rama)
4. [GitFlow por Tipo de Repositorio](#4-gitflow-por-tipo-de-repositorio)
   - [4.1 Backend / Microservicio](#41-backend--microservicio)
   - [4.2 Frontend](#42-frontend)
   - [4.3 Base de Datos](#43-base-de-datos)
   - [4.4 Documentación](#44-documentación)
   - [4.5 Librería / Paquete](#45-librería--paquete)
5. [Pull Requests](#5-pull-requests)
6. [Anti-patterns](#6-anti-patterns)
7. [Referencias](#7-referencias)

---

## Contexto

Un historial de commits inconsistente genera deuda técnica invisible: es imposible generar changelogs automáticos, los bisect de bugs se vuelven manuales y los code reviews pierden contexto. Esta guía establece un único criterio para todos los repositorios, basado en el estándar [Conventional Commits](https://www.conventionalcommits.org/).

---

## 1. Reglas Generales

### 1.1 Formato y lenguaje

| Regla | Valor |
|-------|-------|
| Estándar de commits | Conventional Commits v1.0 |
| Idioma | decisión del equipo — un solo idioma en todo el proyecto |
| Tickets en ramas | Requerido — formato `{PROYECTO}-{número}` |
| Commits directos a `main` | Prohibido |
| Commits directos a `develop` | Prohibido |
| Force push a ramas protegidas | Prohibido |

> **Idioma:** puede ser español o inglés según el contexto del proyecto. Lo que no es negociable es la consistencia: **un solo idioma en todos los commits y ramas del repositorio**. No se permiten repositorios mixtos. Los nombres técnicos (métodos, clases, variables) son independientes de esta decisión.

### 1.2 Atomicidad

**Regla:** cada commit representa un cambio cohesivo y funcional. No se mezclan refactors con features ni fixes con cambios de estilo.

**Por qué:** commits atómicos permiten hacer `git bisect`, `git revert` y `git cherry-pick` de forma quirúrgica sin efectos colaterales.

**Cuándo sí:** un commit puede incluir múltiples archivos si todos son parte del mismo cambio lógico (ej: agregar un endpoint junto con su test y su DTO).

**Cuándo no:** no mezclar en un mismo commit un fix de bug con un cambio de formato no relacionado.

---

## 2. Conventional Commits

### 2.1 Estructura

```
<tipo>(<scope>): <descripción>

[cuerpo opcional]

[footer opcional]
```

| Parte | Regla |
|-------|-------|
| `tipo` | Requerido — ver [tabla 2.2](#22-tipos) |
| `scope` | Opcional pero recomendado. Módulo o área afectada |
| `descripción` | Requerida. Imperativo, minúsculas, sin punto final, máx. 72 caracteres |
| `cuerpo` | Opcional. Explicación del "por qué", no del "qué" |
| `footer` | Opcional. Referencias a tickets, breaking changes |

> **Imperativo:** la descripción debe responder "¿qué hace este commit?". Usar imperativo: "agregar", "corregir", "eliminar" — no "agregado", "se corrigió", "eliminando".

### 2.2 Tipos

| Tipo | Uso | Afecta versión semántica |
|------|-----|--------------------------|
| `feat` | Nueva funcionalidad para el usuario | `MINOR` (1.x.0) |
| `fix` | Corrección de bug que afecta al usuario | `PATCH` (1.0.x) |
| `refactor` | Cambio de código sin impacto funcional | Sin cambio |
| `test` | Agregar o corregir tests | Sin cambio |
| `docs` | Cambios en documentación | Sin cambio |
| `chore` | Tareas de mantenimiento (dependencias, configuración) | Sin cambio |
| `style` | Formato de código, espacios — sin cambio lógico | Sin cambio |
| `perf` | Mejora de performance sin cambio funcional | Sin cambio |
| `ci` | Cambios en pipelines o configuración CI/CD | Sin cambio |
| `build` | Cambios en sistema de build o dependencias externas | Sin cambio |
| `migration` | Scripts de migración de base de datos | `MINOR` o `PATCH` |
| `revert` | Revierte un commit anterior | Depende del revertido |

> **`feat` vs `refactor`:** `feat` agrega comportamiento observable por el usuario o sistema externo. `refactor` cambia la implementación interna sin alterar el comportamiento observable.

> **Hotfixes:** un hotfix usa el tipo `fix` — la urgencia la comunica la rama (`hotfix/*`), no el tipo de commit. Mezclar ambas capas genera inconsistencias en el historial y en la generación de changelogs.

### 2.3 Scope

El scope identifica el módulo o área afectada. Se define por proyecto; valores de referencia:

| Tipo de repo | Scopes típicos |
|--------------|----------------|
| Backend | `auth`, `users`, `orders`, `payments`, `api`, `db` |
| Frontend | `ui`, `auth`, `dashboard`, `forms`, `routing` |
| Base de datos | `schema`, `migrations`, `indexes`, `seed` |
| Librería | nombre del módulo exportado |
| Documentación | nombre del documento o sección |

### 2.4 Breaking Changes

**Regla:** indicar breaking changes en el footer con `BREAKING CHANGE:` o con `!` después del tipo.

```
feat(api)!: cambiar estructura del response de /users

BREAKING CHANGE: el campo `nombre` pasa a llamarse `nombre_completo`.
Los consumidores deben actualizar sus integraciones.
```

Breaking changes afectan la versión `MAJOR` en versionado semántico.

---

## 3. Nomenclatura de Ramas

### 3.1 Formato

**Regla:** `{tipo}/{ticket}-{descripcion-corta}`

| Parte | Regla |
|-------|-------|
| `tipo` | Tipo de rama — ver [tabla 3.2](#32-tipos-de-rama) |
| `ticket` | ID del ticket en el backlog — requerido salvo en `docs` y `chore` sin ticket asociado |
| `descripcion-corta` | `kebab-case`, máx. 4-5 palabras, sin artículos, en el idioma del proyecto |
| Separador tipo/resto | `/` |
| Separador ticket/descripción | `-` |

```
feature/CDF-02-agregar-usuario
fix/CDF-15-error-login
hotfix/CDF-301-calcular-iva
docs/guia-autenticacion
refactor/CDF-88-extraer-servicio-descuento
migration/CDF-55-tabla-categorias

-- Incorrecto
Feature/CDF-02-AgregarUsuario     -- mayúsculas
new-feature-usuario               -- sin tipo ni ticket
feature-CDF-02-agregar-usuario    -- separador incorrecto
feat/CDF-02-agregar-usuario       -- abreviatura (usar nombre completo)
feature/agregar-usuario           -- sin ticket
```

> **Sin ticket:** se permite omitir el ticket solo en ramas `docs` o `chore` cuando el cambio no viene de una historia de usuario. En todos los demás casos es obligatorio.

### 3.2 Tipos de Rama

| Tipo | Cuándo usar | Origen | Destino |
|------|-------------|--------|---------|
| `feature` | Nueva funcionalidad | `develop` | `develop` |
| `fix` | Corrección de bug no urgente | `develop` | `develop` |
| `hotfix` | Corrección urgente a producción | `main` | `main` → `develop` |
| `refactor` | Mejora interna sin cambio funcional | `develop` | `develop` |
| `test` | Agregar o actualizar tests | `develop` | `develop` |
| `docs` | Documentación | `develop` o `main`* | mismo origen |
| `chore` | Mantenimiento, dependencias | `develop` | `develop` |
| `style` | Formato de código — sin cambio lógico | `develop` | `develop` |
| `migration` | Cambios de esquema de base de datos | `develop` | `develop` |
| `release` | Preparación de versión | `develop` | `main` → `develop` |

> *`docs` desde `main` solo en repos de documentación pura, donde no existe rama `develop`.

---

## 4. GitFlow por Tipo de Repositorio

### 4.1 Backend / Microservicio

**Ramas permanentes:** `main`, `develop`
**Ramas temporales:** `feature/*`, `fix/*`, `hotfix/*`, `refactor/*`, `test/*`, `chore/*`, `release/*`

#### Flujo principal

```
feature / fix / refactor / test / chore → develop → release/* → main
```

**Pasos:**

1. Actualizar `develop` y crear la rama de trabajo desde ahí.
2. Desarrollar con commits convencionales.
3. Antes de abrir el PR, sincronizar con `develop`:
   ```
   git fetch origin && git rebase origin/develop
   ```
4. Abrir PR hacia `develop` — mínimo 1 aprobador, CI en verde.
5. Merge a `develop`. Eliminar la rama.

#### Flujo de release

```
develop → release/x.y.z → main (tag) → develop (backmerge obligatorio)
```

La rama `release` solo recibe fixes de estabilización — no nuevas features. Una vez mergeada a `main`, se crea el tag de versión y se hace backmerge a `develop`.

#### Flujo de hotfix

```
main → hotfix/* → main (tag) → develop (backmerge obligatorio)
```

El backmerge a `develop` es **obligatorio**. Un hotfix sin backmerge hace que el fix se pierda en el próximo release.

> Comandos completos para cada flujo: [A.8 — A.10](git-templates.md#a8-flujo-backend--frontend--feature)

---

### 4.2 Frontend

**Ramas permanentes:** `main`, `develop`
**Ramas temporales:** `feature/*`, `fix/*`, `hotfix/*`, `refactor/*`, `style/*`, `test/*`, `chore/*`, `release/*`

El flujo es idéntico al de [Backend 4.1](#41-backend--microservicio). El tipo `style` es específico de frontend: aplica a cambios visuales puros (CSS/SCSS) sin lógica, que no afectan comportamiento.

> **`style` vs `refactor`:** `style` = cambios visuales puros. `refactor` = reorganización de componentes, extracción de lógica, mejora de estructura.

| Situación | Tipo de rama/commit |
|-----------|---------------------|
| Cambio de assets | `chore` si es reemplazo, `feat` si es nuevo |
| Actualización de traducciones (i18n) | `feat` si agrega claves nuevas, `fix` si corrige texto |
| Cambio de design tokens / variables CSS | `style` si solo afecta apariencia, `feat` si cambia el sistema de diseño |
| Feature flags | `feat` para la flag; la feature viaja en su propia rama |

---

### 4.3 Base de Datos

**Ramas permanentes:** `main`, `develop`
**Ramas temporales:** `migration/*`, `fix/*`, `hotfix/*`, `docs/*`

#### Particularidades

Las migraciones son **potencialmente irreversibles en producción**. El flujo agrega una revisión de DBA/Infra antes del merge a `main`.

#### Flujo de migración

```
migration/* → develop → (revisión DBA/Infra) → main
```

**Pasos:**

1. Crear rama `migration/*` desde `develop`.
2. Crear los scripts siguiendo las [convenciones de nomenclatura de la guía de base de datos](../database/sqlserver-naming-conventions.md#8-scripts-ddl-y-dml) — estructura de carpetas, numeración y scripts de rollback.
3. Probar el script en ambiente de desarrollo antes de abrir el PR.
4. El PR debe incluir explícitamente: script de migración, script de rollback y resultado de la ejecución en desarrollo.
5. El equipo DBA/Infra revisa en QA antes del merge a `main`.
6. Nunca ejecutar scripts de migración en producción manualmente — siempre via pipeline.

> Comandos completos: [A.11](git-templates.md#a11-flujo-base-de-datos--migración)

---

### 4.4 Documentación

**Ramas permanentes:** `main`
**Ramas temporales:** `docs/*`

Repositorios de documentación pura (wikis, ADRs, guías técnicas) no tienen `develop` — `main` es la rama de trabajo.

```
docs/* → main
```

PR hacia `main`. Revisión por al menos 1 aprobador técnico.

> **Repositorios mixtos** (código + docs): las ramas de documentación siguen el flujo del tipo principal del repositorio, usando `develop` como base.

> Comandos completos: [A.13](git-templates.md#a13-flujo-documentación)

---

### 4.5 Librería / Paquete

**Ramas permanentes:** `main`, `develop`
**Ramas temporales:** `feature/*`, `fix/*`, `hotfix/*`, `refactor/*`, `test/*`, `release/*`

#### Versionado semántico

Las librerías usan [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

| Cambio | Versión |
|--------|---------|
| Breaking change | `MAJOR` (x.0.0) |
| Nueva funcionalidad compatible | `MINOR` (1.x.0) |
| Fix compatible | `PATCH` (1.0.x) |

#### Flujo de release

```
develop → release/x.y.z → main (tag) → develop (backmerge obligatorio)
```

Cada merge a `main` debe llevar un tag de versión anotado. El pipeline de CI/CD publica automáticamente al detectar el tag. **No publicar desde local.**

> Comandos completos: [A.12](git-templates.md#a12-flujo-librería--release-con-tag)

---

## 5. Pull Requests

### 5.1 Reglas generales

| Regla | Valor |
|-------|-------|
| Aprobadores requeridos | Mínimo 1 (mínimo 2 en librerías y repos de BD) |
| CI obligatorio | Sí — el PR no puede mergearse con CI en rojo |
| Self-merge | Prohibido |
| Squash merge | Recomendado cuando la rama tiene commits intermedios de trabajo |
| Merge commit | Usar cuando los commits están limpios y narran la historia |

### 5.2 Título del PR

Mismo formato que el commit principal (o el commit de squash):

```
feat(auth): agregar autenticación con Google OAuth [CDF-02]
fix(users): corregir validación de email duplicado [CDF-142]
```

> Template del cuerpo del PR: [A.6](git-templates.md#a6-pull_request_templatemd)

---

## 6. Anti-patterns

| Anti-pattern | Síntoma visible | Consecuencia | Solución |
|-------------|----------------|--------------|----------|
| Commits sin tipo (`fix login`, `changes`) | Historial ilegible | Imposible generar changelog automático | ver [Conventional Commits](#2-conventional-commits) |
| Commits WIP mergeados (`wip: guardando`) | Commits sin valor en `develop` | Historial contaminado | Squash antes o durante el merge del PR |
| Rama sin ticket (`feature/agregar-usuario`) | Sin trazabilidad al backlog | Imposible relacionar código con historia | ver [Nomenclatura de Ramas 3.1](#31-formato) |
| Push directo a `develop` o `main` | Branch protection sin efecto | Sin code review, sin CI | Abrir PR, nunca push directo |
| Rama de vida larga (semanas sin merge) | PR con cientos de cambios | Conflictos masivos, review imposible | Ramas cortas, máx. 1 semana — mergear frecuente |
| Force push a `develop` | Historia reescrita | Rompe el trabajo de otros developers | Nunca force push a ramas compartidas |
| PR sin rebase previo | Rama desactualizada con `develop` | CI falla por estado desactualizado | `git rebase origin/develop` antes del PR |
| Hotfix sin backmerge a `develop` | `develop` desactualizado | El fix se pierde en el próximo release | ver [Flujo de hotfix 4.1](#41-backend--microservicio) |
| Tag manual sin pipeline | Versión publicada sin CI | Paquete sin tests, posible inconsistencia | Los tags en `main` disparan el pipeline de release |
| Tipo de rama incorrecto (`feature` para un fix) | Confusión en el historial | SemVer incorrecto en changelogs | ver [Tipos de Rama 3.2](#32-tipos-de-rama) |
| Migración sin script de rollback | No se puede revertir el cambio | Deploy fallido sin salida limpia | ver [Base de Datos 4.3](#43-base-de-datos) |

---

## 7. Referencias

- [Conventional Commits v1.0](https://www.conventionalcommits.org/es/v1.0.0/)
- [Semantic Versioning 2.0.0](https://semver.org/lang/es/)
- [A successful Git branching model — Vincent Driessen](https://nvie.com/posts/a-successful-git-branching-model/)
- [git rebase — documentación oficial](https://git-scm.com/docs/git-rebase)
- [git tag — documentación oficial](https://git-scm.com/docs/git-tag)
- [Convenciones de nomenclatura de base de datos — 8 Scripts DDL y DML](../database/sqlserver-naming-conventions.md#8-scripts-ddl-y-dml)
