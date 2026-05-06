---
name: java-development
description: "Complete Java/Spring Boot development skill: coding standards, naming, immutability, sealed classes, Spring Boot patterns (REST, services, JPA, virtual threads, structured concurrency, weakest-link awareness), unit testing (JUnit 5, Mockito, AssertJ, MockMvc), TDD workflow, code formatting (Spotless), and verification loop (build, static analysis, coverage, security scans)."
---

# Java & Spring Boot Development

Complete development guide for Java 21+ / Spring Boot services in MyProject.

## When to Activate

- Writing, reviewing, or refactoring Java code in Spring Boot projects
- Creating REST APIs, services, repositories, or entities
- Modelling domain hierarchies with sealed classes and pattern matching
- Writing unit tests, web layer tests, or integration tests
- Designing concurrent workloads with virtual threads and structured concurrency
- Running build/test/verification pipelines
- Formatting Java source code

---

# Part 1: Coding Standards

Standards for readable, maintainable Java (21+) code in Spring Boot services.

## Core Principles

- Prefer clarity over cleverness
- Immutable by default; minimize shared mutable state
- Fail fast with meaningful exceptions
- Consistent naming and package structure

## Naming

```java
// Classes/Records: PascalCase
public class MarketService {}
public record Money(BigDecimal amount, Currency currency) {}

// Methods/fields: camelCase
private final MarketRepository marketRepository;
public Market findBySlug(String slug) {}

// Constants: UPPER_SNAKE_CASE
private static final int MAX_PAGE_SIZE = 100;
```

## Immutability

```java
// Favor records and final fields
public record MarketDto(Long id, String name, MarketStatus status) {}

public class Market {
  private final Long id;
  private final String name;
  // getters only, no setters
}
```

## Sealed Classes

Use sealed classes to model closed domain hierarchies — replacing `enum` when variants carry different data, and making exhaustive `switch` expressions compiler-enforced.

```java
// Sealed interface for a domain result hierarchy
public sealed interface MarketResult
    permits MarketResult.Success, MarketResult.NotFound, MarketResult.ValidationFailed {

  record Success(Market market) implements MarketResult {}
  record NotFound(String slug) implements MarketResult {}
  record ValidationFailed(List<String> errors) implements MarketResult {}
}

// Exhaustive switch — compiler enforces all cases
MarketResponse response = switch (result) {
    case MarketResult.Success(var market)           -> MarketResponse.from(market);
    case MarketResult.NotFound(var slug)            -> throw new EntityNotFoundException(slug);
    case MarketResult.ValidationFailed(var errors)  -> throw new ValidationException(errors);
};
```

### When to use sealed classes

| Use sealed | Don't bother |
|---|---|
| Closed set of domain states (e.g. `OrderStatus` variants with payloads) | Simple flag/status with no payload → use `enum` |
| API response discriminated unions | Open extension points / plugin architectures |
| Error / result types carrying context | Pure data objects → use `record` |

### Rules

- Always `sealed` + `permits` — never leave the hierarchy open by accident
- Prefer `record` permits for immutable variants; use `final class` only when mutable state is truly needed
- Pair with pattern-matching `switch` — never `instanceof` chains
- Do **not** use `sealed` for Spring-managed beans (controllers, services, repositories) — Spring proxies require open classes

## Optional Usage

```java
// Return Optional from find* methods
Optional<Market> market = marketRepository.findBySlug(slug);

// Map/flatMap instead of get()
return market
    .map(MarketResponse::from)
    .orElseThrow(() -> new EntityNotFoundException("Market not found"));
```

## Streams Best Practices

```java
// Use streams for transformations, keep pipelines short
List<String> names = markets.stream()
    .map(Market::name)
    .filter(Objects::nonNull)
    .toList();

// Avoid complex nested streams; prefer loops for clarity
```

## Exceptions

- Use unchecked exceptions for domain errors; wrap technical exceptions with context
- Create domain-specific exceptions (e.g., `MarketNotFoundException`)
- Avoid broad `catch (Exception ex)` unless rethrowing/logging centrally

```java
throw new MarketNotFoundException(slug);
```

## Generics and Type Safety

