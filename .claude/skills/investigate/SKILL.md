---
name: investigate
description: "Systematic debugging with root cause investigation for MyProject Spring Boot services. Four phases: investigate, analyze, hypothesize, implement. Iron Law: no fixes without root cause. Covers Kafka consumer failures, HTTP endpoint errors, database issues, external integration failures, Spring Boot runtime failures, virtual thread pinning, memory leaks, structured concurrency, and HIPAA escalation. Adapted from community patterns."
origin: community-adapted
---

# Systematic Debugging

## Iron Law

**NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST.**

Fixing symptoms creates whack-a-mole bugs. Every fix that doesn't address root cause makes the next bug harder to find. Find the root cause, then fix it.

If you are tempted to add a try-catch, add a null check, or tweak a config value without knowing WHY the bug occurs — stop. Gather more evidence.

> Before any fix, you must complete the sentence:
> **"The bug is in `<ClassName>.<method>` at line N because `<specific reason>`."**

---

## When to Activate

- Kafka consumer not processing messages / unexpected DLT routing
- HTTP endpoint returning errors (4xx/5xx)
- External integration failures (SMTP, HTTP client, database)
- Spring Boot startup failures
- Test failures that weren't failing before
- "It was working yesterday"
- Any stack trace or uncaught exception in logs
- Performance degradation or timeout errors
- Virtual threads pinning / carrier thread stalls / latency spikes under load
- Memory usage climbing over time (heap, metaspace, off-heap)
- `StructuredTaskScope` timeout or cancellation failures
- HikariCP pool exhaustion — "Connection is not available, request timed out after 30000ms"

---

## Phase 1: Root Cause Investigation

Gather full context before forming any hypothesis.

### Step 1 — Collect symptoms

Read the error message, stack trace, and reproduction steps completely. If the user hasn't provided enough, ask **one question at a time**:
- "Is this happening on all inputs or specific ones?"
- "Is this local only, or also in dev/staging?"
- "When did it last work?"
- "Has there been a recent dependency upgrade or config change?"

### Step 2 — Check recent changes

```bash
git log --oneline -20
git log --oneline -10 -- src/main/java/
```

A regression means the root cause is in the diff. If something broke after a commit, start there.

### Step 3 — Detect service entry points and architecture

```bash
grep -r "@KafkaListener" src/main/ --include="*.java" -l && echo "ENTRY: Kafka consumer"
grep -r "@RestController\|@Controller" src/main/ --include="*.java" -l && echo "ENTRY: HTTP API"
grep -r "@Scheduled\|@EventListener" src/main/ --include="*.java" -l && echo "ENTRY: Scheduler/Event"
grep -r "@WorkflowInterface\|@ActivityInterface" src/main/ --include="*.java" -l && echo "ENTRY: Temporal"
grep -r "JpaRepository\|@Repository" src/main/ --include="*.java" -l && echo "EXIT: Database"
grep -r "RestClient\|WebClient\|RestTemplate" src/main/ --include="*.java" -l && echo "EXIT: HTTP client"
```

Use Read and Grep to trace the specific code path that is failing. Identify the entry point and follow the call chain to where the error occurs.

### Step 4 — Check service health via Actuator

```bash
# Liveness and readiness probes
curl -s http://localhost:${SERVER_PORT:-8080}/actuator/health | python3 -m json.tool

# JVM thread state
curl -s http://localhost:${SERVER_PORT:-8080}/actuator/metrics/jvm.threads.live
curl -s http://localhost:${SERVER_PORT:-8080}/actuator/metrics/jvm.threads.peak

# HikariCP connection pool
curl -s http://localhost:${SERVER_PORT:-8080}/actuator/metrics/hikaricp.connections.active
curl -s http://localhost:${SERVER_PORT:-8080}/actuator/metrics/hikaricp.connections.pending

# HTTP server request latency
curl -s "http://localhost:${SERVER_PORT:-8080}/actuator/metrics/http.server.requests"

# Tomcat thread pool
curl -s http://localhost:${SERVER_PORT:-8080}/actuator/metrics/tomcat.threads.current

# Circuit breaker status (Resilience4j, if configured)
curl -s http://localhost:${SERVER_PORT:-8080}/actuator/circuitbreakers | python3 -m json.tool
```

