---
name: component-tests
description: Black-box component tests using Cucumber BDD feature files, Testcontainers (app runs as a Docker container), and RestAssured. No Spring context loaded in test JVM. Scenarios simulate real user behavior via HTTP — never inspect databases, Kafka topics, or internal state directly. Use DataTables for parameterized input.
---

# Component Tests Skill

Black-box component tests that treat the running application as an opaque service.
The app is packaged as a Docker image and started via Testcontainers `GenericContainer`.
Tests interact exclusively through the public API (HTTP). No Spring context is loaded in
the test JVM. No database queries, no Kafka consumer assertions, no internal bean access.

## When to Use

- Verifying end-to-end user-facing behavior of the service
- Validating API contracts (request/response shapes, status codes, error formats)
- Testing workflows from a client's perspective (submit order, check status, receive callback)
- Running as a pre-deploy gate in CI

## When NOT to Use

- Unit testing individual classes (use `java-unit-testing` skill)
- Testing internal state transitions or database schemas (that is an integration test)
- Testing Kafka consumer/producer wiring in isolation

---

## Philosophy

### Scenarios Simulate Users, Not Internals

Every scenario must describe what a **user or API client** would do and observe.

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

If the user cannot observe it through the API, it does not belong in a component test scenario.

### No Spring Context

Component tests must **never** use `@SpringBootTest`, `@Autowired`, `@DynamicPropertySource`,
`TestRestTemplate`, or any Spring test annotation. The application runs in a Docker container.
Tests use `RestAssured` or `java.net.http.HttpClient` to make real HTTP calls.

---

## Dependencies (build.gradle)

```groovy
// ── Component test source set ──────────────────────────────────────────
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
}

tasks.register('componentTest', Test) {
    description = 'Runs black-box component tests (Cucumber + Testcontainers)'
    group = 'verification'
    testClassesDirs = sourceSets.componentTest.output.classesDirs
    classpath = sourceSets.componentTest.runtimeClasspath
    useJUnitPlatform()
    jvmArgs '--sun-misc-unsafe-memory-access=allow'

    // Pass tag filter: ./gradlew componentTest -Dcucumber.filter.tags="@smoke"
    systemProperty 'cucumber.filter.tags',
        System.getProperty('cucumber.filter.tags', '@smoke or @full')
}
```

---

## Directory Layout

```
src/
  componentTest/
    java/
      com/polaris/order/component/
        ContainerEnvironment.java       # Testcontainers lifecycle
        CucumberComponentTest.java      # JUnit suite runner
        ScenarioState.java              # Shared state between steps
        steps/
          OrderSteps.java               # Step definitions per feature
          EligibilitySteps.java
          CommonHttpSteps.java          # Reusable HTTP step defs
    resources/
      features/
        order-submission.feature
        eligibility-check.feature
      junit-platform.properties
```

---

## 1. Container Environment

Start the application as a Docker container alongside its dependencies.
All containers start once and are shared across the entire test suite.

```java
package com.polaris.order.component;

import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.containers.Network;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.utility.DockerImageName;

public final class ContainerEnvironment {

    private static final Network NETWORK = Network.newNetwork();

    // ── Infrastructure containers ──────────────────────────────────────
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

    public static final GenericContainer<?> TEMPORAL =
        new GenericContainer<>("temporalio/auto-setup:1.25.2")
            .withNetwork(NETWORK)
            .withNetworkAliases("temporal")
            .withEnv("DB", "postgres12")
            .withEnv("DB_PORT", "5432")
            .withEnv("POSTGRES_USER", "polaris")
            .withEnv("POSTGRES_PWD", "polaris")
            .withEnv("POSTGRES_SEEDS", "postgres")
            .withExposedPorts(7233)
            .dependsOn(POSTGRES)
            .waitingFor(Wait.forListeningPort());

    // ── Application container ──────────────────────────────────────────
    public static final GenericContainer<?> APP =
        new GenericContainer<>("polaris-order-workflow-agent:latest")
            .withNetwork(NETWORK)
            .withExposedPorts(8081)
            .withEnv("SPRING_DATASOURCE_URL",
                "jdbc:postgresql://postgres:5432/polaris")
            .withEnv("SPRING_DATASOURCE_USERNAME", "polaris")
            .withEnv("SPRING_DATASOURCE_PASSWORD", "polaris")
            .withEnv("SPRING_KAFKA_BOOTSTRAP_SERVERS", "kafka:9092")
            .withEnv("TEMPORAL_SERVICE_ADDRESS", "temporal:7233")
            .dependsOn(POSTGRES, KAFKA, TEMPORAL)
            .waitingFor(Wait.forHttp("/actuator/health")
                .forPort(8081)
                .forStatusCode(200));

    static {
        POSTGRES.start();
        KAFKA.start();
        TEMPORAL.start();
        APP.start();
    }

    public static String baseUrl() {
        return "http://" + APP.getHost() + ":" + APP.getMappedPort(8081);
    }

    private ContainerEnvironment() {}
}
```

