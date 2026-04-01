# Convenciones REST

> **Audiencia:** equipo de desarrollo. Esta guía no cubre configuración de API Gateway, CORS ni rate limiting a nivel de infraestructura — esas responsabilidades corresponden al equipo de Infra/DevOps ([rest-setup.md](rest-setup.md)).
>
> **Cómo usar esta guía:** para una API nueva, lee las secciones 1 → 2 → 3 → 10 (OpenAPI-first) antes de escribir código. Para consulta puntual, ve directamente a la sección temática. Los ejemplos y templates ejecutables están en [rest-examples.md](rest-examples.md).

**Versión:** 1.0

---

## Contenido

1. [Convenciones del Proyecto](#1-convenciones-del-proyecto)
   - [1.1 Idioma](#11-idioma)
   - [1.2 Separador en URLs](#12-separador-en-urls)
   - [1.3 Nomenclatura de campos JSON](#13-nomenclatura-de-campos-json)
   - [1.4 Fechas y timestamps](#14-fechas-y-timestamps)
2. [Principios RESTful](#2-principios-restful)
3. [Tipos de Contenido y Negociación](#3-tipos-de-contenido-y-negociación)
4. [Verbos HTTP](#4-verbos-http)
   - [4.1 GET — Obtener recursos](#41-get--obtener-recursos)
   - [4.2 POST — Crear recursos](#42-post--crear-recursos)
   - [4.3 PUT — Actualizar recurso completo](#43-put--actualizar-recurso-completo)
   - [4.4 PATCH — Actualizar recurso parcial](#44-patch--actualizar-recurso-parcial)
   - [4.5 DELETE — Eliminar recurso](#45-delete--eliminar-recurso)
   - [4.6 Soft Delete](#46-soft-delete)
   - [4.7 Tabla de códigos HTTP por verbo](#47-tabla-de-códigos-http-por-verbo)
5. [Diseño de URLs](#5-diseño-de-urls)
   - [5.1 Recursos y colecciones](#51-recursos-y-colecciones)
   - [5.2 Separador en paths compuestos](#52-separador-en-paths-compuestos)
   - [5.3 Parámetros de ruta vs parámetros de query](#53-parámetros-de-ruta-vs-parámetros-de-query)
   - [5.4 Relaciones entre recursos](#54-relaciones-entre-recursos)
   - [5.5 REST pragmático — acciones de negocio](#55-rest-pragmático--acciones-de-negocio)
6. [Query Params](#6-query-params)
   - [6.1 Filtrado](#61-filtrado)
   - [6.2 Paginación](#62-paginación)
   - [6.3 Ordenación](#63-ordenación)
7. [Versionado](#7-versionado)
8. [Headers](#8-headers)
   - [8.1 Casing y estándar RFC](#81-casing-y-estándar-rfc)
   - [8.2 Clasificación por alcance](#82-clasificación-por-alcance)
   - [8.3 Headers estándar del sistema](#83-headers-estándar-del-sistema)
   - [8.4 Qué headers son estándar internacional vs custom](#84-qué-headers-son-estándar-internacional-vs-custom)
9. [Response Pattern](#9-response-pattern)
   - [9.1 Respuesta sin datos](#91-respuesta-sin-datos)
   - [9.2 Respuesta con datos](#92-respuesta-con-datos)
   - [9.3 Respuesta paginada](#93-respuesta-paginada)
   - [9.4 Parámetros de paginación](#94-parámetros-de-paginación)
   - [9.5 Manejo de errores — RFC 7807](#95-manejo-de-errores--rfc-7807)
10. [OpenAPI-First](#10-openapi-first)
    - [10.1 Por qué contract-first](#101-por-qué-contract-first)
    - [10.2 Flujo de trabajo](#102-flujo-de-trabajo)
    - [10.3 Estructura del archivo OpenAPI](#103-estructura-del-archivo-openapi)
    - [10.4 Herramientas recomendadas](#104-herramientas-recomendadas)
11. [Anti-patterns](#11-anti-patterns)
12. [Referencias](#12-referencias)

---

## Contexto

Un API mal diseñada genera deuda técnica desde el primer endpoint: contratos inconsistentes, integraciones frágiles y versiones incompatibles que obligan a mantener código muerto. Esta guía establece un criterio único para el diseño de APIs REST en todos los proyectos, con decisiones explícitas y justificadas. Los principios aquí documentados son agnósticos de tecnología — aplican a cualquier lenguaje, framework o plataforma.

> **Guías relacionadas:**
> - [rest-examples.md](rest-examples.md) — ejemplos HTTP y snippets OpenAPI para cada sección de esta guía
> - [rest-setup.md](rest-setup.md) — configuración de infraestructura: OpenAPI tooling, CORS, rate limiting, contract testing en CI/CD

---

## 1. Convenciones del Proyecto

Las convenciones de nomenclatura son decisiones del equipo. Cada decisión se toma una sola vez, se documenta y se aplica de forma consistente en todo el proyecto. No se permiten mezclas dentro de la misma API.

### 1.1 Idioma

| Regla | Valor |
|-------|-------|
| Idioma del proyecto | Decisión del equipo — un solo idioma en todo el proyecto |
| Idioma de los ejemplos en esta guía | **Inglés** |
| Aplica a | Endpoints, campos JSON, nombres de recursos, parámetros, documentación OpenAPI |

> **Idioma:** puede ser español o inglés según el contexto del proyecto y equipo. Lo que no es negociable es la consistencia: un solo idioma en todo. Los ejemplos en esta guía usan inglés exclusivamente.

### 1.2 Separador en URLs

Los paths y los query params de una URL pueden usar dos estilos de separación:

| Estilo | Ejemplo de path | Ejemplo de query param | Usado por |
|--------|----------------|------------------------|-----------|
| `snake_case` | `/product_categories` | `?page_size=20` | GitHub API, Stripe, Twitter API v2 |
| `kebab-case` | `/product-categories` | `?page-size=20` | Google API Design Guide, Azure API Guidelines |

**Regla:** el equipo elige **un solo estilo** y lo aplica a todos los paths y query params del proyecto. No se permiten mezclas (paths kebab con query params snake, por ejemplo).

> **Guía de elección:** snake_case es el estilo más utilizado en REST APIs del mundo (GitHub, Stripe, Twitter). kebab-case es el recomendado por major API guidelines (Google, Azure). Ambos son válidos. Los ejemplos en esta guía usan `snake_case` por ser el más ampliamente adoptado en la práctica.

### 1.3 Nomenclatura de campos JSON

La nomenclatura de los campos en el cuerpo del request y response es independiente de la convención de URL:

| Estilo | Ejemplo | Usado por |
|--------|---------|-----------|
| `snake_case` | `created_at`, `user_id` | GitHub API, Stripe, Python/Ruby ecosystems |
| `camelCase` | `createdAt`, `userId` | Muchas APIs JavaScript-first, Google APIs modernas |

**Regla:** el equipo elige **un solo estilo** y lo aplica a todos los campos JSON del proyecto. Los ejemplos en esta guía usan `snake_case`.

> **Nota:** la decisión de URL (sección 1.2) y la de campos JSON (1.3) son independientes. Un proyecto puede usar `snake_case` en URLs y `camelCase` en JSON, o usar el mismo estilo para ambos. Lo que no se permite es inconsistencia dentro de cada categoría.

### 1.4 Fechas y timestamps

| Regla | Valor |
|-------|-------|
| Zona horaria | **UTC siempre** — todos los timestamps se almacenan y exponen en UTC |
| Formato | ISO 8601 con sufijo `Z` — `2026-03-31T10:30:00Z` |
| Conversión a zona local | Responsabilidad del cliente — la API nunca expone timestamps localizados |
| Fechas sin hora | `YYYY-MM-DD` — `2026-03-31` |

> **UTC como fuente de verdad:** todos los timestamps en requests y responses son UTC. El servidor nunca aplica conversión de zona horaria. La conversión a la zona local del usuario final es responsabilidad exclusiva del cliente. Esto garantiza consistencia en sistemas distribuidos y entornos multi-zona.

---

## 2. Principios RESTful

| Principio | Descripción |
|-----------|-------------|
| Recursos como sustantivos | Los endpoints nombran recursos, no acciones. `/users`, no `/getUsers` |
| Colecciones en plural | Siempre plural: `/users`, `/orders`, `/products` |
| Jerarquía reflejada en la URL | Las relaciones se expresan en el path: `/users/{user_id}/addresses` |
| Stateless | Cada request es independiente — el servidor no guarda estado de sesión |
| Idempotencia | `GET`, `PUT`, `DELETE` son idempotentes. `POST` no es idempotente por defecto (ver sección 4.2). `PATCH` es idempotente si se aplica correctamente |
| Representación uniforme | JSON como formato por defecto. Ver sección 3 para otros tipos de contenido |

---

## 3. Tipos de Contenido y Negociación

El estándar HTTP permite múltiples tipos de contenido. JSON es el tipo de contenido **por defecto** para APIs REST, pero no el único válido.

### 3.1 Tipos de contenido comunes

| Content-Type | Uso |
|-------------|-----|
| `application/json` | **Default para REST APIs.** Datos estructurados en JSON. Usar en la gran mayoría de endpoints |
| `multipart/form-data` | Carga de archivos. Obligatorio cuando el request incluye archivos binarios junto con campos de formulario |
| `application/octet-stream` | Descarga o subida de archivos binarios puros (sin campos adicionales) |
| `application/x-www-form-urlencoded` | Datos de formularios HTML codificados. Poco común en APIs REST modernas |

### 3.2 Negociación de contenido

La negociación de contenido se realiza a través de los headers `Content-Type` y `Accept`:

| Header | Dirección | Propósito |
|--------|-----------|-----------|
| `Content-Type` | Request → Servidor | Indica el tipo de contenido que envía el cliente |
| `Accept` | Request → Servidor | Indica el tipo de contenido que acepta el cliente en la respuesta |

### 3.3 Regla para endpoints con archivos

**Un endpoint que acepta archivos NO puede requerir `application/json` como `Content-Type`.** Cuando el body contiene archivos, el `Content-Type` debe ser `multipart/form-data`. Los campos de texto se incluyen como partes del multipart, no como JSON.

> **Excepción documentada:** si un endpoint acepta tanto JSON como archivos según el contexto, se deben definir dos operaciones separadas o usar `multipart/form-data` con un campo JSON embebido. Esta decisión debe quedar explícita en el contrato OpenAPI.

Ver ejemplos en [rest-examples.md](rest-examples.md#a3--tipos-de-contenido).

---

## 4. Verbos HTTP

### 4.1 GET — Obtener recursos

Se usa para **recuperar información** sin modificar el estado del servidor.

- Seguro y sin efectos secundarios.
- No modifica datos en el servidor.
- `200 OK`: el recurso existe y se retorna correctamente.
- `200 OK`: array vacío cuando la colección no tiene resultados — la consulta fue exitosa, sin data.
- `404 Not Found`: el recurso específico no existe (aplica a consultas por ID, no a listas).

Ver ejemplos en [rest-examples.md](rest-examples.md#a2--requests-por-verbo).

### 4.2 POST — Crear recursos

Se usa para **crear un nuevo recurso** en el servidor o ejecutar **acciones de negocio**.

- El ID del recurso lo asigna el servidor. No se incluye en el body.
- `201 Created`: nuevo recurso creado exitosamente. **Código preferido para creación.**
- `200 OK`: también válido cuando: se trata de un upsert, la acción ya fue completada anteriormente, o el equipo elige explícitamente `200` por simplicidad. El equipo debe ser consistente en su elección.
- `400 Bad Request`: datos malformados, campos requeridos ausentes o falla de validación de formato.
- `409 Conflict`: el recurso ya existe — conflicto de estado en base de datos.
- `422 Unprocessable Entity`: errores de validación de negocio.

**POST e idempotencia:**

Según RFC 7231, `POST` **no es inherentemente idempotente** — cada llamada puede crear un recurso nuevo. Sin embargo, existen dos patrones para prevenir duplicados:

| Patrón | Mecanismo | Cuándo usar |
|--------|-----------|-------------|
| **Prevención natural** | El servidor retorna `409 Conflict` si se viola una restricción de unicidad (ej. email duplicado) | Cuando el recurso tiene un identificador de negocio único verificable |
| **Idempotency-Key** | El cliente envía un UUID en el header `Idempotency-Key`. El servidor almacena la clave y retorna la misma respuesta para requests repetidos con la misma clave | Pagos, transacciones financieras, operaciones críticas donde el duplicado tiene consecuencias graves |

El patrón `Idempotency-Key` es utilizado por Stripe, Adyen y plataformas de pagos. Es un de facto estándar para operaciones críticas que no deben procesarse dos veces. Ambos patrones son válidos; la elección depende del contexto y la criticidad de la operación.

Ver ejemplos en [rest-examples.md](rest-examples.md#a2--requests-por-verbo).

### 4.3 PUT — Actualizar recurso completo

Se usa para **reemplazar completamente** un recurso existente.

- Reemplaza todo el recurso. Si se omite un campo, ese campo se pierde o vuelve a su valor por defecto.
- Es idempotente — llamarlo varias veces produce el mismo resultado.
- `200 OK`: recurso actualizado, se retorna el recurso modificado.
- `204 No Content`: recurso actualizado sin contenido que retornar.
- `400 Bad Request`: datos malformados o validación de formato fallida.
- `404 Not Found`: el recurso a actualizar no existe.
- `422 Unprocessable Entity`: errores de validación de negocio.

Ver ejemplos en [rest-examples.md](rest-examples.md#a2--requests-por-verbo).

### 4.4 PATCH — Actualizar recurso parcial

Se usa para **modificar solo ciertos campos** de un recurso existente.

- Solo actualiza los campos enviados en el body. Los campos ausentes no se modifican.
- Es idempotente si se aplica correctamente.
- `200 OK`: actualización parcial aplicada, se retorna el recurso modificado.
- `204 No Content`: actualización parcial aplicada sin contenido que retornar.
- `400 Bad Request`: el documento PATCH está malformado — JSON inválido, tipos de campo incorrectos, estructura inválida.
- `404 Not Found`: el recurso a modificar no existe.
- `422 Unprocessable Entity`: errores de validación de negocio.

Ver ejemplos en [rest-examples.md](rest-examples.md#a2--requests-por-verbo).

### 4.5 DELETE — Eliminar recurso

Se usa para **borrar un recurso** del servidor.

- Es idempotente — llamarlo varias veces no cambia el resultado final.
- `204 No Content`: recurso eliminado exitosamente. **Código preferido.** No retorna body.
- `200 OK`: recurso eliminado, retornando el recurso eliminado en el body. Válido cuando el consumidor necesita los datos del recurso eliminado (patrones de auditoría o posible reversión).
- `404 Not Found`: el recurso a eliminar no existe.
- `422 Unprocessable Entity`: errores de validación de negocio (ej. el recurso no puede eliminarse por dependencias activas).

La elección entre `204` y `200` debe ser consistente en toda la API. El equipo debe documentar la decisión en el contrato OpenAPI.

Ver ejemplos en [rest-examples.md](rest-examples.md#a2--requests-por-verbo).

### 4.6 Soft Delete

El soft delete es una **decisión de implementación**, no de diseño de API. Existen dos patrones con superficies REST distintas:

| Patrón | Verbo | Comportamiento visible para el consumidor | Cuándo usar |
|--------|-------|-------------------------------------------|-------------|
| **Patrón A — DELETE implícito** | `DELETE /resources/{id}` | El endpoint retorna `204`. El consumidor no sabe si el borrado fue físico o lógico. La implementación es interna. | Cuando el consumidor no necesita conocer el estado del recurso después del borrado. La superficie REST es idéntica a un hard delete |
| **Patrón B — PATCH de estado** | `PATCH /resources/{id}` con `{ "status": "inactive" }` o `{ "is_active": false }` | El consumidor actualiza explícitamente el estado del recurso. La transición de ciclo de vida es visible en el contrato. | Cuando el estado de lifecycle importa al consumidor — por ejemplo, cuando la reactivación es posible y el consumidor debe poder realizarla |

> **Regla:** elegir el patrón según si el ciclo de vida del recurso es un concepto del dominio que el consumidor necesita operar. Si el consumidor nunca interactúa con el estado "eliminado", usar Patrón A. Si el consumidor puede reactivar recursos, usar Patrón B.

Ver ejemplos en [rest-examples.md](rest-examples.md#a4--soft-delete).

### 4.7 Tabla de códigos HTTP por verbo

| Verbo | Códigos esperados |
|-------|-------------------|
| `GET` | `200 OK`, `404 Not Found` |
| `POST` | `201 Created`, `200 OK`, `400 Bad Request`, `409 Conflict`, `422 Unprocessable Entity` |
| `PUT` | `200 OK`, `204 No Content`, `400 Bad Request`, `404 Not Found`, `422 Unprocessable Entity` |
| `PATCH` | `200 OK`, `204 No Content`, `400 Bad Request`, `404 Not Found`, `422 Unprocessable Entity` |
| `DELETE` | `204 No Content`, `200 OK`, `404 Not Found`, `422 Unprocessable Entity` |

**Referencia completa de códigos:**

| Código | Uso |
|--------|-----|
| `200 OK` | Respuesta exitosa con datos |
| `200 OK` | Lista vacía — la consulta fue exitosa, sin resultados |
| `201 Created` | Recurso creado exitosamente |
| `204 No Content` | Operación exitosa sin contenido que retornar |
| `400 Bad Request` | Error en el formato, parámetros o estructura del request |
| `401 Unauthorized` | Falta de autenticación |
| `403 Forbidden` | Autenticado pero sin autorización para el recurso |
| `404 Not Found` | Recurso no encontrado |
| `409 Conflict` | Conflicto de estado — el recurso ya existe |
| `422 Unprocessable Entity` | Error de validación de negocio |
| `429 Too Many Requests` | Rate limit excedido |
| `500 Internal Server Error` | Error interno del servidor |

---

## 5. Diseño de URLs

### 5.1 Recursos y colecciones

**Regla:** los recursos se nombran en plural. El estilo del separador sigue la convención definida en [sección 1.2](#12-separador-en-urls).

| Regla | Correcto (snake_case) | Correcto (kebab-case) | Incorrecto |
|-------|----------------------|----------------------|------------|
| Plural | `/users` | `/users` | `/user` |
| Sin verbos | `/users` | `/users` | `/get_users`, `/get-users` |
| Separador consistente | `/product_categories` | `/product-categories` | `/productCategories` |
| Minúsculas | `/users` | `/users` | `/Users` |

> La columna muestra ambas opciones — la elección corresponde al equipo ([sección 1.2](#12-separador-en-urls)). Los ejemplos en el resto de esta guía usan `snake_case`.

### 5.2 Separador en paths compuestos

Para paths con más de una palabra, aplicar el separador definido en [sección 1.2](#12-separador-en-urls) de forma consistente en todo el proyecto:

```
-- Con snake_case (ejemplos de esta guía)
/api/product_categories
/api/order_items
/api/shipping_addresses
/api/user_profiles

-- Con kebab-case (si el equipo eligió kebab)
/api/product-categories
/api/order-items
/api/shipping-addresses
/api/user-profiles
```

> Si el recurso compuesto representa una relación directa 1:1, es preferible usar la ruta de relación. Ver sección [5.4](#54-relaciones-entre-recursos).

### 5.3 Parámetros de ruta vs parámetros de query

Los parámetros de ruta y de query tienen **semántica fundamentalmente diferente**. Elegir el tipo incorrecto genera URLs mal diseñadas, difíciles de cachear, documentar y evolucionar.

#### Parámetro de ruta

- **Ubicación:** embebido en el path — `/resources/{param}`
- **Semántica:** identifica **cuál** recurso o sub-recurso específico se accede
- **Naturaleza:** obligatorio — la URL es incompleta sin él
- **Cuándo usar:** el valor es parte de la identidad del recurso; sin él no se puede localizar el recurso en la jerarquía

#### Parámetro de query

- **Ubicación:** después del `?` — `/resources?param=value`
- **Semántica:** modifica **cómo** el servidor responde — filtra, ordena, pagina u ofrece opciones sobre el resultado
- **Naturaleza:** opcional (generalmente tienen valor por defecto)
- **Cuándo usar:** el valor filtra una colección, ordena resultados, pagina, o activa una opción que no cambia qué recurso se accede

#### Regla de decisión

> **"¿Es este valor necesario para identificar el recurso?"** → parámetro de ruta.
> **"¿Este valor filtra, ordena, pagina o modifica el resultado?"** → parámetro de query.

#### Ejemplos

| Request | Parámetro | Tipo | Motivo |
|---------|-----------|------|--------|
| `GET /users/{user_id}` | `user_id` | Ruta | Identifica al usuario específico — sin él, no hay recurso |
| `GET /users?status=active` | `status` | Query | Filtra la colección de usuarios |
| `GET /users/{user_id}/orders` | `user_id` | Ruta | Restringe los pedidos al usuario — identifica la jerarquía |
| `GET /users/{user_id}/orders?status=pending` | `user_id` / `status` | Ruta / Query | `user_id` identifica la jerarquía; `status` filtra dentro de ella |
| `GET /orders/{order_id}/items/{item_id}` | ambos | Ruta | Jerarquía de identidad: pedido → ítem |
| `GET /products?min_price=100&max_price=500` | ambos | Query | Rango de filtro — no identifica un producto específico |
| `GET /users?page=2&page_size=20` | ambos | Query | Paginación — modifica qué porción de la colección se retorna |
| `GET /users?sort=name&order=asc` | ambos | Query | Ordenación — no identifica un recurso |

#### Errores comunes

| Incorrecto | Correcto | Por qué |
|------------|----------|---------|
| `GET /users?id=123` | `GET /users/123` | El ID es la identidad del recurso — va en la ruta |
| `GET /users/active` | `GET /users?status=active` | "active" es un estado/filtro, no un identificador |
| `GET /users/page/2` | `GET /users?page=2` | La paginación no identifica un recurso |
| `GET /users/sort/name` | `GET /users?sort=name` | La ordenación es una opción sobre la colección |

#### Casos en el límite

- **Búsqueda libre:** siempre query param — `GET /users?q=john` es un filtro
- **Recurso identificado por año:** si el año ES el identificador único (`/reports/2024` — un reporte por año) → ruta. Si es un filtro sobre una colección (`/reports?year=2024`) → query
- **Flags opcionales:** siempre query param — `?include_deleted=true`, `?expand=profile`

Ver ejemplos en [rest-examples.md](rest-examples.md#a8--query-params-filtrado-paginación-y-ordenación).

### 5.4 Relaciones entre recursos

Las rutas deben reflejar las relaciones de los recursos de manera intuitiva:

| Acción | Método HTTP | Ejemplo |
|--------|-------------|---------|
| Obtener relación | `GET` | `GET /api/orders/{order_id}/products` |
| Asociar recurso | `POST` | `POST /api/orders/{order_id}/products` |
| Remover relación | `DELETE` | `DELETE /api/orders/{order_id}/products/{product_id}` |

> No usar `PUT` o `PATCH` para agregar o eliminar relaciones cuando no se está modificando todo el recurso raíz.

Ver ejemplos en [rest-examples.md](rest-examples.md#a7--relaciones-entre-recursos).

### 5.5 REST pragmático — acciones de negocio

Cuando la operación no es CRUD sino una **acción de negocio** (validar, activar, aprobar, enviar, cancelar, etc.), se modela con `POST` sobre un sub-recurso sustantivo que representa la acción:

```
-- Acción sobre un recurso específico
POST /api/users/{user_id}/validation
POST /api/orders/{order_id}/approval
POST /api/invoices/{invoice_id}/cancellation

-- Acción sobre múltiples recursos o sin ID específico
POST /api/users/validation
POST /api/orders/bulk_approval
```

**Reglas:**
- Usar **sustantivo** para el sub-recurso de acción — no verbo en la URL.
- El body contiene la información necesaria para ejecutar la operación.
- Usar `POST` — estas acciones generalmente no son idempotentes.
- Si la acción es idempotente y consulta el estado de algo, considerar `GET`.

Ver ejemplos en [rest-examples.md](rest-examples.md#a8--rest-pragmático-acciones-de-negocio).

---

## 6. Query Params

### 6.1 Filtrado

Permite obtener registros que cumplan condiciones específicas.

| Regla | Descripción |
|-------|-------------|
| Nombres descriptivos | `min_price`, `max_price`, `status`, `created_from` |
| Múltiples filtros | Combinados con `&` |
| Rangos | Usar pares `_from` / `_to` o `_min` / `_max` |

Ver ejemplos en [rest-examples.md](rest-examples.md#a6--query-params-filtrado-paginación-y-ordenación).

### 6.2 Paginación

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `page` | `int` | `1` | Número de página |
| `page_size` | `int` | `10` | Registros por página |

Ver [sección 9.3](#93-respuesta-paginada) para la estructura de respuesta.

### 6.3 Ordenación

| Parámetro | Valores | Descripción |
|-----------|---------|-------------|
| `sort` | nombre de campo(s) separados por coma | Campo(s) por los que ordenar |
| `order` | `asc` \| `desc` (separados por coma si varios) | Dirección del orden |

Ver ejemplos en [rest-examples.md](rest-examples.md#a6--query-params-filtrado-paginación-y-ordenación).

---

## 7. Versionado

### 7.1 Semantic Versioning

Las APIs siguen [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`

| Tipo | Incrementa | Ejemplo | Cuándo |
|------|------------|---------|--------|
| `MAJOR` | Cambios incompatibles con versiones anteriores | `2.0.0` | Renombrar campos, cambiar estructura del response, eliminar endpoints |
| `MINOR` | Nuevas funcionalidades sin romper compatibilidad | `1.3.0` | Nuevos endpoints, nuevos campos opcionales |
| `PATCH` | Correcciones y mejoras menores | `1.2.1` | Fixes de comportamiento sin cambio de contrato |

### 7.2 Header de versión

La versión se comunica a través del header `X-Api-Version` en cada request y response:

```
X-Api-Version: 1.2.0
```

> **Por qué header y no URL:** versionar en la URL (`/api/v1/users`) contamina el path del recurso y dificulta la evolución. El header desacopla la versión del recurso, siguiendo el principio REST de URI estable para el recurso.
>
> **Cuándo el versionado en URL es válido:** existen escenarios donde el header no es viable y versionar en URL es la opción correcta:
> - **API Gateways heredados** que enrutan tráfico por path y no admiten enrutamiento por header
> - **APIs públicas de consumo masivo** donde la versión en URL facilita la integración directa desde browser, herramientas HTTP o documentación sin configuración de headers
> - **Webhooks y callbacks** donde el consumer no controla los headers del request entrante
> - **Compatibilidad con clientes legacy** que ya integraron con versión en URL y no pueden migrar sin un ciclo de deprecación prolongado
>
> En cualquiera de estos casos, el versionado en URL es la excepción documentada — no el estándar. El `X-Api-Version` header sigue siendo requerido para comunicar la versión exacta del contrato en el response.

---

## 8. Headers

### 8.1 Casing y estándar RFC

**Referencia normativa:** RFC 7230 (HTTP/1.1) y RFC 9110 (HTTP Semantics, 2022) establecen que los nombres de headers HTTP son **case-insensitive** — el servidor y el cliente deben tratarlos sin distinción de mayúsculas y minúsculas.

**Convención de escritura:** la forma canónica reconocida internacionalmente es **Train-Case** (también llamada HTTP-Header-Case): cada palabra comienza con mayúscula, separadas por guiones.

```
Content-Type
X-Request-Id
X-Correlation-Id
X-Channel-Id
```

> No usar `SCREAMING_SNAKE_CASE`, `camelCase` ni todo en minúsculas para nombres de headers. Train-Case es la convención que usan los estándares HTTP, las herramientas de desarrollo (curl, Postman, navegadores) y la documentación técnica de referencia.

**Prefijo `X-`:** RFC 6648 (2012) deprecó el uso del prefijo `X-` para parámetros MIME. Sin embargo, para headers HTTP de aplicación personalizados, `X-` sigue siendo el estándar de facto — es la convención utilizada por Stripe, AWS, Azure y la industria en general. Se mantiene como convención en esta guía.

### 8.2 Clasificación por alcance

Los headers se clasifican en tres niveles según su alcance en el sistema. El patrón de nomenclatura es `X-{Scope}-{Name}`:

| Nivel | `{Scope}` | Ejemplo | Quién lo define | Obligatoriedad |
|-------|-----------|---------|-----------------|----------------|
| **Global del sistema** | Identifica la naturaleza del header (`Request`, `Correlation`) | `X-Request-Id`, `X-Correlation-Id` | Arquitectura / equipo base | Obligatorio en todo request |
| **Transversal** | Identifica el dominio o canal (`Channel`, `End-User`) | `X-Channel-Id`, `X-End-User-Login` | El equipo del dominio o canal | Obligatorio dentro de su dominio |
| **Por API** | Identifica el contexto específico de la API (`Api`) | `X-Api-Version` | El equipo de esa API | Requerido u opcional según la spec |

**Regla:** si se crea un header nuevo, debe clasificarse en uno de estos tres niveles antes de implementarse y debe documentarse en el contrato OpenAPI con su scope, formato y quién lo genera.

### 8.3 Headers estándar del sistema

#### X-Request-Id

- **Scope:** Global del sistema
- **Clasificación:** De facto estándar (usado por AWS, Azure, Stripe)
- **Dirección:** request (generado y enviado por el cliente)
- **Formato:** UUID v4
- **Propósito:** identifica unívocamente cada HTTP request individual. Permite rastrear la solicitud en todos los sistemas que la procesan.
- **Generado por:** el cliente en cada request — nunca reutilizar entre requests.
- **Ejemplo:** `X-Request-Id: 550e8400-e29b-41d4-a716-446655440000`

#### X-Correlation-Id

- **Scope:** Global del sistema
- **Clasificación:** De facto estándar (usado por AWS, Azure, Stripe)
- **Dirección:** request y propagado entre servicios
- **Formato:** UUID v4
- **Propósito:** correlaciona una solicitud con otros servicios o eventos dentro de un flujo de trabajo distribuido. Se propaga sin modificar entre microservicios a lo largo de toda la transacción.
- **Generado por:** el cliente en el origen del flujo. Cada servicio intermedio lo propaga al siguiente sin modificarlo.
- **Ejemplo:** `X-Correlation-Id: 550e8400-e29b-41d4-a716-446655440000`

> **`X-Request-Id` vs `X-Correlation-Id`:** `X-Request-Id` es único por cada HTTP request individual — cambia en cada llamada. `X-Correlation-Id` es el mismo a lo largo de toda una transacción de negocio, aunque involucre múltiples requests encadenados entre servicios.

#### X-Channel-Id

- **Scope:** Transversal
- **Clasificación:** Custom del sistema — específico del negocio
- **Dirección:** request (enviado por el cliente)
- **Formato:** string identificador del canal — definido por el equipo del sistema
- **Propósito:** identifica el canal desde el cual se origina la solicitud. Útil para auditoría, métricas segmentadas por canal y lógica diferenciada por origen.
- **Generado por:** la aplicación cliente (front-end, sistema externo, etc.)
- **Ejemplo:** `X-Channel-Id: CRM` / `X-Channel-Id: MOBILE` / `X-Channel-Id: BACKOFFICE`

> **Valores válidos:** los valores del canal deben estar documentados y ser finitos. No aceptar strings arbitrarios. Definir el catálogo de canales en la documentación del sistema.

#### X-End-User-Login

- **Scope:** Transversal
- **Clasificación:** Custom del sistema — específico del negocio
- **Dirección:** request (enviado por el cliente)
- **Formato:** string — identificador de login del usuario final en la aplicación front-end
- **Propósito:** identifica al usuario final que origina la petición. Distinto del usuario técnico del servicio — permite auditoría a nivel de usuario final en sistemas con autenticación delegada.
- **Generado por:** la aplicación cliente con la sesión activa del usuario
- **Ejemplo:** `X-End-User-Login: user.frontend`

#### X-Api-Version

- **Scope:** Por API
- **Clasificación:** Custom del sistema — convención del proyecto
- **Dirección:** request y response
- **Formato:** SemVer — `MAJOR.MINOR.PATCH`
- **Propósito:** comunica la versión de la API en uso. El cliente la envía en el request; el servidor la confirma en el response. Ver [sección 7](#7-versionado).
- **Ejemplo:** `X-Api-Version: 1.2.0`

### 8.4 Qué headers son estándar internacional vs custom

Es importante distinguir el origen y el nivel de adopción de cada header:

| Categoría | Headers | Estándar |
|-----------|---------|---------|
| **IANA / HTTP Spec** | `Content-Type`, `Accept`, `Authorization`, `Cache-Control`, `ETag`, `If-None-Match`, `Retry-After` | Registrados en IANA. Parte del estándar HTTP. Comportamiento definido por RFC |
| **De facto — industria** | `X-Request-Id` / `Request-Id`, `X-Correlation-Id`, `Idempotency-Key` | No en el registro IANA. Usados ampliamente por Stripe, AWS, Azure. Comportamiento esperado conocido |
| **Custom del sistema** | `X-Channel-Id`, `X-End-User-Login`, `X-Api-Version` | Definidos por el sistema. No reconocidos fuera de este contexto |
| **Custom por API** | Headers específicos de un endpoint o dominio | Definidos por el equipo de la API para un caso de uso concreto |

> Los headers de la categoría "IANA / HTTP Spec" tienen comportamiento definido por RFC — no se deben usar con semántica diferente a la especificada. Los headers custom deben documentarse completamente en el contrato OpenAPI.

Ver ejemplos en [rest-examples.md](rest-examples.md#a5--headers-por-alcance).

---

## 9. Response Pattern

> **Convención del proyecto:** la estructura de respuesta (`results`, `message`, `total`, `has_next`, etc.) es una **convención interna**, no un mandato internacional. El estándar internacional para errores es RFC 7807 (sección 9.5). La estructura de respuestas exitosas debe aplicarse de forma consistente en todos los endpoints del proyecto. La nomenclatura de campos JSON sigue la convención definida en [sección 1.3](#13-nomenclatura-de-campos-json).

### 9.1 Respuesta sin datos

Usado para operaciones que no retornan un recurso: confirmaciones, operaciones de negocio que solo indican éxito o error.

**Formato del proyecto (convención simplificada):**
```json
{
  "message": "User deleted successfully"
}
```

**Cuándo usar:**
- `DELETE` con `200 OK` y confirmación textual (cuando no se usa `204 No Content`)
- Acciones de negocio que solo confirman ejecución (`POST /api/orders/{id}/approval`)

### 9.2 Respuesta con datos

Usado para operaciones que retornan un recurso o estructura de datos:

```json
{
  "message": "User retrieved successfully",
  "results": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

**Cuándo usar:**
- `GET` por ID — retorna el recurso individual
- `POST` con `201 Created` que retorna el recurso creado
- `PUT` / `PATCH` con `200 OK` que retorna el recurso modificado

> **`message`:** siempre presente y con valor. Describe el resultado de la operación en lenguaje natural. Ejemplos: `"User retrieved successfully"`, `"User created successfully"`, `"User updated successfully"`. Nunca se envía vacío.

### 9.3 Respuesta paginada

Usado para endpoints que retornan colecciones con paginación:

```json
{
  "message": "Users retrieved successfully",
  "results": [ ... ],
  "total": 42,
  "page": 1,
  "page_size": 20,
  "has_next": true,
  "has_prev": false
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `results` | `array` | Lista de recursos de la página actual |
| `total` | `int` | Total de registros que cumplen el filtro aplicado |
| `page` | `int` | Número de página actual |
| `page_size` | `int` | Registros por página |
| `has_next` | `bool` | Si existe una página siguiente |
| `has_prev` | `bool` | Si existe una página anterior |

> **Lista vacía:** retornar `200 OK` con `results: []` y `total: 0`. No retornar `404` para colecciones sin resultados.

> **`total` y performance:** para colecciones de alto volumen (tablas de log, historial de auditoría con millones de registros), un `COUNT(*)` puede ser costoso. En esos casos, el equipo puede omitir `total` y retornar únicamente `has_next`. Esta decisión debe documentarse en el contrato OpenAPI y ser consistente dentro de la misma colección — no alternar entre incluir y omitir `total` según la página.

### 9.4 Parámetros de paginación

Los parámetros de paginación se reciben como query params:

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `page` | `int` | `1` | Número de página |
| `page_size` | `int` | `10` | Registros por página |
| `sort` | `string` | `null` | Campo por el que ordenar |
| `order` | `string` | `null` | Dirección: `asc` o `desc` |

### 9.5 Manejo de errores — RFC 7807

**RFC 7807 "Problem Details for HTTP APIs"** es el estándar internacional para respuestas de error en APIs HTTP. Define una estructura JSON con semántica explícita para cada campo:

```json
{
  "type": "https://example.com/errors/validation",
  "title": "Validation Error",
  "status": 400,
  "detail": "The 'email' field is required",
  "instance": "/api/users"
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `type` | URI | Identificador del tipo de error. Puede ser una URL con documentación del error |
| `title` | string | Descripción breve del tipo de error. Estable entre ocurrencias del mismo tipo |
| `status` | int | Código HTTP del error |
| `detail` | string | Descripción específica de esta ocurrencia del error |
| `instance` | URI | URI que identifica la ocurrencia específica del error |

Para múltiples errores de validación, RFC 7807 puede extenderse con campos adicionales:

```json
{
  "type": "https://example.com/errors/validation",
  "title": "One or more validation errors occurred",
  "status": 400,
  "errors": {
    "email": ["The email field is required"],
    "name": ["The name must be at most 100 characters"]
  }
}
```

**Convención simplificada del proyecto:**

El proyecto puede optar por el formato simplificado `{"message": "..."}` en lugar de RFC 7807 completo. RFC 7807 es **recomendado** para APIs con consumidores externos o errores de validación complejos. La elección debe ser consistente en **todos** los endpoints — no mezclar formatos dentro de la misma API.

> **Regla de seguridad:** nunca exponer stack traces, queries SQL, nombres de tablas ni detalles internos del sistema en los mensajes de error. Los logs internos son el lugar para ese detalle — no el response.

Ver ejemplos en [rest-examples.md](rest-examples.md#a6--responses-error).

---

## 10. OpenAPI-First

### 10.1 Por qué contract-first

> **Regla:** el contrato OpenAPI se define **antes** de escribir código de implementación.

Desarrollar código antes del contrato genera problemas estructurales:

| Problema | Consecuencia |
|----------|-------------|
| Contrato generado desde el código | Refleja detalles de implementación en lugar del diseño deseado |
| Cambios de contrato tardíos | Los consumidores ya están integrados con un contrato incorrecto |
| Falta de alineación entre equipos | Frontend, backend e integraciones descubren discrepancias en QA o producción |
| Sin validación temprana de diseño | Los errores de naming, estructura y semántica se detectan cuando ya tienen costo |

Un contrato OpenAPI definido antes del código:
- Permite que frontend y backend trabajen en paralelo contra el mismo contrato desde el día uno
- Hace que los errores de diseño sean baratos — modificar YAML cuesta segundos; modificar código integrado cuesta horas
- Sirve como documentación viva que no puede quedar desactualizada respecto a la implementación
- Permite generar clientes, mocks y tests de contrato automáticamente

### 10.2 Flujo de trabajo

```
1. Definir el contrato OpenAPI (YAML/JSON)
        ↓
2. Revisar el contrato con el equipo (frontend, backend, QA) — PR del contrato
        ↓
3. Levantar mock server desde el contrato (frontend avanza sin esperar backend)
        ↓
4. Implementar el backend contra el contrato
        ↓
5. Validar la implementación contra el contrato en CI/CD
        ↓
6. Publicar la UI Swagger desde el contrato validado
```

**Paso 1 — Definir el contrato:**
- Crear `openapi.yaml` en la raíz del repositorio o en `/docs/api/`.
- Incluir todos los endpoints con paths, métodos, parámetros, request bodies y responses.
- Usar el response pattern estándar ([sección 9](#9-response-pattern)) en todos los schemas de respuesta.
- Documentar todos los headers requeridos ([sección 8](#8-headers)).

**Paso 2 — Revisión del contrato:**
- El contrato debe revisarse como un PR antes de comenzar la implementación.
- Criterios mínimos: verbos correctos, códigos HTTP apropiados, nombres de campos en `snake_case`, headers documentados, ejemplos incluidos.

**Paso 3 — Mock server:**
- Usar herramientas como Prism, Stoplight o Mockoon para levantar un servidor mock desde el contrato.
- El frontend consume el mock mientras el backend implementa.

**Paso 4 — Implementación:**
- El código implementa el contrato, no al revés.
- Si durante la implementación se detecta que el contrato tiene un error de diseño, se actualiza el YAML, se re-revisa en PR y luego se actualiza el código. No al revés.

**Paso 5 — Validación en CI:**
- El pipeline verifica que la implementación cumple el contrato. Ver [rest-setup.md](rest-setup.md) para la configuración.

### 10.3 Estructura del archivo OpenAPI

Ver el snippet completo en [rest-examples.md](rest-examples.md#a10--openapi-snippet-completo--recurso-users).

**Reglas del archivo OpenAPI:**

| Regla | Descripción |
|-------|-------------|
| Schemas reutilizables | Definir en `components/schemas`. No repetir la misma estructura en múltiples paths |
| Headers documentados | Todos los headers requeridos en `components/headers` |
| Response pattern | Todo schema de respuesta debe envolver el response pattern estándar |
| Ejemplos incluidos | Cada schema debe tener al menos un `example` |
| Errores documentados | Documentar todos los códigos de error posibles por endpoint |
| Responses reutilizables | Los responses de error comunes en `components/responses` |

### 10.4 Herramientas recomendadas

| Herramienta | Propósito |
|-------------|-----------|
| [Stoplight Studio](https://stoplight.io/studio) | Editor visual de contratos OpenAPI — recomendado para diseño inicial |
| [Swagger Editor](https://editor.swagger.io/) | Editor en línea de contratos OpenAPI |
| [Prism](https://stoplight.io/open-source/prism) | Mock server desde contrato OpenAPI |
| [Mockoon](https://mockoon.com/) | Mock server local con UI |
| [Redoc](https://redocly.com/redoc/) | Generación de documentación HTML desde OpenAPI |
| [Spectral](https://stoplight.io/open-source/spectral) | Linting de contratos OpenAPI — detecta violaciones de estilo y estructura |
| [Dredd](https://dredd.org/) | Validación de implementación contra contrato OpenAPI |

---

## 11. Anti-patterns

| Anti-pattern | Síntoma visible | Consecuencia | Solución |
|--------------|----------------|--------------|----------|
| Verbo en la URL (`/getUsers`, `/createUser`) | La URL describe una acción, no un recurso | Viola principios REST, dificulta evolución | Usar sustantivos en plural: `/users` |
| Código HTTP incorrecto (`200` en errores, `200` en creación) | El cliente no puede distinguir éxito de fallo ni tipo de operación | Integraciones rotas, logs engañosos | Ver [tabla 4.7](#47-tabla-de-códigos-http-por-verbo) |
| Contrato generado desde código | El YAML refleja la implementación, no el diseño | Acoplamiento entre implementación y contrato | Definir OpenAPI antes del código — [sección 10](#10-openapi-first) |
| Sin versionado de API | Cambios rompen consumidores sin aviso | Rollbacks forzados, incidentes en producción | Header `X-Api-Version` + SemVer — [sección 7](#7-versionado) |
| Headers sin estructura (`requestid`, `myCustomHeader`) | Difícil de entender, inconsistente entre APIs | Nadie sabe qué headers esperar ni de qué tipo son | Usar formato `X-{Scope}-{Name}` — [sección 8.2](#82-clasificación-por-alcance) |
| Response sin estructura fija | Cada endpoint retorna un formato diferente | Los consumidores deben manejar múltiples formatos | Usar el response pattern estándar — [sección 9](#9-response-pattern) |
| `404` para colección vacía | Se retorna error cuando la lista no tiene resultados | El cliente no puede distinguir "recurso no existe" de "lista vacía" | `200 OK` con `results: []` y `total: 0` |
| `PUT` para agregar/quitar relaciones | Semántica incorrecta del verbo | Efectos secundarios no esperados en el recurso raíz | Usar `POST`/`DELETE` sobre el sub-recurso de relación |
| Stack traces en respuestas de error | Detalles internos expuestos al cliente | Vulnerabilidad de seguridad — information disclosure | Solo mensajes genéricos en producción. Detalle en logs internos |
| Mezcla de estilos en URLs (`/product_categories?page-size=20`) | snake_case en paths pero kebab en query params, o viceversa | Inconsistencia visible en cada request | Definir un solo estilo y aplicarlo a paths y query params — [sección 1.2](#12-separador-en-urls) |
| Mezcla de idiomas en la API | Algunos campos en inglés, otros en español | Inconsistencia que confunde a los consumidores | Un solo idioma en todo el proyecto — [sección 1.1](#11-idioma) |
| `application/json` en endpoint que recibe archivos | Error en tiempo de ejecución al enviar `multipart/form-data` | El servidor rechaza el request | Usar `multipart/form-data` para endpoints con archivos — [sección 3.3](#33-regla-para-endpoints-con-archivos) |

---

## 12. Referencias

- [OpenAPI Specification 3.1.0](https://spec.openapis.org/oas/v3.1.0)
- [Semantic Versioning 2.0.0](https://semver.org/)
- [HTTP Status Codes — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [RFC 7231 — HTTP/1.1 Semantics and Content](https://www.rfc-editor.org/rfc/rfc7231)
- [RFC 9110 — HTTP Semantics (2022)](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 7807 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807)
- [RFC 6648 — Deprecating the X- Prefix](https://www.rfc-editor.org/rfc/rfc6648)
- [RFC 3986 — Uniform Resource Identifier](https://www.rfc-editor.org/rfc/rfc3986)
- [RFC 8594 — The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Microsoft Azure REST API Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md)
- [Stoplight — API Design Best Practices](https://stoplight.io/api-design-guide/basics)
