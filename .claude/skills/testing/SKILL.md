---
name: testing
description: "BDD acceptance tests (Cucumber + SpringBootTest + Testcontainers) and black-box component tests (Cucumber + Docker container + RestAssured). Covers Gherkin feature files, step definitions, DataTables, async polling, WireMock, and Testcontainers lifecycle for both in-process and containerized testing."
---

# Testing Skill

BDD acceptance tests and black-box component tests for Polaris services.

## When to Activate

- Writing acceptance or end-to-end tests
- Writing component tests that treat the app as a black box
- Working with Cucumber feature files or step definitions
- Setting up Testcontainers for test infrastructure
- Testing async workflows (Kafka, Temporal) via polling

---

# Part 1: BDD Acceptance Tests (Cucumber + SpringBootTest)

Full-stack acceptance test patterns using Cucumber + `@SpringBootTest` + real Testcontainers.

## When to Use

- Writing acceptance tests for business scenarios (happy path, error flows, state transitions)
- Verifying end-to-end behaviour against a real database and real Kafka
- QA handoff: scenarios written in Gherkin, validated by engineers
- Running `@smoke` tag on CI for fast gate; `@full` tag pre-deploy

## Dependencies (build.gradle.kts)

```kotlin
testImplementation("io.cucumber:cucumber-java:7.18.0")
testImplementation("io.cucumber:cucumber-spring:7.18.0")
testImplementation("io.cucumber:cucumber-junit-platform-engine:7.18.0")
testImplementation("org.junit.platform:junit-platform-suite:1.11.0")
testImplementation("org.testcontainers:postgresql:1.20.0")
testImplementation("org.testcontainers:kafka:1.20.0")
testImplementation("org.awaitility:awaitility:4.2.2")
```

## Feature Files

Place feature files in `src/test/resources/features/`.

```gherkin
# src/test/resources/features/order-submission.feature
Feature: Order submission

  Background:
    Given the application is running

  @smoke
  Scenario: Valid order is accepted and persisted
    Given a valid order request with customerId "customer-001" and hcpcsCode "A6197"
    When the order is submitted via POST /api/v1/orders
    Then the response status is 201
    And the response body contains a non-empty orderId
    And the order is persisted in the database with status "RECEIVED"

  @smoke
  Scenario: Order with missing customerId is rejected
    Given an order request with no customerId
    When the order is submitted via POST /api/v1/orders
    Then the response status is 400
    And the response body contains a "customerId" violation

  @full
  Scenario: Duplicate order is idempotent
    Given a valid order with id "order-dup-001" already exists
    When the same order is submitted again
    Then the response status is 409
    And only one order record exists for "order-dup-001"
```

