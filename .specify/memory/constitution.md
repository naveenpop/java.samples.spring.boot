# Java Spring Boot Football REST API — Constitution

A REST API for managing football players, implementing CRUD operations with a strict layered architecture, reactive caching, and comprehensive documentation. Part of a cross-language architectural comparison study.

## Core Principles

### I. Strict Layered Architecture (Non-Negotiable)

The application enforces an immutable layered architecture: **Controller → Service → Repository → JPA → Database**. Each layer has exclusive responsibility:

- **Controller Layer** (HTTP): Request/response handling, input validation routing, global exception handling via `@ControllerAdvice`
- **Service Layer** (Business Logic): All business logic, transactional boundaries, caching via `@Cacheable`
- **Repository Layer** (Data Access): Spring Data JPA derived queries, no business logic
- **JPA/Entity Layer**: Entity definitions with `@Entity` annotations, Hibernate-managed relationships

**Violations**:
- Controllers accessing repositories directly → forbidden
- Controllers containing business logic → forbidden
- Services accessing HTTP details → forbidden
- Repositories containing computed values → forbidden

### II. Contractual API Design (Non-Negotiable)

API contracts (HTTP endpoints, status codes, DTO shapes) are fixed and immutable unless explicitly discussed and approved. Breaking changes require:

1. Explicit user approval before implementation
2. Semantic versioning bump
3. Documentation update
4. Migration guide for clients

DTOs are the exclusive contract interface—entities are never exposed in controller signatures. All responses serialize via DTOs.

### III. Test-First Development (Mandatory)

- **Red-Green-Refactor**: Tests written and fail → implementation → tests pass → refactor
- **BDD Naming**: Test method names follow `givenX_whenY_thenZ` convention
- **AssertJ BDD Style**: `then(result).isNotNull()` preferred over `assertNotNull(result)`
- **Coverage Gates**: JaCoCo enforces minimum coverage; all new code requires tests before merge
- **Test Database**: In-memory SQLite auto-clears after each test; Flyway disabled; Spring SQL init (`ddl.sql`, `dml.sql`) seeds test data

### IV. Explicit Dependency Injection (Jakarta Namespace)

- Constructor injection via Lombok `@RequiredArgsConstructor`; never field injection
- Spring beans instantiated by framework; `new` keyword forbidden for beans
- Jakarta Servlet API (`jakarta.servlet.*`), Jakarta Persistence (`jakarta.persistence.*`) exclusively
- No org.apache.servlet or javax.* imports

### V. Observability & Logging

- SLF4J exclusively; `System.out.println` and `System.err` forbidden
- Structured logging; log levels: DEBUG (detailed flow), INFO (significant events), WARN (recoverable issues), ERROR (failures)
- Logback configuration in `src/main/resources/logback-spring.xml`
- All exceptions logged before propagation

### VI. Data Integrity & Transactions

- Read methods marked `@Transactional(readOnly = true)` for clarity and optimization
- Write methods marked `@Transactional`; Hibernate manages flush/rollback
- Schema changes via Flyway migrations (`db/migration/V{N}__description.sql`); manual database edits forbidden
- Production SQLite file (`storage/`) owned by Flyway; never modified directly

### VII. Clean Code & Consistency

- **Naming**: camelCase (methods/variables), PascalCase (classes), UPPER_SNAKE_CASE (constants)
- **Files**: Class name matches file name exactly
- **Comments**: Only for clarification; never state obvious code
- **DTOs**: Immutable, no business logic; Bean Validation annotations only

## Technology Stack (Locked)

