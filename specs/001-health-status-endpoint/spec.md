# Feature Specification: Health and System Resource Status Endpoint

**Feature Branch**: `001-health-status-endpoint`

**Created**: 2026-07-02

**Status**: Clarified (Ready for Planning)

**Input**: User description: "Expose application health and system resource metrics via a public REST endpoint"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Monitor Application Health (Priority: P1)

A DevOps engineer or monitoring system needs to check if the Spring Boot application is running and responsive. They can query the `/api/v1/status` endpoint to verify the application is alive and obtain real-time system health information without requiring authentication.

**Why this priority**: Essential for production monitoring, load balancers, and automated health checks. Every deployed service needs a public health endpoint.

**Independent Test**: Can be fully tested by making a GET request to `/api/v1/status` and validating that the response contains uptime and memory metrics in the expected JSON format.

**Acceptance Scenarios**:

1. **Given** the application has been running, **When** a GET request is sent to `/api/v1/status`, **Then** HTTP 200 OK is returned with a JSON response containing `uptime`, `totalMemoryMB`, `freeMemoryMB`, and `timestamp` fields
2. **Given** the application just started, **When** a GET request is sent to `/api/v1/status`, **Then** `uptime` accurately reflects the seconds elapsed since startup
3. **Given** multiple concurrent requests to `/api/v1/status`, **When** all requests complete, **Then** each receives a valid response and response time is under 10ms per request

---

### User Story 2 - Monitor Memory Usage (Priority: P1)

A developer debugging memory issues needs to understand the current memory utilization of the JVM at any point in time. They query the status endpoint to see both total allocated memory and free memory available, helping them diagnose memory pressure.

**Why this priority**: Critical for identifying memory leaks and capacity planning. Memory metrics are essential operational data.

**Independent Test**: Can be fully tested by calling `/api/v1/status` and verifying that `totalMemoryMB` and `freeMemoryMB` fields are populated with positive integer values.

**Acceptance Scenarios**:

1. **Given** the JVM is running with available memory, **When** the status endpoint is called, **Then** `totalMemoryMB` reflects the total memory allocated to the JVM (e.g., 512 MB)
2. **Given** the JVM has performed garbage collection, **When** the status endpoint is called twice, **Then** `freeMemoryMB` may vary between calls reflecting current heap state
3. **Given** a memory constraint (e.g., 256MB heap), **When** the status endpoint is queried, **Then** `totalMemoryMB` reflects the configured limit

---

### User Story 3 - Record Timestamp for Audit Trail (Priority: P2)

An operations team wants to track when health checks occur for audit purposes and to correlate events with application logs. The status response includes an ISO-8601 timestamp.

**Why this priority**: Useful for audit trails and correlating metrics with log entries. Enhances observability but not strictly required for health checks to function.

**Independent Test**: Can be fully tested by calling `/api/v1/status` and validating that the response contains a valid ISO-8601 formatted `timestamp` field.

**Acceptance Scenarios**:

1. **Given** a GET request to `/api/v1/status`, **When** the response is received, **Then** the `timestamp` field is present and formatted as ISO-8601 (e.g., "2026-07-02T17:11:00Z")
2. **Given** two consecutive calls to `/api/v1/status` within 1 second, **When** both responses are received, **Then** the timestamp of the second response is equal to or later than the first

---

### Edge Cases

- What happens when the endpoint is called immediately after application startup (uptime = 0 seconds)?
- How does the system handle malformed requests (e.g., POST instead of GET, query parameters)?
- What happens if someone attempts to pass authentication headers to a public endpoint?
- How does the system behave under high load (concurrent requests to the status endpoint)?
- What if JVM memory is completely exhausted (freeMemoryMB approaches 0)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST expose a public GET endpoint at `/api/v1/status` with HTTP 200 OK response
- **FR-002**: Endpoint MUST NOT require authentication or authorization; accessible without credentials
- **FR-003**: Response MUST include `uptime` field showing application uptime in human-readable format: `XhYmZs` (e.g., "2h 15m 30s"), always as a string. Omit zero components (e.g., "45m 30s" if no hours, "15s" if seconds only)
- **FR-004**: Response MUST include `totalMemoryMB` field as an integer representing total JVM memory allocation in megabytes
- **FR-005**: Response MUST include `freeMemoryMB` field as an integer representing currently available JVM free memory in megabytes
- **FR-006**: Response MUST include `timestamp` field formatted as ISO-8601 UTC datetime (e.g., "2026-07-02T17:11:00Z")
- **FR-007**: Response MUST be valid JSON with `Content-Type: application/json` header
- **FR-008**: Endpoint MUST follow controller → service → repository layered architecture pattern
- **FR-009**: Implementation MUST use Spring Boot 4.0.0 Jakarta namespace annotations (@RestController, @GetMapping, etc.)
- **FR-010**: Service layer MUST use `@Transactional(readOnly = true)` annotation for read-only operation
- **FR-011**: All method names MUST follow camelCase convention (e.g., `getHealthStatus()`, `calculateUptime()`)
- **FR-012**: API response MUST be documented via SpringDoc OpenAPI 3.0 annotations for Swagger/OpenAPI integration
- **FR-013**: Response time MUST be under 10ms for 99% of requests (no database access, minimal computation)
- **FR-014**: Implementation MUST use Spring's built-in metrics/management capabilities (e.g., `java.lang.Runtime.getRuntime()`) for memory data
- **FR-015**: No new dependencies required; use existing Spring Boot, Jakarta, and Java standard library only
- **FR-016**: If memory metrics cannot be determined (JVM runtime access denied or returns null), service MUST return HTTP 200 with memory fields set to 0, uptime still populated, and log a WARN-level message at application startup: "Unable to access JVM memory metrics; health endpoint will report 0 for memory values"
- **FR-017**: Endpoint MUST NOT implement caching; each request calls `java.lang.Runtime.getRuntime()` directly to ensure real-time metrics
- **FR-018**: Endpoint MUST NOT implement custom error handling for invalid HTTP methods (POST, PUT, DELETE, etc.); Spring's default error handler responds with HTTP 405 Method Not Allowed
- **FR-019**: Implementation MUST NOT use Spring Actuator's `/actuator/health` endpoint; custom `/api/v1/status` is the sole health endpoint for this feature

