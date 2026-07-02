# Research: Health Status Endpoint Feature

**Phase**: 0 (Research & Clarification Resolution)
**Status**: Complete ✅
**Date**: 2026-07-02

---

## Executive Summary

All technical unknowns for the Health Status Endpoint feature have been **resolved in the specification**. No external research required. This document confirms that every design decision is documented, justified, and ready for implementation.

---

## Research Questions & Resolutions

### 1. How to Calculate Application Uptime?

**Question**: Should uptime be calculated from application startup time, process start time, or first request time?

**Decision**: ✅ **Application startup time (Spring ApplicationContext initialization)**
- **Source**: Specification Assumption 4: "Uptime calculation begins when Spring ApplicationContext starts (standard Spring behavior)"
- **Implementation**: Use `ManagementFactory.getRuntimeMXBean().getUptime()` or manually track startup time via `ApplicationContextEvent`
- **Rationale**: Spring Boot starts the ApplicationContext during `SpringApplication.run()`, before first request; this is the canonical "application started" point
- **Format**: Human-readable string: `XhYmZs` (e.g., "2h 15m 30s", "45m 30s", "15s"), omitting zero components
- **Example Calculation**:
  - Uptime in milliseconds: `ManagementFactory.getRuntimeMXBean().getUptime()`
  - Convert to hours, minutes, seconds; format as string with zero-components omitted

**Alternatives Considered**:
- Process start time (JVM startup) — rejected because application code is not ready until Spring starts
- First request time — rejected because it doesn't reflect application health from startup
- Manual stopwatch — rejected because `ManagementFactory` provides native uptime

---

### 2. Memory Metrics Source: `java.lang.Runtime` vs. `ManagementFactory`

**Question**: Should memory metrics come from `Runtime.getRuntime()` or `ManagementFactory.getMemoryMXBean()`?

**Decision**: ✅ **`java.lang.Runtime.getRuntime()`**
- **Source**: Specification FR-014: "Implementation MUST use Spring's built-in metrics/management capabilities (e.g., `java.lang.Runtime.getRuntime()`) for memory data"
- **API Call**:
  ```java
  Runtime runtime = Runtime.getRuntime();
  long totalMemory = runtime.totalMemory(); // bytes
  long freeMemory = runtime.freeMemory();   // bytes
  ```
- **Conversion**: Divide by (1024 * 1024) to get megabytes
- **Rationale**: Simpler API; no additional dependencies; accurate for heap memory representation
- **Note**: `totalMemory()` returns current JVM heap allocation (not maximum); this reflects dynamic heap sizing

**Alternatives Considered**:
- `ManagementFactory.getMemoryMXBean()` — provides more detailed memory stats but adds complexity; `Runtime` sufficient for v1
- Direct memory reading via JNI — rejected; not portable, adds complexity
- Micrometer custom metrics — rejected per spec: "No external metrics system (Prometheus, Micrometer custom) is required"

**Reference Implementation**:
```java
public HealthStatusResponse getHealthStatus() {
    Runtime runtime = Runtime.getRuntime();
    
    long totalMemoryBytes = runtime.totalMemory();
    long freeMemoryBytes = runtime.freeMemory();
    
    int totalMemoryMB = (int) (totalMemoryBytes / (1024L * 1024L));
    int freeMemoryMB = (int) (freeMemoryBytes / (1024L * 1024L));
    
    String uptime = formatUptime(/* calculated from startup time */);
    String timestamp = ZonedDateTime.now(ZoneId.of("UTC")).format(ISO_8601);
    
    return HealthStatusResponse.builder()
        .uptime(uptime)
        .totalMemoryMB(totalMemoryMB)
        .freeMemoryMB(freeMemoryMB)
        .timestamp(timestamp)
        .build();
}
```

---

### 3. Uptime Format: String vs. Long

**Question**: Should uptime be a numeric value (total seconds/milliseconds) or human-readable string?

