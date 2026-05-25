---
name: add-endpoint
description: creates a new endpoint in the current microservice
disable-model-invocation: true

---
You are adding a new REST endpoint to the current microservice.
Endpoint specification: $ARGUMENTS
(format: "{HTTP_METHOD} {path}" — e.g. "POST /orders" or "GET /orders/{id}/items")

Follow ALL rules in CLAUDE.md — especially the API Contract section.

## Steps

### 1. Analyse the endpoint
From $ARGUMENTS, determine:
- HTTP method and path
- Resource entity (infer from path)
- Operation type: create / read / read-list / update / delete / action
- Expected request body (if POST/PUT/PATCH)
- Expected response body
- Required path/query parameters

### 2. Use-case port (if it doesn't exist)
In `domain/port/in/`, create or update:
- Use-case interface: `{Verb}{Entity}UseCase.java`
- Command record (for write operations): `{Verb}{Entity}Command.java`
  with Jakarta Bean Validation annotations

### 3. Application service method
In `application/usecase/{Entity}Service.java`:
- Implement the new use-case interface method
- Add `@Transactional` (writes) or `@Transactional(readOnly = true)` (reads)
- Call the repository port
- Invalidate or update Redis cache as appropriate
- Record a metric increment via the metrics class

### 4. DTO classes
In `adapter/in/web/`:
- Request record: `{Entity}{Operation}Request.java` (if body required)
  Add `@NotNull`, `@NotBlank`, `@Size`, `@Valid` as appropriate
- Response record: `{Entity}{Operation}Response.java`
  (reuse existing response record if shape is identical)

### 5. MapStruct mapping
In the web layer mapper `{Entity}WebMapper.java`:
- Add mapping method: command → domain, domain → response
- Use `@Mapping(target = "...", source = "...")` for field differences

### 6. Controller method
In `adapter/in/web/{Entity}Controller.java`, add:

```java
@{HttpMethod}("{path}")
@Operation(summary = "...", description = "...")
@ApiResponse(responseCode = "{code}", description = "...")
public ResponseEntity<{Response}> {methodName}(
    @PathVariable(required = false) UUID id,   // if path param
    @RequestBody @Valid {Request} request,      // if body
    @PageableDefault Pageable pageable          // if list
) {
    // delegate to use case port
    // return ResponseEntity.{status}(response)
}
```

HTTP status rules:
- POST (creates resource)  → 201 Created
- POST (action/command)    → 200 OK
- GET (single)             → 200 OK
- GET (list/page)          → 200 OK
- PUT/PATCH                → 200 OK
- DELETE                   → 204 No Content

### 7. Error handling
In `adapter/in/web/GlobalExceptionHandler.java` (create if missing):
- Map domain exceptions → correct HTTP status + RFC 7807 body
- Always include `traceId` from current OTel span:
  `Span.current().getSpanContext().getTraceId()`

### 8. OpenAPI spec update
Update `src/main/resources/openapi/{service}-api.yaml`:
- Add the new path + operation
- Add request/response schemas
- Tag with the entity name

### 9. Unit test
In `adapter/in/web/{Entity}ControllerTest.java`:
- Add `@WebMvcTest` test method for the new endpoint
- Test: happy path, validation error (422), not found (404)
- Mock the use-case port with Mockito

### When done
Print a checklist of all modified/created files with their layer.

