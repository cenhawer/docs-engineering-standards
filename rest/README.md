# REST APIs

Lineamientos para el diseño, documentación y consumo de APIs REST con OpenAPI. Estos lineamientos son independientes de la tecnología — aplican a cualquier plataforma, lenguaje o framework.

---

## Guías disponibles

| Guía | Descripción | Estado |
|------|-------------|--------|
| [Convenciones REST](./rest-conventions.md) | Principios RESTful, tipos de contenido, verbos HTTP, diseño de URLs, headers, versionado, response pattern y OpenAPI-first | `Stable` |
| [Ejemplos y Templates](./rest-examples.md) | Referencia rápida: requests HTTP, responses, headers y snippets OpenAPI | `Stable` |
| [Configuración e Infraestructura](./rest-setup.md) | OpenAPI tooling, CORS, rate limiting, SSL/TLS, contract testing en CI/CD | `Stable` |

---

## Estado de los documentos

| Documento | Versión | Audiencia | Contenido |
|-----------|---------|-----------|-----------|
| `rest-conventions.md` | 1.0 | Dev | Convenciones de diseño, tipos de contenido, verbos, URLs, headers, response pattern, OpenAPI-first |
| `rest-examples.md` | 1.0 | Dev | Templates HTTP de requests, responses, headers y snippets OpenAPI |
| `rest-setup.md` | 1.0 | Infra / DevOps | OpenAPI tooling, CORS, seguridad, rate limiting, contract testing en CI/CD |

---

## Separación de responsabilidades

- **Dev**
  - Define el contrato de la API (OpenAPI-first) antes de escribir código
  - Aplica convenciones de diseño: URLs, verbos, códigos HTTP, tipos de contenido, response pattern
  - Documenta y versiona los headers según su alcance
  - Implementa el response pattern estándar en todos los endpoints

- **Infra / DevOps**
  - Configura el API Gateway (ruteo, rate limiting, autenticación a nivel de red)
  - Gestiona certificados SSL/TLS y terminación HTTPS
  - Configura CORS por ambiente
  - Integra contract testing en el pipeline CI/CD
  - Administra el ciclo de vida de versiones de API en producción

---

## Principios

1. **Contract-first**
   El contrato OpenAPI es la fuente de verdad — se define antes del código, no se genera desde él.

2. **Consistency over preference**
   Verbos, códigos HTTP, headers y response pattern son estándar. No se negocian por proyecto.

3. **Shift-left en calidad**
   El diseño de la API se revisa antes de escribir código. Un contrato mal diseñado cuesta más cuanto más tarde se detecta.

4. **Separación de concerns**
   Diseño de API ≠ configuración de infraestructura ≠ seguridad de red.

5. **Idioma consistente**
   El proyecto define el idioma. Los ejemplos en esta guía están en inglés.

6. **Agnóstico de tecnología**
   Las convenciones REST son principios HTTP/OpenAPI universales. No están atadas a ningún lenguaje, framework ni plataforma.

---

## Changelog

| Fecha | Lineamiento |
|-------|------------|
| 2026-03-31 | [Convenciones REST](./rest-conventions.md) |
| 2026-03-31 | [Ejemplos y Templates](./rest-examples.md) |
| 2026-03-31 | [Configuración e Infraestructura](./rest-setup.md) |