## Spring Context Configuration

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
@CucumberContextConfiguration
public class CucumberSpringConfig {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

> `@ServiceConnection` (Spring Boot 3.1+) automatically wires the `PostgreSQLContainer` JDBC URL.

## JUnit Platform Suite Runner

```java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME,
    value = "com.waveum.polaris.<service-name>.bdd")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME,
    value = "pretty, html:build/reports/cucumber/index.html, json:build/reports/cucumber/results.json")
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME,
    value = "@smoke or @full")
public class CucumberTestRunner {}
```

## Step Definitions

```java
@Component
@Slf4j
public class OrderSubmissionSteps {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository orderRepository;

    private ResponseEntity<String> lastResponse;
    private String lastOrderId;

    @Given("a valid order request with customerId {string} and hcpcsCode {string}")
    public void givenValidOrder(String customerId, String hcpcsCode) {
        ScenarioContext.put("customerId", customerId);
        ScenarioContext.put("hcpcsCode", hcpcsCode);
    }

    @When("the order is submitted via POST /api/v1/orders")
    public void submitOrder() {
        var body = """
            {
              "customerId": "%s",
              "hcpcsCode": "%s",
              "quantity": 1
            }
            """.formatted(
            ScenarioContext.get("customerId"),
            ScenarioContext.get("hcpcsCode"));

        var headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        lastResponse = restTemplate.exchange(
            "/api/v1/orders",
            HttpMethod.POST,
            new HttpEntity<>(body, headers),
            String.class);
    }

    @Then("the response status is {int}")
    public void verifyStatus(int expectedStatus) {
        assertThat(lastResponse.getStatusCode().value()).isEqualTo(expectedStatus);
    }

    @Then("the order is persisted in the database with status {string}")
    public void verifyPersistedStatus(String expectedStatus) {
        await().atMost(5, SECONDS).untilAsserted(() -> {
            var order = orderRepository.findById(UUID.fromString(lastOrderId));
            assertThat(order).isPresent();
            assertThat(order.get().getStatus().name()).isEqualTo(expectedStatus);
        });
    }
}
```

## Scenario Context (Thread-Local Shared State)

```java
public final class ScenarioContext {

    private static final ThreadLocal<Map<String, Object>> context =
        ThreadLocal.withInitial(HashMap::new);

    public static void put(String key, Object value) {
        context.get().put(key, value);
    }

    @SuppressWarnings("unchecked")
    public static <T> T get(String key) {
        return (T) context.get().get(key);
    }

    public static void reset() {
        context.get().clear();
    }

    private ScenarioContext() {}
}
```

## Gradle Task

```kotlin
val bddTest by tasks.registering(Test::class) {
    description = "Runs BDD acceptance tests (Cucumber + Testcontainers)"
    group = "verification"
    useJUnitPlatform {
        includeTags("smoke", "full")
    }
    testClassesDirs = sourceSets["test"].output.classesDirs
    classpath = sourceSets["test"].runtimeClasspath
    systemProperty("cucumber.filter.tags", System.getProperty("cucumber.filter.tags", "@smoke or @full"))
}

tasks.check { dependsOn(bddTest) }
```

```bash
# Smoke only
./gradlew bddTest -Dcucumber.filter.tags="@smoke"

# Full suite
./gradlew bddTest -Dcucumber.filter.tags="@smoke or @full"
```

## BDD Rules

- **No mocks** — all collaborators are real (DB, Kafka, HTTP)
- Use `Awaitility` for async assertions — no `Thread.sleep`
- Keep feature files in plain English; step defs in Java
- `@smoke` = fast, always CI; `@full` = slower, pre-deploy
- Each scenario cleans its own data (`@After` hooks or test-scoped transactions)
- Containers start once via static fields — not per test

---

# Part 2: Black-Box Component Tests (Docker + RestAssured)

The app is packaged as a Docker image and started via Testcontainers `GenericContainer`.
Tests interact exclusively through the public API (HTTP). No Spring context in test JVM.

## When to Use

- Verifying end-to-end user-facing behavior of the service
- Validating API contracts (request/response shapes, status codes, error formats)
- Testing workflows from a client's perspective
- Running as a pre-deploy gate in CI

## When NOT to Use

- Unit testing individual classes (use `java-development` skill)
- Testing internal state transitions or database schemas
- Testing Kafka consumer/producer wiring in isolation

## Philosophy

### Scenarios Simulate Users, Not Internals

**GOOD** - user-centric:
```gherkin
When the client submits an order for customer "C-100"
Then the response status is 202
And the response contains a workflowId
```

**BAD** - implementation detail:
```gherkin
Then the database contains a row with status "RECEIVED"
And a message is published to the "orders" Kafka topic
```

### No Spring Context

Component tests must **never** use `@SpringBootTest`, `@Autowired`, `@DynamicPropertySource`,
`TestRestTemplate`, or any Spring test annotation. The application runs in a Docker container.
Tests use `RestAssured` or `java.net.http.HttpClient` to make real HTTP calls.

## Dependencies (build.gradle)

```groovy
sourceSets {
    componentTest {
        java.srcDir 'src/componentTest/java'
        resources.srcDir 'src/componentTest/resources'
        compileClasspath += sourceSets.main.output
        runtimeClasspath += sourceSets.main.output
    }
}

configurations {
    componentTestImplementation.extendsFrom testImplementation
    componentTestRuntimeOnly.extendsFrom testRuntimeOnly
}

dependencies {
    componentTestImplementation 'io.cucumber:cucumber-java:7.22.0'
    componentTestImplementation 'io.cucumber:cucumber-junit-platform-engine:7.22.0'
    componentTestImplementation 'org.junit.platform:junit-platform-suite:1.12.2'
    componentTestImplementation 'org.testcontainers:testcontainers:1.21.1'
    componentTestImplementation 'org.testcontainers:postgresql:1.21.1'
    componentTestImplementation 'org.testcontainers:kafka:1.21.1'
    componentTestImplementation 'io.rest-assured:rest-assured:5.5.1'
    componentTestImplementation 'org.awaitility:awaitility:4.3.0'
    componentTestImplementation 'com.fasterxml.jackson.core:jackson-databind'
    componentTestImplementation 'io.cucumber:cucumber-picocontainer:7.22.0'
}

tasks.register('componentTest', Test) {
    description = 'Runs black-box component tests (Cucumber + Testcontainers)'
    group = 'verification'
    testClassesDirs = sourceSets.componentTest.output.classesDirs
    classpath = sourceSets.componentTest.runtimeClasspath
    useJUnitPlatform()
    systemProperty 'cucumber.filter.tags',
        System.getProperty('cucumber.filter.tags', '@smoke or @full')
}
```

## Container Environment

```java
public final class ContainerEnvironment {

    private static final Network NETWORK = Network.newNetwork();

    public static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withNetwork(NETWORK)
            .withNetworkAliases("postgres")
            .withDatabaseName("polaris")
            .withUsername("polaris")
            .withPassword("polaris");

    public static final KafkaContainer KAFKA =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))
            .withNetwork(NETWORK)
            .withNetworkAliases("kafka");

    public static final GenericContainer<?> APP =
        new GenericContainer<>("polaris-order-workflow-agent:latest")
            .withNetwork(NETWORK)
            .withExposedPorts(8081)
            .withEnv("SPRING_DATASOURCE_URL",
                "jdbc:postgresql://postgres:5432/polaris")
            .withEnv("SPRING_DATASOURCE_USERNAME", "polaris")
            .withEnv("SPRING_DATASOURCE_PASSWORD", "polaris")
            .withEnv("SPRING_KAFKA_BOOTSTRAP_SERVERS", "kafka:9092")
            .dependsOn(POSTGRES, KAFKA)
            .waitingFor(Wait.forHttp("/actuator/health")
                .forPort(8081)
                .forStatusCode(200));

    static {
        POSTGRES.start();
        KAFKA.start();
        APP.start();
    }

    public static String baseUrl() {
        return "http://" + APP.getHost() + ":" + APP.getMappedPort(8081);
    }

    private ContainerEnvironment() {}
}
```

## Feature Files with DataTables

```gherkin
Feature: Order submission

  Background:
    Given the service is healthy

  @smoke
  Scenario: Submit a valid order
    When the client submits an order with:
      | field        | value       |
      | orderId      | ORD-1001    |
      | customerName | Jane Doe    |
      | hcpcsCode    | A6197       |
      | quantity     | 2           |
    Then the response status is 202
    And the response contains:
      | field      | matcher  | value          |
      | workflowId | equals   | order-ORD-1001 |
      | status     | equals   | ACCEPTED       |

  @smoke
  Scenario: Submit order with missing required fields
    When the client submits an order with:
      | field    | value    |
      | orderId  | ORD-1002 |
    Then the response status is 400
```

## Reusable HTTP Steps

```java
public class CommonHttpSteps {

    private final ScenarioState state;

    public CommonHttpSteps(ScenarioState state) {
        this.state = state;
    }

    @Given("the service is healthy")
    public void theServiceIsHealthy() {
        given()
            .baseUri(ContainerEnvironment.baseUrl())
        .when()
            .get("/actuator/health")
        .then()
            .statusCode(200);
    }

    @Then("the response status is {int}")
    public void theResponseStatusIs(int expectedStatus) {
        assertThat(state.getLastResponse().statusCode())
            .isEqualTo(expectedStatus);
    }

    @Then("the response contains:")
    public void theResponseContains(DataTable dataTable) {
        var body = state.getLastResponse().jsonPath();
        for (Map<String, String> row : dataTable.asMaps()) {
            String field = row.get("field");
            String matcher = row.get("matcher");
            String expected = row.get("value");
            String actual = body.getString(field);

            switch (matcher) {
                case "equals" -> assertThat(actual).isEqualTo(expected);
                case "contains" -> assertThat(actual).contains(expected);
                case "notEmpty" -> assertThat(actual).isNotEmpty();
                case "oneOf" -> assertThat(actual).isIn((Object[]) expected.split(","));
                default -> throw new IllegalArgumentException("Unknown matcher: " + matcher);
            }
        }
    }
}
```

## Async Behavior — Polling Pattern

```java
@Then("the order reaches status {string} within {int} seconds")
public void theOrderReachesStatus(String expectedStatus, int timeoutSeconds) {
    String workflowId = state.get("workflowId");

    await()
        .atMost(Duration.ofSeconds(timeoutSeconds))
        .pollInterval(Duration.ofSeconds(2))
        .untilAsserted(() -> {
            var response = given()
                .baseUri(ContainerEnvironment.baseUrl())
            .when()
                .get("/api/orders/status/" + workflowId);

            assertThat(response.jsonPath().getString("status"))
                .isEqualTo(expectedStatus);
        });
}
```

**Never** use `Thread.sleep`. Always poll the public API.

## WireMock for External Dependencies

```java
public static final GenericContainer<?> WIREMOCK =
    new GenericContainer<>("wiremock/wiremock:3.12.1")
        .withNetwork(NETWORK)
        .withNetworkAliases("wiremock")
        .withExposedPorts(8080)
        .waitingFor(Wait.forHttp("/__admin/mappings").forStatusCode(200));
```

Point the app at WireMock via environment variables:

```java
.withEnv("STEDI_BASE_URL", "http://wiremock:8080")
```

## Running Component Tests

```bash
# Smoke (fast CI gate)
./gradlew componentTest -Dcucumber.filter.tags="@smoke"

# Full suite (pre-deploy)
./gradlew componentTest -Dcucumber.filter.tags="@smoke or @full"
```

## Component Test Rules

1. **No Spring context** — never use `@SpringBootTest`, `@Autowired`, or `TestRestTemplate`
2. **No internal assertions** — never query the database, consume Kafka, or inspect beans
3. **User perspective only** — every Then step asserts on HTTP responses
4. **DataTables for input** — use `| field | value |` tables, not long prose
5. **One behavior per scenario** — use Scenario Outline for variants
6. **Background for preconditions** — shared Given steps go in Background
7. **Tags for filtering** — `@smoke` for CI, `@full` for pre-deploy, `@wip` for in-progress
8. **Awaitility for async** — poll the public API; never `Thread.sleep`
9. **Containers start once** — `static {}` block; never restart per scenario
10. **Declarative Gherkin** — domain language, not HTTP verbs
