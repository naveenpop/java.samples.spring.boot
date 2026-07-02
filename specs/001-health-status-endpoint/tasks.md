# Tasks: Health and System Resource Status Endpoint

**Feature**: 001-health-status-endpoint  
**Status**: Ready for Implementation  
**Plan**: [Implementation Plan](./plan.md) | [Specification](./spec.md) | [Data Model](./data-model.md)

---

## Phase 1: Data Transfer Object (DTO) Setup

### Task 1: Create HealthStatusResponse DTO
**ID**: [T001]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `backend`, `models`, `dto`  
**Effort**: 2 story points (30 minutes)  
**Dependencies**: None  

**Description**:
Create immutable DTO `HealthStatusResponse` in `src/main/java/ar/com/nanotaboada/java/samples/spring/boot/models/HealthStatusResponse.java` with Lombok @Value, @Builder annotations. DTO must contain four fields: `uptime` (String), `totalMemoryMB` (Integer), `freeMemoryMB` (Integer), `timestamp` (String).

**Acceptance Criteria**:
- [ ] HealthStatusResponse class created with @Value and @Builder annotations
- [ ] Four fields defined with correct types and immutability (no setters)
- [ ] @Schema annotations added for Swagger documentation
- [ ] @PositiveOrZero validation added to memory fields
- [ ] @JsonFormat annotation on timestamp field with ISO-8601 pattern
- [ ] Class can be instantiated via builder pattern
- [ ] JSON serialization produces camelCase field names
- [ ] Unit test verifies immutability (compile-time check for no setters)
- [ ] Unit test verifies JSON serialization matches expected format

**Test Files**:
- `src/test/java/ar/com/nanotaboada/java/samples/spring/boot/models/HealthStatusResponseTest.java`

**Reference**: [Data Model Spec](./data-model.md) section "DTO Design: HealthStatusResponse"

---

## Phase 2: Repository Layer (Minimal Stub)

### Task 2: Create HealthStatusRepository Interface
**ID**: [T002]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `backend`, `repository`, `architecture`  
**Effort**: 1 story point (15 minutes)  
**Dependencies**: Task 1 (HealthStatusResponse created)

**Description**:
Create minimal repository interface `HealthStatusRepository` in `src/main/java/ar/com/nanotaboada/java/samples/spring/boot/repositories/HealthStatusRepository.java`. This is a stub interface following the layered architecture principle (Controller → Service → Repository). No database queries required; interface can be empty or contain a marker method.

**Acceptance Criteria**:
- [ ] HealthStatusRepository interface created
- [ ] Extends Spring Data Repository interface (or CrudRepository for consistency)
- [ ] Marked with @Repository annotation
- [ ] Interface is empty or contains documentation-only marker method
- [ ] Class follows package naming convention: ar.com.nanotaboada.java.samples.spring.boot.repositories
- [ ] Passes Spring Boot auto-configuration scanning (no errors on startup)

**Reference**: [Implementation Plan](./plan.md) section "Project Structure - Source Code"

---

## Phase 3: Service Layer (Business Logic)

### Task 3: Create HealthStatusService Class
**ID**: [T003]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `backend`, `service`, `metrics`  
**Effort**: 3 story points (1 hour)  
**Dependencies**: Task 1 (HealthStatusResponse), Task 2 (HealthStatusRepository)

**Description**:
Create service class `HealthStatusService` in `src/main/java/ar/com/nanotaboada/java/samples/spring/boot/services/HealthStatusService.java` implementing `getHealthStatus()` method. Service must compute application uptime (human-readable XhYmZs format), fetch JVM memory metrics (totalMemoryMB, freeMemoryMB), generate ISO-8601 UTC timestamp, and return HealthStatusResponse DTO. Uptime must never decrease; memory metrics are fresh on each request (no caching). If memory access fails, return 0 values and log WARN.

