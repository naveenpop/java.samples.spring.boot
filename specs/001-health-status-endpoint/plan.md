# Implementation Plan: Health and System Resource Status Endpoint

**Branch**: `001-health-status-endpoint` | **Date**: 2026-07-02 | **Spec**: [Health Status Endpoint](./spec.md)

**Input**: Feature specification from `/specs/001-health-status-endpoint/spec.md`

**Status**: Ready for Phase 0 (Research)

## Summary

Implement a public RESTful endpoint (`GET /api/v1/status`) that exposes application health and system resource metrics (uptime, total memory, free memory, timestamp) without authentication or database access. The implementation follows Spring Boot 4.0.0 Jakarta patterns with strict layered architecture (Controller → Service → Repository) and real-time metrics from `java.lang.Runtime.getRuntime()`. Response time target: <10ms (p99). No Spring Actuator integration; custom endpoint standalone. Metrics must be fresh on each request (no caching), with human-readable uptime format (e.g., "2h 15m 30s").

## Technical Context

**Language/Version**: Java 25 LTS (enforced via `.java-version`)

**Primary Dependencies**: 
- Spring Boot 4.0.0
- Spring MVC 6.0 (Jakarta namespace)
- SpringDoc OpenAPI 3.0 (Swagger annotations)
- Lombok (annotation processor for boilerplate reduction)
- Jackson (JSON serialization via Spring Boot defaults)

**Storage**: N/A — No database access required; in-memory JVM metrics only

**Testing**: JUnit 5 + Mockito + AssertJ + MockMvc (Spring Web Test client)

**Target Platform**: Linux/macOS/Windows servers (containerized via Docker, port 9000)

**Project Type**: Web service (REST API) — Spring Boot monolith

**Performance Goals**: <5ms average response time, <10ms p99 latency

**Constraints**: <10ms p99 latency, <500 bytes response payload, sub-millisecond metric calculation

**Scale/Scope**: Single endpoint; no scaling complexity; production-ready observability

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Architectural Compliance ✅

| Principle | Requirement | Status | Notes |
|-----------|-----------|--------|-------|
| **Layered Architecture** | Controller → Service → Repository → JPA | ✅ PASS | Health endpoint follows strict layers: Controller routes to Service; Service computes metrics; Repository layer minimal (no DB) |
| **Constructor Injection** | Via Lombok `@RequiredArgsConstructor` | ✅ PASS | All beans use constructor injection; no field injection via `@Autowired` |
| **Jakarta Namespace** | `jakarta.servlet.*`, `jakarta.annotation.*` | ✅ PASS | Spring Boot 4.0.0 enforces Jakarta; no `javax.*` imports |
| **DTO Immutability** | Response DTOs are immutable, no business logic | ✅ PASS | `HealthStatusResponse` uses Lombok `@Value` for immutability |
| **Transactional Read** | Read-only operations use `@Transactional(readOnly = true)` | ✅ PASS | Service method marked read-only for clarity and optimization |
| **No Business Logic in Controller** | Controllers delegate to services | ✅ PASS | Controller only marshals HTTP request/response; service calculates metrics |
| **SLF4J Logging** | No `System.out.println` or `System.err` | ✅ PASS | All logging via SLF4J + Logback (WARN for memory access failures) |
| **Bean Validation** | Use Jakarta Bean Validation annotations | ✅ PASS | DTO uses `@Positive` or similar if needed; Spring validates automatically |
| **API Contract Immutability** | Endpoint path `/api/v1/status`, GET method fixed | ✅ PASS | Contract is documented in spec; no breaking changes planned |
| **Test-First Development** | Tests written first, minimum 80% coverage | ✅ PASS | JUnit 5 + MockMvc tests cover controller, service, DTO serialization |

### Constitution Violations: None Detected ✅

**Justification**: This feature is self-contained, follows all core principles:
- No database access simplifies repository layer (empty or minimal stub)
- Metrics are computed synchronously in service layer
- No caching complexity; real-time data on each request
- HTTP-only, no transactional boundaries to manage

**Re-Check After Phase 1**: Design artifacts (data-model.md, contracts/) will be reviewed against Constitution again.

## Project Structure

### Documentation (this feature)