> Rule: If `jvm.threads.live` is near `hikari.maximum-pool-size`, virtual threads are likely blocking on DB calls.

### Step 5 — Check active configuration

```bash
grep -r "spring.profiles.active\|SPRING_PROFILES_ACTIVE" src/main/resources/ .env 2>/dev/null
grep -r "\${[A-Z_]\+}" src/main/resources/application.yaml | head -20
grep -r "virtual.enabled\|spring.threads" src/main/resources/ --include="*.yaml"
```

### Step 6 — Reproduce deterministically

```bash
./gradlew test --tests "com.example.myapp.<service>.<ClassName>" 2>&1
./gradlew test --info --tests "com.example.myapp.<service>.<ClassName>" 2>&1 | tail -50
```

Output your **root cause hypothesis** before proceeding: a specific, testable claim about what is wrong and why.

---

## Scope Lock

After forming the root cause hypothesis, lock edits to the narrowest relevant scope to prevent scope creep during debugging.

| Bug type | Scope lock |
|---|---|
| Kafka / messaging | consumer/producer package only |
| HTTP / API | controller and service layer |
| Data / persistence | repository and entity classes |
| Config / startup | `src/main/resources/` |
| Integration | specific client/adapter class |
| Virtual thread pinning | class using `synchronized` or blocking I/O |
| Memory leak | class with `ThreadLocal`, static caches, or listeners |

Tell the user: "Edits restricted to `<dir>/` for this debug session to prevent unrelated changes."

---

## Phase 2: Pattern Analysis

Check if this bug matches a known pattern for Spring Boot services (Java 21+, Spring Boot 3.2+):

| Pattern | Signature | Where to look |
|---|---|---|
| **Kafka deserialization failure** | `SerializationException`, schema mismatch | Deserializer config, schema version, `ErrorHandlingDeserializer` |
| **Kafka consumer lag** | Messages not consumed, offset not advancing | Consumer group ID, partition assignment, `DefaultErrorHandler` |
| **DLT overflow** | Messages going to DLT unexpectedly | Error handler, retry config, exception classification |
| **HTTP 5xx** | `NullPointerException`, unhandled exception | Controller → service → repository call chain |
| **HTTP 4xx unexpected** | Validation failure, missing field | Request DTO constraints, `@Valid`, `MethodArgumentNotValidException` |
| **Database connection exhaustion** | `HikariPool-1 - Connection is not available`, timeout after 30s | Pool size, long-held transactions, N+1 queries |
| **External HTTP timeout** | `ReadTimeoutException`, `ConnectTimeoutException` | Client timeout config, target service health |
| **Startup failure** | `ApplicationContext` fails to start | Unresolved `${placeholder}`, missing bean, circular dependency |
| **Race condition** | Intermittent, non-deterministic | Shared mutable state, missing `synchronized`, concurrent access |
| **Config drift** | Works locally, fails in dev/staging | Profile-specific YAML, env vars missing in target env |
| **Null propagation** | `NullPointerException` on data fields | Optional fields from external systems, Avro null unions, JSON null |
| **Transaction rollback** | Data not persisted, silent rollback | `@Transactional` boundaries, checked vs unchecked exceptions |
| **Virtual thread pinning** | Latency spike, carrier threads saturated | `synchronized` blocks, old JDBC drivers, blocking I/O inside virtual threads |
| **Memory leak — ThreadLocal** | Heap grows over time, GC cannot collect | `ThreadLocal` without `remove()` in carrier thread reuse |
| **Memory leak — static cache** | Heap grows, no eviction | `static Map/List/Set` with unbounded growth |
| **Memory leak — listener** | OutOfMemoryError after many events | `addEventListener/addListener` never deregistered |
| **Structured concurrency timeout** | `TimeoutException` from `StructuredTaskScope` | `scope.join()` timeout, one forked task exceeded deadline |

---

## Virtual Thread-Specific Debugging (Java 21+)

Virtual threads (enabled via `spring.threads.virtual.enabled=true`) dramatically improve throughput, but specific patterns cause **carrier thread pinning** — where a virtual thread holds its underlying OS thread and blocks other virtual threads from mounting.