- Avoid raw types; declare generic parameters
- Prefer bounded generics for reusable utilities

```java
public <T extends Identifiable> Map<Long, T> indexById(Collection<T> items) { ... }
```

## Project Structure (Gradle)

```
src/main/java/com/your-org/MyProject/<service-name>/
  config/
  controller/
  service/
  repository/
  entity/
  dto/
  mapper/
  exception/
  util/
src/main/resources/
  application.yaml
src/test/java/... (mirrors main)
```

## Null Handling

- Accept `@Nullable` only when unavoidable; otherwise use `@NonNull`
- Use Bean Validation (`@NotNull`, `@NotBlank`) on inputs

## Code Smells to Avoid

- Long parameter lists → use DTO/builders
- Deep nesting → early returns
- Magic numbers → named constants
- Static mutable state → prefer dependency injection
- Silent catch blocks → log and act or rethrow
- `synchronized` blocks → use `java.util.concurrent` primitives (`ReentrantLock`, `ConcurrentHashMap`, `AtomicReference`) or redesign to eliminate shared mutable state
- `static { }` initializer blocks → move initialization to constructors, `@PostConstruct`, or Spring `@Bean` factory methods; they hide failures and complicate testing
- Reactive types (`Mono`, `Flux`) → use virtual threads (see Part 2 — Async & Concurrency)
- Unclosed resources → always use `try-with-resources` for any `Closeable`
- `ThreadLocal` without `remove()` → leaks per-request data; use `ScopedValue` (Java 21+) where possible
- `static` collections / caches without eviction → use `Caffeine` with `maximumSize` + TTL
- `HttpClient` / `RestClient` created inside methods → declare as singleton `@Bean`
- Event listeners registered but never deregistered → call deregister in `@PreDestroy`
- `EAGER` fetch on large `@OneToMany` associations → default to `LAZY`

---

# Part 2: Spring Boot Patterns

Architecture and API patterns for scalable, production-grade services.

## REST API Structure

```java
@RestController
@RequestMapping("/api/markets")
@Validated
class MarketController {
  private final MarketService marketService;

  MarketController(MarketService marketService) {
    this.marketService = marketService;
  }

  @GetMapping
  ResponseEntity<Page<MarketResponse>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    Page<Market> markets = marketService.list(PageRequest.of(page, size));
    return ResponseEntity.ok(markets.map(MarketResponse::from));
  }

  @PostMapping
  ResponseEntity<MarketResponse> create(@Valid @RequestBody CreateMarketRequest request) {
    Market market = marketService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(MarketResponse.from(market));
  }
}
```

## Repository Pattern (Spring Data JPA)

```java
public interface MarketRepository extends JpaRepository<MarketEntity, Long> {
  @Query("select m from MarketEntity m where m.status = :status order by m.volume desc")
  List<MarketEntity> findActive(@Param("status") MarketStatus status, Pageable pageable);
}
```

## Service Layer with Transactions

```java
@Service
public class MarketService {
  private final MarketRepository repo;

  public MarketService(MarketRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public Market create(CreateMarketRequest request) {
    MarketEntity entity = MarketEntity.from(request);
    MarketEntity saved = repo.save(entity);
    return Market.from(saved);
  }
}
```

## DTOs and Validation

```java
public record CreateMarketRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant endDate,
    @NotEmpty List<@NotBlank String> categories) {}

public record MarketResponse(Long id, String name, MarketStatus status) {
  static MarketResponse from(Market market) {
    return new MarketResponse(market.id(), market.name(), market.status());
  }
}
```

## Exception Handling

```java
@RestControllerAdvice
class GlobalExceptionHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
    ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    problem.setTitle("Validation Failed");
    problem.setProperty("violations", ex.getBindingResult().getFieldErrors().stream()
        .map(e -> Map.of("field", e.getField(), "reason", e.getDefaultMessage()))
        .toList());
    return problem;
  }

  @ExceptionHandler(Exception.class)
  ProblemDetail handleGeneric(Exception ex) {
    log.error("unhandled_exception", ex);
    return ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
  }
}
```

## Async & Concurrency — Virtual Threads

