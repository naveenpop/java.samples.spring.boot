# Data Model: Health Status Endpoint

**Phase**: 1 (Design & Contracts)
**Status**: Complete ✅
**Date**: 2026-07-02

---

## Overview

This feature has **no database entities** and **no persistent data model**. It exposes a single, immutable **DTO (Data Transfer Object)** that represents the application's current health and system resource metrics. The DTO is computed on every request from in-memory JVM state; no storage, caching, or entity relationships are involved.

---

## Entity Model: None

**Decision**: This feature does NOT introduce any JPA `@Entity` classes.

**Reasoning**:
1. Health metrics are ephemeral (computed per-request)
2. No persistence required (no database table)
3. No relationships to other entities
4. DTO-only approach is sufficient and aligns with Constitution

---

## Data Transfer Object (DTO)

### HealthStatusResponse

**Location**: `src/main/java/com/example/models/HealthStatusResponse.java`

**Purpose**: Immutable response object representing application health at a point in time.

**Class Definition**:

```java
package com.example.models;

import com.fasterxml.jackson.annotation.JsonFormat;
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.PositiveOrZero;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Value;

/**
 * Immutable DTO representing the application's current health and resource status.
 * 
 * Instances are created per-request by {@link HealthStatusService#getHealthStatus()}
 * and serialized to JSON via Jackson.
 * 
 * No instances are persisted; all data is computed from JVM runtime metrics.
 */
@Value
@Builder
@AllArgsConstructor
public class HealthStatusResponse {

    /**
     * Application uptime in human-readable format.
     * 
     * Format: XhYmZs (e.g., "2h 15m 30s", "45m 30s", "15s")
     * Zero components are omitted (no "0h" prefix if < 1 hour).
     * 
     * Calculated from application startup time (Spring ApplicationContext initialization).
     */
    @Schema(
        description = "Application uptime in human-readable format (e.g., '2h 15m 30s')",
        example = "45m 30s",
        required = true
    )
    private final String uptime;

    /**
     * Total JVM memory allocated to the application, in megabytes.
     * 
     * Obtained from {@code Runtime.getRuntime().totalMemory() / (1024 * 1024)}.
     * 
     * Note: This is the current heap allocation, not the maximum heap size.
     * For a JVM with -Xmx512m, totalMemoryMB might be 256 (if heap hasn't expanded yet).
     * 
     * If memory metrics cannot be retrieved (rare exception), value is 0.
     */
    @Schema(
        description = "Total JVM memory allocated in megabytes",
        example = "512",
        required = true
    )
    @PositiveOrZero
    private final Integer totalMemoryMB;

    /**
     * Currently available (free) JVM heap memory, in megabytes.
     * 
     * Obtained from {@code Runtime.getRuntime().freeMemory() / (1024 * 1024)}.
     * 
     * Represents space available for new object allocation before garbage collection.
     * 
     * Constraint: freeMemoryMB <= totalMemoryMB (always)
     * 
     * If memory metrics cannot be retrieved (rare exception), value is 0.
     */
    @Schema(
        description = "Currently available JVM free memory in megabytes",
        example = "256",
        required = true
    )
    @PositiveOrZero
    private final Integer freeMemoryMB;

    /**
     * Server-side timestamp indicating when this response was generated.
     * 
     * Format: ISO-8601 UTC (e.g., "2026-07-02T17:11:00Z").
     * 
     * Always UTC; never client's local time.
     * Useful for audit trails and correlating metrics with application logs.
     */
    @Schema(
        description = "ISO-8601 UTC timestamp when response was generated",
        example = "2026-07-02T17:11:00Z",
        required = true
    )
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
    private final String timestamp;

    // Lombok @Value generates:
    // - private final fields (shown above)
    // - all-args constructor (via @AllArgsConstructor, used by @Builder)
    // - immutable getters: uptime(), totalMemoryMB(), freeMemoryMB(), timestamp()
    // - equals(), hashCode(), toString()
    // - no setters (immutable)

    /**
     * Validation constraints applied via Jakarta Bean Validation:
     * 
     * - uptime: Non-empty String; must match format "XhYmZs" (validated in service layer)
     * - totalMemoryMB: >= 0 (via @PositiveOrZero)
     * - freeMemoryMB: >= 0 and <= totalMemoryMB (via @PositiveOrZero, runtime check in service)
     * - timestamp: ISO-8601 UTC format (validated via @JsonFormat; Jackson enforces on deserialization if ever needed)
     * 
     * Serialization Behavior:
     * Jackson auto-serializes to JSON with camelCase field names:
     * {
     *   "uptime": "...",
     *   "totalMemoryMB": ...,
     *   "freeMemoryMB": ...,
     *   "timestamp": "..."
     * }
     */
}
```