### Detecting pinned virtual threads

```bash
# Enable diagnostic JVM flags at startup
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintVirtualThreadSchedulerStats \
     -jar app.jar

# Watch for pinned thread warnings in logs — they appear as:
# Thread[#42,ForkJoinPool-1-worker-1,5,CarrierThreads] PINNED
```

### Common causes of pinning

| Cause | Diagnosis | Fix |
|---|---|---|
| `synchronized` block in hot path | Grep for `synchronized` in service classes | Replace with `ReentrantLock` or redesign |
| Old JDBC driver (PgJDBC < 42.7) | Check `build.gradle.kts` for driver version | Upgrade to `org.postgresql:postgresql:42.7+` |
| `Object.wait()` / `Thread.sleep()` in synchronized | Stack trace shows both | Remove `synchronized`, use virtual-thread-safe constructs |
| JNI calls | Stack trace shows native frames | Offload to a dedicated platform thread pool |

### Grep for pinning candidates

```bash
grep -rn "synchronized" src/main/java/ --include="*.java" | grep -v "//.*synchronized"
grep -rn "Object\.wait\(\)\|Thread\.sleep\(" src/main/java/ --include="*.java"
```

### Case study: API suddenly slow — virtual threads pinned by JDBC driver

**Symptoms:** P99 latency jumps from 50ms to 2s under load; `jvm.threads.peak` metric spikes.

**Diagnosis:**
1. Check JDBC driver version: `grep "postgresql" build.gradle.kts`
2. If version < 42.7, the driver uses `synchronized` blocks internally — pinning carrier threads.
3. Confirm: Run with `-XX:+PrintVirtualThreadSchedulerStats`, look for `PINNED` entries.

**Fix:** Upgrade driver in `build.gradle.kts`:
```kotlin
implementation("org.postgresql:postgresql:42.7.3")
```

**Verification:** Latency returns to baseline; no `PINNED` log lines.

---

## Memory Leak Investigation Patterns

### Common memory leak patterns

| Pattern | Detection command | Fix |
|---|---|---|
| `ThreadLocal` without `remove()` | `grep -rn "ThreadLocal" src/main/ --include="*.java"` | Call `threadLocal.remove()` in `finally`, or use `ScopedValue` (Java 21+) |
| Static caches without eviction | `grep -rn "private static final.*Map\|List\|Set" src/main/ --include="*.java"` | Use `Caffeine` with TTL/size limit, or `WeakHashMap` |
| Listeners registered but never removed | `grep -rn "addEventListener\|addListener" src/main/ --include="*.java"` | Deregister on `@PreDestroy` or use `@EventListener` (Spring manages lifecycle) |
| `HttpClient` / `RestClient` created inside methods | `grep -rn "RestClient\|HttpClient" src/main/ --include="*.java" -B2 -A2` | Move to `@Bean` singleton |
| Unclosed resources (`Stream`, `Reader`, `Connection`) | `grep -rn "new FileInputStream\|new BufferedReader\|new Connection" src/main/ --include="*.java"` | Use try-with-resources |

### Diagnostic commands

```bash
# ThreadLocal usage
grep -rn "ThreadLocal" src/main/ --include="*.java"

# Static collections that could grow unbounded
grep -rn "private static final.*Map\|List\|Set" src/main/ --include="*.java"

# Listener registration without deregistration
grep -rn "addEventListener\|addListener" src/main/ --include="*.java"

# RestClient/HttpClient instantiation (should be beans)
grep -rn "RestClient\.create\(\)\|HttpClient\.newHttpClient\(\)" src/main/ --include="*.java"
```

### Java Flight Recorder for heap analysis

```bash
# Start app with JFR enabled
java -XX:StartFlightRecording=delay=20s,duration=120s,filename=app.jfr \
     -jar build/libs/app.jar

# Analyze with JDK Mission Control or:
java -jar jfr-analyzer.jar app.jfr
```

### Heap growth check via Actuator

```bash
# Heap used
curl -s http://localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap

# If heap used keeps growing between requests with no GC relief -> leak confirmed
```

---

## Spring Boot Actuator Deep Dive

Actuator is your first stop before touching any code. Every diagnostic session should start here.