### Building the Docker Image Before Tests

Add to `build.gradle`:

```groovy
tasks.register('buildDockerImage', Exec) {
    description = 'Builds the application Docker image for component tests'
    group = 'build'
    dependsOn bootJar
    commandLine 'docker', 'build', '-t', 'polaris-order-workflow-agent:latest', '.'
}

tasks.named('componentTest') {
    dependsOn 'buildDockerImage'
}
```

Minimal `Dockerfile`:

```dockerfile
FROM eclipse-temurin:25-jre-alpine
COPY build/libs/polaris-order-workflow-agent-*.jar /app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

---

## 2. JUnit Suite Runner

```java
package com.polaris.order.component;

import static io.cucumber.junit.platform.engine.Constants.*;

import org.junit.platform.suite.api.*;

@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME,
    value = "com.polaris.order.component")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME,
    value = "pretty, html:build/reports/component-tests/index.html, "
          + "json:build/reports/component-tests/results.json")
public class CucumberComponentTest {}
```

`src/componentTest/resources/junit-platform.properties`:

```properties
cucumber.publish.quiet=true
```

---

## 3. Scenario State

Plain Java object shared between steps within a single scenario. No Spring, no ThreadLocal needed
because Cucumber runs steps sequentially within a scenario.

```java
package com.polaris.order.component;

import io.restassured.response.Response;
import java.util.HashMap;
import java.util.Map;

public class ScenarioState {

    private Response lastResponse;
    private final Map<String, Object> data = new HashMap<>();

    public Response getLastResponse() { return lastResponse; }
    public void setLastResponse(Response response) { this.lastResponse = response; }

    public void put(String key, Object value) { data.put(key, value); }
    @SuppressWarnings("unchecked")
    public <T> T get(String key) { return (T) data.get(key); }

    public void reset() {
        lastResponse = null;
        data.clear();
    }
}
```

Reset between scenarios:

```java
package com.polaris.order.component.steps;

import com.polaris.order.component.ScenarioState;
import io.cucumber.java.Before;

public class Hooks {

    private final ScenarioState state;

    public Hooks(ScenarioState state) {
        this.state = state;
    }

    @Before
    public void resetState() {
        state.reset();
    }
}
```

> Cucumber-JVM creates a new instance of each step definition class per scenario by default
> (using its built-in PicoContainer). Inject `ScenarioState` via constructor for sharing across
> step definition classes.

Add PicoContainer for Cucumber DI (no Spring needed):

```groovy
componentTestImplementation 'io.cucumber:cucumber-picocontainer:7.22.0'
```

---

## 4. Feature Files

### Writing Good Scenarios

| Do | Don't |
|---|---|
| Describe what the user/client does and sees | Assert on database rows or Kafka messages |
| Use domain language ("submit an order") | Use technical language ("POST to /api/orders") |
| One behavior per scenario | Multiple unrelated assertions |
| Use DataTables for structured input | Hardcode every field in prose |
| Use Background for shared preconditions | Repeat Given steps in every scenario |
| Use tags (`@smoke`, `@full`, `@wip`) | Leave scenarios untagged |

### Example: Order Submission

```gherkin
# src/componentTest/resources/features/order-submission.feature
Feature: Order submission

  As an API client
  I want to submit medical supply orders
  So that they are processed through the workflow

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
      | field      | matcher  | value       |
      | workflowId | equals   | order-ORD-1001 |
      | status     | equals   | ACCEPTED    |
      | message    | contains | submitted   |

  @smoke
  Scenario: Submit order with missing required fields
    When the client submits an order with:
      | field    | value    |
      | orderId  | ORD-1002 |
    Then the response status is 400
    And the response contains:
      | field   | matcher  | value          |
      | error   | contains | customerName   |

  @full
  Scenario Outline: Submit orders for different product codes
    When the client submits an order with:
      | field        | value          |
      | orderId      | <orderId>      |
      | customerName | <customerName> |
      | hcpcsCode    | <hcpcsCode>    |
      | quantity     | <quantity>     |
    Then the response status is <status>

    Examples:
      | orderId  | customerName | hcpcsCode | quantity | status |
      | ORD-2001 | Alice Smith  | A6197     | 1        | 202    |
      | ORD-2002 | Bob Jones    | E0601     | 3        | 202    |
      | ORD-2003 | Carol White  | INVALID   | 1        | 400    |

  @full
  Scenario: Check order processing status
    Given the client submitted an order with:
      | field        | value       |
      | orderId      | ORD-3001    |
      | customerName | Jane Doe    |
      | hcpcsCode    | A6197       |
      | quantity     | 1           |
    When the client checks the order status for workflow "order-ORD-3001"
    Then the response status is 200
    And the response contains:
      | field  | matcher | value     |
      | status | oneOf   | RUNNING,COMPLETED |