---

## Field Specifications

### Field: `uptime`

| Aspect | Value | Notes |
|--------|-------|-------|
| **Type** | `String` | Immutable; never null |
| **Format** | `XhYmZs` | e.g., "2h 15m 30s", "45m 30s", "15s", "2h 30s" |
| **Zero Components** | Omitted | No "0h" prefix; no "0m" if seconds only |
| **Min Value** | "0s" | At application startup |
| **Monotonicity** | Always increasing | Never decreases |
| **Source** | Application startup time | Calculated from `ManagementFactory.getRuntimeMXBean().getUptime()` |
| **Calculation** | Total milliseconds since startup → hours:minutes:seconds | See service layer implementation |

**Examples**:
```
Just started:              "0s"
After 30 seconds:          "30s"
After 1 minute 45 seconds: "1m 45s"
After 2 hours:             "2h"
After 2.5 hours:           "2h 30m"
After 2h 45m 30s:          "2h 45m 30s"
```

---

### Field: `totalMemoryMB`

| Aspect | Value | Notes |
|--------|-------|-------|
| **Type** | `Integer` (nullable? No; always present) | Primitive wrapper; never null |
| **Range** | ≥ 0 | Typically 256–2048 (depends on `-Xmx` JVM flag) |
| **Units** | Megabytes | Converted from bytes via division by (1024 × 1024) |
| **Source** | `Runtime.getRuntime().totalMemory()` | Current heap allocation (not max) |
| **Precision** | Truncated to integer | `(totalBytes / 1048576)` |
| **Degraded Value** | 0 | If memory access fails (rare) |

**Calculation**:
```java
long totalMemoryBytes = Runtime.getRuntime().totalMemory();
int totalMemoryMB = (int) (totalMemoryBytes / (1024L * 1024L));
```

**Examples** (for typical JVM with `-Xmx512m`):
```
Heap just allocated:    256 MB
After some allocation:  384 MB
After garbage collection: 512 MB (max expansion)
```

---

### Field: `freeMemoryMB`

| Aspect | Value | Notes |
|--------|-------|-------|
| **Type** | `Integer` (never null) | Primitive wrapper |
| **Range** | 0 to `totalMemoryMB` | Constraint: `freeMemoryMB <= totalMemoryMB` |
| **Units** | Megabytes | Converted from bytes |
| **Source** | `Runtime.getRuntime().freeMemory()` | Heap space available for new allocations |
| **Precision** | Truncated to integer | `(freeBytes / 1048576)` |
| **Degraded Value** | 0 | If memory access fails (rare) |
| **Volatility** | Highly variable | Changes with garbage collection, allocation |

**Calculation**:
```java
long freeMemoryBytes = Runtime.getRuntime().freeMemory();
int freeMemoryMB = (int) (freeMemoryBytes / (1024L * 1024L));
```

**Examples**:
```
Just started (low allocation):      384 MB
After heavy allocation:             64 MB
After garbage collection:           400 MB
Under memory pressure:              8 MB
```

**Constraint Validation** (service layer):
```java
if (freeMemoryMB > totalMemoryMB) {
    throw new IllegalStateException("freeMemory exceeds totalMemory");
}
```

---

### Field: `timestamp`