### Key endpoints

| Endpoint | What it tells you |
|---|---|
| `GET /actuator/health` | Liveness, readiness, DB/Kafka/external connectivity |
| `GET /actuator/metrics` | Full list of available metric names |
| `GET /actuator/metrics/jvm.threads.live` | Current virtual + platform threads |
| `GET /actuator/metrics/jvm.threads.peak` | Peak thread count since startup |
| `GET /actuator/metrics/hikaricp.connections.active` | Active DB connections |
| `GET /actuator/metrics/hikaricp.connections.pending` | Waiting for a connection |
| `GET /actuator/metrics/http.server.requests` | Latency percentiles per endpoint |
| `GET /actuator/metrics/tomcat.threads.current` | Tomcat thread pool usage |
| `GET /actuator/circuitbreakers` | Resilience4j circuit breaker states |
| `GET /actuator/env` | Resolved config values (sanitized) |
| `GET /actuator/loggers` | Current log levels (change at runtime) |

### Diagnostic decision tree using Actuator

```bash
# Step 1 — Is the service alive?
curl -s http://localhost:8080/actuator/health | python3 -m json.tool

# Step 2 — Is the DB reachable?
curl -s http://localhost:8080/actuator/health/db

# Step 3 — Are threads near capacity?
curl -s http://localhost:8080/actuator/metrics/jvm.threads.live
# If count near hikari.maximum-pool-size -> external calls likely blocking

# Step 4 — Which endpoint is slow?
curl -s "http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/orders"

# Step 5 — Is HikariCP waiting?
curl -s http://localhost:8080/actuator/metrics/hikaricp.connections.pending
# If > 0 consistently -> pool exhaustion
```

### Change log level at runtime (no restart needed)

```bash
curl -X POST http://localhost:8080/actuator/loggers/com.example.myapp \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

---

## Kafka Consumer Debugging

### Consumer lag — is the consumer keeping up?

```bash
# Check if consumer group is lagging
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group <group-id>

# LAG > 1000 is critical; LAG growing over time = consumer is stuck or overwhelmed
```

| LAG reading | Interpretation |
|---|---|
| 0 | Consumer healthy, keeping up |
| Growing slowly | Consumer slower than producer — check processing time |
| Stuck (LAG constant) | Consumer paused, rebalancing, or throwing exceptions |
| Sudden jump | Spike in messages, or consumer restarted and lost position |

### Check DLT for error patterns

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic <topic-name>.DLT \
  --from-beginning \
  --max-messages 10
```

Look for:
- `SerializationException` with offset info → schema mismatch or bad payload
- Same exception type repeating → systemic bug, not transient

### Deserialization failure diagnosis

```bash
# Find deserialization errors in logs
grep -r "SerializationException\|DeserializationException" logs/ 2>/dev/null | tail -20

# Check ErrorHandlingDeserializer config
grep -r "ErrorHandlingDeserializer\|value.deserializer" src/main/resources/ --include="*.yaml"
```

### Consumer rebalance diagnosis

```bash
# Is the consumer rebalancing constantly?
grep -r "Rebalancing\|Consumer group.*rebalance" logs/ 2>/dev/null | tail -10

# Check session.timeout.ms and max.poll.interval.ms
grep -r "session.timeout\|max.poll.interval\|heartbeat.interval" src/main/resources/ --include="*.yaml"
```

> Rule: If processing one message takes longer than `max.poll.interval.ms` (default 5 min), the consumer will be kicked out of the group and trigger rebalance. Increase the interval or reduce batch size.

---

## HTTP Client Timeout Debugging

### Locate all HTTP client configurations

```bash
# Find all HTTP clients
grep -r "RestClient\|RestTemplate\|WebClient\|HttpClient" src/main/ --include="*.java" -B2 -A2

# Find timeout configs
grep -r "connectTimeout\|readTimeout\|writeTimeout" src/main/resources/ --include="*.yaml"
grep -r "connectTimeout\|readTimeout" src/main/ --include="*.java" -A3
```

### Check if the target service is healthy

```bash
curl -s http://<target-host>:<port>/actuator/health | python3 -m json.tool

# If no Actuator, check raw response
curl -v --max-time 5 http://<target-host>:<port>/health
```

