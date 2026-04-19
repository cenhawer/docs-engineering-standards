# Keycloak — Referencia de Configuración

> **Audiencia:** equipo de desarrollo.
>
> **No cubre:** instalación, variables de entorno, separación de ambientes, pipelines → [keycloak-setup.md](./keycloak-setup.md).
>
> **Convenciones de naming y decisiones de diseño** → [keycloak-conventions.md](./keycloak-conventions.md).

**Versión:** 1.0 · **Fecha:** 2026-04-15 · **Referencia:** Keycloak 26.x (Quarkus)

> **Cómo usar esta guía:** si estás configurando un dominio desde cero, sigue el [Modelo de dependencias](#modelo-de-dependencias) en orden. Si incorporas un dominio a una plataforma existente, ve directamente al [Checklist de validación de dominio](#checklist-de-validación-de-dominio). Para convenciones de naming y decisiones de diseño, consulta [keycloak-conventions.md](./keycloak-conventions.md).

---

## Contenido

- [Modelo de dependencias](#modelo-de-dependencias)
- [Variables de entorno — consumo de la aplicación](#variables-de-entorno--consumo-de-la-aplicación)
- [Realm](#realm)
- [Client API backend](#client-api-backend)
- [Client SPA](#client-spa)
- [Service account M2M](#service-account-m2m)
- [Client roles](#client-roles)
- [Composite role — platform:admin](#composite-role--platformadmin)
- [Usuario de prueba](#usuario-de-prueba)
- [Audience mapper](#audience-mapper)
- [Client Scope](#client-scope)
- [Scope mapper](#scope-mapper)
- [Protocol mapper — custom claim](#protocol-mapper--custom-claim)
- [Access token — contrato](#access-token--contrato)
- [Checklist de validación de dominio](#checklist-de-validación-de-dominio)

---

## Modelo de dependencias

Al configurar un dominio desde cero, el orden de creación importa porque cada objeto depende del anterior.

```
Realm
└── Client API (confidencial)
    ├── Client roles → {dominio}:{acción}
    ├── Composite role (platform:admin actualizado)
    ├── Audience mapper → claim aud del access token
    └── Protocol mappers → claims custom (si aplica)
└── Client Scope → {dominio}:{acción}
    └── Scope mapper → roles incluidos en el token según scope solicitado
└── Client SPA (público)
    └── Optional scopes → {dominio}:{acción} asignados desde Client Scope
└── Service account M2M (si el dominio tiene workers)
    └── Roles asignados directamente al service account
└── Usuario de prueba → client roles asignados + atributos configurados
```

Para incorporar un nuevo dominio a una plataforma existente: ir directamente al [Checklist de validación de dominio](#checklist-de-validación-de-dominio).

---

## Variables de entorno — consumo de la aplicación

Al completar la configuración del realm, la aplicación necesita estos valores inyectados como variables de entorno. No son parámetros de código — son configuración de infraestructura.

| Variable | Origen |
|---|---|
| URL base de Keycloak | Host por ambiente — ver [keycloak-setup.md — Separación de Ambientes](./keycloak-setup.md#separación-de-ambientes) |
| Realm | Nombre del realm creado |
| Client ID | Client ID del client del servicio |
| Client secret | Solo clients confidenciales — inyectar desde vault, nunca hardcodear |

> **SPA:** no tiene `client_secret`. El redirect URI configurado en el client debe coincidir exactamente con el de la aplicación.

---

## Realm

El realm es el namespace completo de identidad de la plataforma. Se crea una vez por plataforma.

| Parámetro | Valor | Razón |
|---|---|---|
| Realm ID | `{nombre-plataforma}` en `kebab-case` | Sin sufijo de ambiente, sin prefijo de empresa |
| User registration | `OFF` | Los usuarios se crean via IdP externo o gestión manual |
| Login with email | `ON` | El email es el identificador único garantizado por `Duplicate emails OFF` |
| Duplicate emails | `OFF` | Integridad de identidad |
| Forgot password | `ON` | Recuperación de contraseña sin intervención del administrador |
| Brute force detection | `ON` | Seguridad mínima no negociable |
| Access token lifespan | 5 minutos | Balance entre seguridad y overhead de refresh |
| SSO session idle | 30 minutos | Equilibrio entre seguridad y UX — el usuario inactivo cierra sesión sin forzar re-login en uso normal |
| SSO session max | 10 horas | Sesión SSO de realm (`ssoSessionMaxLifespan`) — cubre una jornada laboral completa |
| Client session max | 8 horas | Sesión por client (`clientSessionMaxLifespan`) — inferior al SSO max para forzar re-validación por client |
| Password: longitud mínima | 12 caracteres | Umbral mínimo contra ataques de fuerza bruta según NIST SP 800-63B |
| Password: complejidad | 1 mayúscula, 1 minúscula, 1 dígito, 1 especial | Aumenta el espacio de búsqueda efectivo sin imponer cambios frecuentes |

> El realm `master` es exclusivo para administración de Keycloak. Prohibido usarlo para workloads de aplicación.
>
> **Default client scopes:** el scope `roles` debe estar configurado como default client scope del realm. Sin él, el claim `resource_access` no aparece en ningún token del realm y todos los backends responden `403` independientemente de los roles asignados al usuario.

---

## Client API backend

Representa el servicio que recibe y valida tokens. Un client por dominio.

| Parámetro | Valor | Razón |
|---|---|---|
| Client ID | `{dominio}-api` | Sin sufijo de ambiente — ver 3.1 de conventions |
| Client authentication | `ON` (confidencial) | Las APIs no son públicas |
| Standard flow | `OFF` | Las APIs no redirigen usuarios al login — el flujo de autenticación lo gestiona el client SPA |
| Direct access grants | `OFF` | Evita que un client obtenga tokens con credenciales de usuario directamente, sin pasar por el consentimiento del usuario |
| Service accounts | `OFF` | Activar solo si este API llama a otras APIs con su propia identidad |

El `client_secret` se copia una sola vez al crearlo y se inyecta en vault o pipeline. Nunca se hardcodea.

---

## Client SPA

Representa el frontend que autentica usuarios. Un client por dominio.

| Parámetro | Valor | Razón |
|---|---|---|
| Client ID | `{dominio}-web` | Sin sufijo de ambiente |
| Client authentication | `OFF` (público) | El bundle JavaScript es texto plano para cualquier usuario |
| Standard flow | `ON` | Es el único flujo válido para SPAs — authorization code + PKCE |
| Implicit flow | `OFF` | Deprecated — RFC 9700 / BCP 212 |
| Direct access grants | `OFF` | Evita que un client obtenga tokens con credenciales de usuario directamente, sin pasar por el consentimiento del usuario |
| PKCE | `S256` obligatorio | Reemplaza el `client_secret` en clients públicos |
| Valid Redirect URIs | URIs exactas por ambiente | Prohibido `*` — vector de robo de token |
| Web Origins | Dominio exacto del frontend | Prohibido `*` y `+` |
| Valid post logout redirect URIs | URI exacta del frontend | Controla adónde redirige Keycloak tras el logout — sin esta configuración, el usuario queda en una página en blanco de Keycloak |

> Para desarrollo local: agregar `http://localhost:{puerto}/callback` y el origen correspondiente solo en la instancia de dev.

---

## Service account M2M

Representa un worker o job que requiere tokens sin usuario activo.

| Parámetro | Valor | Razón |
|---|---|---|
| Client ID | `{nombre}-worker` | Refleja el dominio y la naturaleza del proceso, no la tecnología |
| Client authentication | `ON` | El worker guarda el secret en entorno de servidor — no es público |
| Service accounts | `ON` | Habilita `client_credentials` para obtener tokens sin usuario activo |
| Standard flow | `OFF` | Los workers no tienen usuario ni navegador — no necesitan el flujo de redirección |
| Direct access grants | `OFF` | Evita que un client obtenga tokens con credenciales de usuario directamente |

Los roles se asignan directamente al service account del client, no a usuarios. Los workers no reciben `platform:admin` — solo los roles del dominio en que operan.

> **Asignación de roles al service account:** los client roles se asignan desde el client M2M en "Service accounts" → "Assign role" → filtrar por client `{dominio}-api`. A diferencia de los usuarios, el service account no tiene perfil propio — la asignación vive en el client que lo representa.

---

## Client roles

Los roles de dominio se definen como client roles en el client API que los evalúa. Patrón de naming: `{dominio}:{acción}`.

Ejemplo para el dominio `orders`:

| Role | Semántica |
|---|---|
| `orders:read` | Consultar órdenes |
| `orders:create` | Crear nuevas órdenes |
| `orders:approve` | Aprobar órdenes pendientes |
| `orders:cancel` | Cancelar órdenes |

Los client roles solo aparecen en tokens con `aud` coincidente con el client que los define — no inflan el token de otros clients del realm.

---

## Composite role — platform:admin

Agrupa todos los client roles de la plataforma para perfiles que requieren acceso de lectura y acción en todos los dominios (soporte transversal, administradores de sistema). Ver criterio completo en [keycloak-conventions.md — 4.3 Patrón super admin](./keycloak-conventions.md#43-patrón-super-admin).

```
Realm role: platform:admin (composite, máximo 1 nivel de anidamiento)
  ├── {dominio-a}-api → {dominio-a}:read, {dominio-a}:create, ...
  └── {dominio-b}-api → {dominio-b}:read, {dominio-b}:write, ...
```

Al incorporar un nuevo dominio: actualizar `platform:admin` con sus client roles.

> **Límite de escala:** con más de 8 dominios, el mantenimiento manual de este composite es propenso a errores. Evaluar el patrón de Keycloak Groups — ver [keycloak-conventions.md — 4.3](./keycloak-conventions.md#43-patrón-super-admin).

---

## Usuario de prueba

Para validar la configuración del dominio se necesita al menos un usuario con los roles y atributos del dominio asignados. Sin usuario no es posible emitir un token real a través del client SPA ni verificar que `aud`, `resource_access` y claims custom se incluyen correctamente.

| Parámetro | Valor | Razón |
|---|---|---|
| Username | Email recomendado | `Login with email: ON` — el email es el identificador de login en el realm |
| Email | Email válido | Requerido para el flujo de autenticación |
| Email verified | `ON` en ambientes no productivos | Sin verificación activa, el login puede fallar o requerir flujo de confirmación por correo |
| Password temporal | `OFF` en dev y QA | Evita que Keycloak fuerce cambio de contraseña en el primer login durante pruebas |
| Client roles asignados | Roles de `{dominio}-api` a validar | El token incluye solo los roles del client cuyo `aud` coincide con el client API |

**Atributos de usuario**

Si el dominio usa protocol mappers custom, el atributo debe existir en el perfil del usuario antes de emitir el token. Sin el atributo configurado, el claim no aparece en el JWT — Keycloak lo omite silenciosamente sin producir error.

| Atributo | Valor de prueba |
|---|---|
| Nombre del atributo | Cualquier valor representativo — debe coincidir exactamente con el campo `User attribute` definido en el protocol mapper |

> **Asignación de roles:** los client roles se asignan desde el perfil del usuario en "Role mapping" → "Assign role" → filtrar por client `{dominio}-api`. Los realm roles (como `platform:admin`) se asignan en la misma sección sin filtro de client.
>
> Un usuario sin roles asignados genera un token con `aud` correcto pero con `resource_access.{dominio}-api.roles` vacío — el backend responde `403 Forbidden`, no `401 Unauthorized`.

---

## Audience mapper

**Obligatorio en cada client API.** Sin este mapper, el access token no incluye el client en el claim `aud` — el API responde `401` aunque los roles sean correctos y el token sea válido.

Vive en el **dedicated scope** del client API (`{client-id}-dedicated`), que Keycloak crea automáticamente al crear el client.

| Parámetro | Valor | Razón |
|---|---|---|
| Nombre | `audience-mapper` | Nombre canónico — identifica el mapper en el dedicated scope del API |
| Included Client Audience | `{client-id}` del API | El client ID que se incluye en el claim `aud` del access token |
| Add to access token | `ON` | El `aud` se valida en el access token — sin esto el API responde `401` |
| Add to ID token | `OFF` | El `aud` del ID token es el client frontend, no el API backend |

> **Advertencia operacional:** este mapper no se incluye en el export JSON del realm. Debe configurarse explícitamente en cada ambiente después del import — vía script idempotente de Admin REST API.
>
> **Requisito post-import:** este mapper debe configurarse vía Admin REST API como paso obligatorio en el pipeline de cada ambiente, inmediatamente después del import. La implementación del paso es responsabilidad del equipo — debe ser idempotente y el pipeline debe fallar si produce error. Ver [keycloak-setup.md — Limitación del export JSON](./keycloak-setup.md#limitación-del-export-json).

---

## Client Scope

Un client scope es un objeto del realm que agrupa un permiso OAuth2 y el mapper que lo traduce a roles en el token. Se crea una vez en el realm y se asigna como `Optional` al client SPA que lo solicita en el `authorization_request`.

| Parámetro | Valor | Razón |
|---|---|---|
| Nombre | `{dominio}:{acción}` en `kebab-case` | Mismo patrón que los roles — ver 5.1 de conventions |
| Protocol | `openid-connect` | — |
| Display on consent screen | `OFF` en plataformas internas | Activar solo si la plataforma requiere consentimiento explícito del usuario por acción |
| Include in token scope | `ON` | Hace visible el scope en el claim `scope` del access token |

Después de crear el client scope:

1. Agregar el scope mapper dentro del client scope — ver [Scope mapper](#scope-mapper)
2. Asignar el client scope como `Optional` en el client SPA correspondiente

> Si el client scope se asigna como `Default` en lugar de `Optional`, el scope se incluye en todos los tokens del client sin que el frontend lo solicite explícitamente — comportamiento correcto solo para scopes como `openid`, `profile` o `email`. Los scopes de dominio (`orders:read`, `payments:write`) deben ser siempre `Optional`.

---

## Scope mapper

Vincula un scope con uno o más roles. El naming coincidente entre scope y role **no** produce el mapeo automáticamente — el mapper es obligatorio.

El mapper vive dentro del client scope (`{dominio}:{acción}`) creado en la sección anterior. El scope se asigna como `Optional` al client SPA.

| Objeto | Responsabilidad |
|---|---|
| Scope | El client frontend lo solicita en el `authorization_request` |
| Mapper | Keycloak incluye en el token los client roles del dominio cuando se solicita el scope |
| Role | El backend lo evalúa en `resource_access` — nunca evalúa el scope directamente |

| Parámetro | Valor | Razón |
|---|---|---|
| Mapper type | User Client Role | Incluye los client roles del dominio en el token cuando se solicita el scope |
| Client ID | `{dominio}-api` | El client API cuyos roles se incluyen |
| Add to access token | `ON` | Los roles se evalúan en el access token |
| Add to ID token | `OFF` | Los roles son claims de autorización, no de identidad del usuario |
| Multivalued | `ON` | El usuario puede tener varios roles del mismo client |

Un scope puede agrupar múltiples roles según lo que el usuario tenga asignado en ese client:

```
scope orders:write → incluye: orders:create, orders:approve  (si el usuario tiene esos roles en orders-api)
scope orders:read  → incluye: orders:read                    (si el usuario tiene ese rol en orders-api)
```

---

## Protocol mapper — custom claim

Incluye un atributo del usuario como claim en el access token. Vive en el dedicated scope del client API donde se valida el claim.

| Parámetro | Valor |
|---|---|
| Nombre | `{claim-name}-mapper` |
| User attribute | nombre del atributo en el perfil del usuario |
| Token claim name | `snake_case` — ver 6.3 de conventions |
| Claim JSON type | tipo primitivo del valor (`String`, `int`, etc.) |
| Add to access token | `ON` |
| Add to ID token | `OFF` |

Prerequisito: el atributo debe existir en el usuario antes de que el token lo incluya.

> **Límite de tamaño:** máximo 5 claims custom por token. Un JWT que supera 8 KB puede ser rechazado por proxies HTTP con error 400/431. Si el dominio requiere más claims, revisar qué claims son estrictamente necesarios en cada servicio — no todos los servicios necesitan todos los claims.

> **Advertencia operacional:** igual que el audience mapper — no persiste en el export JSON. Requiere configuración explícita en cada ambiente.

---

## Access token — contrato

El access token emitido correctamente para un usuario del dominio `orders` debe contener:

```json
{
  "iss": "https://keycloak-dev.empresa.internal/realms/ecommerce",
  "sub": "<uuid-del-usuario>",
  "aud": ["orders-api"],
  "azp": "orders-web",
  "scope": "openid profile email orders:read",
  "resource_access": {
    "orders-api": {
      "roles": ["orders:read", "orders:approve"]
    }
  },
  "tenant_id": "acme-corp",
  "iat": 1713000000,
  "exp": 1713000300
}
```

| Claim | Qué valida el backend |
|---|---|
| `aud` | Debe contener el `client_id` del API — si está ausente, el token fue emitido sin audience mapper |
| `resource_access.{client-id}.roles` | Roles efectivos del usuario para ese client — el backend evalúa este campo, nunca el campo `scope` |
| Claims custom (`tenant_id`, etc.) | Solo presentes si el usuario tiene el atributo configurado y el protocol mapper está activo |
| `azp` | Client que solicitó el token — útil para auditoría |

> **Scope en el `authorization_request`:** el SPA solicita scopes opcionales con el parámetro `scope` del protocolo OAuth2. Los scopes de dominio se añaden al valor base de `openid`:
>
> ```
> scope=openid orders:read
> scope=openid orders:read orders:write
> ```
>
> Sin el scope explícito en el request, Keycloak no incluye los roles del dominio en el token aunque el usuario los tenga asignados.

---

## Checklist de validación de dominio

Al incorporar un nuevo dominio `{x}` a la plataforma:

`[auto]` = verificable via Admin REST API, debe ser gate de pipeline. `[manual]` = requiere revisión humana en PR.

- [ ] `[auto]` Scope `roles` presente como default client scope del realm — `GET /admin/realms/{realm}/default-default-client-scopes` debe incluir una entrada con `name: roles`
- [ ] `[auto]` Client API `{x}-api` creado: confidencial, `standardFlowEnabled: false`, `directAccessGrantsEnabled: false`
- [ ] `[auto]` Client SPA `{x}-web` creado: `publicClient: true`, `pkce.code.challenge.method: S256`, `redirectUris` sin `*`
- [ ] `[manual]` Client roles creados en `{x}-api` con patrón `{x}:{acción}` — naming revisado en PR
- [ ] `[manual]` `platform:admin` actualizado con los nuevos client roles de `{x}-api`
- [ ] `[auto]` Audience mapper presente en el dedicated scope de `{x}-api` — paso post-import de mappers ejecutado sin errores
- [ ] `[manual]` Protocol mappers configurados si el dominio requiere claims custom — máximo 5 claims en total
- [ ] `[manual]` Client scopes creados con nombre `{x}:{acción}`, scope mapper configurado dentro de cada uno, asignados como `Optional` en el client SPA
- [ ] `[manual]` Usuario de prueba creado con client roles de `{x}-api` asignados y atributos configurados si el dominio usa protocol mappers
- [ ] `[auto]` `offline_access` no está en los default client scopes del realm — `GET /admin/realms/{realm}/default-default-client-scopes` no debe incluir una entrada con `name: offline_access`
- [ ] `[auto]` Token validado: `aud` incluye `{x}-api`, roles aparecen en `resource_access.{x}-api.roles`
- [ ] `[auto]` Mappers configurados vía Admin REST API en cada ambiente post-import sin errores — DevOps responsable
- [ ] `[manual]` Realm exportado y PR abierto antes de promover a QA — ver [keycloak-setup.md](./keycloak-setup.md#realm-export--import)
