---
name: api-design
description: REST API design patterns for Polaris Spring Boot services. Covers OpenAPI-first workflow, SLA contracts, RFC 7807 error responses, tracing headers, URL naming, status codes, pagination, filtering, versioning, and rate limiting.
---

# API Design Patterns

Conventions and best practices for designing consistent, production-grade REST APIs in Polaris
Spring Boot 4 services. All new APIs MUST follow the OpenAPI-first workflow below.

## When to Activate

- Designing new API endpoints or resources
- Reviewing existing API contracts for compliance
- Adding pagination, filtering, or sorting to an endpoint
- Implementing error handling or tracing headers
- Planning API versioning strategy
- Onboarding a new Spring Boot service

---

## OpenAPI-First Workflow (MANDATORY for All Polaris Services)

The OpenAPI spec is the single source of truth for every API contract. Models and interface
skeletons are generated from the spec — never handwritten.

### Full Flow

```
1. Write/update spec in src/main/resources/openapi/<service-name>.openapi.yaml
2. ./gradlew openApiGenerateHelloApi   (or ./gradlew compileJava — wired as dependency)
3. Generated interface: com.waveum.polaris.<service-name>.api.<Tag>Api
4. Generated models:    com.waveum.polaris.<service-name>.api.model.<Model>
5. Controller implements <Tag>Api — @Valid on @RequestBody is still required
6. Tests verify response shape against spec
```

### `gradle/openApiGenerator.gradle` Plugin Configuration

```kotlin
// Applied via: apply(from = "gradle/openApiGenerator.gradle")
plugins {
    id("org.openapi.generator") version "7.10.0"
}

// Custom task name — keeps generator task names meaningful when a service has multiple specs
val openApiGenerateHelloApi by tasks.registering(org.openapitools.generator.gradle.plugin.tasks.GenerateTask::class) {
    generatorName.set("spring")
    inputSpec.set("$projectDir/src/main/resources/openapi/<service-name>.openapi.yaml")
    outputDir.set("$buildDir/generated/src/main/java")
    apiPackage.set("com.waveum.polaris.<service-name>.api")
    modelPackage.set("com.waveum.polaris.<service-name>.api.model")
    configOptions.set(mapOf(
        "interfaceOnly"         to "true",
        "useSpringBoot3"        to "true",
        "useTags"               to "true",
        "delegatePattern"       to "false",
        "serializableModel"     to "true",
        "skipDefaultInterface"  to "true",
        "beanValidations"       to "true",
        "useOptional"           to "false"
    ))
}

sourceSets.main {
    java.srcDirs("$buildDir/generated/src/main/java")
}

// Always regenerate before compile
tasks.compileJava { dependsOn(openApiGenerateHelloApi) }
```

### Spec Excerpt — `<service-name>.openapi.yaml`

```yaml
openapi: "3.1.0"
info:
  title: Polaris Order API
  version: "1.0.0"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    CreateOrderRequest:
      type: object
      required: [customerId, hcpcsCode, quantity]
      properties:
        customerId:
          type: string
          format: uuid
        hcpcsCode:
          type: string
          pattern: "^[A-Z][0-9]{4}$"   # Bean Validation @Pattern generated from this
          example: "A6197"
        quantity:
          type: integer
          minimum: 1
          maximum: 999

    OrderResponse:
      type: object
      properties:
        orderId:
          type: string
          format: uuid
        status:
          $ref: "#/components/schemas/OrderStatus"
        createdAt:
          type: string
          format: date-time

paths:
  /api/v1/orders:
    post:
      tags: [Orders]
      summary: Create an order
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateOrderRequest"
      responses:
        "201":
          description: Order created
          headers:
            Location:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/OrderResponse"
        "400":
          $ref: "#/components/responses/ValidationError"
        "401":
          $ref: "#/components/responses/Unauthorized"
```

### Controller Implements Generated Interface

```java
// CORRECT: implements generated interface — no @RequestMapping on controller
@RestController
@RequiredArgsConstructor
public class OrderController implements OrdersApi {

    private final OrderService orderService;

    @Override
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest req) {
        // @Valid is STILL required — triggers runtime Bean Validation on generated model
        OrderResponse response = orderService.create(req);
        URI location = URI.create("/api/v1/orders/" + response.getOrderId());
        return ResponseEntity.created(location).body(response);
    }

    @Override
    public ResponseEntity<OrderResponse> getOrder(UUID orderId) {
        return ResponseEntity.ok(orderService.findById(orderId));
    }
}
```

---

## SLA Contract

Every Polaris synchronous API endpoint MUST meet these thresholds in production.