### Timeout configuration for RestClient (Spring Boot 3.2+)

```java
// Expected pattern — if missing, this is likely the root cause
RestClient.builder()
    .requestFactory(
        new JdkClientHttpRequestFactory(
            HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(5))
                .build()
        )
    )
    .build();
```

### Circuit breaker diagnosis (Resilience4j)

```bash
curl -s http://localhost:8080/actuator/circuitbreakers | python3 -m json.tool
# States: CLOSED (healthy) | OPEN (failing fast) | HALF_OPEN (testing recovery)

# If OPEN -> target service is unhealthy or circuit breaker threshold too sensitive
grep -r "resilience4j\|circuitbreaker" src/main/resources/ --include="*.yaml"
```

---

## Database Connection Pool Exhaustion

**Symptoms:** `HikariPool-1 - Connection is not available, request timed out after 30000ms`; `HikariPool` exceptions in logs; requests queuing with no DB errors on the DB side.

### Step-by-step investigation

```bash
# Step 1 — Check pool config
grep -A10 "hikari:" src/main/resources/application.yaml

# Step 2 — Check active and pending connections via Actuator
curl -s http://localhost:8080/actuator/metrics/hikaricp.connections.active
curl -s http://localhost:8080/actuator/metrics/hikaricp.connections.pending
curl -s http://localhost:8080/actuator/metrics/hikaricp.connections.max

# Step 3 — Look for long-held transactions in code
grep -rn "@Transactional" src/main/java/ --include="*.java" -A5 | grep -E "for|while|sleep|http"

# Step 4 — Check for N+1 query patterns
grep -rn "findAll\(\)\|findBy" src/main/java/ --include="*.java" -B2 -A5
```

### Root causes and fixes

| Root cause | Signature | Fix |
|---|---|---|
| Long-held `@Transactional` | DB call inside HTTP call inside transaction | Move HTTP calls outside `@Transactional` boundary |
| N+1 queries | Each entity triggers child query | Use `@EntityGraph` or `JOIN FETCH` |
| Virtual threads + too many concurrent requests | Pool size < concurrency | Tune `hikari.maximum-pool-size` (recommend: 10–20 for virtual threads) |
| No connection timeout | Requests queue forever | Set `hikari.connection-timeout=3000` |

### Recommended HikariCP config for virtual threads

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # Virtual threads don't need large pools
      minimum-idle: 5
      connection-timeout: 3000     # Fail fast rather than queue forever
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 5000  # Warn if connection held > 5s
```

---

## Structured Concurrency Failures (Java 21+)

`StructuredTaskScope` is used in MyProject for parallel calls with automatic cancellation on failure or timeout.

### Anatomy of a structured concurrency failure

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var task1 = scope.fork(() -> callServiceA());
    var task2 = scope.fork(() -> callServiceB());

    scope.joinUntil(Instant.now().plusSeconds(10))  // <- timeout here
         .throwIfFailed();                           // <- or exception from subtask here

    return new Result(task1.get(), task2.get());
} catch (TimeoutException e) {
    // One or more tasks exceeded 10 seconds
} catch (ExecutionException e) {
    // One task threw an exception, scope cancelled others
}
```

### Diagnosing which task timed out

```bash
# Look for TimeoutException with task name in logs
grep -r "TimeoutException\|StructuredTaskScope\|ShutdownOnFailure" logs/ 2>/dev/null | tail -20

# Enable DEBUG logging for the structured concurrency package
curl -X POST http://localhost:8080/actuator/loggers/java.util.concurrent \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

### Flamegraph with async-profiler

```bash
# Profile to see where forked tasks are stuck
java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=profile.html \
     -jar build/libs/app.jar