```

### Example: Eligibility Check

```gherkin
# src/componentTest/resources/features/eligibility-check.feature
Feature: Coverage eligibility check

  As an API client
  I want to verify patient insurance coverage
  So that I know if a supply order will be covered

  Background:
    Given the service is healthy

  @smoke
  Scenario: Check eligibility for a covered patient
    When the client requests eligibility for:
      | field      | value          |
      | memberId   | MBR-ACTIVE-001 |
      | providerId | NPI-1234567893 |
      | serviceCode| A6197          |
    Then the response status is 200
    And the response contains:
      | field    | matcher | value  |
      | eligible | equals  | true   |
      | status   | equals  | active |

  @smoke
  Scenario: Check eligibility for an inactive member
    When the client requests eligibility for:
      | field      | value            |
      | memberId   | MBR-INACTIVE-001 |
      | providerId | NPI-1234567893   |
      | serviceCode| A6197            |
    Then the response status is 200
    And the response contains:
      | field    | matcher | value    |
      | eligible | equals  | false    |
      | status   | equals  | inactive |

  @full
  Scenario: Eligibility check with missing member ID
    When the client requests eligibility for:
      | field      | value          |
      | providerId | NPI-1234567893 |
      | serviceCode| A6197          |
    Then the response status is 400
    And the response contains:
      | field | matcher  | value    |
      | error | contains | memberId |

  @full
  Scenario Outline: Eligibility for various coverage scenarios
    When the client requests eligibility for:
      | field      | value        |
      | memberId   | <memberId>   |
      | providerId | <providerId> |
      | serviceCode| <serviceCode>|
    Then the response status is <status>
    And the response contains:
      | field    | matcher | value      |
      | eligible | equals  | <eligible> |

    Examples:
      | memberId         | providerId     | serviceCode | status | eligible |
      | MBR-ACTIVE-001   | NPI-1234567893 | A6197       | 200    | true     |
      | MBR-ACTIVE-001   | NPI-1234567893 | E0601       | 200    | true     |
      | MBR-INACTIVE-001 | NPI-1234567893 | A6197       | 200    | false    |
      | MBR-NOTFOUND-001 | NPI-1234567893 | A6197       | 404    | false    |
```

---

## 5. Step Definitions

### Reusable HTTP Steps

Generic steps that any feature can use. Keep domain-specific logic in dedicated step classes.

```java
package com.polaris.order.component.steps;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import com.polaris.order.component.ContainerEnvironment;
import com.polaris.order.component.ScenarioState;

import io.cucumber.datatable.DataTable;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.restassured.http.ContentType;

import java.util.Map;

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
            .as("HTTP status code")
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
                case "equals" -> assertThat(actual)
                    .as("Field '%s'", field).isEqualTo(expected);
                case "contains" -> assertThat(actual)
                    .as("Field '%s'", field).contains(expected);
                case "notEmpty" -> assertThat(actual)
                    .as("Field '%s'", field).isNotEmpty();
                case "oneOf" -> assertThat(actual)
                    .as("Field '%s'", field).isIn((Object[]) expected.split(","));
                default -> throw new IllegalArgumentException(
                    "Unknown matcher: " + matcher);
            }
        }
    }
}
```

### Domain Step Definitions

```java
package com.polaris.order.component.steps;

import static io.restassured.RestAssured.given;

import com.polaris.order.component.ContainerEnvironment;
import com.polaris.order.component.ScenarioState;

import io.cucumber.datatable.DataTable;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.When;
import io.restassured.http.ContentType;

import java.util.Map;

public class OrderSteps {

    private final ScenarioState state;

    public OrderSteps(ScenarioState state) {
        this.state = state;
    }

    @When("the client submits an order with:")
    public void theClientSubmitsAnOrderWith(DataTable dataTable) {
        Map<String, String> fields = dataTable.asMap(String.class, String.class);

        var response = given()
            .baseUri(ContainerEnvironment.baseUrl())
            .contentType(ContentType.JSON)
            .body(fields)
        .when()
            .post("/api/orders");

        state.setLastResponse(response);
    }

    @Given("the client submitted an order with:")
    public void theClientSubmittedAnOrderWith(DataTable dataTable) {
        theClientSubmitsAnOrderWith(dataTable);
        // Store the workflowId for subsequent steps
        var workflowId = state.getLastResponse().jsonPath().getString("workflowId");
        state.put("workflowId", workflowId);
    }