| Component | Version/Choice | Rationale |
|-----------|-----------------|-----------|
| **Language** | Java 25 (LTS) | Long-term support, modern features, garbage collection improvements |
| **Framework** | Spring Boot 4.0.0 | Modular architecture; `jakarta.*` namespace; `spring-boot-starter-webmvc` |
| **Web Framework** | Spring MVC 6.0 | Jakarta Servlet API; no reactive requirements (Webflux not used) |
| **ORM** | Spring Data JPA 6.0 + Hibernate | Derived queries, reduced boilerplate, JPA contract |
| **Connection Pooling** | Apache DBCP2 | Robust pooling with idle eviction, connection validation |
| **Database (Runtime)** | SQLite (file-based) | Single-file deployment, zero configuration |
| **Database (Test)** | SQLite (in-memory) | Isolation, auto-cleanup between tests, parity with production |
| **Migrations** | Flyway 10.x | Version-controlled schema evolution; production only (disabled in tests) |
| **Dependency Injection** | Spring Core 6.0 | Constructor injection, container-managed lifecycle |
| **Validation** | Jakarta Bean Validation (JSR-380) | Declarative constraints via annotations |
| **Caching** | Spring `@Cacheable` (simple in-memory) | No TTL; manual invalidation via `@CacheEvict` |
| **Testing** | JUnit 5 + Mockito + AssertJ | Modern assertions, mocking, parameterized tests |
| **Testing (Web)** | MockMvc + WebTestClient | HTTP layer testing without network I/O |
| **Coverage** | JaCoCo 0.8.x | Code coverage reporting; gating on minimum threshold |
| **Mapping** | ModelMapper | DTO ↔ Entity conversion, reducible by build plugins if needed |
| **API Docs** | SpringDoc OpenAPI 3.0 | Auto-generated Swagger/OpenAPI from annotations |
| **Build** | Maven 3 (wrapper `./mvnw`) | Reproducible builds, dependency management |
| **Logging** | SLF4J + Logback | Structured logging, log levels, appender flexibility |
| **Boilerplate** | Lombok (annotation processor) | Reduces verbosity; `@Data`, `@Builder`, `@RequiredArgsConstructor` |
| **Runtime** | Docker | Single-stage production image; port 9000 |

## Project Structure

```text
src/main/java/
├── controllers/        HTTP handlers; delegate to services; no business logic
├── services/           Business logic; @Transactional; @Cacheable
├── repositories/       Spring Data JPA; derived queries only
├── models/             @Entity (Player); DTOs; validators
└── converters/         JPA AttributeConverter (e.g., ISO-8601 date)

src/main/resources/
├── application.properties
├── application-{profile}.properties
├── logback-spring.xml
└── db/migration/       Flyway versions (V1 schema; V2+ data)

src/test/java/          Unit/integration tests mirroring main structure
src/test/resources/     application-test.properties, ddl.sql, dml.sql

storage/                SQLite file (runtime; Flyway-managed; never edit manually)
docs/adr/               Architectural Decision Records (14 ADRs)
```

## Development Workflow

### Naming & Commits

- **Format**: `type(scope): description (#issue)` — max 80 characters
- **Types**: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `ci`, `perf`
- **Example**: `feat(api): add player stats endpoint (#42)`
- **Co-author**: Include co-author trailer (Copilot or contributor)

### Pre-Commit Checklist

Before every commit:

1. `./mvnw clean install` succeeds (no compilation errors/warnings)
2. All tests pass: `./mvnw clean test`
3. Coverage meets threshold: `./mvnw clean test jacoco:report` → `target/site/jacoco/index.html`
4. `CHANGELOG.md` `[Unreleased]` section updated (added under Added/Changed/Fixed/Removed)
5. Commit message follows Conventional Commits format (enforced by commitlint)
6. If architectural: update `CLAUDE.md` + create/amend ADR in `docs/adr/`

### Adding an Endpoint (Workflow)

1. **DTO** (models/): Define request/response with Bean Validation annotations
2. **Service**: Implement business logic in `services/PlayerService.java` with `@Transactional`
3. **Repository**: Add derived query if needed (rarely; most use Spring Data defaults)
4. **Controller**: Create endpoint with `@RestController`, `@Operation` (Swagger), route to service
5. **Test**: Write MockMvc integration test; given-when-then style
6. **Validate**: `./mvnw clean test jacoco:report`; ensure new code covered

### Modifying Database Schema (Workflow)

1. **Migration**: Create `src/main/resources/db/migration/V{N}__description.sql` (production path only)
2. **Entity**: Update `@Entity` in `models/Player.java`; adjust relationships
3. **DTOs**: Update request/response DTOs if API contract changes
4. **Test Init**: Also update `src/test/resources/ddl.sql` and `dml.sql` (tests use Spring SQL init, not Flyway)
5. **Service/Repo/Tests**: Update service methods, repository, and integration tests
6. **Validate**: `./mvnw clean test` (Flyway disabled; Spring SQL init runs instead)
7. **Never**: Manually edit `storage/` SQLite file; Flyway owns schema versioning

### Quick Commands