**Decision**: ✅ **Human-readable string**
- **Source**: Specification FR-003, Assumption 7: "Response format is ALWAYS human-readable string for uptime (e.g., "2h 15m 30s"), not total seconds"
- **Format**: `XhYmZs` with zero components omitted
  - Examples: `"2h 15m 30s"`, `"45m 30s"`, `"15s"`, `"2h 30s"` (no minutes)
- **Rationale**: Operational-friendly for DevOps dashboards; easier to read in logs and monitoring UIs
- **Parsing**: Parseable via regex or fixed format; straightforward for clients

**Alternatives Considered**:
- Total seconds (e.g., `7530`) — rejected as less operational-friendly
- ISO-8601 duration (e.g., `PT2H15M30S`) — rejected as overly complex for this use case
- Milliseconds — rejected as not human-readable

**Implementation Helper Function**:
```java
private String formatUptime(long uptimeMillis) {
    long totalSeconds = uptimeMillis / 1000;
    long hours = totalSeconds / 3600;
    long minutes = (totalSeconds % 3600) / 60;
    long seconds = totalSeconds % 60;
    
    StringBuilder sb = new StringBuilder();
    if (hours > 0) sb.append(hours).append("h ");
    if (minutes > 0) sb.append(minutes).append("m ");
    if (seconds > 0) sb.append(seconds).append("s");
    
    return sb.toString().trim();
}
```

---

### 4. Response Time Performance: Is <10ms Achievable?

**Question**: Can the endpoint respond in <10ms given memory calculations?

**Decision**: ✅ **Yes, easily achievable**
- **Source**: Specification FR-013: "Response time MUST be under 10ms for 99% of requests"
- **Reasoning**:
  1. `Runtime.getRuntime()` calls are native Java methods — sub-microsecond
  2. No I/O operations (no database, no network calls)
  3. No thread contention (simple read-only operations)
  4. Uptime calculation: subtraction + string formatting ~ <1ms
  5. Memory conversion: bit shifts and division ~ <1ms
  6. Spring MVC routing and JSON serialization ~ <5ms
  7. **Total**: <10ms typical; <5ms average
- **Measurement**: Will be validated in integration tests using MockMvc `StopWatch`

**Performance Constraints Met**: ✅
- No database access
- No caching complexity (each request is fresh calculation)
- No thread pool contention
- JSON payload < 500 bytes

---

### 5. Caching Strategy: Should We Cache Metrics?

**Question**: Should metrics be cached or computed fresh on each request?

**Decision**: ✅ **NO CACHING—compute fresh on each request**
- **Source**: Specification FR-017: "Endpoint MUST NOT implement caching; each request calls `java.lang.Runtime.getRuntime()` directly to ensure real-time metrics"
- **Rationale**: 
  - Metrics must reflect real-time JVM state
  - Response time is sub-millisecond; caching adds no performance benefit
  - Caching introduces complexity (TTL, invalidation, consistency issues)
- **No `@Cacheable` annotation** on service method
- **No Redis/Caffeine cache** configuration needed

**Why Not Cache?**:
- Memory metrics can change rapidly (garbage collection, allocation)
- Stale cache would mask true application state
- Response time (<10ms) is already fast enough; caching not needed for performance
- Per spec: "caching complexity not justified"

---

### 6. Error Handling: What If Memory Metrics Are Unavailable?

**Question**: How should the endpoint behave if `Runtime.getRuntime()` returns null or throws exception?

**Decision**: ✅ **Graceful degradation: HTTP 200 with memory = 0**
- **Source**: Specification FR-016: "If memory metrics cannot be determined (JVM runtime access denied or returns null), service MUST return HTTP 200 with memory fields set to 0, uptime still populated, and log a WARN-level message at application startup"
- **Behavior**:
  1. Catch any exception from `Runtime.getRuntime()`
  2. Log WARN: `"Unable to access JVM memory metrics; health endpoint will report 0 for memory values"` at **application startup** (once, not per request)
  3. Return HTTP 200 OK with:
     - `uptime`: Still calculated (not affected by memory access failure)
     - `totalMemoryMB`: 0
     - `freeMemoryMB`: 0
     - `timestamp`: Current time
  4. Never return error status (5xx or 4xx) — health checks must not fail