| Aspect | Value | Notes |
|--------|-------|-------|
| **Type** | `String` (ISO-8601 UTC) | Immutable; never null |
| **Format** | `yyyy-MM-dd'T'HH:mm:ss'Z'` | Example: "2026-07-02T17:11:00Z" |
| **Timezone** | UTC (Zulu time) | Always `Z` suffix; never client's local time |
| **Precision** | Seconds | No milliseconds (keeps payload compact) |
| **Source** | Server system clock at request processing time | Generated in service layer |
| **Validation** | ISO-8601 format check | Jackson `@JsonFormat` annotation |

**Calculation**:
```java
String timestamp = ZonedDateTime.now(ZoneId.of("UTC"))
    .format(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss'Z'"));
```

**Examples**:
```
"2026-07-02T00:00:00Z"
"2026-07-02T12:30:45Z"
"2026-07-02T23:59:59Z"
```

**Why Server Time?**:
- Ensures consistent audit trail (no client clock skew)
- Correlates with server-side logs
- Prevents replay attacks if timestamps were client-provided

---

## Serialization Contract

### JSON Representation

```json
{
  "uptime": "2h 45m 30s",
  "totalMemoryMB": 512,
  "freeMemoryMB": 256,
  "timestamp": "2026-07-02T17:11:00Z"
}
```

**Properties**:
- **Content-Type**: `application/json; charset=UTF-8`
- **Encoding**: UTF-8 (no special characters in this DTO)
- **Payload Size**: ~89–150 bytes (compact; no extra whitespace in production)
- **Field Order**: Not guaranteed (JSON object property order is unspecified); clients must not rely on order
- **Null Handling**: No fields are null; all fields always present
- **Number Format**: Integers (no decimals for memory fields)

### Jackson Configuration

**Auto-configured by Spring Boot**:
- `ObjectMapper` handles serialization/deserialization
- camelCase property names (default Jackson behavior)
- ISO-8601 timestamps via `@JsonFormat` annotations
- No custom serializers/deserializers needed

**Example Spring Boot Jackson Configuration** (optional, for production):
```properties
# application.properties
spring.jackson.serialization.write-dates-as-timestamps=false
spring.jackson.time-zone=UTC
spring.jackson.serialization.indent-output=false
```

---

## Validation Rules

### Service Layer Validation

The following checks are performed in `HealthStatusService.getHealthStatus()`:

```java
// 1. Uptime format validation
if (!uptime.matches("^(\\d+h )?( \\d+m)?( \\d+s)?$")) {
    throw new IllegalArgumentException("Invalid uptime format: " + uptime);
}

// 2. Memory constraint validation
if (freeMemoryMB > totalMemoryMB) {
    throw new IllegalStateException(
        "Free memory (" + freeMemoryMB + "MB) exceeds total (" + totalMemoryMB + "MB)"
    );
}

// 3. Memory non-negativity (redundant; @PositiveOrZero enforces at binding time)
if (totalMemoryMB < 0 || freeMemoryMB < 0) {
    throw new IllegalArgumentException("Memory values cannot be negative");
}
```

### REST Contract Validation

**Spring MVC automatic validation** (if DTO were in request body):
- Bean Validation framework triggers via `@Valid` annotation
- `@PositiveOrZero` constraints enforced by Hibernate Validator
- Violations return HTTP 400 Bad Request

**Note**: For this feature, `HealthStatusResponse` is **response only**; no request validation needed.

---

## No Database Schema

**Decision**: No SQL migrations, no Flyway versions, no schema changes.

**Reasoning**:
- Health metrics are computed in-memory
- No persistent data
- No entity relationships
- No need for database storage

**Implication**:
- No `V{N}__description.sql` migration file
- No changes to `src/test/resources/ddl.sql` or `dml.sql`
- No Flyway version tracking
- Production `storage/` database unchanged

---

## Example Request/Response Flows

### Flow 1: Healthy Application (Normal Case)

