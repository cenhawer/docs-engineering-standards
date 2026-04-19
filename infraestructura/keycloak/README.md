# Keycloak

Lineamientos para convenciones, configuración y operación de Keycloak como proveedor de identidad OIDC.

---

## Guías disponibles

| Guía | Descripción | Audiencia | Versión |
|------|-------------|-----------|---------|
| [keycloak-conventions.md](./keycloak-conventions.md) | Naming de realms, clients, roles, scopes y mappers; estrategias de realm; tipos de client; reglas de seguridad; anti-patterns | Dev | 1.0 |
| [keycloak-examples.md](./keycloak-examples.md) | Templates UI y Admin REST API: cliente API, SPA, service account, roles, composite role, audience mapper, scope mapper, protocol mapper, usuario; checklist de validación; realm JSON mínimo | Dev | 1.0 |
| [keycloak-setup.md](./keycloak-setup.md) | Docker Compose local, variables de servidor `KC_*`, PostgreSQL, separación de ambientes, realm export/import, gestión de secrets, anti-patterns de infraestructura | Infra / DevOps | 1.0 |

---

## Separación de responsabilidades

### Dev

- Aplica convenciones de naming — [keycloak-conventions.md](./keycloak-conventions.md)
- Crea clients, roles, scopes y mappers siguiendo los templates — [keycloak-examples.md](./keycloak-examples.md)
- Define el realm JSON y abre PRs para cambios de configuración antes de promover a QA

### Infra / DevOps

- Instala y opera Keycloak — [keycloak-setup.md](./keycloak-setup.md)
- Configura PostgreSQL, variables de entorno y separación de instancias por ambiente
- Gestiona secrets y rotación de `client_secret`
- Promueve realm JSON entre ambientes (dev → QA → staging → prod) vía CI/CD

---

## Fuera de scope

- Kubernetes, Helm, Operators, orchestration → infraestructura general
- SSL/TLS, networking, proxy inverso → infraestructura general
- Alta disponibilidad, clustering, load balancing → infraestructura general
- Integración con IdP externo (LDAP, Active Directory, Google) → decisión de proyecto, contactar arquitectura
- Observabilidad y monitoring → infraestructura general
- Troubleshooting de errores de aplicación (`401`, `403`) → responsabilidad del equipo de desarrollo
