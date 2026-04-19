# Keycloak — Convenciones

> **Audiencia:** equipo de desarrollo.
>
> **No cubre:** instalación, variables de entorno, separación de ambientes, export/import de realms → [keycloak-setup.md](./keycloak-setup.md).
>
> **Referencia de configuración** (parámetros y valores por componente) → [keycloak-examples.md](./keycloak-examples.md).

**Versión:** 1.0 · **Fecha:** 2026-04-15 · **Referencia:** Keycloak 26.x (Quarkus)

> **Cómo usar esta guía:** lee 1 (Reglas Generales) antes de crear cualquier objeto en Keycloak — define el idioma, el formato y las prohibiciones que aplican a toda la plataforma. Para naming específico de un tipo de objeto ve directamente a su sección. Para incorporar un nuevo dominio, usa el checklist en [keycloak-examples.md — Checklist de validación de dominio](./keycloak-examples.md#checklist-de-validación-de-dominio).

---

## Contenido

- [Contexto](#contexto)
1. [Reglas Generales](#1-reglas-generales)
   - [1.1 Idioma](#11-idioma)
   - [1.2 Formato](#12-formato)
   - [1.3 Prohibiciones](#13-prohibiciones)
   - [1.4 Verificación automática](#14-verificación-automática)
2. [Realms](#2-realms)
   - [2.1 Estrategias](#21-estrategias)
   - [2.2 Nomenclatura](#22-nomenclatura)
3. [Clients](#3-clients)
   - [3.1 Nomenclatura](#31-nomenclatura)
   - [3.2 Tipos de client](#32-tipos-de-client)
   - [3.3 Reglas de seguridad](#33-reglas-de-seguridad)
   - [3.4 Patrón frontend + backend por dominio](#34-patrón-frontend--backend-por-dominio)
   - [3.5 Comunicación backend-to-backend](#35-comunicación-backend-to-backend)
   - [3.6 Token lifetime por tipo](#36-token-lifetime-por-tipo)
4. [Roles](#4-roles)
   - [4.1 Naming](#41-naming)
   - [4.2 Realm roles vs Client roles](#42-realm-roles-vs-client-roles)
   - [4.3 Patrón super admin](#43-patrón-super-admin)
   - [4.4 Composite roles](#44-composite-roles)
5. [Client Scopes](#5-client-scopes)
   - [5.1 Naming](#51-naming)
   - [5.2 Relación scope → role](#52-relación-scope--role)
   - [5.3 Default vs Optional](#53-default-vs-optional)
6. [Protocol Mappers](#6-protocol-mappers)
   - [6.1 Naming](#61-naming)
   - [6.2 Claims: access token vs ID token](#62-claims-access-token-vs-id-token)
   - [6.3 Claim names](#63-claim-names)
   - [6.4 Audience mapper](#64-audience-mapper)
7. [Anti-patterns](#7-anti-patterns)
8. [Referencias](#8-referencias)

---

## Contexto

Keycloak es el SSO de la plataforma: todos los microservicios y microfrontends autentican y autorizan a través de él. Los identificadores de Keycloak —realms, clients, roles, scopes— forman parte del contrato de seguridad de la plataforma, no son decisiones internas de cada equipo.

Cuando son inconsistentes, el resultado es acoplamiento invisible, auditorías imposibles de trazar y configuraciones que no pueden reproducirse entre ambientes.

> **Ejemplo de referencia:** esta guía usa como ejemplo consistente una plataforma `ecommerce` con dos dominios — `orders` y `payments`. Adaptar los nombres a la plataforma real.

---

## 1. Reglas Generales

### 1.1 Idioma

El idioma de los identificadores es una decisión de proyecto: se toma una sola vez y aplica a toda la plataforma.

| Situación | Decisión |
|-----------|----------|
| Proyecto nuevo | Inglés (recomendado) o español — documentar en `realms/{realm-name}/README.md` del repositorio |
| Proyecto existente | Mantener el idioma existente |
| Equipo internacional o producto global | Inglés |
| Producto y equipo 100% hispanohablante | Español es válido |

**Alcance:** aplica a realm names, client IDs, roles, scopes, claim names y atributos de usuario. No aplica a display names ni descripciones.

### 1.2 Formato

| Identificador | Formato | Ejemplo |
|---------------|---------|---------|
| Realm | `kebab-case` | `ecommerce`, `b2b-portal` |
| Client ID | `kebab-case` | `orders-api`, `payments-web` |
| Role | `{dominio}:{acción}` | `orders:approve`, `payments:refund` |
| Client Scope | `{dominio}:{acción}` | `orders:read`, `payments:write` |
| Authentication Flow | `kebab-case` | `mfa-browser-flow` |
| Claim name | `snake_case` | `tenant_id`, `cost_center` |
| Atributo de usuario | `snake_case` | `employee_id`, `department_code` |
| Group path | `kebab-case` con `/` | `/finance/accounts-payable` |
| Mapper | `{claim-name}-mapper` | `tenant-id-mapper` |

**Regla universal:** minúsculas en todos los identificadores. Espacios prohibidos.

### 1.3 Prohibiciones

| Prohibición | Razón |
|-------------|-------|
| Ambiente en el identificador (`orders-api-dev`, `ecommerce-qa`) | El ambiente se separa por instancia de Keycloak, no por nombre |
| Prefijos húngaros: `role_`, `scope_`, `client_` | El tipo de objeto es conocido por contexto — son ruido sin valor |
| Roles internos de Keycloak en lógica de negocio (`uma_authorization`, `default-roles-{realm}`) | Son artefactos del protocolo — pueden cambiar en upgrades de Keycloak |
| `offline_access` en guards o checks de negocio | Es un scope OAuth2 estándar, no un rol de dominio |
| Client ID igual al nombre del framework (`spring-security-client`, `keycloak-angular`) | Refleja implementación, no dominio |

### 1.4 Verificación automática

Las siguientes reglas son verificables vía Admin REST API y deben incluirse como gates en el pipeline de onboarding de dominio. Un pipeline que omite estos gates no cumple el lineamiento.

| Regla | Endpoint | Campo a verificar |
|---|---|---|
| Wildcard en redirect URIs | `GET /admin/realms/{realm}/clients` | `redirectUris` no contiene `*` |
| Direct access grants OFF | `GET /admin/realms/{realm}/clients` | `directAccessGrantsEnabled: false` |
| Implicit flow OFF | `GET /admin/realms/{realm}/clients` | `implicitFlowEnabled: false` |
| PKCE S256 en clients públicos | `GET /admin/realms/{realm}/clients` | `attributes["pkce.code.challenge.method"]: "S256"` en clients con `publicClient: true` |
| Audience mapper presente | `GET /admin/realms/{realm}/client-scopes/{id}/protocol-mappers/models` | existe entrada con `protocolMapper: oidc-audience-mapper` en el dedicated scope del API |
| Composite máximo 1 nivel | `GET /admin/realms/{realm}/roles/{role}/composites` | ningún role del resultado tiene composites propios |

Las reglas no cubiertas por Admin REST API (naming de identificadores, idioma, documentación de excepciones) requieren revisión humana en PR.

---

## 2. Realms

Un realm es un **namespace completo de identidad**: tiene sus propios usuarios, clients, roles y políticas. No hay sesión compartida entre realms distintos — el SSO opera dentro de un realm, no entre realms.

### 2.1 Estrategias

| | Opción A — Un realm por plataforma | Opción B — Un realm por línea de negocio | Opción C — Un realm por bounded context |
|---|---|---|---|
| **Cuándo** | Usuarios compartidos entre módulos | Audiencias completamente separadas (B2B vs B2C) | Bounded contexts como productos independientes |
| **SSO** | Sí — sesión única en toda la plataforma | Por línea de negocio | No — sin SSO cross-context |
| **Políticas diferenciadas por dominio** | No | Por línea | Sí |
| **Complejidad operacional** | Baja | Media | Alta |
| **Token exchange entre dominios** | No necesario | No necesario | Requerido (RFC 8693) |

**Default:** Opción A. Opción B si se cumplen las tres condiciones simultáneamente: (1) los usuarios de una audiencia nunca acceden a recursos de la otra, (2) las políticas de autenticación son distintas (MFA, password policy, IdP), (3) no hay necesidad de SSO entre audiencias. Opción C requiere aprobación de arquitectura antes de implementar. Proceso: abrir un ADR con el diseño propuesto (estrategia de realm, mecanismo de propagación de identidad, impacto en SSO), enviarlo al canal de arquitectura del proyecto antes de iniciar la configuración. SLA de respuesta: 5 días hábiles.

**Ejemplo Opción A:**

```
Realm: ecommerce
├── clients: orders-web, orders-api, payments-web, payments-api, billing-worker
├── client roles (orders-api): orders:read, orders:create, orders:approve
├── client roles (payments-api): payments:read, payments:refund
└── realm roles: platform:admin (composite)
```

**Ejemplo Opción B:**

```
Realm: b2b  → clients: backoffice-web, orders-api, catalog-api  · usuarios: empresas clientes
Realm: b2c  → clients: shop-web, mobile-app, support-api        · usuarios: consumidores finales
```

### 2.2 Nomenclatura

El nombre del realm refleja el scope de identidad que agrupa, sin sufijo de ambiente y sin prefijo de empresa.

```
Correcto:   ecommerce  ·  b2b  ·  b2c  ·  crm-platform
Incorrecto: ecommerce-dev  ·  EcommerceRealm  ·  master  ·  my-company-ecommerce
```

El mismo nombre de realm (`ecommerce`) existe en todos los ambientes. Lo que cambia es el host de la instancia — ver [keycloak-setup.md — Separación de Ambientes](./keycloak-setup.md#separación-de-ambientes).

---

## 3. Clients

Un client representa una aplicación (SPA, API backend, worker, móvil) que interactúa con Keycloak. No es un usuario ni un rol.

### 3.1 Nomenclatura

El client ID es el nombre del servicio en lenguaje ubicuo del dominio, sin sufijos técnicos y sin ambiente.

```
Correcto:   orders-web  ·  orders-api  ·  payments-web  ·  payments-api  ·  billing-worker
Incorrecto: OrdersWebApp  ·  orders-api-dev  ·  my-orders-client  ·  service1  ·  keycloak-angular
```

El client ID aparece en el claim `azp` del access token y en logs de auditoría. Un nombre sin semántica de dominio hace imposible correlacionar un token con un servicio de negocio.

### 3.2 Tipos de client

| Tipo | Client authentication | Standard flow | PKCE | Service accounts | Ejemplo |
|------|-----------------------|---------------|------|-----------------|---------|
| SPA / móvil | OFF (público) | ON | S256 obligatorio | OFF | `orders-web`, `mobile-app` |
| API backend | ON (confidencial) | OFF | — | OFF (salvo que llame a otras APIs) | `orders-api` |
| Worker / job M2M | ON (confidencial) | OFF | — | ON | `billing-worker` |

### 3.3 Reglas de seguridad

**Clients públicos (SPA y móvil):**

| Parámetro | Regla |
|-----------|-------|
| `client_secret` | Prohibido — el bundle es texto plano para cualquier usuario |
| PKCE | `S256` obligatorio |
| Implicit flow | Desactivado — deprecated en RFC 9700 / BCP 212 |
| Valid Redirect URIs | URIs exactas por ambiente. Prohibido `*`. Prohibido `http://` en producción y staging. Para aplicaciones móviles nativas: custom URI scheme (`myapp://callback`) es válido en todos los ambientes — no aplica la restricción de `http://` |
| Web Origins | Dominios exactos del frontend. Prohibido `*` y `+` |

**Clients confidenciales (APIs y workers):**

| Parámetro | Regla |
|-----------|-------|
| `client_secret` | Inyectado por variable de entorno o vault. Nunca hardcodeado |
| Standard flow | OFF — las APIs no redirigen usuarios al login |
| Direct access grants | OFF |

> **JWKS y rotación de claves:** Keycloak rota las signing keys automáticamente. Los servicios backend deben configurar un JWKS refresh interval en su librería OAuth2/OIDC. Sin refresh, el servicio rechazará tokens válidos firmados con la nueva clave hasta que se reinicie.

### 3.4 Patrón frontend + backend por dominio

Cada dominio tiene al menos dos clients: frontend (público) y API (confidencial). El usuario se autentica una sola vez vía SSO; cada SPA obtiene su propio token con el audience de su backend.

```
Realm: ecommerce

  orders-web  (público, PKCE)   →  solicita token con aud=orders-api
  orders-api  (confidencial)    →  valida el token, evalúa roles

  payments-web (público, PKCE)  →  solicita token con aud=payments-api
  payments-api (confidencial)   →  valida el token, evalúa roles
```

| Situación | Comportamiento |
|-----------|----------------|
| Usuario navega de `orders-web` a `payments-web` | No se reautentica — SSO reutiliza la sesión |
| `orders-web` llama a `orders-api` | Token con `aud=orders-api` — aceptado |
| `orders-web` intenta usar su token en `payments-api` | Rechazado — `aud=orders-api`, no `payments-api` |

### 3.5 Comunicación backend-to-backend

Cuando `orders-api` llama a `payments-api`: obtiene un token propio con `grant_type=client_credentials` (service accounts habilitados en `orders-api`), con `aud=payments-api`. Nunca reenvía el token del usuario — ese token tiene `aud=orders-api` y será rechazado por `payments-api`.

**Propagación de identidad de usuario:** si `payments-api` necesita la identidad del usuario original, el patrón es Token Exchange (RFC 8693). Requiere configuración específica en el realm y aprobación de arquitectura antes de implementar. Proceso: ADR con el diseño del flujo de identidad (qué claims propagar, qué servicio actúa como exchanger), enviado al canal de arquitectura del proyecto antes de iniciar la configuración.

### 3.6 Token lifetime por tipo

| Client type | Access token | Refresh token | Sesión idle / max |
|-------------|-------------|---------------|-------------------|
| SPA (`-web`) | 5 min | 30 min | 30 min / 10 h |
| API backend (`-api`) | 5 min | No aplica | — |
| Service account | 5 min | No aplica | — |
| `offline_access` habilitado | 5 min | 30 días | — |

> **Nota:** "Sesión idle / max" refiere a la sesión SSO de realm (`ssoSessionIdleTimeout` / `ssoSessionMaxLifespan`). El `clientSessionMaxLifespan` (sesión por client) es 8 h. Los valores de referencia están en [keycloak-examples.md — Realm](./keycloak-examples.md#realm).

Valores distintos al default requieren documentación como excepción en `realms/{realm-name}/README.md` con justificación.

---

## 4. Roles

Un role es un permiso asignable a usuarios o service accounts, evaluado por los servicios backend. Los roles no modelan identidades — modelan permisos.

### 4.1 Naming

**Regla:** `{dominio}:{acción}` en `kebab-case`, minúsculas, separador `:`.

```
Correcto:   orders:read  ·  orders:approve  ·  payments:process-refund  ·  platform:admin
```

| Incorrecto | Motivo |
|------------|--------|
| `ADMIN` | Mayúsculas, sin contexto de dominio |
| `role_read` | Prefijo húngaro, sin dominio |
| `uma_authorization` | Rol interno de Keycloak |
| `read` | Ambiguo — ¿leer qué? |
| `orders_read` | Separador incorrecto — usar `:` |
| `orders-read` | Separador incorrecto — el guion va dentro de la acción: `orders:process-refund` |
| `Orders:Read` | Capitalización incorrecta |

### 4.2 Realm roles vs Client roles

| Tipo | Alcance | Cuándo usar |
|------|---------|-------------|
| Client role | Específico del client que lo define | Roles evaluados por un solo servicio: `orders:approve` en `orders-api` |
| Realm role | Todo el realm | Roles transversales o composites que agrupan client roles (`platform:admin`) |

**Regla:** roles de negocio como client roles del API que los evalúa. Los client roles solo aparecen en tokens con `aud` coincidente con el client que los define — no inflan el token de otros clients.

### 4.3 Patrón super admin

Para perfiles que requieren acceso de lectura y acción en todos los dominios de la plataforma (soporte transversal, administradores de sistema):

```
Realm role: platform:admin  (composite)
  ├── orders-api  → orders:read, orders:create, orders:approve, orders:cancel
  └── payments-api → payments:read, payments:refund, payments:void
```

Al incorporar un nuevo dominio, actualizar `platform:admin` para incluir sus client roles. Los service accounts de workers no deben recibir `platform:admin` — solo los roles del dominio en el que operan.

> **Límite de escala:** a partir de 8 dominios, el mantenimiento manual de `platform:admin` es propenso a errores y dificulta la auditoría. Con 8+ dominios, aplicar el patrón de Keycloak Groups: definir un grupo por perfil transversal (ej: `platform-read-only`, `platform-admin`), asignar los client roles directamente al grupo, y gestionar la pertenencia de usuarios al grupo en lugar de asignar roles individuales. El onboarding de un nuevo dominio agrega sus client roles al grupo sin modificar asignaciones de usuarios. Requiere ADR antes de implementar — proceso en 2.1.

### 4.4 Composite roles

**Regla:** máximo 1 nivel de anidamiento.

```
Correcto:
  orders:manager → orders:read, orders:create, orders:approve, orders:cancel

Incorrecto:
  orders:manager → orders:operator → orders:read   (grafo imposible de auditar)
```

Los composite roles se resuelven en tiempo de emisión del token. Anidamiento profundo hace imposible responder "¿qué permisos efectivos tiene este usuario?" sin recorrer el grafo completo.

---

## 5. Client Scopes

Un scope es un permiso OAuth2 solicitado por un client al pedir un token. Sigue el mismo patrón de naming que los roles.

### 5.1 Naming

```
Correcto:   orders:read  ·  orders:write  ·  payments:read  ·  offline_access (nombre canónico OAuth2)
Incorrecto: read_orders  ·  scope-orders-read  ·  orders
```

### 5.2 Relación scope → role

| Objeto | Quién lo solicita | Quién lo evalúa |
|--------|-------------------|-----------------|
| Scope | El client frontend en el `authorization_request` | Keycloak — decide qué incluir en el token |
| Role | No se solicita — se asigna al usuario en el realm | El backend al procesar la petición |

**El mapeo scope → role requiere un mapper explícito configurado en Keycloak.** El naming coincidente entre scope y role no produce el mapeo automáticamente. Ver [keycloak-examples.md — Scope mapper](./keycloak-examples.md#scope-mapper).

Un scope puede agrupar múltiples roles:

```
scope orders:write  →  roles: orders:create, orders:approve
scope orders:read   →  roles: orders:read
```

El backend evalúa siempre los roles individuales en `resource_access`, nunca el scope directamente.

### 5.3 Default vs Optional

| Tipo | Ejemplos | Regla |
|------|----------|-------|
| Default | `openid`, `profile`, `email`, `roles` | Solo lo que siempre se necesita |
| Optional | `orders:read`, `payments:write`, `offline_access` | El client los solicita explícitamente en el `authorization_request` |

El scope `roles` (protocolo de Keycloak) habilita el claim `resource_access` en el token. Incluirlo como default es correcto — sin él, los client roles no aparecen en el JWT.

`offline_access` habilita refresh tokens de larga duración. Usar solo en aplicaciones que requieren acceso real cuando el usuario está desconectado (daemons, sincronización en background). No activar por defecto.

---

## 6. Protocol Mappers

### 6.1 Naming

`{claim-name}-mapper`

```
Correcto:   tenant-id-mapper  ·  cost-center-mapper  ·  employee-id-mapper
Incorrecto: mapper1  ·  add-tenant-claim  ·  TenantIdMapper
```

### 6.2 Claims: access token vs ID token

| Token | Qué incluir |
|-------|------------|
| Access token | Roles, scopes, claims de negocio (`tenant_id`, `cost_center`) |
| ID token | Información de perfil del usuario (`name`, `email`, `locale`) |
| Refresh token | Sin claims custom |

**Regla:** claims de negocio solo en el access token. El ID token no se envía a APIs backend — si incluye claims de negocio y llega a un servicio, expone información de autorización sensible.

> **Límite de tamaño:** un JWT que supera 8 KB puede ser rechazado por proxies HTTP con error 400/431. Limitar a 5 claims custom por token. Si el dominio requiere más, revisar qué claims son realmente necesarios en cada servicio — no todos los servicios necesitan todos los claims.

### 6.3 Claim names

`snake_case` para todos los claims custom, consistente con el estándar OAuth2/OIDC (`access_token`, `expires_in`, `sub`, `iss`). La convención de campos JSON del proyecto REST es independiente de esta decisión.

```json
{ "tenant_id": "acme-corp", "cost_center": "FIN-001" }   // Correcto
{ "tenantId": "acme-corp",  "CostCenter": "FIN-001" }    // Incorrecto
```

### 6.4 Audience mapper

**Regla:** cada API backend requiere un audience mapper que incluya su client ID en el claim `aud` del access token.

Sin este mapper, el token no incluye el client en `aud` y el API rechazará el request con `401 Unauthorized` aunque los roles sean correctos — es el error de configuración más frecuente.

Ver template en [keycloak-examples.md — Audience mapper](./keycloak-examples.md#audience-mapper).

---

## 7. Anti-patterns

| Anti-pattern | Síntoma visible | Consecuencia | Solución |
|---|---|---|---|
| Wildcard en Valid Redirect URIs (`*`) | El authorization code puede enviarse a cualquier `redirect_uri`, incluso a dominios externos | Authorization code redirigible a dominio externo — vector de robo de token | URIs exactas por ambiente — ver 3.3 |
| `Web Origins = *` o `+` en clients públicos | Cualquier origen supera el CORS preflight al endpoint de token | Cualquier dominio puede solicitar tokens en nombre del usuario | Dominios exactos del frontend — ver 3.3 |
| Implicit flow habilitado | El token aparece en la URL del navegador y en el header `Referer` | Token expuesto en historial del navegador — deprecated RFC 9700 | Standard flow + PKCE S256 — ver 3.2 |
| `client_secret` en client público SPA | El secret es visible inspeccionando el bundle JavaScript | Credencial expuesta públicamente en el bundle | Client público sin secret + PKCE — ver 3.2 |
| `offline_access` como default scope | Todos los tokens llevan refresh de larga duración aunque el client no lo necesite | Superficie de ataque ampliada innecesariamente en todos los clients | Mover a optional scope — ver 5.3 |
| Ambiente en el client ID (`orders-api-dev`) | Los logs de ambientes distintos son incomparables; aparece lógica condicional en código | Acoplamiento del código al nombre del client; auditoría cruzada entre ambientes imposible | Mismo client ID en todos los ambientes — ver 3.1 |
| N realms para N microservicios | El usuario re-autentica al navegar entre servicios | Sin SSO; token exchange complejo y frágil | Opción A: un realm por plataforma — ver 2.1 |
| Ausencia de audience mapper en la API | El API responde `401` aunque el token sea válido y los roles correctos | Servicio inoperativo por configuración ausente | Audience mapper en cada client API — ver 6.4 |
| Roles internos de Keycloak en lógica de negocio (`uma_authorization`) | El servicio falla tras un upgrade de Keycloak sin cambios en el código | Acoplamiento con internos del protocolo — inestable entre versiones | Roles de dominio propios con `{dominio}:{acción}` — ver 4.1 |
| Roles de dominio definidos como realm roles | El JWT crece en todos los clients del realm, no solo los que evalúan el rol | Token inflado; latencia en validación; posible rechazo por proxies | Client roles en el API que los evalúa — ver 4.2 |
| Composite roles anidados profundamente | Imposible responder "¿qué permisos tiene este usuario?" sin recorrer el grafo completo | Permisos efectivos inauditables | Máximo 1 nivel de composición — ver 4.4 |
| Claims de negocio en el ID token | Un servicio que recibe el ID token por error obtiene claims de autorización | Información de autorización expuesta fuera del flujo esperado | Claims de negocio solo en access token — ver 6.2 |
| IdP externo (AD, Google) consumido directamente por servicios | Cambiar de IdP requiere modificar todos los servicios que lo consumen | Acoplamiento con el IdP externo — migración costosa | Keycloak como broker único — ver 2.1 para proceso de aprobación |
| Modificar el built-in `browser` flow | Los cambios no aparecen en el export del realm ni en el historial de configuración | Cambios invisibles en exports; puede romperse en upgrades | Copiar → renombrar → modificar la copia |
| Reenviar el token del usuario en llamadas B2B | El servicio destino responde `401` | El token tiene `aud` del client original — rechazado por el destino | Token propio con `client_credentials` — ver 3.5 |

---

## 8. Referencias

- [Keycloak 26 — Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/)
- [RFC 6749 — OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 8693 — OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)
- [RFC 9700 / BCP 212 — OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/rfc9700)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [keycloak-examples.md](./keycloak-examples.md) — Referencia de Configuración
- [keycloak-setup.md](./keycloak-setup.md) — Lineamiento DevOps