**Rationale**: Production monitoring systems rely on health endpoints; an error response could trigger unnecessary alerts or failovers. Degraded metrics are better than no metrics.

**Implementation Pattern**:
```java
@Service
@Transactional(readOnly = true)
public class HealthStatusService {
    private static final Logger logger = LoggerFactory.getLogger(HealthStatusService.class);
    private volatile boolean memoryAccessFailed = false;
    
    @PostConstruct
    public void validateMemoryAccess() {
        try {
            Runtime.getRuntime();  // test access
        } catch (Exception e) {
            logger.warn("Unable to access JVM memory metrics; health endpoint will report 0 for memory values");
            memoryAccessFailed = true;
        }
    }
    
    public HealthStatusResponse getHealthStatus() {
        int totalMemoryMB = 0;
        int freeMemoryMB = 0;
        
        try {
            Runtime runtime = Runtime.getRuntime();
            totalMemoryMB = (int) (runtime.totalMemory() / (1024L * 1024L));
            freeMemoryMB = (int) (runtime.freeMemory() / (1024L * 1024L));
        } catch (Exception e) {
            // Silently degrade; warning logged at startup
        }
        
        return HealthStatusResponse.builder()
            .uptime(/* calculated */)
            .totalMemoryMB(totalMemoryMB)
            .freeMemoryMB(freeMemoryMB)
            .timestamp(/* current time */)
            .build();
    }
}
```

---

### 7. API Path and Method: Is `/api/v1/status` the Right Choice?

**Question**: Should this be part of existing API versioning scheme or standalone?

**Decision**: ✅ **`GET /api/v1/status` (aligns with existing API v1 versioning)**
- **Source**: Specification FR-001: "System MUST expose a public GET endpoint at `/api/v1/status`"
- **Rationale**: 
  - Aligns with existing project versioning (`/api/v1/players`, etc.)
  - Version 1 is current; future versions can evolve independently
  - Status is a non-player resource; separate from CRUD endpoints
- **Method**: GET only (POST/PUT/DELETE return 405 via Spring default)

**Alternatives Considered**:
- `/health` — rejected; less specific, conflicts with potential Spring Actuator integration
- `/status` (no `/api/v1`) — rejected; doesn't align with project versioning
- `/api/v1/health` — rejected; specification explicitly requires `/api/v1/status`

---

### 8. Spring Actuator Integration: Should We Use It?

**Question**: Should the endpoint integrate with Spring Actuator's `/actuator/health` or be standalone?

**Decision**: ✅ **Standalone; NO Spring Actuator integration**
- **Source**: Specification FR-019: "Implementation MUST NOT use Spring Actuator's `/actuator/health` endpoint; custom `/api/v1/status` is the sole health endpoint for this feature"
- **Rationale**:
  - Simpler implementation (no Actuator dependency configuration)
  - Custom endpoint focused on specific metrics (uptime + memory)
  - Actuator provides broader metrics not needed for v1
  - Actuator may be added in future features independently; no coupling
- **No `/actuator/health` endpoint** for this feature
- **No `@Endpoint` annotation** or Actuator framework used

---

### 9. Thread Safety and Concurrent Requests

**Question**: Is `Runtime.getRuntime()` thread-safe? Can endpoint handle concurrent requests?

**Decision**: ✅ **Yes, thread-safe; no concurrency concerns**
- **Research**: `Runtime.getRuntime()` is a singleton and thread-safe (JDK guarantee)
- **Source**: Specification SC-005: "Endpoint successfully handles 100 concurrent requests without errors or timeouts"
- **Implementation**: Service method has no mutable state; each request is independent
- **Uptime Calculation**: Uses `ManagementFactory.getRuntimeMXBean().getUptime()` (thread-safe singleton)
- **No Synchronization Needed**: No locks or `synchronized` blocks required

**Testing**: Integration test will fire 10-100 concurrent requests and verify all succeed.

---

### 10. DTO Immutability and Serialization

**Question**: How should the response DTO be structured for immutability and JSON serialization?