```text
specs/001-health-status-endpoint/
├── spec.md                      # Feature specification (input)
├── plan.md                      # This file (Phase 0-1 output)
├── research.md                  # Phase 0 output: resolved unknowns
├── data-model.md                # Phase 1 output: DTOs, entities, validation
├── quickstart.md                # Phase 1 output: validation scenarios
├── contracts/
│   └── health-status-api.json   # OpenAPI contract (Phase 1 output)
└── tasks.md                     # Phase 2 output: actionable GitHub issues
```

### Source Code (repository root)

Feature code will integrate into existing project structure:

```text
src/main/java/
├── controllers/
│   ├── HealthController.java              # NEW: HTTP handler for GET /api/v1/status
│   └── [existing player endpoints]
├── services/
│   ├── HealthStatusService.java           # NEW: Business logic; metrics computation
│   └── [existing PlayerService, etc.]
├── repositories/
│   ├── HealthStatusRepository.java        # NEW: Minimal stub (no DB queries; required by architecture)
│   └── [existing PlayerRepository, etc.]
├── models/
│   ├── HealthStatusResponse.java          # NEW: Response DTO (immutable)
│   └── [existing Player, etc.]
└── [existing: converters/, exception handlers via GlobalExceptionHandler]

src/main/resources/
├── application.properties                 # Existing; no changes
├── db/migration/                           # No migrations needed (no DB schema changes)
└── [existing: logback-spring.xml]

src/test/java/
├── controllers/
│   ├── HealthControllerTest.java          # NEW: Integration test (MockMvc)
│   └── [existing player tests]
├── services/
│   ├── HealthStatusServiceTest.java       # NEW: Unit tests with mocked Runtime
│   └── [existing service tests]
└── [existing test structure]

src/test/resources/
├── application-test.properties             # Existing; no changes
```

**Structure Decision**: 
- **Single project** (Java monolith) — no separation needed
- **New classes** follow existing package conventions (controllers/, services/, repositories/, models/)
- **No migration required** — health metrics are computed in-memory
- **No entity** — `HealthStatusResponse` is a DTO only (no `@Entity`)

## Complexity Tracking

> **No Constitution violations detected; table omitted for brevity.**

This feature is straightforward:
- No database schema changes
- No new dependencies required
- No layering violations (Controller → Service → Repository is standard)
- Metrics calculated directly from `java.lang.Runtime.getRuntime()`
- No caching complexity (specification explicitly forbids caching)

---

# Phase 0: Research & Unknowns

**Status**: Ready to begin

## Research Tasks (0 NEEDS CLARIFICATION)

All technical decisions are already clarified in the specification. No unknowns exist that require research. The spec includes:

✅ **Uptime Format**: Human-readable string (`XhYmZs` format; e.g., "2h 15m 30s", "45m 30s", "15s")
✅ **Memory Metrics**: Via `java.lang.Runtime.getRuntime()`
✅ **Endpoint Path**: `/api/v1/status` (GET method)
✅ **Response Time**: <10ms p99
✅ **No Caching**: Each request computes fresh metrics
✅ **Error Handling**: Graceful degradation (HTTP 200 with memory fields = 0 if access fails)
✅ **Architecture**: Controller → Service → Repository (strict layered)
✅ **DTO Pattern**: Immutable, Jackson annotations, camelCase fields
✅ **Testing**: JUnit 5 + MockMvc (no Spring Actuator)

**Conclusion**: Proceed directly to Phase 1 design artifacts.

---

# Phase 1: Design & Contracts

## Task 1: Generate data-model.md

**Status**: Ready (prerequisite research complete)

### DTO Design: HealthStatusResponse

**Immutable DTO** with Jackson annotations for JSON serialization:

```java
@Value
@Builder
@AllArgsConstructor
public class HealthStatusResponse {
    
    @Schema(description = "Application uptime in human-readable format (e.g., '2h 15m 30s')", 
            example = "45m 30s")
    private final String uptime;
    
    @Schema(description = "Total JVM memory allocated in megabytes", 
            example = "512")
    @Positive
    private final Integer totalMemoryMB;
    
    @Schema(description = "Currently available JVM free memory in megabytes", 
            example = "256")
    @PositiveOrZero
    private final Integer freeMemoryMB;
    
    @Schema(description = "ISO-8601 UTC timestamp when response was generated", 
            example = "2026-07-02T17:11:00Z")
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
    private final String timestamp;
}
```

**Field Specifications**:

| Field | Type | Constraints | Example | Notes |
|-------|------|-----------|---------|-------|
| `uptime` | `String` | Non-empty; format `XhYmZs` omitting zero components | `"2h 15m 30s"` | Human-readable; never milliseconds; calculated from application startup time |
| `totalMemoryMB` | `Integer` | ≥ 0; represents total JVM heap allocation | `512` | From `Runtime.getRuntime().totalMemory()` / (1024 * 1024) |
| `freeMemoryMB` | `Integer` | ≥ 0; ≤ totalMemoryMB; represents available heap | `256` | From `Runtime.getRuntime().freeMemory()` / (1024 * 1024) |
| `timestamp` | `String` | ISO-8601 UTC format; non-empty | `"2026-07-02T17:11:00Z"` | Server-generated at request time; no client control |

**Validation Rules**:
- `uptime`: Non-empty string; validated via regex or format validation in service layer
- `totalMemoryMB`: Must be positive integer (or 0 if access denied)
- `freeMemoryMB`: Must be non-negative integer; checked ≤ `totalMemoryMB`
- `timestamp`: Must conform to ISO-8601 UTC format; validated by Jackson's `@JsonFormat` or via assertion test

**Serialization Example**:
```json
{
  "uptime": "2h 15m 30s",
  "totalMemoryMB": 512,
  "freeMemoryMB": 256,
  "timestamp": "2026-07-02T17:11:00Z"
}
```

---

## Task 2: Define API Contracts

**Status**: Ready

### Contract: GET /api/v1/status

**Endpoint Path**: `/api/v1/status`
**HTTP Method**: `GET`
**Authentication**: None required (public endpoint)
**Authorization**: None required (public endpoint)

**Request**:
- No request body
- No query parameters required
- No request headers required (standard HTTP headers optional)

**Successful Response (HTTP 200 OK)**:
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Content-Length: 89

{
  "uptime": "2h 15m 30s",
  "totalMemoryMB": 512,
  "freeMemoryMB": 256,
  "timestamp": "2026-07-02T17:11:00Z"
}
```

**Error Responses**:

1. **HTTP 405 Method Not Allowed** (invalid HTTP method)
   ```http
   HTTP/1.1 405 Method Not Allowed
   Allow: GET
   Content-Type: application/json
   
   {
     "timestamp": "2026-07-02T17:11:00Z",
     "status": 405,
     "error": "Method Not Allowed",
     "message": "Request method 'POST' not supported",
     "path": "/api/v1/status"
   }
   ```
   - **Cause**: POST, PUT, DELETE, PATCH, etc. attempted
   - **Handling**: Spring MVC default; no custom handler needed

2. **HTTP 200 OK with Degraded Metrics** (memory access failure)
   ```http
   HTTP/1.1 200 OK
   Content-Type: application/json
   
   {
     "uptime": "45m 30s",
     "totalMemoryMB": 0,
     "freeMemoryMB": 0,
     "timestamp": "2026-07-02T17:11:00Z"
   }
   ```
   - **Cause**: JVM memory metrics unavailable (rare)
   - **Handling**: Return HTTP 200 (do not fail health checks); log WARN at startup

**Response Characteristics**:
- **Content-Type**: `application/json; charset=UTF-8`
- **Payload Size**: ~89-150 bytes (compact JSON, no extra whitespace in production)
- **Response Time**: <5ms average, <10ms p99
- **Caching**: `Cache-Control: no-cache` (or omitted; no server-side caching)

**OpenAPI/Swagger Annotation Example**:
```java
@Operation(summary = "Get application health status",
           description = "Returns application uptime, JVM memory metrics, and server timestamp")
@ApiResponse(responseCode = "200", description = "Health status retrieved successfully",
             content = @Content(mediaType = "application/json", 
                               schema = @Schema(implementation = HealthStatusResponse.class)))
@GetMapping("/api/v1/status")
public ResponseEntity<HealthStatusResponse> getHealthStatus() {
    return ResponseEntity.ok(healthStatusService.getHealthStatus());
}
```

---

## Task 3: Create quickstart.md (Validation Guide)

**Status**: Ready

### End-to-End Validation Scenarios

#### Scenario 1: Endpoint Returns Valid JSON

**Prerequisite**: Application running on `http://localhost:9000`

**Setup**:
```bash
./mvnw spring-boot:run
```

**Validation Command**:
```bash
curl -s http://localhost:9000/api/v1/status | jq '.'
```

