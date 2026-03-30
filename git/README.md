# Git

Lineamientos para el uso de Git: convenciones de commits, nomenclatura de ramas, flujos GitFlow por tipo de repositorio y configuración de infraestructura de versionado.

---

## Guías disponibles

| Guía | Descripción | Estado |
|------|-------------|--------|
| [Convenciones de Git](./git-conventions.md) | Conventional commits, nombres de ramas, GitFlow por tipo de repositorio | `Stable` |
| [Templates y Ejemplos](./git-templates.md) | Plantillas de commits, PRs y referencia rápida de ramas | `Stable` |
| [Configuración de Repositorios](./git-setup.md) | Branch protection, hooks, permisos, CI/CD | `Stable` |

---

## Estado de los documentos

| Documento | Versión | Audiencia | Contenido |
|-----------|---------|-----------|-----------|
| `git-conventions.md` | 1.0 | Dev | Commits, ramas, GitFlow por tipo de repo |
| `git-templates.md` | 1.0 | Dev | Templates de commits, PRs, referencia rápida |
| `git-setup.md` | 1.0 | Infra / DevOps | Configuración de repos, branch protection, hooks |

---

## Separación de responsabilidades

- **Dev**
  - Aplica convenciones de commits (Conventional Commits)
  - Sigue la nomenclatura de ramas del proyecto
  - Usa el flujo GitFlow según el tipo de repositorio
  - Crea PRs con el template estándar

- **Infra / DevOps**
  - Configura repositorios (visibilidad, permisos, acceso)
  - Establece reglas de protección de ramas (`main`, `develop`)
  - Configura hooks y pipelines CI/CD
  - Define la estrategia de releases y tags

---

## Principios

1. **Consistency over preference**
   Los flujos y convenciones son estándar, no decisiones por equipo o proyecto.

2. **Commits como documentación**
   El historial de commits debe narrar la evolución del proyecto de forma legible.

3. **Shift-left en calidad**
   El formato y los controles se aplican desde el primer commit, no en revisión.

4. **Separación de concerns**
   Cada tipo de repositorio tiene un flujo adaptado a su ciclo de vida.

5. **Reproducibilidad**
   Cualquier integrante debe poder entender el estado del repo y su historia sin contexto adicional.

---

## Changelog

| Fecha | Lineamiento |
|-------|------------|
| 2026-03-30 | [Convenciones de Git](./git-conventions.md) |
| 2026-03-30 | [Templates y Ejemplos](./git-templates.md) |
| 2026-03-30 | [Configuración de Repositorios](./git-setup.md) |
