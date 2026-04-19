# Configuración e Infraestructura — REST APIs

> **Audiencia:** equipo de Infra / DevOps. Esta guía no cubre diseño de endpoints, response patterns ni convenciones de nomenclatura — esas responsabilidades corresponden al equipo de desarrollo ([rest-conventions.md](rest-conventions.md)).
>
> **Cómo usar esta guía:** para un proyecto nuevo, lee las secciones 1 → 2 → 3 en orden. Para configuración puntual, ve directamente a la sección temática. Esta guía es agnóstica de tecnología — los conceptos aplican a cualquier plataforma, lenguaje o framework. Los workflows de CI/CD se ejemplifican en GitHub Actions; adaptar la sintaxis a la plataforma del equipo (GitLab CI, Azure DevOps, Bitbucket Pipelines, etc.).

**Versión:** 1.0

---

## Contenido

1. [OpenAPI Tooling y Documentación de API](#1-openapi-tooling-y-documentación-de-api)
   - [1.1 Herramientas de visualización](#11-herramientas-de-visualización)
   - [1.2 Exposición de documentación por ambiente](#12-exposición-de-documentación-por-ambiente)
   - [1.3 Integrar el contrato OpenAPI externo](#13-integrar-el-contrato-openapi-externo)
2. [CORS](#2-cors)
   - [2.1 Qué es CORS y por qué importa](#21-qué-es-cors-y-por-qué-importa)
   - [2.2 Configuración por ambiente](#22-configuración-por-ambiente)
3. [Versionado de API en infraestructura](#3-versionado-de-api-en-infraestructura)
   - [3.1 Ciclo de vida de versiones](#31-ciclo-de-vida-de-versiones)
4. [Rate Limiting](#4-rate-limiting)
   - [4.1 Concepto y propósito](#41-concepto-y-propósito)
   - [4.2 Headers estándar de rate limiting](#42-headers-estándar-de-rate-limiting)
   - [4.3 Límites recomendados por tipo de endpoint](#43-límites-recomendados-por-tipo-de-endpoint)
5. [Contract Testing en CI/CD](#5-contract-testing-en-cicd)
   - [5.1 Por qué contract testing](#51-por-qué-contract-testing)
   - [5.2 Etapas del pipeline de contract testing](#52-etapas-del-pipeline-de-contract-testing)
   - [5.3 Herramientas recomendadas](#53-herramientas-recomendadas)
6. [Seguridad a nivel de infraestructura](#6-seguridad-a-nivel-de-infraestructura)
   - [6.1 SSL/TLS](#61-ssltls)
   - [6.2 Autenticación y autorización en el gateway](#62-autenticación-y-autorización-en-el-gateway)
7. [Referencias](#7-referencias)

---

## Contexto

La configuración de infraestructura para APIs REST es responsabilidad de Infra/DevOps, separada del diseño de API que hace el equipo de desarrollo. Esta guía cubre conceptos y decisiones de configuración: OpenAPI tooling, CORS, rate limiting, contract testing en pipelines y seguridad a nivel de red. La implementación concreta en la plataforma elegida (gateway, middleware, runtime) es responsabilidad del equipo de Infra que debe mapear estos conceptos a las herramientas disponibles.

---

## 1. OpenAPI Tooling y Documentación de API

### 1.1 Herramientas de visualización

El contrato OpenAPI (`openapi.yaml` / `openapi.json`) puede visualizarse con distintas herramientas. La elección depende del contexto y las necesidades del equipo:

| Herramienta | Tipo | Propósito |
|-------------|------|-----------|
| [Swagger UI](https://swagger.io/tools/swagger-ui/) | UI interactiva | Explorar y probar endpoints desde el navegador. Integrable como middleware en la aplicación |
| [Redoc](https://redocly.com/redoc/) | Documentación HTML | Generación de documentación navegable y limpia desde el contrato OpenAPI |
| [Stoplight Elements](https://stoplight.io/open-source/elements) | UI embebible | Componente web para embeber documentación en cualquier sitio |
| [Scalar](https://scalar.com/) | UI moderna | Alternativa moderna a Swagger UI, con mejor experiencia de usuario |

> **Regla fundamental:** la herramienta de visualización consume el contrato OpenAPI como fuente de verdad. No genera el contrato — lo sirve. **El contrato vive en el repositorio**, no en el servidor. Esta regla aplica a todos los ambientes.

### 1.2 Exposición de documentación por ambiente

La documentación interactiva expone la estructura completa de todos los endpoints, parámetros y schemas. Esta información es útil en desarrollo y staging, pero representa un riesgo de seguridad en producción.

| Ambiente | UI de documentación | Endpoint OpenAPI | Acceso |
|----------|---------------------|-----------------|--------|
| Desarrollo | Habilitado | Habilitado | Sin restricción |
| Staging | Habilitado | Habilitado | Autenticado — ver mecanismos abajo |
| Producción | **Deshabilitado** | **Deshabilitado** | No exponer |

**Regla:** en producción, la UI de documentación y el endpoint `/openapi.yaml` no deben ser accesibles.

> **Por qué deshabilitar en producción:** la UI expone la estructura completa de la API, incluyendo todos los endpoints, parámetros y schemas. En producción, esto es información útil para atacantes (enumeración de endpoints, comprensión de la estructura de datos, identificación de endpoints no protegidos).

**Control de acceso en staging:**

Los mecanismos disponibles para proteger el acceso a la documentación, ordenados de mayor a menor preferencia:

| Mecanismo | Cuándo usar | Cómo funciona |
|-----------|-------------|---------------|
| **OAuth2 + OIDC** | **Recomendado.** Entornos corporativos con IdP activo (Azure AD, Okta, Keycloak, Auth0) | El acceso a `/docs` requiere autenticación via proveedor de identidad. El token se valida en el gateway antes de servir la UI. Soporta SSO, control por rol y audit trail |
| **API Key** | Equipos técnicos que necesitan acceso simple sin sesión | Se emite una key por equipo o persona. Se valida como header (`X-Api-Key`) en el gateway. Sin gestión de sesión, fácil de rotar por key individual |
| **IP Allowlist** | Complemento a cualquiera de los anteriores | Restringir el path de documentación a rangos de IP corporativas en el API Gateway o balanceador. Primera línea de defensa — no suficiente como único control |
| **HTTP Basic Auth** | Último recurso — entornos sin infraestructura de identidad | Usuario/contraseña en base64. Solo válido **sobre HTTPS**. No genera audit trail, las credenciales son globales (rotar afecta a todos los usuarios) y no se integra con gestión de identidad corporativa. Evitar si hay alternativa disponible |

> **Por qué OAuth2 + OIDC es la recomendación principal:** en equipos que ya operan un IdP corporativo, el costo de integrar OAuth2 es bajo y el beneficio es alto — audit trail, revocación individual, control por rol y compatibilidad con SSO. HTTP Basic Auth no ofrece ninguna de estas capacidades.

### 1.3 Integrar el contrato OpenAPI externo

Cuando el equipo sigue el flujo OpenAPI-first ([sección 10 de rest-conventions.md](rest-conventions.md#10-openapi-first)), el contrato (`openapi.yaml`) existe antes del código. La UI de documentación debe servir este contrato directamente.

**Flujo recomendado:**
1. El archivo `openapi.yaml` está versionado en el repositorio.
2. En el pipeline CI/CD, el archivo se publica como artefacto o se copia a un directorio de archivos estáticos.
3. La UI de documentación se configura para apuntar al archivo publicado — no al spec generado desde el código.

> Si la plataforma genera un spec desde el código (anotaciones, decorators, etc.), ese spec generado debe ser compatible con el contrato original. La validación ocurre en el pipeline — ver [sección 5](#5-contract-testing-en-cicd).

---

## 2. CORS

### 2.1 Qué es CORS y por qué importa

**Cross-Origin Resource Sharing (CORS)** es un mecanismo de seguridad del navegador que restringe las peticiones HTTP entre orígenes distintos (dominio, protocolo o puerto diferentes). Por defecto, los navegadores bloquean estas peticiones.

Para que una aplicación web en `https://app.example.com` pueda consumir una API en `https://api.example.com`, la API debe incluir los headers CORS apropiados en sus respuestas.

**Headers CORS relevantes:**

| Header | Dirección | Propósito |
|--------|-----------|-----------|
| `Access-Control-Allow-Origin` | Response | Orígenes permitidos para acceder a la API |
| `Access-Control-Allow-Methods` | Response | Métodos HTTP permitidos |
| `Access-Control-Allow-Headers` | Response | Headers que el cliente puede enviar |
| `Access-Control-Allow-Credentials` | Response | Si se permiten cookies/credenciales cross-origin |
| `Access-Control-Max-Age` | Response | Tiempo en segundos que el navegador puede cachear la respuesta preflight |

> **CORS es una restricción del navegador, no del servidor.** Las herramientas como curl, Postman o clientes HTTP programáticos no están sujetas a CORS. Solo los navegadores la imponen.

### 2.2 Configuración por ambiente

| Ambiente | Orígenes permitidos | Credenciales |
|----------|---------------------|--------------|
| Desarrollo | `*` (cualquier origen) | No requerido |
| Staging | Lista explícita de dominios de staging | Según necesidad |
| Producción | Lista explícita de dominios de producción | Solo si se requieren cookies |

**Reglas:**

- En producción, nunca usar `Access-Control-Allow-Origin: *` con `Access-Control-Allow-Credentials: true` — es una vulnerabilidad de seguridad (permite que cualquier sitio realice peticiones autenticadas a la API).
- La lista de orígenes permitidos debe estar externalizada como configuración del ambiente — no hardcodeada.
- Los headers custom del sistema (`X-Request-Id`, `X-Correlation-Id`, `X-Channel-Id`, `X-Api-Version`, etc.) deben estar incluidos en `Access-Control-Allow-Headers`.
- Configurar `Access-Control-Max-Age` para reducir el volumen de peticiones preflight en producción.

**Configuración de CORS recomendada para producción:**

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-Request-Id, X-Correlation-Id, X-Channel-Id, X-End-User-Login, X-Api-Version
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 3600
```

---

## 3. Versionado de API en infraestructura

### 3.1 Ciclo de vida de versiones

El versionado semántico lo gestiona el equipo de desarrollo ([sección 7 de rest-conventions.md](rest-conventions.md#7-versionado)). Infra/DevOps gestiona el ciclo de vida de las versiones en producción:

| Fase | Descripción | Acción de Infra |
|------|-------------|-----------------|
| **Active** | Versión actual en uso | Disponible en todos los ambientes |
| **Deprecated** | Versión antigua con plazo de fin de vida | Agregar header `Sunset` en los responses con la fecha de corte |
| **Retired** | Versión fuera de soporte | Retornar `410 Gone` con mensaje de migración |

**Períodos mínimos de deprecación:**

| Tipo de consumidor | Período mínimo en fase Deprecated |
|--------------------|----------------------------------|
| APIs internas (misma organización) | 90 días |
| APIs con consumidores externos | 180 días |

> El período puede extenderse por acuerdo con los consumidores, pero **nunca acortarse** una vez comunicado. La fecha de sunset es un SLA, no una estimación.

**Header `Sunset` para versiones deprecadas (RFC 8594):**

El header `Sunset` informa a los consumidores cuándo la versión dejará de estar disponible:

```
HTTP/1.1 200 OK
Sunset: Tue, 31 Dec 2026 23:59:59 GMT
Deprecation: true
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

Este header debe agregarse automáticamente por el API Gateway o un middleware para todas las respuestas de la versión deprecada.

**Comunicación del sunset:**
- Agregar el header `Sunset` en todas las respuestas de la versión deprecada desde que se anuncia el fin de vida.
- Notificar a los consumidores registrados por correo o canal oficial.
- Mantener la versión deprecada funcionando hasta la fecha de sunset — no retirarla antes.

---

## 4. Rate Limiting

### 4.1 Concepto y propósito

El rate limiting limita la cantidad de requests que un cliente puede realizar en un período de tiempo. Protege la API de:

- Abuso intencional (ataques de denegación de servicio a nivel de aplicación)
- Consumo excesivo involuntario (bugs en el cliente que generan bucles)
- Equidad entre consumidores (evitar que un cliente monopolice recursos)

> **Alcance:** el rate limiting protege contra abuso a nivel de aplicación. Para ataques volumétricos (DDoS de red), se requiere protección dedicada en la capa de infraestructura (CDN, DDoS mitigation services) — fuera del alcance de esta guía.

El rate limiting se implementa típicamente en el API Gateway o en un middleware de la aplicación, **antes** de que el request llegue a la lógica de negocio.

**Estrategias comunes:**

| Estrategia | Descripción | Cuándo usar |
|-----------|-------------|-------------|
| Fixed window | Límite fijo por ventana de tiempo (ej. 100 req/min) | Simplicidad. Puede tener picos en el borde de la ventana |
| Sliding window | Ventana deslizante — suaviza los picos | Mayor precisión. Más costoso computacionalmente |
| Token bucket | Tokens se acumulan con el tiempo hasta un máximo | Permite bursts controlados |
| Leaky bucket | Queue con tasa de salida constante | Tasa de proceso absolutamente uniforme |

**Clave de partición:** el límite puede aplicarse por IP, por usuario autenticado, por API key, o por cualquier combinación. Elegir la clave según el modelo de autenticación de la API.

### 4.2 Headers estándar de rate limiting

Cuando se excede el límite, el servidor retorna `429 Too Many Requests`. Los siguientes headers informan al cliente sobre el estado actual y cuándo puede reintentar:

| Header | Tipo | Descripción |
|--------|------|-------------|
| `X-RateLimit-Limit` | Número | Límite total de requests permitidos en la ventana |
| `X-RateLimit-Remaining` | Número | Requests restantes en la ventana actual |
| `X-RateLimit-Reset` | Unix timestamp | Momento en que se reinicia el contador (epoch seconds, UTC) |
| `Retry-After` | Segundos | Cuántos segundos debe esperar el cliente antes de reintentar. Incluir en respuestas `429` |

> `X-RateLimit-Limit`, `X-RateLimit-Remaining` y `X-RateLimit-Reset` son de facto estándar — usados por GitHub, Stripe, Twitter/X y la mayoría de APIs públicas. `Retry-After` es un header estándar HTTP (RFC 9110).

> **Nota sobre estandarización:** el IETF tiene un draft activo (`draft-ietf-httpapi-ratelimit-headers`) que propone headers sin prefijo `X-` (`RateLimit-Policy`, `RateLimit`). A la fecha de esta guía el draft no ha alcanzado estatus de RFC. Para APIs nuevas con horizonte de largo plazo, monitorear su avance. Documentar explícitamente si `X-RateLimit-Reset` contiene timestamp absoluto o segundos restantes — ambas convenciones existen en la industria.

**Response de rate limit excedido:**

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1743429600

{
  "type": "https://tools.ietf.org/html/rfc6585#section-4",
  "title": "Too Many Requests",
  "status": 429,
  "detail": "Rate limit exceeded. Retry after 30 seconds."
}
```

> El cuerpo de error sigue el mismo formato `ProblemDetails` que usa el resto de la API — consistente con el manejo de excepciones del backend. Ver [rest-conventions.md 9.5](rest-conventions.md#95-manejo-de-errores--rfc-7807).

### 4.3 Límites recomendados por tipo de endpoint

| Tipo de endpoint | Límite sugerido | Ventana |
|------------------|-----------------|---------|
| Lectura (GET) | 200 req | 1 minuto |
| Escritura (POST, PUT, PATCH) | 60 req | 1 minuto |
| Eliminación (DELETE) | 30 req | 1 minuto |
| Acciones de negocio | 10 req | 1 minuto |

> **Acciones de negocio:** endpoints que ejecutan operaciones con efecto irreversible, costoso o de alto impacto — por ejemplo: `/payments/process`, `/emails/send`, `/reports/generate`, `/contracts/sign`. Se identifican en el contrato OpenAPI con la etiqueta `x-rate-limit-tier: business`. Los límites son puntos de partida; ajustar según el perfil de uso real y el tipo de cliente (usuario final, sistema de integración, batch process).

---

## 5. Contract Testing en CI/CD

### 5.1 Por qué contract testing

El flujo OpenAPI-first ([sección 10 de rest-conventions.md](rest-conventions.md#10-openapi-first)) define el contrato antes del código. Sin validación automatizada, el contrato y la implementación pueden divergir silenciosamente:

| Sin contract testing | Con contract testing |
|---------------------|---------------------|
| El contrato y el código divergen sin que nadie lo detecte | El pipeline falla si la implementación no cumple el contrato |
| Los consumidores reciben respuestas distintas a las documentadas | El contrato es garantía verificable, no documentación opcional |
| Los errores se detectan en QA o producción | Los errores se detectan en el PR, antes del merge |

### 5.2 Etapas del pipeline de contract testing

El pipeline de contract testing tiene tres etapas diferenciadas:

**Etapa 1 — Linting del contrato (en PR):**
- Se ejecuta en cada PR que modifica el archivo `openapi.yaml`.
- Valida que el contrato es sintácticamente correcto y cumple las reglas de estilo definidas.
- Herramienta: Spectral con reglas personalizadas.
- **Fallo bloquea el merge.**

**Etapa 2 — Validación de la implementación (en CI):**
- Se ejecuta después de construir la aplicación.
- Levanta el servidor y valida que los requests y responses cumplen el contrato.
- Detecta endpoints faltantes, schemas incompatibles, códigos HTTP incorrectos.
- Herramientas: Schemathesis o Prism.
- **Fallo bloquea el despliegue.**

**Etapa 3 — Breaking change detection:**
- Compara el contrato actual contra la versión anterior (main branch).
- Detecta cambios incompatibles: campos eliminados, tipos cambiados, endpoints removidos.
- Un breaking change sin incremento de MAJOR en SemVer debe bloquear el merge.
- Herramienta: openapi-diff.

**Reglas del pipeline:**

| Regla | Descripción |
|-------|-------------|
| Lint en todo PR que toque el contrato | Spectral ejecuta en cada PR que modifique `openapi.yaml` |
| Fallo por `error`, warning por `warn` | Los errores bloquean el merge; los warnings son informativos |
| Comparación de specs | El spec generado o expuesto por la implementación debe ser compatible con el contrato |
| Breaking changes bloquean el merge | Un cambio incompatible requiere incremento de `MAJOR` en SemVer |

### 5.3 Herramientas recomendadas

> **Versión de referencia:** OpenAPI Specification **3.2.0** (lanzada septiembre 2025, última versión estable a la fecha de esta guía). Las herramientas listadas tienen soporte activo para esta versión.

| Herramienta | Propósito | Etapa |
|-------------|-----------|-------|
| [Spectral](https://stoplight.io/open-source/spectral) | Linting del contrato OpenAPI — valida estilo y estructura | Etapa 1 |
| [Prism](https://stoplight.io/open-source/prism) | Mock server y validación de requests/responses contra contrato | Etapa 2 |
| [Schemathesis](https://schemathesis.readthedocs.io/) | Property-based testing contra la implementación real — detecta casos borde que el contrato no anticipa | Etapa 2 |
| [openapi-diff](https://github.com/OpenAPITools/openapi-diff) | Detecta breaking changes entre versiones del contrato | Etapa 3 |

**Configuración de Spectral (`.spectral.yaml`):**

```yaml
extends: ["spectral:oas"]

rules:
  # Todos los paths deben tener operationId
  operation-operationId: error

  # Todos los responses deben tener descripción
  operation-description: warn
```

> **OpenAPI 3.1+ y `$ref`:** a partir de OpenAPI 3.1, los `$ref` pueden tener propiedades hermanas (siblings) — esto es válido por spec, alineado con JSON Schema 2020-12. No agregar reglas que prohíban este patrón.

**Workflow completo en GitHub Actions** *(ejemplo de referencia — adaptar a la plataforma CI/CD del equipo):*

Los tres jobs van en el mismo archivo. `validate-implementation` y `detect-breaking-changes` tienen dependencia sobre `lint-contract` — si el contrato no pasa linting, no tiene sentido continuar.

```yaml
# .github/workflows/contract-validation.yml
name: Contract Validation

on:
  pull_request:
    paths:
      - 'openapi.yaml'
      - 'openapi.json'
      - 'docs/api/**'

jobs:

  # Etapa 1 — Linting del contrato
  lint-contract:
    name: Lint OpenAPI contract
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Spectral
        run: npm install -g @stoplight/spectral-cli

      - name: Lint OpenAPI contract
        run: spectral lint openapi.yaml --ruleset .spectral.yaml --fail-severity error

  # Etapa 2 — Validación de implementación contra contrato
  validate-implementation:
    name: Validate implementation against contract
    runs-on: ubuntu-latest
    needs: lint-contract
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: docker compose up -d --wait   # ajustar al método de arranque del proyecto

      - name: Install Schemathesis
        run: pip install schemathesis

      - name: Run contract validation
        run: st run openapi.yaml --base-url http://localhost:8080 --checks all --hypothesis-max-examples 50

  # Etapa 3 — Detección de breaking changes
  detect-breaking-changes:
    name: Detect breaking changes
    runs-on: ubuntu-latest
    needs: lint-contract
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Export contract from main branch
        run: git show origin/main:openapi.yaml > openapi-main.yaml

      - name: Download openapi-diff
        run: |
          curl -L https://github.com/OpenAPITools/openapi-diff/releases/latest/download/openapi-diff-cli.jar \
            -o openapi-diff.jar

      - name: Check for breaking changes
        run: java -jar openapi-diff.jar openapi-main.yaml openapi.yaml --fail-on-incompatible
```

---

## 6. Seguridad a nivel de infraestructura

### 6.1 SSL/TLS

**Reglas:**

| Regla | Descripción |
|-------|-------------|
| HTTPS obligatorio | Todo tráfico en staging y producción debe ir sobre HTTPS. HTTP sin cifrado no es aceptable |
| Redirección HTTP → HTTPS | Todo request HTTP debe redirigirse automáticamente a HTTPS con `301 Moved Permanently` |
| Versión preferida de TLS | **TLS 1.3** — elimina cipher suites legacy por diseño, handshake en 1-RTT, sin negociación de algoritmos débiles (RFC 8446). Recomendado para todas las APIs nuevas |
| Versión mínima de TLS | **TLS 1.2** como mínimo aceptable. TLS 1.0 y 1.1 deben estar deshabilitados — deprecados por AWS y Azure (2024–2026) y requerido por PCI DSS, HIPAA y SOC 2 |
| Terminación TLS | La terminación TLS ocurre en el API Gateway o load balancer — no en la aplicación directamente en producción |
| HSTS | Agregar el header `Strict-Transport-Security` en producción para indicar al navegador que solo use HTTPS |
| Certificados | Usar certificados de una CA reconocida. Renovar antes del vencimiento (automatizar con Let's Encrypt o equivalente) |
| Desarrollo local | HTTPS también recomendado en desarrollo para detectar problemas de mixed content antes de producción |

**Header HSTS recomendado:**

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

> **Sobre la directiva `preload`:** omitida intencionalmente. `preload` requiere registrar el dominio en [hstspreload.org](https://hstspreload.org/) y es **difícil de revertir** — una vez en la lista, removerlo puede tomar meses. Solo considerar si el equipo ha evaluado este compromiso y completado el proceso de registro.

**Verificación de configuración TLS:**

La configuración TLS del servidor puede verificarse con herramientas como [SSL Labs](https://www.ssllabs.com/ssltest/) (para servidores públicos) o [testssl.sh](https://testssl.sh/) (para redes internas). El objetivo es obtener calificación A o superior en SSL Labs.

**Cipher suites:**

Deshabilitar cipher suites obsoletos o con vulnerabilidades conocidas:
- Deshabilitar: RC4, DES, 3DES, export ciphers, NULL ciphers
- Preferir en TLS 1.2: AES-GCM, ChaCha20-Poly1305 con forward secrecy (ECDHE)
- Con TLS 1.3: la selección de cipher suites es automática por spec — solo se permiten AES-128-GCM-SHA256, AES-256-GCM-SHA384 y ChaCha20-Poly1305-SHA256. No requiere configuración manual.

### 6.2 Autenticación y autorización en el gateway

El gateway es la primera línea de validación de identidad. La autenticación no debe ocurrir en la lógica de negocio de cada servicio — debe resolverse en el gateway antes de que el request llegue a la aplicación.

**Mecanismos por tipo de integración:**

| Mecanismo | Tipo de cliente | Responsabilidad del gateway |
|-----------|----------------|----------------------------|
| **JWT / OAuth2 Bearer** | Usuarios autenticados (web, mobile) | Validar firma del token, expiración y claims mínimos. El gateway rechaza con `401` si el token es inválido o ausente |
| **API Key** | Sistemas de integración (B2B, servicios internos) | Validar la key contra el registro de keys activas. Asociar la key a un cliente para rate limiting y audit trail |
| **mTLS** | Comunicación servicio-a-servicio | Validar el certificado del cliente en el handshake TLS. El gateway actúa como CA intermediaria o valida contra la CA interna |
| **OAuth2 Client Credentials** | Servicios automatizados sin usuario | Validar client_id y client_secret contra el IdP. Emite token de acceso para el flujo machine-to-machine |

**Responsabilidades del gateway vs. la aplicación:**

| Responsabilidad | Gateway | Aplicación |
|-----------------|---------|-----------|
| Validar que el token existe y tiene firma válida | X |  |
| Validar expiración del token | X |  |
| Validar claims de autorización de negocio (permisos, roles) |  | X |
| Retornar `401 Unauthorized` por token ausente o inválido | X |  |
| Retornar `403 Forbidden` por permisos insuficientes |  | X |

> **Principio:** el gateway autentica (¿quién eres?). La aplicación autoriza (¿qué puedes hacer?). Mezclar ambas responsabilidades en la aplicación elimina la capacidad de centralizar y auditar el acceso.

**Gestión de API Keys:**

| Práctica | Descripción |
|----------|-------------|
| Emisión por cliente | Una key por sistema o equipo consumidor — nunca una key compartida entre múltiples clientes |
| Rotación | Las keys deben poder rotarse sin downtime — el cliente usa la nueva key mientras la antigua sigue activa por un período de gracia (máx. 24–48 h) |
| Revocación inmediata | El gateway debe poder revocar una key en tiempo real sin redeploy |
| Alcance (scopes) | Asociar cada key a los scopes que necesita — no emitir keys con acceso total por comodidad |
| Transmisión | Las keys se envían en header, nunca en URL: `X-Api-Key: <key>`. Las URLs se loguean en proxies y sistemas intermedios |

**Validación JWT — configuración mínima en gateway:**

```
Algoritmo recomendado      : RS256 o ES256 — asimétricos: el IdP firma con clave privada, el gateway
                             verifica con clave pública (JWKS). La clave privada nunca sale del IdP.
Algoritmo a evitar         : HS256 en APIs con múltiples consumidores — HS256 es simétrico: el mismo
                             secreto firma y verifica. Compartirlo con consumidores externos les permite
                             también emitir tokens válidos. Solo aceptable en contextos internos donde un
                             único sistema controla emisión y verificación.
Algoritmo a rechazar       : "none" — tokens sin firma que cualquier cliente puede fabricar.
Claims obligatorios        : iss (issuer), exp (expiration), aud (audience). El gateway rechaza tokens
                             con aud que no corresponda a esta API.
JWKS endpoint              : configurar rotación automática de claves públicas desde el IdP — sin
                             hardcodear la clave pública en el gateway.
```

> La guía de diseño de endpoints con autenticación (scopes, roles, claims) está en [rest-conventions.md](rest-conventions.md).

---

## 7. Referencias

- [OpenAPI Specification 3.2.0](https://spec.openapis.org/oas/v3.2.0)
- [Swagger UI — documentación oficial](https://swagger.io/tools/swagger-ui/)
- [Redoc — documentación oficial](https://redocly.com/redoc/)
- [Spectral — OpenAPI linting](https://stoplight.io/open-source/spectral)
- [Prism — mock server](https://stoplight.io/open-source/prism)
- [Schemathesis — property-based contract testing](https://schemathesis.readthedocs.io/)
- [openapi-diff — breaking change detection](https://github.com/OpenAPITools/openapi-diff)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [RFC 6749 — OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 7519 — JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519)
- [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 8594 — Sunset Header](https://www.rfc-editor.org/rfc/rfc8594)
- [RFC 8705 — OAuth 2.0 mTLS](https://www.rfc-editor.org/rfc/rfc8705)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [MDN — CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [SSL Labs — SSL Server Test](https://www.ssllabs.com/ssltest/)
- [IANA — HTTP Header Field Registrations](https://www.iana.org/assignments/http-fields/http-fields.xhtml)
- [IETF draft-ietf-httpapi-ratelimit-headers](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-ratelimit-headers)