# Look for wide flat sections in the flamegraph -> code stuck waiting
```

### Common root causes

| Root cause | Signature | Fix |
|---|---|---|
| Subtask blocked on slow HTTP call | Flamegraph shows HTTP read wait | Set timeout on `RestClient`; reduce `joinUntil` duration |
| Subtask blocked on DB query | Flamegraph shows JDBC wait | Optimize query; add index |
| `TimeoutException` on subtask | `scope.joinUntil()` throws | Increase deadline or add fallback via `ShutdownOnSuccess` |
| Scope used inside `@Transactional` | Transaction held open during fork | Move structured concurrency outside `@Transactional` |

---

## Spring Boot Startup Failures (Phase 2 — Startup Patterns)

### Common startup failure patterns

| Pattern | Signature | Diagnosis |
|---|---|---|
| Missing bean / circular dependency | `ApplicationContext` fails to start; `NoSuchBeanDefinitionException` or `BeanCurrentlyInCreationException` | Check `@Bean` definitions; use `@Lazy` to break circular deps |
| Unresolved placeholder | `Could not resolve placeholder '${VAR}'` | Missing env var or `application.yaml` key; check `spring.config.import` |
| Profile-specific config missing | Properties not loaded | Check `spring.profiles.active` matches a `application-{profile}.yaml` |
| Flyway migration failure | `FlywayException: Validate failed` | SQL script changed after applying; use `flyway.repair` or fix migration |
| Port already in use | `Address already in use: bind` | Kill the existing process on `server.port` |

### Startup debug commands

```bash
# Run with full debug output to see bean wiring sequence
./gradlew bootRun --args='--debug' 2>&1 | grep -E "Creating|Registering|No bean|Error" | head -40

# Show all loaded configuration properties
./gradlew bootRun --args='--trace' 2>&1 | grep "Loaded config" | head -20

# Check for unresolved placeholders
grep -rn "\${" src/main/resources/ --include="*.yaml" --include="*.properties"
```

---

## Phase 3: Root Cause Confirmation

Before fixing, confirm the root cause with evidence.

### Write a failing test

```java
@Test
@DisplayName("should [describe the bug scenario]")
void should_reproduce_bug() {
    // given — exact conditions that trigger the bug
    // when  — invoke the code path
    // then  — observe the failure (assertThrows, assertEquals, etc.)
}
```

### Confirm the hypothesis

Does the test fail for the reason you predicted? If yes, proceed. If no, restart Phase 1.

### Check for existing tests (false negative)

```bash
./gradlew test 2>&1 | grep -E "PASSED|FAILED" | head -30
```

### Test failure regression investigation (CI vs local)

When a test passes locally but fails in CI:

```bash
# Step 1 — Is the test flaky? Run 10x locally
./gradlew test --tests "path.to.FlakyTest" --rerun-tasks -i 2>&1 | tail -100

# Step 2 — Check for clock sensitivity (use Awaitility, not Thread.sleep)
grep -rn "Thread\.sleep\|System\.currentTimeMillis" src/test/ --include="*.java"

# Step 3 — Check Awaitility timeout (default 1s may be too short in CI)
grep -rn "await()\|Awaitility\|atMost" src/test/ --include="*.java" | head -10
```

**Common CI-specific failures:**
- Awaitility default timeout (1s) passes locally (fast machine) but fails on slow CI runners → increase `atMost(10, SECONDS)`
- Test uses wall clock time → replace with Awaitility
- Test depends on port availability → use `@SpringBootTest(webEnvironment = RANDOM_PORT)`
- Test uses `@EmbeddedKafka` but CI has limited resources → add `@DirtiesContext`

Do NOT proceed until you can answer: **"The bug is in `<ClassName>.<method>` at line N because `<specific reason>`."**

---

## Phase 4: Implement the Fix

1. **Fix the narrowest possible scope** — one file, one method if possible
2. **Run Spotless** to avoid formatting noise:
   ```bash
   ./gradlew spotlessApply
   ```
3. **Run the specific test** that was failing:
   ```bash
   ./gradlew test --tests "com.example.myapp.<service>.<ClassName>" 2>&1
   ```
4. **Run the full suite** to check for regressions:
   ```bash
   ./gradlew clean test 2>&1 | tail -20
   ```
5. **Add a regression test** if one didn't exist
6. **Save context before switching** if this was a complex investigation:
   ```
   /context-save "fixed: <root cause in 5 words>"
   ```

---

## Verification Loop

```bash
./gradlew clean assemble -x test
./gradlew test jacocoTestReport
./gradlew spotlessCheck
git diff --stat && git diff
```

---

## Escalation Rules

- **3 failed fix attempts on the same hypothesis** → hypothesis is wrong; restart Phase 1 with fresh eyes
- **Bug is in Spring Boot / Kafka / library internals** → don't patch library behavior; upgrade the library or apply a workaround
- **Fix requires a schema change** (Avro, DB migration) → flag for team review before applying
- **Fix touches shared config or infrastructure** → needs explicit team review

### HIPAA Compliance Escalation

**If at any point during debugging you discover:**
- Patient data (MRN, SSN, email, NPI, diagnosis codes) written to application logs
- PHI in a Kafka DLT topic (unmasked)
- Patient data sent to an unintended system or endpoint
- PHI accessible to an unauthorized user or role

**STOP IMMEDIATELY. Do not attempt to fix the bug.**

Follow this protocol:

```
1. Document:
   - Timestamp of discovery
   - What data was found
   - Where it was found (log file, topic, endpoint)
   - Systems affected
   - How many records may be affected