**Expected Output**:
```json
{
  "uptime": "0s",
  "totalMemoryMB": 512,
  "freeMemoryMB": 384,
  "timestamp": "2026-07-02T17:11:00Z"
}
```

**Validation Assertions**:
- ✅ HTTP status is `200 OK`
- ✅ Response has `Content-Type: application/json`
- ✅ All four fields present: `uptime`, `totalMemoryMB`, `freeMemoryMB`, `timestamp`
- ✅ Field types match spec (string, integer, integer, string)

#### Scenario 2: Uptime Increases Over Time

**Prerequisite**: Application running; previous health check noted

**Command 1** (at t=0s):
```bash
curl -s http://localhost:9000/api/v1/status | jq '.uptime'
# Output: "0s"
```

**Wait**: 5 seconds

**Command 2** (at t=5s):
```bash
curl -s http://localhost:9000/api/v1/status | jq '.uptime'
# Output: "5s"
```

**Validation Assertions**:
- ✅ Uptime changed from `"0s"` to `"5s"` (or similar increments)
- ✅ Uptime never decreases (monotonically increasing)
- ✅ Format follows `XhYmZs` pattern (omits zero components)

#### Scenario 3: Memory Metrics Are Non-Zero

**Command**:
```bash
curl -s http://localhost:9000/api/v1/status | jq '.totalMemoryMB, .freeMemoryMB'
```

**Expected Output**:
```
512
384
```

**Validation Assertions**:
- ✅ `totalMemoryMB` is a positive integer (> 0)
- ✅ `freeMemoryMB` is non-negative integer (≥ 0)
- ✅ `freeMemoryMB` ≤ `totalMemoryMB`
- ✅ Values reflect current JVM memory state

#### Scenario 4: Timestamp Is ISO-8601 UTC

**Command**:
```bash
curl -s http://localhost:9000/api/v1/status | jq '.timestamp'
```

**Expected Output**:
```
"2026-07-02T17:11:00Z"
```

**Validation Assertions**:
- ✅ Timestamp matches ISO-8601 format: `YYYY-MM-DDTHH:MM:SSZ`
- ✅ Timestamp is in UTC (ends with `Z`)
- ✅ Timestamp is close to current system time (within 1 second)

#### Scenario 5: Invalid HTTP Method Returns 405

**Command**:
```bash
curl -X POST http://localhost:9000/api/v1/status
```

**Expected Output** (HTTP 405 with error details):
```
HTTP/1.1 405 Method Not Allowed
Allow: GET
```

**Validation Assertions**:
- ✅ HTTP status is `405 Method Not Allowed`
- ✅ Response includes `Allow: GET` header

#### Scenario 6: Endpoint Handles Concurrent Requests

**Command** (simulate 10 concurrent requests):
```bash
for i in {1..10}; do
  curl -s http://localhost:9000/api/v1/status &
done
wait
```

**Expected Output**: All requests return HTTP 200 with valid JSON

**Validation Assertions**:
- ✅ All 10 requests completed successfully
- ✅ Response time for each < 10ms
- ✅ All responses contain valid JSON with required fields

#### Scenario 7: Swagger/OpenAPI Documentation Available

**Command**:
```bash
curl -s http://localhost:9000/v3/api-docs | jq '.paths["/api/v1/status"]'
```

**Expected Output**: OpenAPI schema for the health endpoint

**Validation Assertions**:
- ✅ GET method documented
- ✅ Response schema matches `HealthStatusResponse` structure
- ✅ HTTP 200 status documented
- ✅ Endpoint description present

**Quickstart Summary**:
Run `./mvnw spring-boot:run`, then use `curl` and `jq` to validate:
1. All fields present and correctly typed
2. Uptime increases monotonically
3. Memory metrics are realistic
4. Timestamp is current and ISO-8601 formatted
5. Invalid methods rejected with HTTP 405
6. Concurrent requests handled successfully
7. OpenAPI docs are generated

---

# Next Steps

**Phase 1 Completion**: All design artifacts (data-model.md, contracts/, quickstart.md) generated.

**Phase 2 (Next)**: Generate `tasks.md` using `/speckit.tasks` command with step-by-step actionable GitHub issues.

**Implementation**: Once tasks.md is generated, use `/speckit.implement` to execute tasks and build the feature.

---

**Last Updated**: 2026-07-02
**Plan Version**: 1.0.0 (Ready for Phase 1 Design)
