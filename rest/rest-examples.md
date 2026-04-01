# Ejemplos y Templates — REST

> Templates de referencia para [Convenciones REST](rest-conventions.md).
> Todos los ejemplos están en inglés. El equipo mapea los nombres al idioma definido para el proyecto.
> Para el contexto y decisiones detrás de cada ejemplo, consultar la guía principal.

**Versión:** 1.0

---

## Contenido

- [A.1 — Referencia rápida: verbos y códigos HTTP](#a1--referencia-rápida-verbos-y-códigos-http)
- [A.2 — Requests por verbo](#a2--requests-por-verbo)
- [A.3 — Tipos de contenido](#a3--tipos-de-contenido)
- [A.4 — Soft delete](#a4--soft-delete)
- [A.5 — Responses: éxito](#a5--responses-éxito)
- [A.6 — Responses: error](#a6--responses-error)
- [A.7 — Headers por alcance](#a7--headers-por-alcance)
- [A.8 — Query params: filtrado, paginación y ordenación](#a8--query-params-filtrado-paginación-y-ordenación)
- [A.9 — Relaciones entre recursos](#a9--relaciones-entre-recursos)
- [A.10 — REST pragmático: acciones de negocio](#a10--rest-pragmático-acciones-de-negocio)
- [A.11 — OpenAPI snippet completo — recurso Users](#a11--openapi-snippet-completo--recurso-users)

---

## A.1 — Referencia rápida: verbos y códigos HTTP

| Verbo | Acción | Código éxito | Códigos error comunes |
|-------|--------|-------------|----------------------|
| `GET` colección | Listar recursos | `200 OK` | `400` |
| `GET` por ID | Obtener recurso | `200 OK` | `404` |
| `POST` | Crear recurso | `201 Created` / `200 OK` | `400`, `409`, `422` |
| `PUT` | Reemplazar recurso | `200 OK` / `204 No Content` | `400`, `404`, `422` |
| `PATCH` | Actualizar parcial | `200 OK` / `204 No Content` | `400`, `404`, `422` |
| `DELETE` | Eliminar recurso | `204 No Content` / `200 OK` | `404`, `422` |
| `POST` (acción) | Acción de negocio | `200 OK` | `400`, `404`, `422` |

---

## A.2 — Requests por verbo

### GET — Listar colección con filtros y paginación

```
GET /api/users?status=active&page=1&page_size=20&sort=created_at&order=desc
Accept: application/json
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
X-Correlation-Id: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
X-Channel-Id: WEB
X-Api-Version: 1.0.0
```

### GET — Obtener recurso por ID

```
GET /api/users/550e8400-e29b-41d4-a716-446655440000
Accept: application/json
X-Request-Id: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
X-Correlation-Id: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
X-Channel-Id: WEB
X-Api-Version: 1.0.0
```

### POST — Crear recurso (201 Created — preferido)

```
POST /api/users
Content-Type: application/json
X-Request-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Correlation-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Channel-Id: CRM
X-End-User-Login: admin.user
X-Api-Version: 1.0.0

{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "role": "viewer"
}
```

Response:
```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/550e8400-e29b-41d4-a716-446655440000

{
  "message": "",
  "results": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "role": "viewer",
    "created_at": "2026-03-31T10:30:00Z"
  }
}
```

### POST — Crear recurso con Idempotency-Key

```
POST /api/payments
Content-Type: application/json
Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890
X-Request-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Correlation-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Channel-Id: WEB
X-Api-Version: 1.0.0

{
  "amount": 1500,
  "currency": "USD",
  "description": "Order #42"
}
```

> Si el servidor recibe una segunda request con el mismo `Idempotency-Key`, retorna la misma respuesta que la primera vez sin procesar el pago nuevamente.

### PUT — Reemplazar recurso completo

```
PUT /api/users/550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json
X-Request-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3301
X-Correlation-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3301
X-Channel-Id: BACKOFFICE
X-End-User-Login: ops.admin
X-Api-Version: 1.0.0

{
  "name": "John Doe Updated",
  "email": "john.updated@example.com",
  "role": "admin"
}
```

### PATCH — Actualizar campos específicos

```
PATCH /api/users/550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json
X-Request-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3302
X-Correlation-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3302
X-Channel-Id: WEB
X-Api-Version: 1.0.0

{
  "role": "admin"
}
```

### DELETE — Eliminar recurso (204 No Content — preferido)

```
DELETE /api/users/550e8400-e29b-41d4-a716-446655440000
X-Request-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3303
X-Correlation-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3303
X-Channel-Id: BACKOFFICE
X-End-User-Login: ops.admin
X-Api-Version: 1.0.0
```

Response:
```
HTTP/1.1 204 No Content
```

### DELETE — Eliminar recurso retornando el recurso eliminado (200 OK)

```
DELETE /api/users/550e8400-e29b-41d4-a716-446655440000
X-Request-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3304
X-Correlation-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3304
X-Channel-Id: BACKOFFICE
X-End-User-Login: ops.admin
X-Api-Version: 1.0.0
```

Response:
```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "User deleted successfully",
  "results": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "deleted_at": "2026-03-31T14:00:00Z"
  }
}
```

---

## A.3 — Tipos de contenido

### Subida de archivo (multipart/form-data)

```
POST /api/documents
Content-Type: multipart/form-data; boundary=----FormBoundary7MA4YWxkTrZu0gW
X-Request-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Correlation-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Channel-Id: WEB
X-Api-Version: 1.0.0

------FormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="contract.pdf"
Content-Type: application/pdf

[binary content]
------FormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="description"

Annual contract 2026
------FormBoundary7MA4YWxkTrZu0gW--
```

### Descarga de archivo binario (application/octet-stream)

```
GET /api/documents/550e8400-e29b-41d4-a716-446655440000/download
Accept: application/octet-stream
X-Request-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae8
X-Correlation-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae8
X-Api-Version: 1.0.0
```

Response:
```
HTTP/1.1 200 OK
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="contract.pdf"
Content-Length: 204800

[binary content]
```

### Negociación de contenido — cliente indica preferencia

```
GET /api/reports/550e8400-e29b-41d4-a716-446655440000
Accept: application/json
X-Api-Version: 1.0.0
```

```
GET /api/reports/550e8400-e29b-41d4-a716-446655440000
Accept: application/octet-stream
X-Api-Version: 1.0.0
```

> El servidor retorna `406 Not Acceptable` si no puede servir el tipo de contenido solicitado en el `Accept`.

### OpenAPI — endpoint con archivo (multipart/form-data)

```yaml
/api/documents:
  post:
    summary: Upload a document
    requestBody:
      required: true
      content:
        multipart/form-data:
          schema:
            type: object
            required: [file]
            properties:
              file:
                type: string
                format: binary
              description:
                type: string
    responses:
      '201':
        description: Document uploaded
```

---

## A.4 — Soft delete

### Patrón A — DELETE implícito (la implementación es interna)

```
DELETE /api/users/550e8400-e29b-41d4-a716-446655440000
X-Request-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3303
X-Correlation-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3303
X-End-User-Login: ops.admin
X-Api-Version: 1.0.0
```

Response:
```
HTTP/1.1 204 No Content
```

> El consumidor no sabe si el borrado fue físico o lógico. La superficie REST es idéntica a un hard delete.

### Patrón B — PATCH de estado (transición de ciclo de vida explícita)

```
PATCH /api/users/550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json
X-Request-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3305
X-Correlation-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3305
X-End-User-Login: ops.admin
X-Api-Version: 1.0.0

{
  "status": "inactive"
}
```

Response:
```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "",
  "results": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "status": "inactive",
    "updated_at": "2026-03-31T14:00:00Z"
  }
}
```

Reactivación (solo posible con Patrón B):
```
PATCH /api/users/550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json
X-Request-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3306
X-Correlation-Id: 3f2504e0-4f89-11d3-9a0c-0305e82c3306
X-Api-Version: 1.0.0

{
  "status": "active"
}
```

---

## A.5 — Responses: éxito

### Respuesta sin datos (ej. DELETE con 200, acción de negocio)

```json
{
  "message": "User deleted successfully"
}
```

### Respuesta con datos — recurso individual (ej. GET por ID, POST, PUT, PATCH)

```json
{
  "message": "",
  "results": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "role": "viewer",
    "created_at": "2026-03-31T10:30:00Z",
    "updated_at": "2026-03-31T10:30:00Z"
  }
}
```

### Respuesta paginada — colección (ej. GET lista)

```json
{
  "message": "",
  "results": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "John Doe",
      "email": "john.doe@example.com",
      "role": "viewer"
    },
    {
      "id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
      "name": "Jane Smith",
      "email": "jane.smith@example.com",
      "role": "admin"
    }
  ],
  "total": 42,
  "page": 1,
  "page_size": 20,
  "has_next": true,
  "has_prev": false
}
```

### Lista vacía — colección sin resultados

```json
{
  "message": "",
  "results": [],
  "total": 0,
  "page": 1,
  "page_size": 20,
  "has_next": false,
  "has_prev": false
}
```

> Código HTTP: `200 OK`. No retornar `404` para listas sin resultados.

---

## A.6 — Responses: error

### Formato simplificado del proyecto

#### 400 Bad Request — parámetros inválidos

```json
{
  "message": "The field 'email' is required"
}
```

#### 401 Unauthorized

```json
{
  "message": "Authentication is required to access this resource"
}
```

#### 403 Forbidden

```json
{
  "message": "You do not have permission to perform this action"
}
```

#### 404 Not Found

```json
{
  "message": "User with id '550e8400-e29b-41d4-a716-446655440000' not found"
}
```

#### 409 Conflict — recurso ya existe

```json
{
  "message": "A user with email 'john.doe@example.com' already exists"
}
```

#### 422 Unprocessable Entity — validación de negocio

```json
{
  "message": "The user account is inactive and cannot be assigned the 'admin' role"
}
```

#### 500 Internal Server Error

```json
{
  "message": "An internal error occurred. Please try again later"
}
```

---

### Formato RFC 7807 — Problem Details

#### 400 Bad Request — campo requerido ausente (RFC 7807)

```json
{
  "type": "https://example.com/errors/validation",
  "title": "Validation Error",
  "status": 400,
  "detail": "The 'email' field is required",
  "instance": "/api/users"
}
```

#### 400 Bad Request — múltiples errores de validación (RFC 7807 extendido)

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

#### 404 Not Found (RFC 7807)

```json
{
  "type": "https://example.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with id '550e8400-e29b-41d4-a716-446655440000' not found",
  "instance": "/api/users/550e8400-e29b-41d4-a716-446655440000"
}
```

#### 409 Conflict (RFC 7807)

```json
{
  "type": "https://example.com/errors/conflict",
  "title": "Resource Already Exists",
  "status": 409,
  "detail": "A user with email 'john.doe@example.com' already exists",
  "instance": "/api/users"
}
```

---

## A.7 — Headers por alcance

### Headers globales del sistema (presentes en todo request)

```
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
X-Correlation-Id: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
```

> Ambos en formato UUID v4. `X-Request-Id` es único por request. `X-Correlation-Id` se propaga sin cambiar a lo largo de todo un flujo de negocio.

### Headers transversales (por dominio o canal)

```
X-Channel-Id: CRM
X-End-User-Login: user.frontend
```

> `X-Channel-Id`: valores posibles definidos por el sistema (ej. `WEB`, `MOBILE`, `CRM`, `BACKOFFICE`).
> `X-End-User-Login`: login del usuario final en la aplicación cliente. Distinto del usuario técnico del servicio.

### Headers por API

```
X-Api-Version: 1.2.0
```

### Request completo con todos los headers

```
GET /api/users/550e8400-e29b-41d4-a716-446655440000
Accept: application/json
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
X-Correlation-Id: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
X-Channel-Id: WEB
X-End-User-Login: john.doe
X-Api-Version: 1.0.0
```

---

## A.8 — Query params: filtrado, paginación y ordenación

### Filtrado

```
-- Filtro simple
GET /api/products?category=electronics

-- Filtros combinados
GET /api/products?category=electronics&status=active

-- Rango de valores
GET /api/products?min_price=100&max_price=5000

-- Rango de fechas
GET /api/orders?created_from=2026-01-01&created_to=2026-12-31
```

### Paginación

```
-- Página 1, 20 registros por página
GET /api/users?page=1&page_size=20

-- Página 3, 10 registros por página (default)
GET /api/users?page=3
```

### Ordenación

```
-- Un campo
GET /api/products?sort=price&order=asc

-- Múltiples campos
GET /api/products?sort=price,created_at&order=asc,desc
```

### Combinado — filtros + paginación + ordenación

```
GET /api/products?category=electronics&status=active&min_price=100&page=2&page_size=20&sort=price&order=asc
```

---

## A.9 — Relaciones entre recursos

```
-- Obtener productos de un pedido
GET /api/orders/550e8400-e29b-41d4-a716-446655440000/products

-- Asociar un producto a un pedido
POST /api/orders/550e8400-e29b-41d4-a716-446655440000/products
Content-Type: application/json

{
  "product_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "quantity": 2
}

-- Remover un producto de un pedido
DELETE /api/orders/550e8400-e29b-41d4-a716-446655440000/products/6ba7b810-9dad-11d1-80b4-00c04fd430c8

-- Obtener categoría de un producto
GET /api/products/550e8400-e29b-41d4-a716-446655440000/category
```

---

## A.10 — REST pragmático: acciones de negocio

```
-- Validar un usuario específico
POST /api/users/550e8400-e29b-41d4-a716-446655440000/validation
Content-Type: application/json

{
  "validation_type": "identity"
}

-- Aprobar un pedido
POST /api/orders/6ba7b810-9dad-11d1-80b4-00c04fd430c8/approval
Content-Type: application/json

{
  "approved_by": "ops.admin",
  "comments": "All conditions met"
}

-- Cancelar una factura
POST /api/invoices/3f2504e0-4f89-11d3-9a0c-0305e82c3301/cancellation
Content-Type: application/json

{
  "reason": "Duplicate invoice"
}

-- Validación en lote (sin ID específico)
POST /api/users/validation
Content-Type: application/json

{
  "user_ids": [
    "550e8400-e29b-41d4-a716-446655440000",
    "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
  ]
}
```

Response de acción de negocio exitosa:

```json
{
  "message": "User validation completed successfully"
}
```

---

## A.11 — OpenAPI snippet completo — recurso Users

```yaml
openapi: 3.1.0
info:
  title: Users API
  version: 1.0.0
  description: API for user management

servers:
  - url: https://api.example.com
    description: Production
  - url: https://api.staging.example.com
    description: Staging

paths:
  /api/users:
    get:
      summary: List users
      operationId: listUsers
      parameters:
        - $ref: '#/components/parameters/XRequestId'
        - $ref: '#/components/parameters/XCorrelationId'
        - $ref: '#/components/parameters/XChannelId'
        - $ref: '#/components/parameters/XApiVersion'
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/PageSizeParam'
        - $ref: '#/components/parameters/SortParam'
        - $ref: '#/components/parameters/OrderParam'
        - name: status
          in: query
          schema:
            type: string
            enum: [active, inactive]
      responses:
        '200':
          description: Paginated list of users
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserPaginatedResponse'
              example:
                message: ""
                results:
                  - id: "550e8400-e29b-41d4-a716-446655440000"
                    name: "John Doe"
                    email: "john.doe@example.com"
                    role: "viewer"
                total: 42
                page: 1
                page_size: 20
                has_next: true
                has_prev: false

    post:
      summary: Create user
      operationId: createUser
      parameters:
        - $ref: '#/components/parameters/XRequestId'
        - $ref: '#/components/parameters/XCorrelationId'
        - $ref: '#/components/parameters/XChannelId'
        - $ref: '#/components/parameters/XEndUserLogin'
        - $ref: '#/components/parameters/XApiVersion'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserRequest'
            example:
              name: "John Doe"
              email: "john.doe@example.com"
              role: "viewer"
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          $ref: '#/components/responses/Conflict'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'

  /api/users/{user_id}:
    parameters:
      - name: user_id
        in: path
        required: true
        schema:
          type: string
          format: uuid

    get:
      summary: Get user by ID
      operationId: getUserById
      parameters:
        - $ref: '#/components/parameters/XRequestId'
        - $ref: '#/components/parameters/XCorrelationId'
        - $ref: '#/components/parameters/XApiVersion'
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '404':
          $ref: '#/components/responses/NotFound'

    put:
      summary: Replace user completely
      operationId: replaceUser
      parameters:
        - $ref: '#/components/parameters/XRequestId'
        - $ref: '#/components/parameters/XCorrelationId'
        - $ref: '#/components/parameters/XEndUserLogin'
        - $ref: '#/components/parameters/XApiVersion'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserRequest'
      responses:
        '200':
          description: User replaced
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'

    patch:
      summary: Partially update user
      operationId: updateUser
      parameters:
        - $ref: '#/components/parameters/XRequestId'
        - $ref: '#/components/parameters/XCorrelationId'
        - $ref: '#/components/parameters/XEndUserLogin'
        - $ref: '#/components/parameters/XApiVersion'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserPatchRequest'
      responses:
        '200':
          description: User updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'

    delete:
      summary: Delete user
      operationId: deleteUser
      parameters:
        - $ref: '#/components/parameters/XRequestId'
        - $ref: '#/components/parameters/XCorrelationId'
        - $ref: '#/components/parameters/XEndUserLogin'
        - $ref: '#/components/parameters/XApiVersion'
      responses:
        '204':
          description: User deleted successfully
        '404':
          $ref: '#/components/responses/NotFound'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'

  /api/users/{user_id}/validation:
    post:
      summary: Validate user identity
      operationId: validateUser
      parameters:
        - name: user_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - $ref: '#/components/parameters/XRequestId'
        - $ref: '#/components/parameters/XCorrelationId'
        - $ref: '#/components/parameters/XApiVersion'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                validation_type:
                  type: string
                  enum: [identity, email, phone]
      responses:
        '200':
          description: Validation completed
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ResultDto'
              example:
                message: "User validation completed successfully"
        '404':
          $ref: '#/components/responses/NotFound'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        email:
          type: string
          format: email
        role:
          type: string
          enum: [viewer, editor, admin]
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

    UserRequest:
      type: object
      required: [name, email, role]
      properties:
        name:
          type: string
        email:
          type: string
          format: email
        role:
          type: string
          enum: [viewer, editor, admin]

    UserPatchRequest:
      type: object
      properties:
        name:
          type: string
        email:
          type: string
          format: email
        role:
          type: string
          enum: [viewer, editor, admin]

    ResultDto:
      type: object
      properties:
        message:
          type: string

    UserResponse:
      allOf:
        - $ref: '#/components/schemas/ResultDto'
        - type: object
          properties:
            results:
              $ref: '#/components/schemas/User'

    UserPaginatedResponse:
      type: object
      properties:
        message:
          type: string
        results:
          type: array
          items:
            $ref: '#/components/schemas/User'
        total:
          type: integer
        page:
          type: integer
        page_size:
          type: integer
        has_next:
          type: boolean
        has_prev:
          type: boolean

  parameters:
    XRequestId:
      name: X-Request-Id
      in: header
      required: true
      schema:
        type: string
        format: uuid
      description: Unique identifier for the request (UUID v4)
    XCorrelationId:
      name: X-Correlation-Id
      in: header
      required: true
      schema:
        type: string
        format: uuid
      description: Correlation identifier across distributed services (UUID v4)
    XChannelId:
      name: X-Channel-Id
      in: header
      required: true
      schema:
        type: string
        enum: [WEB, MOBILE, CRM, BACKOFFICE]
      description: Channel identifier for the request origin
    XEndUserLogin:
      name: X-End-User-Login
      in: header
      schema:
        type: string
      description: Login of the end user in the frontend application
    XApiVersion:
      name: X-Api-Version
      in: header
      required: true
      schema:
        type: string
        example: "1.0.0"
      description: API version in SemVer format
    PageParam:
      name: page
      in: query
      schema:
        type: integer
        default: 1
        minimum: 1
    PageSizeParam:
      name: page_size
      in: query
      schema:
        type: integer
        default: 10
        minimum: 1
        maximum: 100
    SortParam:
      name: sort
      in: query
      schema:
        type: string
    OrderParam:
      name: order
      in: query
      schema:
        type: string
        enum: [asc, desc]

  responses:
    BadRequest:
      description: Bad request — malformed parameters or missing required fields
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ResultDto'
          example:
            message: "The field 'email' is required"
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ResultDto'
          example:
            message: "User with id '550e8400-e29b-41d4-a716-446655440000' not found"
    Conflict:
      description: Conflict — resource already exists
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ResultDto'
          example:
            message: "A user with email 'john.doe@example.com' already exists"
    UnprocessableEntity:
      description: Business validation error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ResultDto'
          example:
            message: "The user account is inactive and cannot be modified"
```