2. Notify:
   - Privacy Officer — before any further action
   - Do NOT commit code changes related to the issue before notification

3. Assess:
   - Was this a breach under 45 CFR §164.402 (HIPAA Breach Notification Rule)?
   - HHS notification required within 60 days of discovery
   - Media notification required if > 500 individuals in a state are affected

4. Check DLT for PHI:
   kafka-console-consumer.sh \
     --bootstrap-server localhost:9092 \
     --topic <topic>.DLT \
     --from-beginning \
     --max-messages 10 | grep -iE "patient|mrn|ssn|npi|email"
   # If hits -> potential breach, escalate immediately
```

---

## Scenario-Based Debugging Runbooks

### Runbook 1: API returns 500 — logs show NullPointerException

**Symptoms:** `HTTP 500` on `POST /api/orders`; stack trace shows `NullPointerException` in service layer.

```
INVESTIGATION STEPS:

1. Read the full stack trace — identify the exact line
   grep -r "NullPointerException" logs/ 2>/dev/null | tail -20

2. Trace the call chain from the controller to where null appears
   - Is it a missing field in the request DTO?
   - Is it a null return from a repository?
   - Is it a null injected bean?

3. Check if the null comes from an external system (Avro field, HTTP response, DB column)
   grep -rn "Optional\|nullable\|@Column" src/main/java/ --include="*.java" -B2 -A2

4. Write a failing test with the exact null input that triggers the bug

5. Fix: Add Optional.orElseThrow(), null guard, or fix the upstream source

6. Root cause template:
   "NPE at OrderService.process() line N because customerEmail was null
    when the incoming Avro event had null in the email field —
    which is valid in schema v1 but not handled in service code."
```

---

### Runbook 2: Kafka consumer stuck — messages piling up in DLT

**Symptoms:** Consumer lag growing; DLT topic has new messages; no visible exceptions in consumer log.

```
INVESTIGATION STEPS:

1. Check consumer lag
   kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
     --describe --group <group-id>

2. Read DLT messages
   kafka-console-consumer.sh --bootstrap-server localhost:9092 \
     --topic <topic>.DLT --from-beginning --max-messages 10

3. Identify exception type from DLT headers
   grep -r "deserializ\|SerializationException" logs/ 2>/dev/null | tail -20

4. Check DefaultErrorHandler config
   grep -r "DefaultErrorHandler\|BackOff\|DltPublishingErrorHandler" src/main/ --include="*.java"

5. Check if exception is classified as retryable or not:
   - Non-retryable -> routed to DLT immediately (correct behavior)
   - Retryable -> retried N times then DLT (check retry exhausted logs)

6. Root cause template:
   "Consumer stuck because order payload has a field 'orderType' not in Avro schema v2.
    ErrorHandlingDeserializer catches it and routes to DLT.
    Fix: update schema or add backward-compatible field."
```

---

### Runbook 3: Database connection pool exhausted

**Symptoms:** `HikariPool-1 - Connection is not available, request timed out after 30000ms`; API requests timing out.

```
INVESTIGATION STEPS:

1. Check pool metrics
   curl -s http://localhost:8080/actuator/metrics/hikaricp.connections.active
   curl -s http://localhost:8080/actuator/metrics/hikaricp.connections.pending