### Key Entities *(include if feature involves data)*

- **HealthStatusResponse DTO**: Immutable data transfer object containing health metrics
  - `uptime`: String or Long - application uptime
  - `totalMemoryMB`: Integer - total allocated JVM memory
  - `freeMemoryMB`: Integer - currently available JVM memory
  - `timestamp`: String (ISO-8601) - time of response generation

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Endpoint responds to GET `/api/v1/status` with HTTP 200 and valid JSON containing all four required fields
- **SC-002**: Response time averages under 5ms and 99th percentile under 10ms for non-cacheable, real-time metrics
- **SC-003**: Memory metrics (totalMemoryMB, freeMemoryMB) reflect current JVM runtime state within 100ms of measurement
- **SC-004**: Uptime value increases monotonically with application runtime (no backward jumps)
- **SC-005**: Endpoint successfully handles 100 concurrent requests without errors or timeouts
- **SC-006**: Response payload size is under 500 bytes (compact JSON representation)
- **SC-007**: Unit test coverage for HealthStatusService is minimum 90%
- **SC-008**: Integration test (MockMvc) verifies endpoint contract: status code, content type, required fields
- **SC-009**: Swagger/OpenAPI documentation auto-generated from code annotations, accessible via `/swagger-ui.html`
- **SC-010**: Zero breaking changes to existing API contracts; new endpoint isolated from current `/api/v1/players` endpoints

## Clarifications *(encoded from review)*

### Memory Metrics Failure Handling
- **Decision**: If `java.lang.Runtime.getRuntime()` fails or returns 0, return HTTP 200 OK with `totalMemoryMB: 0` and `freeMemoryMB: 0`, while still providing valid `uptime` and `timestamp` fields
- **Rationale**: Ensures endpoint never breaks production monitoring; failure is logged so ops teams are alerted
- **Log Entry**: WARN-level message at application startup if memory access fails

### Uptime Format
- **Decision**: Uptime MUST be a human-readable string formatted as `XhYmZs` (e.g., "2h 15m 30s", "45m 30s", "15s")
- **Rationale**: More operational-friendly for DevOps dashboards and monitoring systems
- **Zero Components**: Omit zero values (no leading "0h" if less than 1 hour)

### Error Response Handling
- **Decision**: Invalid HTTP methods (POST, PUT, DELETE, etc.) MUST use Spring's default error handler
- **Rationale**: No custom error handling needed; Spring MVC provides HTTP 405 Method Not Allowed automatically
- **Implementation**: Use `@GetMapping` only; no custom `@ExceptionHandler` for method mismatch

### Caching Strategy
- **Decision**: NO caching; each request calls `java.lang.Runtime.getRuntime()` directly
- **Rationale**: Metrics must reflect real-time JVM state; response time is sub-millisecond; caching complexity not justified
- **No `@Cacheable`**: Service method MUST NOT use Spring Cache annotations

### Spring Actuator Integration
- **Decision**: Custom `/api/v1/status` endpoint is standalone; Spring Actuator is NOT used
- **Rationale**: Simpler implementation; focused on application health + memory; Actuator's broader metrics not needed for v1
- **Independence**: Actuator may be added in future features independently; no coupling

## Assumptions

- Application startup time (bootstrap time) is captured via Spring Boot's built-in application context initialization tracking
- JVM memory metrics are obtained from `java.lang.Runtime.getRuntime()` without additional dependencies
- Uptime calculation begins when Spring ApplicationContext starts (standard Spring behavior)
- Timestamp should be current system time at request processing time (server-side generation, not client-provided)
- **Response format is ALWAYS human-readable string for uptime** (e.g., "2h 15m 30s"), not total seconds
- No external metrics system (Prometheus, Micrometer custom) is required; basic Runtime API is sufficient for v1
- "No authentication required" means the endpoint is public and unrestricted; no Spring Security rules apply
- **No caching is implemented**; metrics reflect real-time JVM state on each request (no @Cacheable)
- DTO naming convention follows existing project pattern: `HealthStatusResponse` (not `StatusResponseDTO` or `StatusDto`)
- All new code follows Constitution principle: Constructor injection via Lombok `@RequiredArgsConstructor`
- Exceptions from the service layer are handled by existing `@ControllerAdvice` (GlobalExceptionHandler) with no special handling needed
- **Invalid HTTP methods (POST, PUT, DELETE) are handled by Spring's default error handler** (HTTP 405); no custom handling
- **Custom `/api/v1/status` is the sole health endpoint**; Spring Actuator (`/actuator/health`) is NOT integrated
- If memory metrics cannot be retrieved, endpoint returns HTTP 200 with memory fields = 0 and logs WARN at startup
- No database migration required; feature is purely in-memory calculation
- Build pipeline (Maven, JaCoCo, GitHub Actions) remains unchanged; new tests must meet existing coverage threshold (minimum 80% for service layer)