**Acceptance Criteria**:
- [ ] Service class created with @Service annotation
- [ ] Constructor injection of HealthStatusRepository via Lombok @RequiredArgsConstructor
- [ ] getHealthStatus() method implemented with @Transactional(readOnly = true)
- [ ] Uptime calculation using application startup time (ManagementFactory.getRuntimeMXBean().getUptime())
- [ ] Uptime formatted as human-readable XhYmZs string with zero components omitted
- [ ] Memory metrics obtained via Runtime.getRuntime().totalMemory() and freeMemory()
- [ ] Memory values converted to MB (integer division by 1024 * 1024)
- [ ] Timestamp generated in UTC using ZonedDateTime.now(ZoneId.of("UTC"))
- [ ] Timestamp formatted as ISO-8601 pattern: yyyy-MM-dd'T'HH:mm:ss'Z'
- [ ] Validation: freeMemoryMB <= totalMemoryMB
- [ ] Error handling: If memory access fails, return 0 values and log WARN-level message
- [ ] Response time < 5ms average, < 10ms p99
- [ ] No caching (@Cacheable annotation NOT used)
- [ ] Unit tests with mocked Runtime verify uptime, memory, timestamp calculations
- [ ] Unit test verifies memory constraint (free <= total)
- [ ] Unit test verifies error handling (memory access failure)
- [ ] Minimum 90% code coverage for service class

**Test Files**:
- `src/test/java/ar/com/nanotaboada/java/samples/spring/boot/services/HealthStatusServiceTest.java`

**Reference**: [Implementation Plan](./plan.md) section "Phase 1 Design - Task 1" and [Data Model](./data-model.md) field specifications

---

## Phase 4: Controller Layer (HTTP Handler)

### Task 4: Create HealthController
**ID**: [T004]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `backend`, `controller`, `api`, `rest`  
**Effort**: 2 story points (30 minutes)  
**Dependencies**: Task 3 (HealthStatusService)

**Description**:
Create controller class `HealthController` in `src/main/java/ar/com/nanotaboada/java/samples/spring/boot/controllers/HealthController.java` with GET endpoint `/api/v1/status`. Controller must delegate to HealthStatusService for business logic, return ResponseEntity with HTTP 200 OK status, and include SpringDoc OpenAPI annotations for Swagger documentation. No authentication or authorization required (public endpoint).

**Acceptance Criteria**:
- [ ] Controller class created with @RestController annotation
- [ ] Constructor injection of HealthStatusService via Lombok @RequiredArgsConstructor
- [ ] GET endpoint `getHealthStatus()` mapped to `/api/v1/status` via @GetMapping
- [ ] Method returns ResponseEntity<HealthStatusResponse> with HTTP 200 status
- [ ] Method calls healthStatusService.getHealthStatus() and wraps result in ResponseEntity.ok()
- [ ] @Operation annotation added with summary and description
- [ ] @ApiResponse annotations document HTTP 200 response
- [ ] @Schema annotation on HealthStatusResponse DTO in response
- [ ] No @RequestMapping class-level annotation needed (uses @RestController convention)
- [ ] No authorization checks; endpoint is public
- [ ] No error handling in controller (@ControllerAdvice handles errors globally)
- [ ] Only @GetMapping present (POST, PUT, DELETE, PATCH not supported; Spring returns HTTP 405)
- [ ] Integration test with MockMvc verifies endpoint returns HTTP 200
- [ ] Integration test verifies Content-Type is application/json
- [ ] Integration test verifies all four required fields in response JSON
- [ ] Integration test verifies field values are non-empty/non-null

**Test Files**:
- `src/test/java/ar/com/nanotaboada/java/samples/spring/boot/controllers/HealthControllerTest.java`

**Reference**: [Implementation Plan](./plan.md) section "Phase 1 Design - Task 2 (Contracts)" and [Data Model](./data-model.md) section "Example Request/Response Flows"

---

## Phase 5: Integration Testing

### Task 5: Write Integration Tests for HealthStatusService
**ID**: [T005]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `testing`, `unit-test`, `service`, `metrics`  
**Effort**: 2 story points (45 minutes)  
**Dependencies**: Task 3 (HealthStatusService)

**Description**:
Create unit test class `HealthStatusServiceTest` with MockedStatic<Runtime> to verify service layer logic. Tests must validate uptime calculation (human-readable format), memory metric computation, timestamp generation (ISO-8601 UTC), constraint validation (free <= total), and error handling when memory access fails.