**Request**:
```http
GET /api/v1/status HTTP/1.1
Host: localhost:9000
Accept: application/json
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Content-Length: 89
Date: Wed, 02 Jul 2026 17:11:00 GMT

{
  "uptime": "2h 15m 30s",
  "totalMemoryMB": 512,
  "freeMemoryMB": 256,
  "timestamp": "2026-07-02T17:11:00Z"
}
```

---

### Flow 2: Just-Started Application

**Request**:
```http
GET /api/v1/status HTTP/1.1
Host: localhost:9000
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8

{
  "uptime": "0s",
  "totalMemoryMB": 384,
  "freeMemoryMB": 384,
  "timestamp": "2026-07-02T17:11:00Z"
}
```

---

### Flow 3: Low Memory Condition

**Request**:
```http
GET /api/v1/status HTTP/1.1
Host: localhost:9000
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8

{
  "uptime": "5h 30m 15s",
  "totalMemoryMB": 512,
  "freeMemoryMB": 8,
  "timestamp": "2026-07-02T22:41:15Z"
}
```

**Implication**: Client monitoring systems should set alerts for `freeMemoryMB < 50`, for example.

---

### Flow 4: Memory Access Failure (Degraded)

**Request**:
```http
GET /api/v1/status HTTP/1.1
Host: localhost:9000
```

**Response** (rare; memory access denied):
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8

{
  "uptime": "1h 15m 30s",
  "totalMemoryMB": 0,
  "freeMemoryMB": 0,
  "timestamp": "2026-07-02T18:26:30Z"
}
```

**Server Logs**:
```
WARN [startup] HealthStatusService: Unable to access JVM memory metrics; health endpoint will report 0 for memory values
```

**Client Interpretation**: Memory metrics unavailable; uptime is valid. Operations team should investigate why memory access failed.

---

## Testing Data

### Unit Test Scenarios

```java
// Test: Uptime format
HealthStatusResponse response = HealthStatusResponse.builder()
    .uptime("2h 15m 30s")
    .totalMemoryMB(512)
    .freeMemoryMB(256)
    .timestamp("2026-07-02T17:11:00Z")
    .build();

then(response.getUptime()).isEqualTo("2h 15m 30s");
then(response.getTotalMemoryMB()).isEqualTo(512);
then(response.getFreeMemoryMB()).isEqualTo(256);
then(response.getTimestamp()).isEqualTo("2026-07-02T17:11:00Z");

// Test: Immutability (compile-time check)
// response.setUptime("x");  // Won't compile; no setter

// Test: JSON serialization
String json = objectMapper.writeValueAsString(response);
then(json).contains("\"uptime\":\"2h 15m 30s\"");
```

### Integration Test Scenarios

```java
// MockMvc test: GET /api/v1/status
mockMvc.perform(get("/api/v1/status"))
    .andExpect(status().isOk())
    .andExpect(content().contentType(MediaType.APPLICATION_JSON))
    .andExpect(jsonPath("$.uptime").isString())
    .andExpect(jsonPath("$.totalMemoryMB").isNumber())
    .andExpect(jsonPath("$.freeMemoryMB").isNumber())
    .andExpect(jsonPath("$.timestamp").isString())
    .andExpect(jsonPath("$.freeMemoryMB").value(lessThanOrEqualTo(512))); // Assume totalMemoryMB = 512
```

---

## Summary

| Aspect | Value |
|--------|-------|
| **Entities** | None (no `@Entity` classes) |
| **DTOs** | 1: `HealthStatusResponse` (immutable, response-only) |
| **Database Tables** | None (no migrations) |
| **Database Relationships** | None |
| **Caching** | None (fresh calculation per request) |
| **Validation** | Field-level constraints + service-layer logic checks |
| **Serialization Format** | JSON (via Jackson) |
| **Payload Size** | ~89–150 bytes |
| **Response Time** | <5ms average, <10ms p99 |

---

**Data Model Complete**: 2026-07-02
**Ready for**: Phase 1 Contracts & Phase 2 Tasks
