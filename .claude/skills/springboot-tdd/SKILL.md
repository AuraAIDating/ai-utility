---
name: springboot-tdd
description: Unit-test-driven development for Spring Boot using JUnit 5, Mockito, MockMvc, and H2. Use when adding features, fixing bugs, or refactoring. For full-stack BDD acceptance tests, see the bdd-cucumber skill.
---

# Spring Boot TDD Workflow

Unit-test TDD guidance for Spring Boot services — fast, isolated, no containers.

## When to Use

- New features or endpoints
- Bug fixes or refactors
- Adding data access logic

## Workflow

1) Write tests first (they should fail)
2) Implement minimal code to pass
3) Refactor with tests green
4) Enforce coverage (JaCoCo)

## Unit Tests (JUnit 5 + Mockito)

```java
@ExtendWith(MockitoExtension.class)
class MarketServiceTest {
  @Mock MarketRepository repo;
  @InjectMocks MarketService service;

  @Test
  void createsMarket() {
    CreateMarketRequest req = new CreateMarketRequest("name", "desc", Instant.now(), List.of("cat"));
    when(repo.save(any())).thenAnswer(inv -> inv.getArgument(0));

    Market result = service.create(req);

    assertThat(result.name()).isEqualTo("name");
    verify(repo).save(any());
  }
}
```

Patterns:

- Arrange-Act-Assert
- Avoid partial mocks; prefer explicit stubbing
- Use `@ParameterizedTest` for variants

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

## Integration Tests (SpringBootTest)

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class MarketIntegrationTest {
  @Autowired MockMvc mockMvc;

  @Test
  void createsMarket() throws Exception {
    mockMvc.perform(post("/api/markets")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""
          {"name":"Test","description":"Desc","endDate":"2030-01-01T00:00:00Z","categories":["general"]}
        """))
      .andExpect(status().isCreated());
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

> For tests that need a real PostgreSQL container, use the `bdd-cucumber` skill.

## Coverage (JaCoCo)

Gradle snippet (`build.gradle.kts`):

```kotlin
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

## Assertions

- Prefer AssertJ (`assertThat`) for readability
- For JSON responses, use `jsonPath`
- For exceptions: `assertThatThrownBy(...)`

## Test Data Builders

```java
class MarketBuilder {
  private String name = "Test";
  MarketBuilder withName(String name) { this.name = name; return this; }
  Market build() { return new Market(null, name, MarketStatus.ACTIVE); }
}
```

## CI Commands

```bash
./gradlew test jacocoTestReport
```

**Remember**: Keep tests fast, isolated, and deterministic. Test behavior, not implementation details.
