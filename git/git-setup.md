# Configuración de Repositorios — Git

> **Audiencia:** Infra / DevOps. Este documento cubre la configuración de repositorios Git: nomenclatura, branch protection, permisos, hooks y estrategia de CI/CD.
>
> Para convenciones de commits y nomenclatura de ramas, ver [git-conventions.md](git-conventions.md). Los scripts y archivos listos para usar están en [git-templates.md](git-templates.md).

**Versión:** 1.0

---

## Contenido

1. [Creación de Repositorios](#1-creación-de-repositorios)
   - [1.1 Nomenclatura de Repositorios](#11-nomenclatura-de-repositorios)
   - [1.2 Estructura Inicial](#12-estructura-inicial)
2. [Branch Protection](#2-branch-protection)
   - [2.1 Reglas para `main`](#21-reglas-para-main)
   - [2.2 Reglas para `develop`](#22-reglas-para-develop)
3. [Permisos y Acceso](#3-permisos-y-acceso)
4. [Hooks](#4-hooks)
   - [4.1 commit-msg](#41-commit-msg)
   - [4.2 pre-push](#42-pre-push)
5. [Estrategia de Tags y Releases](#5-estrategia-de-tags-y-releases)
6. [Integración CI/CD](#6-integración-cicd)
   - [6.1 Pipeline de PR](#61-pipeline-de-pr)
   - [6.2 Pipeline de Release](#62-pipeline-de-release)
7. [Configuración por Tipo de Repositorio](#7-configuración-por-tipo-de-repositorio)

---

## 1. Creación de Repositorios

### 1.1 Nomenclatura de Repositorios

**Regla:** `{tipo}-{proyecto}[-{nombre}]`

| Parte | Regla |
|-------|-------|
| `tipo` | Tipo de repositorio — ver tabla |
| `proyecto` | Código del proyecto — minúsculas, igual al usado en los tickets del backlog |
| `nombre` | Nombre descriptivo opcional — `kebab-case` |

| Tipo | Prefijo | Ejemplo |
|------|---------|---------|
| Backend / API | `api-` | `api-cdf`, `api-cdf-users` |
| Frontend web | `web-` | `web-cdf`, `web-cdf-admin` |
| Frontend móvil | `mobile-` | `mobile-cdf` |
| Base de datos | `db-` | `db-cdf` |
| Librería | `lib-` | `lib-cdf-validation` |
| Documentación | `docs-` | `docs-cdf` |
| Infraestructura | `infra-` | `infra-cdf` |
| Configuración | `config-` | `config-cdf` |

```
# Correcto
api-cdf
api-cdf-users
web-cdf-admin
db-cdf
lib-cdf-validation

# Incorrecto
CDF_API                -- mayúsculas y guiones bajos
cdf-backend            -- tipo como sufijo y nombre genérico
backend-cdf            -- tipo genérico sin semántica
```

### 1.2 Estructura Inicial

Cada repositorio nuevo debe incluir desde el primer commit los archivos base. Los scripts listos para copiar están en:

- `.gitattributes` → [A.2](git-templates.md#a2-gitattributes)
- `.githooks/commit-msg` → [A.3](git-templates.md#a3-githookscommit-msg)
- `.githooks/pre-push` → [A.4](git-templates.md#a4-githookspre-push)
- `.github/PULL_REQUEST_TEMPLATE.md` → [A.6](git-templates.md#a6-pull_request_templatemd)
- `.github/CODEOWNERS` (recomendado) → [A.7](git-templates.md#a7-codeowners)

> **`text=auto eol=lf` en `.gitattributes`:** normaliza todos los archivos de texto a LF en el repositorio independientemente del OS del desarrollador. Evita diffs espurios por diferencia de line endings entre Windows y macOS/Linux.

---

## 2. Branch Protection

### 2.1 Reglas para `main`

Configurar en GitHub / GitLab / Azure DevOps:

| Regla | Valor | Por qué |
|-------|-------|---------|
| Require pull request before merging | `true` | Sin push directo a producción |
| Required approvals | `1` mínimo — `2` en libs y DB | Code review obligatorio |
| Dismiss stale reviews | `true` | Nuevos commits invalidan aprobaciones anteriores |
| Require review from Code Owners | `true` si hay CODEOWNERS | Revisión por responsables del área |
| Require status checks to pass | `true` | CI debe pasar antes del merge |
| Require branches to be up to date | `true` | La rama debe estar al día con `main` antes del merge |
| Restrict who can push | Solo Infra/DevOps | Merges de `release/*` y `hotfix/*` solamente |
| Allow force pushes | `false` | Nunca — protege la historia del repositorio |
| Allow deletions | `false` | `main` es permanente |

### 2.2 Reglas para `develop`

| Regla | Valor | Por qué |
|-------|-------|---------|
| Require pull request before merging | `true` | Sin push directo |
| Required approvals | `1` | Code review obligatorio |
| Dismiss stale reviews | `true` | |
| Require status checks to pass | `true` | CI debe pasar |
| Require branches to be up to date | `true` | |
| Allow force pushes | `false` | Rama compartida — nunca force push |
| Allow deletions | `false` | `develop` es permanente |

---

## 3. Permisos y Acceso

**Principio:** mínimo privilegio. Cada rol accede solo a lo que necesita para su función.

| Rol | Permiso |
|-----|---------|
| Developer | `Write` — puede hacer push a ramas de trabajo, abrir PRs |
| Tech Lead / Senior | `Write` + aprobador configurado en branch protection |
| Infra / DevOps | `Maintain` — configura repositorio y branch protection |
| Pipeline CI/CD | Token con `Write` limitado al repositorio, sin acceso de admin |
| Bots (Dependabot, Renovate) | `Write` limitado para PRs automáticos |

> **Service accounts para CI/CD:** usar tokens de acceso dedicados (PAT o GitHub App), nunca credenciales personales. El token tiene alcance mínimo necesario y se rota periódicamente.

---

## 4. Hooks

Los hooks se distribuyen como scripts en `.githooks/` dentro del repositorio y se instalan localmente con un paso de setup. No se usa gestión externa de hooks por defecto.

Los scripts completos están en:
- `commit-msg` → [A.3](git-templates.md#a3-githookscommit-msg)
- `pre-push` → [A.4](git-templates.md#a4-githookspre-push)
- Instalación → [A.5](git-templates.md#a5-instalación-de-hooks)

### 4.1 commit-msg

**Propósito:** valida que el mensaje sigue Conventional Commits antes de completar el commit.

Actúa como red de seguridad local. La validación definitiva ocurre en el servidor (branch protection + CI).

### 4.2 pre-push

**Propósito:** previene push directo a `main` y `develop`.

> Los hooks pueden omitirse con `--no-verify`. Solo en casos de emergencia documentados — queda registrado en el log del PR.

---

## 5. Estrategia de Tags y Releases

### 5.1 Formato de tags

```
v{MAJOR}.{MINOR}.{PATCH}

v1.0.0
v1.4.0
v2.0.0
v2.0.0-rc.1        # release candidate
v2.0.0-beta.1      # beta
```

### 5.2 Reglas

| Regla | Valor |
|-------|-------|
| Tags solo en `main` | Siempre — nunca en `develop` ni ramas de trabajo |
| Tags anotados | Siempre usar `git tag -a` — no tags ligeros |
| Mensaje del tag | Incluir resumen de cambios del release |
| Quién crea el tag | El pipeline CI/CD automáticamente al mergear `release/*` o `hotfix/*` a `main` |

### 5.3 Automatización

Al detectar el merge de una rama `release/*` o `hotfix/*` a `main`, el pipeline debe:

1. Leer la versión desde el archivo de versión del proyecto.
2. Crear el tag anotado.
3. Generar el changelog desde los commits convencionales.
4. Publicar el release en GitHub/GitLab Releases.

---

## 6. Integración CI/CD

### 6.1 Pipeline de PR

Ejecutar en cada PR contra `main` o `develop`. Configurar como status check requerido — el PR no puede mergearse con alguno en rojo.

```yaml
on:
  pull_request:
    branches: [main, develop]

jobs:
  validate:
    steps:
      - name: Validar formato de commits
        # Verificar que todos los commits del PR siguen Conventional Commits

      - name: Build

      - name: Tests

      - name: Cobertura
        # Umbral mínimo definido por proyecto (referencia: 80%)

      - name: Análisis estático
        # Linting, calidad de código
```

### 6.2 Pipeline de Release

Ejecutar al detectar un tag `v*` en `main`.

```yaml
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    steps:
      - name: Build de release
      - name: Tests completos (incluyendo integración)
      - name: Publicar paquete       # si es librería
      - name: Desplegar a producción # si es servicio
      - name: Generar Release notes con changelog automático
```

---

## 7. Configuración por Tipo de Repositorio

| Configuración | Backend / Frontend | Base de Datos | Librería | Documentación |
|---------------|-------------------|---------------|----------|---------------|
| Ramas permanentes | `main`, `develop` | `main`, `develop` | `main`, `develop` | `main` |
| Branch protection `main` | Sí | Sí + revisión DBA | Sí | Sí |
| Branch protection `develop` | Sí | Sí | Sí | N/A |
| Aprobaciones en `main` | 1 | 2 | 2 | 1 |
| CODEOWNERS | Recomendado | Obligatorio | Obligatorio | Recomendado |
| Tags / SemVer | Opcional | No | Obligatorio | No |
| Pipeline de PR | Build + Tests | Validar scripts SQL | Build + Tests | Validar links |
| Pipeline de release | Deploy a prod | Ejecutar migración | Publicar paquete | N/A |

### Repositorios de Base de Datos — consideraciones adicionales

- Los PRs de migración **siempre** requieren aprobación de alguien del equipo DBA/Infra.
- Configurar CODEOWNERS para que `migrations/` requiera aprobación de `@dba-team` — ver [A.7](git-templates.md#a7-codeowners).
- El pipeline de PR debe ejecutar el script de migración en un ambiente de prueba y verificar que el script de rollback revierte correctamente.
- Nunca ejecutar scripts de migración en producción manualmente — siempre via pipeline con aprobación registrada.

### Repositorios de Librería — consideraciones adicionales

- El pipeline de release publica automáticamente — no publicar desde local.
- Ejecutar el equivalente a `dry-run` de publicación en el pipeline de PR para detectar errores de packaging antes del merge.
- El `CHANGELOG.md` se genera automáticamente desde los commits convencionales en cada release.