**Acceptance Criteria**:
- [ ] Test class created: HealthStatusServiceTest.java
- [ ] @ExtendWith(MockitoExtension.class) annotation for JUnit 5
- [ ] Unit test for uptime calculation: "0s" at startup, increases monotonically
- [ ] Unit test for uptime format: XhYmZs pattern, zero components omitted
- [ ] Unit test for totalMemoryMB: positive integer, calculated from Runtime
- [ ] Unit test for freeMemoryMB: non-negative integer, <= totalMemoryMB
- [ ] Unit test for timestamp: ISO-8601 UTC format (yyyy-MM-dd'T'HH:mm:ss'Z')
- [ ] Unit test for memory constraint: throws IllegalStateException if free > total
- [ ] Unit test for error handling: returns 0 for memory values if Runtime access fails
- [ ] Unit test verifies log message (WARN level) when memory fails
- [ ] All unit tests use @Test annotation (JUnit 5)
- [ ] Test assertions use AssertJ (then(...).isEqualTo(...), etc.)
- [ ] Mocking uses Mockito MockedStatic for Runtime.getRuntime()
- [ ] Code coverage for HealthStatusService >= 90%

**Test Files**:
- `src/test/java/ar/com/nanotaboada/java/samples/spring/boot/services/HealthStatusServiceTest.java`

**Reference**: [Data Model](./data-model.md) section "Testing Data"

---

### Task 6: Write Integration Tests for HealthController
**ID**: [T006]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `testing`, `integration-test`, `controller`, `rest`  
**Effort**: 2 story points (45 minutes)  
**Dependencies**: Task 4 (HealthController)

**Description**:
Create integration test class `HealthControllerTest` using Spring MockMvc to verify HTTP endpoint contract. Tests must validate HTTP 200 response, Content-Type application/json, all required JSON fields present, field types (String, Integer, String), and concurrent request handling.

**Acceptance Criteria**:
- [ ] Test class created: HealthControllerTest.java
- [ ] @SpringBootTest annotation for integration testing
- [ ] @AutoConfigureMockMvc annotation for MockMvc auto-configuration
- [ ] Test method: GET /api/v1/status returns HTTP 200 OK
- [ ] Test verifies Content-Type header is application/json
- [ ] Test verifies JSON response contains four required fields: uptime, totalMemoryMB, freeMemoryMB, timestamp
- [ ] Test uses MockMvc.perform(get("/api/v1/status"))
- [ ] Test assertions use status().isOk() and content().contentType(MediaType.APPLICATION_JSON)
- [ ] Test uses jsonPath() to verify field presence and types
- [ ] Test verifies freeMemoryMB <= totalMemoryMB
- [ ] Test for concurrent requests: simulates 10+ concurrent GET requests (ThreadPoolTaskExecutor or thread loop)
- [ ] All concurrent requests complete successfully with HTTP 200
- [ ] Test verifies response time < 10ms for each request
- [ ] Test for invalid HTTP method: POST /api/v1/status returns HTTP 405
- [ ] Test response payload size < 500 bytes
- [ ] MockMvc test verifies no request body required
- [ ] MockMvc test verifies no query parameters required
- [ ] Tests use @Test annotation (JUnit 5)

**Test Files**:
- `src/test/java/ar/com/nanotaboada/java/samples/spring/boot/controllers/HealthControllerTest.java`

**Reference**: [Data Model](./data-model.md) section "Example Request/Response Flows"

---

## Phase 6: Documentation & API Contracts

### Task 7: Verify Swagger/OpenAPI Generation
**ID**: [T007]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `documentation`, `swagger`, `api-contract`  
**Effort**: 1 story point (20 minutes)  
**Dependencies**: Task 4 (HealthController with @Operation annotations)

**Description**:
Verify that Swagger/OpenAPI documentation is auto-generated from code annotations. The endpoint `/api/v1/status` must appear in OpenAPI schema (`/v3/api-docs`) with correct HTTP method (GET), response status (200), and HealthStatusResponse schema.

