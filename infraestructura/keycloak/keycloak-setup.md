# Keycloak — Lineamiento DevOps

> **Audiencia:** Infra / DevOps. Esta guía no cubre nomenclatura de realms, clients ni roles — esas convenciones corresponden al equipo de desarrollo ([keycloak-conventions.md](./keycloak-conventions.md)).

**Versión:** 1.0 · **Fecha:** 2026-04-15 · **Referencia:** Keycloak 26.x (Quarkus)

> **Cómo usar esta guía:** configura las variables de entorno y la base de datos antes del primer arranque. Define la separación de ambientes desde el inicio — es la decisión que más impacto tiene en la operación diaria. Para el pipeline CI/CD, sigue el orden de pasos de la sección correspondiente ya que tienen dependencias explícitas entre sí.

---

## Contenido

- [Contexto](#contexto)
1. [Uso local vs producción](#uso-local-vs-producción)
2. [Variables de entorno obligatorias](#variables-de-entorno-obligatorias)
3. [PostgreSQL](#postgresql)
   - [3.1 Requisitos](#requisitos)
   - [3.2 Pool de conexiones](#pool-de-conexiones)
4. [Separación de ambientes](#separación-de-ambientes)
5. [Realm Export / Import](#realm-export--import)
   - [5.1 Estrategia](#estrategia)
   - [5.2 Métodos por situación](#métodos-por-situación)
   - [5.3 Limitación del export JSON](#limitación-del-export-json)
   - [5.4 Requerimientos del pipeline CI/CD](#requerimientos-del-pipeline-cicd)
   - [5.5 Service account CI/CD](#service-account-cicd)
   - [5.6 Rollback](#rollback)
6. [Secretos y configuración sensible](#secretos-y-configuración-sensible)
7. [Backup](#backup)
8. [Anti-patterns](#anti-patterns)

---

## Contexto

Keycloak 26.x usa Quarkus como runtime. La configuración se maneja exclusivamente por variables de entorno prefijadas con `KC_` — no hay archivos `standalone.xml`.

Esta guía cubre:
- Decisiones de configuración para desarrollo local, QA y producción
- Requisitos de base de datos
- Separación de ambientes
- Estrategia de export/import de realms y pipelines CI/CD
- Gestión de secrets y rotación
- Backup

**NO cubre:**
- Kubernetes, Helm, Operators, orchestration
- SSL/TLS, networking, proxy inverso
- Alta disponibilidad, clustering, load balancing
- Observabilidad, monitoring, Prometheus, Loki

---

## Uso local vs producción

| Contexto | Modo de inicio | Restricción |
|---|---|---|
| Desarrollo local | `start-dev` | Prohibido en QA, staging y producción — deshabilita TLS y usa defaults inseguros |
| QA / staging / producción | `kc.sh start` | Obligatorio — `start-dev` en estos ambientes es un anti-pattern de seguridad |

**Variable adicional para desarrollo local:**

| Variable | Valor | Razón |
|---|---|---|
| `KC_HOSTNAME_STRICT` | `false` | En `start-dev`, sin esta variable Keycloak puede devolver URLs incorrectas en el discovery OIDC (`.well-known/openid-configuration`) cuando el acceso proviene de otra máquina en la red — por ejemplo, un contenedor accedido desde el host o un servicio que resuelve el host interno. Con `false`, Keycloak usa el host del request para construir las URLs del discovery |

> Esta variable aplica solo en `start-dev`. En `kc.sh start` con `KC_HOSTNAME` configurado correctamente no es necesaria.

---

## Variables de entorno obligatorias

Para servidor dedicado, VM o contenedor en QA/producción.

| Variable | Propósito | Restricción |
|---|---|---|
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | Contraseña del administrador inicial | Solo en el primer arranque de una instancia nueva — desde vault, nunca hardcodeada. Ignorada si ya existe un administrador en la BD |
| `KC_DB` | Motor de base de datos | `postgres` — prohibido H2 fuera de local |
| `KC_DB_URL` | JDBC URL de conexión | — |
| `KC_DB_USERNAME` | Usuario de la BD | Usuario dedicado — prohibido compartir con otras aplicaciones |
| `KC_DB_PASSWORD` | Contraseña de la BD | Desde vault o secrets manager — nunca hardcodeada |
| `KC_HOSTNAME` | FQDN público de la instancia | Sin scheme (`https://`) ni barra final — los endpoints OIDC se derivan de este valor |
| `KC_PROXY_HEADERS` | Header de proxy inverso | `xforwarded` — obligatorio si hay proxy o load balancer delante |
| `KC_HTTP_ENABLED` | Habilitar HTTP | `true` cuando TLS termina en el proxy |
| `KC_HEALTH_ENABLED` | Endpoint de health | `true` — expone `GET /health/ready` y `GET /health/live` |
| `KC_METRICS_ENABLED` | Endpoint de métricas | `true` — expone `GET /metrics` en formato Prometheus |
| `KC_LOG_CONSOLE_OUTPUT` | Formato de logs | `json` — necesario para ingesta en ELK/EFK |

**Convención de hostname:** FQDN sin scheme ni barra final.

```
Correcto:   keycloak.empresa.com  ·  keycloak-dev.empresa.internal
Incorrecto: https://keycloak.empresa.com  ·  keycloak.empresa.com/
```

---

## PostgreSQL

### Requisitos

| Elemento | Valor | Razón |
|---|---|---|
| Versión | 14+ (preferir 16) | Soporte oficial Keycloak |
| Encoding | UTF8 | Internacionalización |
| Collation | `en_US.UTF-8` | Consistencia de queries |
| `max_connections` | ≥ 100 | Keycloak abre múltiples conexiones por nodo |
| BD exclusiva | Prohibido compartir con otras aplicaciones | Imposible aislar configuración por ambiente si se comparte |

### Pool de conexiones

| Variable | Regla |
|---|---|
| `KC_DB_POOL_MIN_SIZE` | Valor de referencia: `5`. Ajustar según carga observada en métricas de conexiones activas |
| `KC_DB_POOL_MAX_SIZE` | Regla: `max_connections_PG > KC_DB_POOL_MAX_SIZE × N_nodos + 10` |
| `KC_DB_POOL_INITIAL_SIZE` | Igual a `KC_DB_POOL_MIN_SIZE` en todos los ambientes — ajustar solo si métricas muestran latencia de conexión en el arranque |

---

## Separación de ambientes

Cada ambiente (dev, QA, staging, prod) tiene su **propia instancia de Keycloak + BD PostgreSQL independientes**. El nombre del realm es el mismo en todos los ambientes — lo que cambia es el host de la instancia.

| Ambiente | Host | Realm | BD |
|---|---|---|---|
| Desarrollo | `keycloak-dev.{empresa}.internal` | `{realm}` | `keycloak_dev` |
| QA | `keycloak-qa.{empresa}.internal` | `{realm}` | `keycloak_qa` |
| Staging | `keycloak-staging.{empresa}.internal` | `{realm}` | `keycloak_staging` |
| Producción | `keycloak.{empresa}.com` | `{realm}` | `keycloak_prod` |

**Razón:** evitar contaminación de políticas de seguridad (MFA, brute-force, password policy) entre ambientes y eliminar el riesgo de cambios accidentales en producción.

La URL del host se inyecta como variable de entorno en cada servicio. Sin lógica condicional en código para determinar el ambiente.

---

## Realm Export / Import

### Estrategia

La configuración del realm (sin usuarios) se versiona como JSON en el repositorio. Flujo obligatorio antes de promover a un ambiente superior:

```
dev (Admin UI) → export JSON → commit → PR → merge → CI/CD importa en QA → prod
```

### Métodos por situación

| Situación | Método |
|---|---|
| Ambiente nuevo con BD vacía | Auto-import vía volumen montado al arrancar el contenedor |
| CI/CD — actualizar realm existente | Admin REST API: `PUT /admin/realms/{realm}` |
| Recuperación manual o setup one-shot | Import desde dentro del contenedor con `kc.sh import` |

### Limitación del export JSON

Los **audience mappers** y **protocol mappers** no se incluyen en el export JSON del realm — es una limitación de Keycloak. Deben configurarse explícitamente en cada ambiente después del import, vía script idempotente de Admin REST API.

**Requisito post-import:** los audience mappers y protocol mappers deben configurarse vía Admin REST API como paso obligatorio post-import en cada ambiente. La implementación del paso es responsabilidad del equipo — debe ser idempotente y el pipeline debe fallar si produce error. Ver [keycloak-examples.md — Audience mapper](./keycloak-examples.md#audience-mapper).

### Requerimientos del pipeline CI/CD

El pipeline de importación debe, en este orden:

1. Autenticarse con el service account `ci-admin` (ver sección siguiente) mediante `client_credentials`
2. Importar el JSON del realm vía Admin REST API
3. Configurar los audience mappers y protocol mappers vía Admin REST API
4. Validar que el realm está habilitado antes de marcar el pipeline como exitoso
5. Fallar explícitamente si cualquiera de los pasos anteriores produce error

### Service account CI/CD

Crear una vez por instancia de Keycloak, en el **master realm**. El pipeline usa este service account — nunca credenciales de usuario con `grant_type=password`.

| Parámetro | Valor |
|---|---|
| Client ID | `ci-admin` |
| Client authentication | `ON` (confidencial) |
| Service accounts | `ON` |
| Standard flow | `OFF` |
| Direct access grants | `OFF` |
| Roles del service account | `manage-realm` y `manage-clients` del client `realm-management` |

El secret se almacena en vault o secrets manager del CI/CD. Nunca en variables de entorno visibles en logs ni en el repositorio.

### Rollback

Si un import introduce una regresión: identificar el commit anterior funcional en Git, extraer el JSON de ese commit y re-importar vía Admin REST API con el mismo flujo del pipeline. Configurar los mappers vía Admin REST API después del rollback. Verificar que el realm está habilitado antes de dar por completado.

---

## Secretos y configuración sensible

| Recurso | Prohibición | Regla |
|---|---|---|
| `client_secret` | Hardcodear en imagen o código | Inyectar desde vault o pipeline |
| `KC_DB_PASSWORD` | Plain text en archivos versionados | Variable de entorno desde secrets manager |
| Admin password | Versionar en el repositorio | Solo en primer arranque vía `KC_BOOTSTRAP_ADMIN_PASSWORD` — luego gestionar desde vault |
| Secret del pipeline (`ci-admin`) | Compartir entre ambientes | Secret independiente por instancia de Keycloak |

### Rotación

| Secret | Frecuencia máxima |
|---|---|
| `client_secret` de aplicaciones | 90 días |
| Admin password | 30 días |
| Secret del service account CI/CD | 30 días |

Keycloak detecta cambios en variables de entorno al reiniciar el contenedor.

---

## Backup

| Qué | Estrategia | Cuándo |
|---|---|---|
| Configuración del realm | Export JSON versionado en Git | Antes de cada promoción a QA o prod |
| Base de datos | Dump completo de PostgreSQL | Diario en QA/staging/prod |

**Retención:** 30 días en QA/staging, 90 días en producción.

**Verificación:** restaurar desde backup en staging al menos una vez por trimestre. Un backup no verificado no es un backup.

> En ambientes donde los usuarios vienen de un IdP externo (LDAP, Active Directory), la BD de Keycloak es menos crítica. En ambientes con usuarios locales en producción, la BD es la fuente de verdad y requiere backup propio con plan de recuperación definido.

**Contenido mínimo del plan de recuperación (ambientes con usuarios locales en producción):**
- RTO máximo: tiempo máximo tolerable hasta que Keycloak esté operativo y los usuarios puedan autenticarse
- RPO máximo: antigüedad máxima aceptable del backup a restaurar
- Responsable de ejecución del restore: rol o persona asignada
- Canal de comunicación en incidente: dónde se notifica y escala
- Procedimiento de validación post-restore: qué verificar antes de declarar recuperado

---

## Anti-patterns

| Anti-pattern | Síntoma visible | Consecuencia | Solución |
|---|---|---|---|
| H2 en producción | Los datos de sesión y usuarios desaparecen al reiniciar el contenedor | Pérdida total de datos — Keycloak inoperativo tras cada restart | Usar PostgreSQL 14+ con BD exclusiva — ver sección PostgreSQL |
| `start-dev` fuera del dev local | Keycloak arranca sin TLS y acepta cualquier conexión | TLS deshabilitado, defaults inseguros de autenticación activos en producción | `kc.sh start` con todas las variables obligatorias configuradas |
| Compartir BD con otras aplicaciones | Un cambio de `max_connections` en la BD impacta a Keycloak sin notificación | Imposible aislar pool ni configuración por ambiente | BD exclusiva por instancia de Keycloak — ver sección PostgreSQL |
| Hardcodear secrets en imagen o código | El secret queda visible en el historial de Git o en el registro de contenedores | Exposición permanente incluso después de rotar el secret | Inyectar desde vault o secrets manager — nunca en el repositorio |
| `KC_HOSTNAME` con scheme (`https://`) o barra final | Los endpoints OIDC en el `.well-known/openid-configuration` tienen URLs malformadas | JWKS discovery falla — los servicios no pueden validar tokens | FQDN sin scheme ni barra final — ver convención de hostname |
| Versionar secrets en el repositorio | El secret aparece en `git log` aunque se elimine del archivo actual | Exposición permanente — el historial de Git es inmutable | Eliminar del historial con `git filter-repo` y rotar el secret inmediatamente |
| `grant_type=password` en pipelines CI/CD | Las credenciales del usuario administrador quedan en variables de entorno del CI/CD | Credenciales de usuario expuestas en logs si falla el enmascaramiento | Service account `ci-admin` con `client_credentials` — ver sección Service account CI/CD |
| Ambiente compartido entre equipos | Un cambio de política de MFA o brute-force en QA bloquea a todos los equipos | Contaminación cruzada de configuración de seguridad entre equipos | Instancia independiente por ambiente — ver sección Separación de ambientes |

> Anti-patterns de nomenclatura (realm `master` para workloads, ambiente en el client ID, etc.) → [keycloak-conventions.md — Anti-patterns](./keycloak-conventions.md#7-anti-patterns).