Spring Boot 3.2+ runs on virtual threads out of the box when configured. **Do not use `Mono`/`Flux` (Project Reactor) or `synchronized` blocks.** Write plain blocking code; the JVM schedules it on virtual threads efficiently.

### Enable virtual threads (Spring Boot 3.2+)

```yaml
# application.yaml
spring:
  threads:
    virtual:
      enabled: true
```

This single flag makes Tomcat, `@Async`, and `TaskExecutor` all use virtual threads — no code changes required for standard request handling.

### `@Async` with virtual threads

```java
// config/AsyncConfig.java
@Configuration
@EnableAsync
public class AsyncConfig {
  @Bean
  TaskExecutor taskExecutor() {
    // Virtual-thread-per-task executor — Spring Boot auto-configures this
    // when spring.threads.virtual.enabled=true; define explicitly only when
    // you need a named bean.
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
  }
}

// service/NotificationService.java
@Service
public class NotificationService {
  @Async
  public CompletableFuture<Void> sendAsync(Notification notification) {
    // plain blocking call — runs on a virtual thread
    emailClient.send(notification);
    return CompletableFuture.completedFuture(null);
  }
}
```

### Structured concurrency (Java 21+)

Use `StructuredTaskScope` when you need fan-out + join semantics:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
  var priceTask  = scope.fork(() -> priceService.fetch(marketId));
  var volumeTask = scope.fork(() -> volumeService.fetch(marketId));

  scope.join().throwIfFailed();

  return new MarketSummary(priceTask.get(), volumeTask.get());
}
```

### ⚠️ The "Weakest Link" problem with blocking I/O

Virtual threads are cheap but **they still block their carrier thread when a virtual thread performs non-cooperative blocking** (e.g. holding a `ReentrantLock` while waiting on I/O, or calling a driver that uses `synchronized` internally).

Rules to avoid the weakest-link problem:

| Rule | Why |
|---|---|
| Use JDBC drivers that support virtual-thread-friendly locking (e.g. PgJDBC ≥ 42.7 with `preferQueryMode=simple` or Loom-aware builds) | Old JDBC drivers use `synchronized` inside, pinning the carrier thread |
| Do **not** hold `ReentrantLock` / any lock across I/O calls | Pinned virtual threads stall the carrier, defeating concurrency |
| Size HikariCP pool to match **actual DB concurrency** (not virtual thread count) | HikariCP itself is virtual-thread-safe, but the DB has limited connections |
| Prefer `java.util.concurrent` (`Semaphore`, `CountDownLatch`) over `synchronized` | These cooperate with virtual thread scheduler |
| Set `spring.datasource.hikari.maximum-pool-size` conservatively | Virtual threads make it easy to flood the DB; the connection pool is the real throttle |
| Keep `@Transactional` boundaries short | Long-held DB connections under virtual threads still exhaust the pool |

```yaml
# application.yaml — safe defaults for virtual-thread + HikariCP
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # tune to DB capacity, not thread count
      connection-timeout: 3000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### What NOT to do

```java
// ❌ Reactive — don't use; virtual threads replace this need
Mono<Market> market = marketRepository.findBySlug(slug);

// ❌ synchronized — pins carrier thread under virtual threads
synchronized (this) { ... }

// ❌ static { } initializer — hides startup failures, hard to test
static {
  config = loadConfig();
}

// ✅ Plain blocking — reads clearly, runs efficiently on virtual threads
Market market = marketRepository.findBySlug(slug)
    .orElseThrow(() -> new MarketNotFoundException(slug));
```

## Logging (SLF4J)

```java
private static final Logger log = LoggerFactory.getLogger(MarketService.class);
log.info("fetch_market slug={}", slug);
log.error("failed_fetch_market slug={}", slug, ex);
```