**Acceptance Criteria**:
- [ ] Application starts without errors
- [ ] `/v3/api-docs` endpoint responds with valid OpenAPI 3.0.1 JSON schema
- [ ] OpenAPI schema contains path entry for `/api/v1/status`
- [ ] GET method is documented under /api/v1/status
- [ ] Response 200 is documented with HealthStatusResponse schema
- [ ] Response schema includes all four fields: uptime, totalMemoryMB, freeMemoryMB, timestamp
- [ ] Field types are correct: string, integer, integer, string
- [ ] @Operation summary and description are present in schema
- [ ] Swagger UI (`/swagger-ui.html`) displays endpoint documentation
- [ ] Swagger UI "Try it out" button allows test requests
- [ ] Test endpoint via Swagger UI returns HTTP 200 with valid JSON

**Reference**: [Implementation Plan](./plan.md) section "Phase 1 Design - Task 2 (Contracts)"

---

### Task 8: Update CHANGELOG.md
**ID**: [T008]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `documentation`, `changelog`  
**Effort**: 1 story point (15 minutes)  
**Dependencies**: Task 7 (Verification complete)

**Description**:
Add entry to `CHANGELOG.md` under `[Unreleased]` section documenting the new health status endpoint feature. Entry must reference feature branch, issue number (if applicable), and summary of changes (new GET /api/v1/status endpoint, HealthStatusResponse DTO, HealthStatusService, HealthController).

**Acceptance Criteria**:
- [ ] CHANGELOG.md updated with [Unreleased] section entry
- [ ] Entry added under "Added" subsection
- [ ] Entry references feature: "Health and System Resource Status Endpoint"
- [ ] Entry references endpoint: GET /api/v1/status
- [ ] Entry lists new classes: HealthStatusResponse DTO, HealthStatusService, HealthController
- [ ] Entry references branch: 001-health-status-endpoint
- [ ] Entry includes link to issue (if applicable)
- [ ] CHANGELOG format follows existing pattern (Markdown, ISO date format)
- [ ] No trailing whitespace or formatting errors

**Reference**: Contributing guide for CHANGELOG format

---

### Task 9: Update README.md with Health Endpoint Documentation
**ID**: [T009]  
**Priority**: [P2]  
**Story**: [US1]  
**Labels**: `documentation`, `readme`  
**Effort**: 1 story point (20 minutes)  
**Dependencies**: Task 7 (Verification complete)

**Description**:
Add health endpoint documentation to `README.md` in an "API Endpoints" or "Health Check" section. Documentation must include endpoint path, HTTP method, description, example request/response, and curl command for manual testing.

**Acceptance Criteria**:
- [ ] README.md updated with new "Health Endpoint" or "API Endpoints" section
- [ ] Section includes endpoint path: GET /api/v1/status
- [ ] Section includes description: public endpoint for application health and JVM metrics
- [ ] Example curl command: curl -s http://localhost:9000/api/v1/status | jq '.'
- [ ] Example JSON response shown with all four fields
- [ ] Response fields documented: uptime (human-readable format), totalMemoryMB, freeMemoryMB, timestamp (ISO-8601 UTC)
- [ ] HTTP 200 status documented
- [ ] Note about no authentication required
- [ ] Note about response time: <10ms p99
- [ ] README follows existing formatting and structure

---

## Phase 7: Final Verification & Merge

### Task 10: End-to-End Manual Testing (Quickstart Validation)
**ID**: [T010]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `testing`, `qa`, `verification`  
**Effort**: 2 story points (45 minutes)  
**Dependencies**: All tasks 1-9 complete

**Description**:
Execute manual end-to-end validation scenarios from `quickstart.md` to verify the complete implementation. Tests include: endpoint returns valid JSON, uptime increases over time, memory metrics are non-zero, timestamp is ISO-8601 UTC formatted, invalid HTTP methods return 405, concurrent requests handled, and Swagger docs are available.

