---
name: bdd-cucumber
description: Full-stack BDD acceptance tests using Cucumber, @SpringBootTest, and real Testcontainers (PostgreSQL + Kafka). Use when writing end-to-end scenario tests covering the full application stack. For fast unit and web-layer tests, see the springboot-tdd skill.
---

# BDD Cucumber Skill

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

---

## 1. Feature Files

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

---

## 2. Spring Context Configuration

```java
// src/test/java/com/waveum/polaris/<service-name>/bdd/CucumberSpringConfig.java
package com.waveum.polaris.<service-name>.bdd;

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

> `@ServiceConnection` (Spring Boot 3.1+) automatically wires the `PostgreSQLContainer` JDBC URL
> into the application context — no manual `@DynamicPropertySource` needed for Postgres.

---

## 3. JUnit Platform Suite Runner

```java
// src/test/java/com/waveum/polaris/<service-name>/bdd/CucumberTestRunner.java
package com.waveum.polaris.<service-name>.bdd;

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

---

## 4. Step Definitions

```java
// src/test/java/com/waveum/polaris/<service-name>/bdd/steps/OrderSubmissionSteps.java
package com.waveum.polaris.<service-name>.bdd.steps;

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

    @Then("the response body contains a non-empty orderId")
    public void verifyOrderIdPresent() throws Exception {
        var node = new ObjectMapper().readTree(lastResponse.getBody());
        lastOrderId = node.path("orderId").asText();
        assertThat(lastOrderId).isNotBlank();
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

---

## 5. Scenario Context (Thread-Local Shared State)

```java
// src/test/java/com/waveum/polaris/<service-name>/bdd/ScenarioContext.java
package com.waveum.polaris.<service-name>.bdd;

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

Clean the context between scenarios:

```java
@Component
public class ScenarioHooks {

    @Before
    public void beforeScenario() {
        ScenarioContext.reset();
    }
}
```

---

## 6. Gradle Task for BDD Tests

```kotlin
// build.gradle.kts
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

// Run smoke tests as part of CI check (fast gate)
tasks.check { dependsOn(bddTest) }
```

Run smoke suite only:

```bash
./gradlew bddTest -Dcucumber.filter.tags="@smoke"
```

Run full suite pre-deploy:

```bash
./gradlew bddTest -Dcucumber.filter.tags="@smoke or @full"
```

---

## 7. Rules

- **No mocks in BDD tests** — all collaborators are real (DB, Kafka, HTTP)
- Use `Awaitility` for async assertions (Kafka consumers, events) — no `Thread.sleep`
- Keep feature files in plain English; step definitions in Java — do not leak Java in Gherkin
- `@smoke` = fast, stateless, always run in CI; `@full` = stateful, slower, run pre-deploy
- Each scenario must clean up its own data (use `@After` hooks or test-scoped transactions)
- Containers are started once per test suite via static fields — not per test