2. Check pool config
   grep -A10 "hikari:" src/main/resources/application.yaml

3. Look for long @Transactional methods that call external HTTP APIs
   grep -rn "@Transactional" src/main/java/ --include="*.java" -A10 | \
     grep -E "RestClient\|RestTemplate\|WebClient"

4. Look for N+1 queries
   grep -rn "findAll\(\)" src/main/java/ --include="*.java" -B5 -A5

5. Enable HikariCP leak detection
   Add: spring.datasource.hikari.leak-detection-threshold=5000

6. Root cause template:
   "Pool exhausted because OrderService.createOrder() holds a transaction
    while making a 10s HTTP call to an external service.
    Fix: move HTTP call outside @Transactional boundary."
```

---

### Runbook 4: Virtual threads pinning — latency spikes under load

**Symptoms:** P99 latency spikes from 50ms to 2s+ under load; `jvm.threads.peak` high; Actuator shows thread count growing.

```
INVESTIGATION STEPS:

1. Enable virtual thread pinning diagnostics
   Start app with: -XX:+UnlockDiagnosticVMOptions -XX:+PrintVirtualThreadSchedulerStats
   Look for: Thread[...CarrierThreads] PINNED

2. Search for synchronized blocks in hot paths
   grep -rn "synchronized" src/main/java/ --include="*.java" | grep -v "//.*synchronized"

3. Check JDBC driver version
   grep "postgresql\|mysql\|mariadb" build.gradle.kts

4. Check for Thread.sleep in synchronized or locked context
   grep -rn "Thread\.sleep" src/main/java/ --include="*.java" -B5 | grep "synchronized\|lock"

5. Use async-profiler to get a flamegraph
   java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=flame.html -jar app.jar
   # Wide flat sections = code stuck blocking

6. Root cause template:
   "Virtual threads pinned because PostgreSQL JDBC driver 42.5.x uses synchronized
    blocks internally. Upgrading to 42.7.3 removes synchronization, eliminating pinning."
```

---

### Runbook 5: Spring Boot startup failure on configuration

**Symptoms:** Application fails to start; `ApplicationContext` startup exception; no HTTP traffic possible.

```
INVESTIGATION STEPS:

1. Read the full startup exception — Spring prints the root cause clearly
   ./gradlew bootRun 2>&1 | grep -A20 "APPLICATION FAILED TO START"

2. Missing placeholder?
   grep -rn "\${" src/main/resources/ --include="*.yaml"
   # Compare against .env or environment variables provided

3. Missing bean or circular dependency?
   ./gradlew bootRun --args='--debug' 2>&1 | grep "NoSuchBeanDefinition\|BeanCurrentlyInCreation"

4. Profile not matching?
   grep -r "spring.profiles.active" src/main/resources/ .env 2>/dev/null
   ls src/main/resources/application-*.yaml

5. Flyway migration failure?
   ./gradlew flywayInfo 2>/dev/null
   grep -r "FlywayException\|Validate failed" logs/ 2>/dev/null | tail -10

6. Root cause template:
   "Startup fails because DATABASE_URL env var is not set in the local .env file.
    The application.yaml references \${DATABASE_URL} with no default.
    Fix: add DATABASE_URL to .env or provide a default: \${DATABASE_URL:jdbc:postgresql://localhost/MyProject}"
```

---

## Completion Status

- **DONE** — root cause found and fixed, tests passing, no regressions
- **DONE_WITH_CONCERNS** — fixed, but underlying design issue worth noting
- **BLOCKED** — cannot reproduce or access required system (state what's needed)
- **NEEDS_CONTEXT** — missing information to form a hypothesis (state exactly what)
- **ESCALATED** — PHI/HIPAA issue discovered; Privacy Officer notified; investigation paused

---

## Integration Notes

- After a complex investigation, save the session state:
  `/context-save "fixed: <root cause in 5 words>"`
- When resuming a debugging session from a previous day:
  `/context-restore` then check the "Investigation Notes" section
- Cross-reference with `java-development` skill for coding standards when implementing the fix
- Cross-reference with `kafka-patterns` skill for DLT and retry configuration patterns
- Cross-reference with `jpa-patterns` skill for N+1 and transaction boundary issues