**Decision**: ✅ **Lombok `@Value` + Jackson annotations**
- **Source**: Constitution: "DTOs are immutable, no business logic; Constructor injection via Lombok `@RequiredArgsConstructor`"
- **Annotations**:
  ```java
  @Value
  @Builder
  public class HealthStatusResponse {
      String uptime;
      Integer totalMemoryMB;
      Integer freeMemoryMB;
      @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
      String timestamp;
  }
  ```
- **Immutability**: `@Value` generates immutable getters and private constructor
- **Serialization**: Jackson auto-configures via Spring Boot; camelCase fields match JSON by default
- **Validation**: Bean Validation annotations (`@Positive`, `@PositiveOrZero`) optional but recommended

**Alternatives Considered**:
- `@Data` — rejected; not immutable by default
- `record` (Java 17+) — rejected; project uses full Lombok compatibility
- `@Entity` — rejected; this is a DTO only, not a JPA entity

---

### 11. Swagger/OpenAPI Documentation

**Question**: How should the endpoint be documented for Swagger/OpenAPI?

**Decision**: ✅ **SpringDoc OpenAPI 3.0 annotations**
- **Source**: Specification FR-012: "API response MUST be documented via SpringDoc OpenAPI 3.0 annotations for Swagger/OpenAPI integration"
- **Annotations**:
  ```java
  @Operation(summary = "Get application health status", 
             description = "Returns application uptime, JVM memory metrics, and current timestamp")
  @ApiResponse(responseCode = "200", description = "Health status retrieved successfully",
               content = @Content(mediaType = "application/json", 
                                 schema = @Schema(implementation = HealthStatusResponse.class)))
  @GetMapping("/api/v1/status")
  public ResponseEntity<HealthStatusResponse> getHealthStatus() { ... }
  ```
- **Auto-Documentation**: Swagger UI at `/swagger-ui.html` auto-generates endpoint docs
- **Schema**: HealthStatusResponse fields auto-documented from class properties

**Validation**: Swagger endpoint must return valid OpenAPI 3.0 spec at `/v3/api-docs`.

---

### 12. Transactional Boundaries

**Question**: Should `getHealthStatus()` be marked `@Transactional`?

**Decision**: ✅ **Yes, `@Transactional(readOnly = true)`**
- **Source**: Constitution III, Specification FR-010: "Service layer MUST use `@Transactional(readOnly = true)` annotation for read-only operation"
- **Rationale**:
  - Marks method as read-only for clarity and potential optimization
  - Spring optimizes read-only transactions (disables flush, optimizes connection handling)
  - No database access, but annotation maintains architectural consistency
- **No Side Effects**: Method computes uptime/memory; no state modification

**Implementation**:
```java
@Service
@Transactional(readOnly = true)
public class HealthStatusService {
    public HealthStatusResponse getHealthStatus() { ... }
}
```

---

## Best Practices Identified

### 1. **Real-Time Metrics Over Caching**
Even with sub-millisecond response times, fresh calculations are critical for operational metrics. Stale data can mask true application state.

### 2. **Graceful Degradation**
If memory metrics are unavailable, return HTTP 200 with degraded values rather than error. Health endpoints must be reliable.

### 3. **Human-Readable Uptime**
Operational teams need readable formats. "2h 15m 30s" is more useful than "8130" seconds or "PT2H15M30S" (ISO-8601 duration).

### 4. **Immutable DTOs**
Immutable response objects prevent accidental mutations and improve thread safety. Lombok `@Value` is clean and enforces immutability.

### 5. **Sub-Millisecond Response Times**
No I/O, no complex computation — just native Java memory APIs. This achieves <10ms easily without caching complexity.

### 6. **Single Responsibility**
This endpoint has one job: report application health. No complex aggregation, no database joins, no external calls.

---

## Conclusion

**No External Research Required** ✅

All technical decisions are grounded in the specification, Constitution, and Java best practices. The feature is ready to proceed to **Phase 1: Design & Contracts**, then **Phase 2: Tasks** and **Phase 3: Implementation**.

---

**Research Completed**: 2026-07-02
**Reviewed By**: Architecture Review
**Status**: ✅ READY FOR PHASE 1 DESIGN