```bash
./mvnw spring-boot:run                  # Start on port 9000
./mvnw clean test                       # Run all tests
./mvnw clean test jacoco:report         # Tests + coverage
open target/site/jacoco/index.html      # View coverage report
./mvnw clean package -DskipTests        # Build JAR
docker compose up && docker compose down -v  # Local container
```

## Agent Permissions

### ✅ Proceed Freely

- Route handlers and controller endpoints
- Service layer business logic
- Repository custom queries (Spring Data derived)
- Unit and integration tests
- Exception handling in `@ControllerAdvice`
- Documentation updates, bug fixes, refactoring
- Utility classes and helpers
- Logging and observability enhancements

### ⚠️ Ask Before Changing

- Database schema (entity fields, relationships, migrations)
- Dependencies (`pom.xml`), dependency versions
- CI/CD configuration (`.github/workflows/`)
- Docker setup (Dockerfile, compose.yaml)
- Application properties and profiles
- API contracts (endpoint signatures, status codes, DTO shapes)
- Caching strategy, TTL values, cache invalidation rules
- Package structure or module boundaries
- Port configuration (9000 reserved for API, 9001 for test)

### 🚫 Never Modify

- `.java-version` — JDK 25 is fixed
- Maven wrapper scripts (`mvnw`, `mvnw.cmd`)
- Port 9000 (API) or 9001 (test) — reserved
- Test database configuration (in-memory SQLite)
- Flyway baseline or history (production database versioning)
- Production configuration or deployment secrets

## Issue & PR Workflow

### Spec-Driven Development

**Mandatory Process**: Discuss → Issue (GitHub artifact) → Implementation

**Feature Request** (label: `enhancement`):
- **Problem**: Specific pain point being solved
- **Proposed Solution**: Expected behavior and functionality
- **Suggested Approach** (optional): Implementation plan
- **Acceptance Criteria**: Minimum—behaves as proposed, tests added, no regressions
- **References**: Related issues, documentation, examples

**Bug Report** (label: `bug`):
- **Description**: Clear summary
- **Steps to Reproduce**: Numbered, minimal steps
- **Expected/Actual Behavior**: Side-by-side
- **Environment**: Java version, OS, Spring Boot version
- **Additional Context**: Logs, stack traces
- **Possible Solution** (optional): Suggested fix

## Invariants (Immutable)

These invariants define the project's identity and may only change via explicit, documented discussion:

- **Port**: 9000 (HTTP API), 9001 (test infrastructure)
- **API Contract**: Endpoints, HTTP status codes, DTO shapes fixed; breaking changes require explicit approval
- **Commit Format**: `type(scope): description (#issue)` — max 80 characters
- **Conventional Types**: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `ci`, `perf` (no custom types)
- **Changelog**: `CHANGELOG.md` `[Unreleased]` section updated before every commit (Added/Changed/Fixed/Removed subsections)
- **Language**: Java 25 LTS (enforced via `.java-version`)
- **Framework**: Spring Boot 4.0.0 with `jakarta.*` namespace
- **Layer Rule**: `Controller → Service → Repository → JPA`; no shortcuts

## Governance

### Constitution Authority

This constitution supersedes all other development guidance (CLAUDE.md, ADRs, README.md). In the case of conflict, **Constitution > CLAUDE.md > ADR > README**.

### Amendment Process

1. **Proposal**: Document rationale for change
2. **Approval**: Team consensus on impact and migration path
3. **Documentation**: Update this file + CLAUDE.md
4. **Migration**: Create issue(s) for implementation; track compliance
5. **Versioning**: Increment constitution version (semantic versioning)

### Compliance Verification

- **Code Review**: All PRs verified against Architecture Decision Records and this constitution
- **Linting/Testing**: Pre-commit checklist automated via GitHub Actions
- **Coverage**: JaCoCo gates enforce minimum thresholds
- **Migrations**: Flyway manages schema evolution; manual database changes rejected

### Related Documentation

- **CLAUDE.md**: Runtime development guidance (less formal than constitution; may be updated freely)
- **Architecture Decision Records** (docs/adr/): The "why" behind major choices (framework selection, layering, persistence)
- **README.md**: User-facing project overview and quick-start guide
- **CONTRIBUTING.md**: Contributor guidelines, PR process, local development setup

---

**Version**: 1.0.0 | **Ratified**: 2026-07-02 | **Last Amended**: 2026-07-02

**Last Updated By**: Copilot | **Review Cadence**: Quarterly (or as needed per amendment process)