| Operation                          | P99 Target | Alert at P95 |
|------------------------------------|------------|--------------|
| All synchronous REST APIs           | < 100ms    | > 80ms       |
| DB reads (indexed single row)       | < 10ms     | > 8ms        |
| DB writes (single row)             | < 20ms     | > 15ms       |
| External HTTP calls (circuit broken)| < 500ms    | > 400ms      |
| Cache reads (Redis)                 | < 2ms      | > 1.5ms      |

Endpoints that consistently exceed P99 targets must be profiled and fixed before merge to main.
Use `@Timed(percentiles = {0.5, 0.95, 0.99})` on service methods and alert via Prometheus.

---

## Tracing Headers

Every Polaris API request and response MUST carry tracing headers for distributed debugging.

### Request Headers (inbound)

| Header          | Required | Description                                         |
|----------------|----------|-----------------------------------------------------|
| `X-Request-ID`  | Yes      | Caller-supplied or gateway-assigned UUID per request |
| `Authorization` | Yes      | Bearer JWT                                          |

### Response Headers (outbound)

| Header          | Description                                         |
|----------------|-----------------------------------------------------|
| `X-Request-ID`  | Echo back the inbound `X-Request-ID`                |
| `Location`      | URI of created resource (201 responses)             |
| `Retry-After`   | Seconds to wait on 429 / 503 responses              |

`X-Request-ID` is injected and echoed by `MdcLoggingFilter` and stored in MDC as `requestId`
for correlation across all log lines in a request.

---

## RFC 7807 ProblemDetail Error Responses

All Polaris error responses use Spring Boot's built-in `ProblemDetail` (RFC 7807).
Content-Type: `application/problem+json`.

### Error Response Shape

```json
{
  "type": "about:blank",
  "title": "Validation Failed",
  "status": 400,
  "detail": "One or more fields failed validation.",
  "instance": "/api/v1/orders",
  "violations": [
    { "field": "hcpcsCode", "reason": "must match \"^[A-Z][0-9]{4}$\"" },
    { "field": "quantity",  "reason": "must be greater than or equal to 1" }
  ]
}
```