## Middleware / Filters

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
  private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    long start = System.currentTimeMillis();
    try {
      filterChain.doFilter(request, response);
    } finally {
      long duration = System.currentTimeMillis() - start;
      log.info("req method={} uri={} status={} durationMs={}",
          request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }
  }
}
```

## Pagination and Sorting

```java
PageRequest page = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());
Page<Market> results = marketService.list(page);
```

## Production Defaults

- Prefer constructor injection, avoid field injection
- Enable `spring.mvc.problemdetails.enabled=true` for RFC 7807 errors (Spring Boot 3+)
- Enable `spring.threads.virtual.enabled=true` — virtual threads for all request handling and `@Async`
- Configure HikariCP conservatively (`maximum-pool-size` ≤ DB connection limit); virtual threads do **not** reduce DB concurrency needs
- Use `@Transactional(readOnly = true)` for queries; keep transaction boundaries short
- Enforce null-safety via `@NonNull` and `Optional` where appropriate
- Never use `synchronized`, `static { }` blocks, or reactive types (`Mono`/`Flux`) — see Async & Concurrency section

---

# Part 3: Unit Testing

JUnit 5 + Mockito + AssertJ patterns for Spring Boot services.

## Tech Stack

- **JUnit 5** (`org.junit.jupiter`) for test framework
- **Mockito** (`org.mockito`) for mocking dependencies
- **AssertJ** (`org.assertj.core.api.Assertions`) for fluent assertions
- **Parameterized Tests** (`@ParameterizedTest`) for testing multiple inputs

## Test File Placement

- Place test files under `src/test/java` mirroring the source package structure
- Name test classes as `{ClassName}Test.java`

## Class Structure

```java
@ExtendWith(MockitoExtension.class)
class FooServiceTest {

    @Mock
    private BarRepository barRepository;

    @InjectMocks
    private FooService fooService;

    @Nested
    @DisplayName("methodName")
    class MethodName {

        @Test
        @DisplayName("should do something when condition is met")
        void shouldDoSomething_whenConditionIsMet() {
            // given
            // when
            // then
        }
    }
}
```

## Rules

1. **Use AssertJ for all assertions** — never use JUnit `assertEquals`/`assertTrue`
2. **Use `@ParameterizedTest`** when testing multiple inputs. Prefer `@CsvSource` for simple values, `@MethodSource` for complex objects, `@ValueSource`/`@NullAndEmptySource` for single-parameter variations
3. **Annotate every test with `@DisplayName`** using a human-readable sentence
4. **Follow given/when/then pattern** — use comments `// given`, `// when`, `// then`
5. **Use `@Mock` and `@InjectMocks`** — do not manually construct mocks unless specifically required
6. **Verify interactions when meaningful** — use `verify()` for collaborator calls
7. **Test one behavior per test method**
8. **Name test methods descriptively** — pattern: `should{Expected}_when{Condition}`
9. **Group related tests with `@Nested` classes**
10. **No Javadoc on tests** — `@DisplayName` serves as documentation
11. **Keep tests independent** — no shared mutable state, no order dependency
12. **Use strict stubbing** — only stub methods that the test exercises
13. **Assert exceptions with AssertJ** — `assertThatThrownBy(() -> ...)` or `assertThatCode(() -> ...).doesNotThrowAnyException()`

## Parameterized Test Examples

### Simple values with `@CsvSource`
```java
@ParameterizedTest
@DisplayName("should validate order status transitions")
@CsvSource({
    "PENDING, CONFIRMED, true",
    "CONFIRMED, SHIPPED, true",
    "SHIPPED, PENDING, false"
})
void shouldValidateStatusTransition(String from, String to, boolean expected) {
    boolean result = service.isValidTransition(from, to);
    assertThat(result).isEqualTo(expected);
}
```

### Complex objects with `@MethodSource`
```java
@ParameterizedTest
@DisplayName("should process order correctly for various inputs")
@MethodSource("provideOrders")
void shouldProcessOrder(Order order, String expectedStatus) {
    OrderResult result = service.process(order);
    assertThat(result.getStatus()).isEqualTo(expectedStatus);
}

private static Stream<Arguments> provideOrders() {
    return Stream.of(
        Arguments.of(new Order("item1", 1), "CONFIRMED"),
        Arguments.of(new Order("item2", 0), "REJECTED")
    );
}
```

### Null and empty inputs
```java
@ParameterizedTest
@DisplayName("should throw exception for invalid input")
@NullAndEmptySource
@ValueSource(strings = {"  ", "\t"})
void shouldThrowException_whenInputIsBlank(String input) {
    assertThatThrownBy(() -> service.process(input))
        .isInstanceOf(IllegalArgumentException.class);
}
```