    @When("the client checks the order status for workflow {string}")
    public void theClientChecksOrderStatus(String workflowId) {
        var response = given()
            .baseUri(ContainerEnvironment.baseUrl())
        .when()
            .get("/api/orders/status/" + workflowId);

        state.setLastResponse(response);
    }
}
```

---

## 6. DataTable Best Practices

### Use `| field | value |` for Single-Entity Input

Best for submitting a single request with named fields:

```gherkin
When the client submits an order with:
  | field        | value    |
  | orderId      | ORD-1001 |
  | customerName | Jane Doe |
```

```java
Map<String, String> fields = dataTable.asMap(String.class, String.class);
```

### Use Column Headers for Multi-Row Data

Best for Scenario Outlines embedded in Examples, or for asserting on lists:

```gherkin
Then the response contains orders:
  | orderId  | status    |
  | ORD-1001 | COMPLETED |
  | ORD-1002 | RUNNING   |
```

```java
List<Map<String, String>> rows = dataTable.asMaps();
```

### Use `| field | matcher | value |` for Flexible Assertions

Allows different assertion types per field without writing new step defs:

```gherkin
Then the response contains:
  | field      | matcher  | value       |
  | workflowId | equals   | order-ORD-1 |
  | status     | equals   | ACCEPTED    |
  | message    | contains | submitted   |
  | createdAt  | notEmpty |             |
```

### Use Scenario Outline + Examples for Parameterized Variants

```gherkin
Scenario Outline: Validate order input
  When the client submits an order with:
    | field        | value          |
    | orderId      | <orderId>      |
    | customerName | <customerName> |
  Then the response status is <status>

  Examples:
    | orderId  | customerName | status |
    | ORD-V001 | Valid Name   | 202    |
    |          | Valid Name   | 400    |
    | ORD-V003 |              | 400    |
```

---

## 7. Async Behavior — Polling Pattern

For workflows and async operations, poll the public API with Awaitility:

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

In the feature file:

```gherkin
Scenario: Order completes processing
  Given the client submitted an order with:
    | field        | value    |
    | orderId      | ORD-4001 |
    | customerName | Jane Doe |
    | hcpcsCode    | A6197    |
    | quantity     | 1        |
  Then the order reaches status "COMPLETED" within 30 seconds
```

**Never** use `Thread.sleep`. Always poll the public API.

---

## 8. WireMock for External Dependencies

When the application depends on external APIs (Stedi, FHIR payers), start a WireMock
Testcontainer and initialize its stubs from the test. Point the application container
at it via environment variables.

```java
public static final GenericContainer<?> WIREMOCK =
    new GenericContainer<>("wiremock/wiremock:3.12.1")
        .withNetwork(NETWORK)
        .withNetworkAliases("wiremock")
        .withExposedPorts(8080)
        .waitingFor(Wait.forHttp("/__admin/mappings")
            .forStatusCode(200));
```

Register stubs programmatically using the WireMock admin API from the test:

```java
int wiremockPort = WIREMOCK.getMappedPort(8080);
WireMock client = new WireMock("localhost", wiremockPort);
client.register(post(urlEqualTo("/path"))
    .willReturn(okJson("{...}")));
```

Then point the app at WireMock via environment variables:

```java
public static final GenericContainer<?> APP =
    new GenericContainer<>("polaris-order-workflow-agent:latest")
        // ...existing env vars...
        .withEnv("STEDI_BASE_URL", "http://wiremock:8080")
        .withEnv("FHIR_BASE_URL", "http://wiremock:8080");
```

---

## 9. Running Tests

```bash
# Run smoke tests (fast CI gate)
./gradlew componentTest -Dcucumber.filter.tags="@smoke"

# Run full suite (pre-deploy)
./gradlew componentTest -Dcucumber.filter.tags="@smoke or @full"

# Run a specific feature
./gradlew componentTest -Dcucumber.filter.tags="@smoke" \
  -Dcucumber.features="src/componentTest/resources/features/order-submission.feature"
```

---

## Rules

1. **No Spring context** — never use `@SpringBootTest`, `@Autowired`, or `TestRestTemplate`
2. **No internal assertions** — never query the database, consume Kafka, or inspect beans
3. **User perspective only** — every Then step asserts on HTTP responses the client would see
4. **DataTables for input** — use `| field | value |` tables, not long prose Given steps
5. **One behavior per scenario** — a scenario tests one thing; use Scenario Outline for variants
6. **Background for preconditions** — shared Given steps go in Background, not repeated
7. **Tags for filtering** — `@smoke` for fast CI, `@full` for comprehensive pre-deploy, `@wip` for in-progress
8. **Awaitility for async** — poll the public API; never `Thread.sleep`
9. **Containers start once** — use `static {}` block in `ContainerEnvironment`; never restart per scenario
10. **Declarative Gherkin** — domain language ("submit an order"), not HTTP verbs ("POST to /api/orders")
