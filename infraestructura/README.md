# Infraestructura

Lineamientos para la configuración y operación de servicios de infraestructura de plataforma: gestión de identidad y acceso (IAM), secretos, API gateways y service mesh.

---

## Guías disponibles

| Guía | Descripción | Estado |
|------|-------------|--------|
| [Keycloak](./keycloak/README.md) | Configuración de Identity & Access Management con Keycloak — realms, clients, roles, scopes, templates y setup DevOps | `Stable` |

---

## Estado de los documentos

| Documento | Versión | Audiencia | Contenido |
|-----------|---------|-----------|-----------|
| `keycloak/keycloak-conventions.md` | 1.0 | Dev | Realms, clients, roles, scopes, mappers — convenciones, estrategias de realm, tipos de client, reglas de seguridad, anti-patterns |
| `keycloak/keycloak-examples.md` | 1.0 | Dev | Templates de configuración: client API, SPA, service account, roles, composite role, audience mapper, scope mapper, protocol mapper, usuario; checklist de validación |
| `keycloak/keycloak-setup.md` | 1.0 | Infra / DevOps | Variables de entorno `KC_*`, PostgreSQL, separación de ambientes, realm export/import, gestión de secrets, backups, anti-patterns de infraestructura |

---

Ver [keycloak/README.md — Separación de responsabilidades](./keycloak/README.md#separación-de-responsabilidades).

---

## Changelog

| Fecha | Cambio |
|-------|--------|
| 2026-04-15 | Versión inicial 1.0 — keycloak-conventions.md, keycloak-examples.md, keycloak-setup.md |