## AssertJ Assertion Patterns

```java
// Object equality
assertThat(actual).isEqualTo(expected);

// Null checks
assertThat(actual).isNull();
assertThat(actual).isNotNull();

// Collections
assertThat(list).hasSize(3);
assertThat(list).containsExactly("a", "b", "c");
assertThat(list).extracting("name").containsOnly("Alice", "Bob");

// Exceptions
assertThatThrownBy(() -> service.call())
    .isInstanceOf(RuntimeException.class)
    .hasMessageContaining("failed");

// Optional
assertThat(optional).isPresent().hasValue(expected);
assertThat(optional).isEmpty();
```

---

# Part 4: TDD Workflow

Unit-test-driven development — fast, isolated, no containers.

## Cycle

1. Write tests first (they should fail)
2. Implement minimal code to pass
3. Refactor with tests green
4. Enforce coverage (JaCoCo)

## Web Layer Tests (MockMvc)

```java
@WebMvcTest(MarketController.class)
class MarketControllerTest {
  @Autowired MockMvc mockMvc;
  @MockitoBean MarketService marketService;

  @Test
  void returnsMarkets() throws Exception {
    when(marketService.list(any())).thenReturn(Page.empty());

    mockMvc.perform(get("/api/markets"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.content").isArray());
  }
}
```

## Persistence Tests (DataJpaTest)

H2 in-memory database — fast, no containers needed for unit-level repo tests.

```java
@DataJpaTest
class MarketRepositoryTest {
  @Autowired MarketRepository repo;

  @Test
  void savesAndFinds() {
    MarketEntity entity = new MarketEntity();
    entity.setName("Test");
    repo.save(entity);

    Optional<MarketEntity> found = repo.findByName("Test");
    assertThat(found).isPresent();
  }
}
```

> For tests that need real PostgreSQL + Kafka containers, see the `testing` skill.

## Test Data Builders

```java
class MarketBuilder {
  private String name = "Test";
  MarketBuilder withName(String name) { this.name = name; return this; }
  Market build() { return new Market(null, name, MarketStatus.ACTIVE); }
}
```

## Coverage (JaCoCo)

```kotlin
// build.gradle.kts
plugins {
    jacoco
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required = true
        html.required = true
    }
}
```

---

# Part 5: Code Formatting

## Rule

After writing or editing Java files, **always** run:

```bash
./gradlew spotlessApply
```

This formats all Java sources using **Google Java Style** via the Eclipse JDT formatter configured in `spotless.properties` at the project root.

- Run **once** at the end of a batch of changes, not after every individual file edit
- If `spotlessApply` reports errors, fix them before proceeding
- Do **not** manually format code — let Spotless handle it

---

# Part 6: Verification Loop

Run before PRs, after major changes, and pre-deploy.

## Phase 1: Build

```bash
./gradlew clean assemble -x test
```

If build fails, stop and fix.

## Phase 2: Static Analysis

```bash
./gradlew checkstyleMain pmdMain spotbugsMain
```

## Phase 3: Tests + Coverage

```bash
./gradlew test jacocoTestReport
```

Report: total tests, passed/failed, coverage % (lines/branches).

## Phase 4: Security Scan

```bash
# Dependency CVEs
./gradlew dependencyCheckAnalyze

# Secrets in source
grep -rn "password\s*=\s*\"" src/ --include="*.java" --include="*.yml" --include="*.properties"
grep -rn "sk-\|api_key\|secret" src/ --include="*.java" --include="*.yml"

# Check for System.out.println (use logger instead)
grep -rn "System\.out\.print" src/main/ --include="*.java"

# Check for raw exception messages in responses
grep -rn "e\.getMessage()" src/main/ --include="*.java"
```

## Phase 4b: Memory Leak Checks

### Static grep checks (run locally, no extra tooling)