### `@RestControllerAdvice` for Centralized Handling

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex,
                                          HttpServletRequest req) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        problem.setInstance(URI.create(req.getRequestURI()));
        problem.setProperty("violations", ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> Map.of("field", fe.getField(), "reason", fe.getDefaultMessage()))
            .toList());
        return problem;
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        problem.setDetail(ex.getMessage());
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex, HttpServletRequest req) {
        log.error("unhandled_exception path={}", req.getRequestURI(), ex);
        return ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

Enable `application/problem+json` content type for all error responses:

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

---

## Resource Design

### URL Structure

```
# Resources are nouns, plural, lowercase, kebab-case
GET    /api/v1/orders
GET    /api/v1/orders/{orderId}
POST   /api/v1/orders
PUT    /api/v1/orders/{orderId}
PATCH  /api/v1/orders/{orderId}
DELETE /api/v1/orders/{orderId}

# Sub-resources for clear ownership relationships
GET    /api/v1/orders/{orderId}/line-items
POST   /api/v1/orders/{orderId}/line-items

# Actions that don't map to CRUD — use nouns where possible, verbs sparingly
POST   /api/v1/orders/{orderId}/submission
POST   /api/v1/auth/token/refresh
```

### Naming Rules

```
# CORRECT
/api/v1/order-submissions          # kebab-case for multi-word resources
/api/v1/orders?status=RECEIVED     # query params for filtering
/api/v1/orders/{orderId}/documents # nested for ownership

# WRONG
/api/v1/getOrders                  # verb in URL
/api/v1/order                      # singular (use plural)
/api/v1/order_submissions          # snake_case in URLs
/api/v1/orders/123/getDocuments    # verb in nested resource
```

---

## HTTP Methods and Status Codes

### Method Semantics

| Method | Idempotent | Safe | Use For                            |
|--------|------------|------|------------------------------------|
| GET    | Yes        | Yes  | Retrieve resources                 |
| POST   | No         | No   | Create resources, trigger actions  |
| PUT    | Yes        | No   | Full replacement of a resource     |
| PATCH  | No*        | No   | Partial update of a resource       |
| DELETE | Yes        | No   | Remove a resource                  |

### Status Code Reference

```
# Success
200 OK             — GET, PUT, PATCH (with response body)
201 Created        — POST (include Location header)
204 No Content     — DELETE (no response body)

# Client Errors
400 Bad Request           — Bean Validation failure, malformed JSON
401 Unauthorized          — Missing or invalid JWT
403 Forbidden             — Authenticated but not authorized
404 Not Found             — Resource doesn't exist
409 Conflict              — Duplicate entry, state conflict
422 Unprocessable Entity  — Semantically invalid (valid JSON, bad business data)
429 Too Many Requests     — Rate limit exceeded (include Retry-After header)

# Server Errors
500 Internal Server Error — Unexpected failure (never expose details)
502 Bad Gateway           — Upstream service failed
503 Service Unavailable   — Temporary overload (include Retry-After header)
```

---

## Pagination

All collection endpoints MUST return `Page<T>` with `Pageable` — never bare `List<T>`.

### Spring Boot `Page<T>` Pattern

```java
// Controller — Pageable parameter resolved from query params automatically
@GetMapping("/orders")
ResponseEntity<Page<OrderSummary>> listOrders(
        @RequestParam(required = false) OrderStatus status,
        Pageable pageable) {
    return ResponseEntity.ok(orderService.findAll(status, pageable));
}
```

Query parameters: `?page=0&size=20&sort=createdAt,desc`

### Page Response Shape

```json
{
  "content": [
    { "orderId": "abc-123", "status": "RECEIVED" }
  ],
  "page": {
    "size": 20,
    "number": 0,
    "totalElements": 142,
    "totalPages": 8
  }
}
```

### Pagination Strategy by Use Case

| Use Case                          | Pagination Type  |
|-----------------------------------|-----------------|
| Admin dashboards, small datasets  | Offset (Page<T>) |
| Infinite scroll, large datasets   | Cursor-based     |
| Search results                    | Offset           |
| Audit logs, event feeds           | Cursor-based     |

Cursor-based — encode as opaque Base64 string, never expose raw DB ID:

```
GET /api/v1/orders?cursor=eyJpZCI6MTIzfQ&size=20
```

---

## Filtering and Sorting

```
# Simple equality filter
GET /api/v1/orders?status=RECEIVED&customerId=abc-123

# Date range (ISO-8601 instants)
GET /api/v1/orders?createdAfter=2025-01-01T00:00:00Z

# Sorting (Spring Pageable convention)
GET /api/v1/orders?sort=createdAt,desc&sort=status,asc

# Multiple values — comma-separated or repeated param
GET /api/v1/orders?status=RECEIVED,L1_VALIDATING
```

Never pass filter values as path segments — use query parameters.

---

## Versioning Strategy

Polaris uses URL path versioning as the primary strategy.

```
/api/v1/orders    — current stable
/api/v2/orders    — next version (when breaking change required)
```

Non-breaking changes DO NOT require a new version:
- Adding new optional fields to response
- Adding new optional query parameters
- Adding new endpoints

Breaking changes REQUIRE a new version:
- Removing or renaming fields
- Changing field types
- Changing URL structure

Deprecation timeline:
1. Announce deprecation via `Deprecation` response header
2. Add `Sunset: <date>` header (minimum 6-month notice for external APIs)
3. Return `410 Gone` after sunset date

---

## Rate Limiting

Per API key (extracted from JWT `sub` claim). Resilience4j `RateLimiter` enforced at service layer.

### Rate Limit Tiers

| Tier          | Limit     | Window   | Use Case            |
|---------------|-----------|----------|---------------------|
| Unauthenticated | 30/min  | Per IP   | Health check, docs  |
| Authenticated   | 100/min | Per user | Standard API access |
| Service-to-service | 1000/min | Per service key | Internal calls |

### Rate Limit Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1740000000

# When exceeded
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/problem+json
{
  "type": "about:blank",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "Rate limit exceeded. Retry after 60 seconds."
}
```

---

## Authentication and Authorization

### Bearer Token

```
GET /api/v1/orders
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
```

All Polaris APIs require JWT Bearer authentication except:
- `GET /actuator/health`
- `GET /actuator/info`

### Authorization at Service Layer

```java
// CORRECT: @PreAuthorize on service method
@PreAuthorize("@orderSecurity.canRead(#orderId, authentication)")
public OrderResponse findById(UUID orderId) { ... }

// WRONG: authorization in controller
@GetMapping("/orders/{id}")
ResponseEntity<OrderResponse> getOrder(@PathVariable UUID id) {
    if (!isOwner(id)) return ResponseEntity.status(403).build();  // BLOCKED — move to service
}
```

---

## API Design Checklist

Before shipping a new endpoint:

- [ ] Spec written/updated in `<service-name>.openapi.yaml` BEFORE writing controller code
- [ ] `./gradlew generateOpenApiDocs` passes cleanly
- [ ] Controller implements generated `*Api` interface
- [ ] `@Valid` present on `@RequestBody`
- [ ] URL is plural, kebab-case nouns (no verbs)
- [ ] Correct HTTP method and status codes
- [ ] Collection endpoint uses `Page<T>` with `Pageable`
- [ ] Error responses use `ProblemDetail` (RFC 7807)
- [ ] `X-Request-ID` echoed in response via `MdcLoggingFilter`
- [ ] `Location` header on 201 responses
- [ ] `@SecurityRequirement(name = "bearerAuth")` in spec for protected operations
- [ ] Rate limiting configured for public-facing endpoints
- [ ] P99 latency verified < 100ms in staging before merge