**Acceptance Criteria**:
- [ ] Application built and started: `./mvnw spring-boot:run`
- [ ] Scenario 1: GET /api/v1/status returns JSON with all required fields
- [ ] Scenario 2: Uptime increases monotonically when queried twice with delay
- [ ] Scenario 3: totalMemoryMB and freeMemoryMB are positive integers
- [ ] Scenario 4: Timestamp matches ISO-8601 format and is current (within 1 second)
- [ ] Scenario 5: POST /api/v1/status returns HTTP 405 Method Not Allowed
- [ ] Scenario 6: 10 concurrent requests to /api/v1/status all succeed with HTTP 200
- [ ] Scenario 7: Swagger documentation available at /swagger-ui.html
- [ ] All tests passed with no errors or exceptions
- [ ] Response time confirmed < 10ms for each request
- [ ] No log warnings or errors observed

**Test Files**: Quickstart scenarios in [quickstart.md](./quickstart.md)

**Reference**: [Implementation Plan](./plan.md) and [Quickstart Guide](./quickstart.md)

---

### Task 11: Code Review & Quality Checks
**ID**: [T011]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `code-review`, `quality`, `verification`  
**Effort**: 2 story points (1 hour)  
**Dependencies**: Tasks 1-9 complete, Task 10 (manual testing) passed

**Description**:
Perform code review and quality checks: verify layered architecture (Controller → Service → Repository), confirm no business logic in controller, validate Lombok usage (@Value, @Builder, @RequiredArgsConstructor), check Jakarta namespace imports (no javax.*), verify SLF4J logging (no System.out), confirm 80%+ test coverage, run Maven build and tests, verify no new dependencies added.

**Acceptance Criteria**:
- [ ] All code follows layered architecture: Controller → Service → Repository
- [ ] No business logic in HealthController (only HTTP marshaling)
- [ ] HealthStatusResponse uses @Value for immutability
- [ ] HealthStatusService uses @RequiredArgsConstructor for constructor injection
- [ ] All imports use Jakarta namespace (jakarta.*, not javax.*)
- [ ] All logging uses SLF4J (@Slf4j or LoggerFactory), no System.out.println
- [ ] No @Autowired field injection (only constructor injection)
- [ ] No @Cacheable on service method
- [ ] Maven build passes: `./mvnw clean verify`
- [ ] All unit and integration tests pass
- [ ] Code coverage >= 80% for new service and controller classes
- [ ] No new dependencies added to pom.xml
- [ ] No compilation warnings or errors
- [ ] Code formatted per project conventions (if applicable)
- [ ] No commented-out code blocks

**Reference**: [Implementation Plan](./plan.md) section "Constitution Check"

---

### Task 12: Merge to Main Branch
**ID**: [T012]  
**Priority**: [P1]  
**Story**: [US1]  
**Labels**: `release`, `merge`, `ci-cd`  
**Effort**: 1 story point (15 minutes)  
**Dependencies**: Tasks 1-11 complete, code review approved

**Description**:
Merge feature branch `001-health-status-endpoint` to main branch after all quality checks pass and code review is approved. Verify CI/CD pipeline runs successfully and all GitHub Actions checks pass (build, test, coverage).

**Acceptance Criteria**:
- [ ] Code review approved by at least one reviewer
- [ ] All GitHub Actions checks pass (CI/CD pipeline green)
- [ ] Maven build succeeds with `./mvnw clean verify`
- [ ] All tests pass (unit and integration)
- [ ] Code coverage meets minimum threshold (80%)
- [ ] Feature branch `001-health-status-endpoint` rebased/up-to-date with main
- [ ] Merge commit created with message referencing feature and issues (if applicable)
- [ ] CHANGELOG.md and README.md updated and committed
- [ ] No merge conflicts
- [ ] Post-merge CI/CD pipeline runs without errors
- [ ] Feature is live on main branch

**Reference**: Contributing guide for merge process

---

## Implementation Dependencies & Execution Order

### Dependency Graph

```
T001 (HealthStatusResponse DTO)
  ↓
T002 (Repository Stub) ← T001
T003 (Service)         ← T001, T002
  ↓
T004 (Controller)      ← T003
  ↓
T005 (Service Tests)   ← T003
T006 (Controller Tests) ← T004
  ↓
T007 (Swagger Verification) ← T004
T008 (CHANGELOG)       ← T007
T009 (README)          ← T007
  ↓
T010 (E2E Manual Testing) ← T001-T009
T011 (Code Review)      ← T001-T010
  ↓
T012 (Merge to Main)    ← T001-T011
```