```bash
# Streams / Readers / Writers opened without try-with-resources
grep -rn "new FileInputStream\|new FileOutputStream\|new BufferedReader\|new InputStreamReader" \
     src/main/ --include="*.java"

# HttpClient / RestClient / WebClient instances created inside methods (should be beans)
grep -rn "HttpClient\.newHttpClient\|RestClient\.create\|WebClient\.create" \
     src/main/ --include="*.java"

# ThreadLocal without remove() — leaks under virtual-thread or thread-pool reuse
grep -rn "ThreadLocal" src/main/ --include="*.java"

# Static collections that grow unboundedly
grep -rn "private static final.*Map\|private static final.*List\|private static final.*Set" \
     src/main/ --include="*.java"

# Listeners / observers registered but never deregistered
grep -rn "addEventListener\|addListener\|register.*Listener" src/main/ --include="*.java"

# ScheduledExecutorService / Timer created but not shut down
grep -rn "new ScheduledThreadPoolExecutor\|new Timer(" src/main/ --include="*.java"
```

### Runtime detection (CI / staging)

Add the following JVM flags to your test task to enable JDK Flight Recorder leak profiling:

```kotlin
// build.gradle.kts
tasks.test {
    jvmArgs(
        "-XX:+HeapDumpOnOutOfMemoryError",
        "-XX:HeapDumpPath=build/reports/heap-dump.hprof",
        "-Xlog:gc*:file=build/reports/gc.log:time,uptime:filecount=5,filesize=20m"
    )
}
```

Analyse heap dumps with:

```bash
# jmap heap histogram (JDK built-in)
jmap -histo:live <pid> | head -40

# Or open build/reports/heap-dump.hprof in IntelliJ / Eclipse MAT
```

### Common memory leak patterns and fixes

| Pattern | Problem | Fix |
|---|---|---|
| `static` cache / map with no eviction | Grows unboundedly | Use `Caffeine` / `@Cacheable` with `maximumSize` + TTL |
| `ThreadLocal` not removed after request | Leaked under virtual-thread recycling | Call `threadLocal.remove()` in `finally`, or use `ScopedValue` (Java 21+) |
| Resource opened outside `try-with-resources` | Stream never closed on exception | Wrap every `Closeable` in `try (var x = ...)` |
| Inner class holding outer class reference | GC cannot collect outer object | Use `static` nested classes or lambdas that don't capture `this` |
| Event listeners never deregistered | Publisher holds strong ref to subscriber | Deregister in `@PreDestroy` / `DisposableBean.destroy()` |
| `HttpClient` / `RestClient` created per-call | Connection pool recreated every request | Declare as `@Bean` (singleton) |
| Hibernate `@OneToMany` with `EAGER` on large collections | Entire collection loaded into heap | Default to `LAZY`; load explicitly when needed |
| `CompletableFuture` chains not completed or cancelled | Pending stages never GC'd | Always complete / cancel in `finally`; set timeouts |

### SpotBugs memory rules (already in Phase 2)

Ensure these SpotBugs bug categories are **not** suppressed in `spotbugs-exclude.xml`:

```xml
<!-- spotbugs-exclude.xml — do NOT exclude these -->
<!-- DM_GC, IS2_INCONSISTENT_SYNC, OBL_UNSATISFIED_OBLIGATION,
     ST_WRITE_TO_STATIC_FROM_INSTANCE_METHOD -->
```

## Phase 5: Lint/Format

```bash
./gradlew spotlessApply
```

## Phase 6: Diff Review

```bash
git diff --stat
git diff
```

Checklist:
- No debugging logs left (`System.out`, `log.debug` without guards)
- Meaningful errors and HTTP statuses
- Transactions and validation present where needed
- Config changes documented

## Verification Output Template

```
VERIFICATION REPORT
===================
Build:     [PASS/FAIL]
Static:    [PASS/FAIL] (spotbugs/pmd/checkstyle)
Tests:     [PASS/FAIL] (X/Y passed, Z% coverage)
Security:  [PASS/FAIL] (CVE findings: N)
Memory:    [PASS/FAIL] (leaks: static caches, unclosed resources, ThreadLocal, listeners)
Diff:      [X files changed]

Overall:   [READY / NOT READY]

Issues to Fix:
1. ...
2. ...
```

---

**Remember**: Keep controllers thin, services focused, repositories simple, tests fast and deterministic. Optimize for maintainability and testability.