### Parallel Execution Opportunities

**Can run in parallel**:
- T005 (Service Tests) and T006 (Controller Tests) can run simultaneously after their dependencies are satisfied
- T008 (CHANGELOG) and T009 (README) can run in parallel after T007

**Suggested parallel phases**:
- **Phase 1**: T001 (DTO) 
- **Phase 2**: T002 (Repository) + T003 (Service) 
- **Phase 3**: T004 (Controller) + T005 (Service Tests) + T006 (Controller Tests) 
- **Phase 4**: T007 (Swagger) + T008 (CHANGELOG) + T009 (README) 
- **Phase 5**: T010 (E2E) 
- **Phase 6**: T011 (Code Review) 
- **Phase 7**: T012 (Merge)

---

## Quickstart Execution

### For a Developer Starting This Feature

1. **Create feature branch**: `git checkout -b 001-health-status-endpoint`

2. **Phase 1 - DTO** (30 minutes):
   - Implement T001: HealthStatusResponse.java
   - Run tests: `./mvnw test -Dtest=HealthStatusResponseTest`

3. **Phase 2 - Repository** (15 minutes):
   - Implement T002: HealthStatusRepository.java

4. **Phase 3 - Service** (1 hour):
   - Implement T003: HealthStatusService.java
   - Run tests: `./mvnw test -Dtest=HealthStatusServiceTest`

5. **Phase 4 - Controller** (30 minutes):
   - Implement T004: HealthController.java

6. **Phase 5 - Integration Tests** (1.5 hours, parallel):
   - Implement T005: HealthStatusServiceTest.java
   - Implement T006: HealthControllerTest.java
   - Run all tests: `./mvnw test`

7. **Phase 6 - Documentation** (1 hour, parallel):
   - T007: Verify Swagger (automated)
   - T008: Update CHANGELOG.md
   - T009: Update README.md

8. **Phase 7 - Verification & Merge** (2 hours):
   - T010: Manual E2E testing via curl and Swagger UI
   - T011: Code review and quality checks
   - T012: Merge to main

### Total Estimated Time

- **Ideal (with parallelization)**: ~4-5 hours
- **Sequential**: ~6-7 hours
- **With code review and CI/CD**: ~8-10 hours (including wait time)

---

## Success Metrics

Upon completion of all tasks, the implementation will achieve:

✅ **Functional**: Public health endpoint operational at GET /api/v1/status  
✅ **Performance**: Response time < 5ms average, < 10ms p99  
✅ **Quality**: Unit and integration test coverage >= 80%  
✅ **Architecture**: Strict layered design (Controller → Service → Repository)  
✅ **Documentation**: Swagger/OpenAPI, README, CHANGELOG updated  
✅ **Reliability**: Error handling for memory access failures with graceful degradation  
✅ **Observability**: Real-time JVM metrics exposed; no caching  
✅ **Production-Ready**: Zero breaking changes; feature isolated from existing endpoints

---

## GitHub Issues Template

For each task above, use this template when creating GitHub issues:

```markdown
## Title
[Feature] {Task Title}

## Description
{Task Description}

## Acceptance Criteria
- [ ] {Criterion 1}
- [ ] {Criterion 2}
...

## Effort Estimate
{Story Points} story points ({Time estimate})

## Dependencies
- [ ] Depends on issue #{Related Issue Number} ({Task Title})

## Labels
- `backend`
- `{other labels}`

## Related Documents
- [Specification](./spec.md)
- [Implementation Plan](./plan.md)
- [Data Model](./data-model.md)

## Implementation Notes
{Any additional notes or implementation guidance}
```

---

**Document Version**: 1.0  
**Last Updated**: 2026-07-02  
**Status**: Ready for GitHub Issues Creation  
**Total Tasks**: 12  
**Estimated Total Effort**: 20 story points (~4-5 hours with parallelization)
