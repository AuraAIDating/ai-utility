---
name: java-unit-testing
description: Skill for writing Java unit tests using JUnit 5, Mockito, and AssertJ. Use this when asked to write, generate, or create unit tests for Java classes.
---

# Java Unit Testing Skill

## Tech Stack

- **JUnit 5** (`org.junit.jupiter`) for test framework
- **Mockito** (`org.mockito`) for mocking dependencies
- **AssertJ** (`org.assertj.core.api.Assertions`) for fluent assertions
- **Parameterized Tests** (`@ParameterizedTest`) for testing multiple inputs

## Test File Placement

- Place test files under `src/test/java` mirroring the source package structure.
- Name test classes as `{ClassName}Test.java`.

## Imports

Always use these imports (add only what is needed per test class):

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.MethodSource;
import org.junit.jupiter.params.provider.NullAndEmptySource;
import org.junit.jupiter.params.provider.ValueSource;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;
```

## Class Structure

```java
@ExtendWith(MockitoExtension.class)
class FooServiceTest {

    @Mock
    private BarRepository barRepository;

    @InjectMocks
    private FooService fooService;

    // Group related tests using @Nested classes
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

1. **Use AssertJ for all assertions** — never use JUnit `assertEquals`/`assertTrue` etc.
2. **Use `@ParameterizedTest`** when a test would otherwise be duplicated for multiple inputs. Prefer `@CsvSource` for simple value combinations, `@MethodSource` for complex objects, and `@ValueSource`/`@NullAndEmptySource` for single-parameter variations.
3. **Annotate every test with `@DisplayName`** using a human-readable sentence that describes the expected behavior.
4. **Follow the given/when/then pattern** — use comments `// given`, `// when`, `// then` to separate the three phases in each test.
5. **Use `@Mock` and `@InjectMocks`** — do not manually construct mocks with `Mockito.mock()` unless the test specifically requires it.
6. **Verify interactions when meaningful** — use `verify()` to assert that the correct collaborator methods were called with expected arguments. Use `verifyNoMoreInteractions()` sparingly, only when important.
7. **Test one behavior per test method** — keep tests focused on a single logical assertion or a closely related group of assertions.
8. **Name test methods descriptively** — use the pattern `should{Expected}_when{Condition}` (e.g., `shouldReturnEmpty_whenInputIsNull`).
9. **Group related tests with `@Nested` classes** — group by the method under test or by scenario.
10. **Do not generate Javadoc comments** on test classes or methods — `@DisplayName` serves as the documentation.
11. **Do not generate summary files** unless explicitly asked for.
12. **Keep tests independent** — no test should depend on the execution order or outcome of another test. Avoid shared mutable state.
13. **Use `@BeforeEach`** only for setup logic shared across multiple tests in the same class or nested class.
14. **Use strict stubbing** — only stub methods that the test actually exercises. Rely on Mockito's default strict stubbing behavior.
15. **Assert exceptions with AssertJ** — use `assertThatThrownBy(() -> ...)` or `assertThatCode(() -> ...).doesNotThrowAnyException()`.
16. **For Temporal workflow/activity tests** — mock the activity stubs and use Temporal's `TestWorkflowEnvironment` or mock activities directly. Follow the Temporal Java SDK testing patterns.

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
    // when
    boolean result = service.isValidTransition(from, to);

    // then
    assertThat(result).isEqualTo(expected);
}
```

### Complex objects with `@MethodSource`
```java
@ParameterizedTest
@DisplayName("should process order correctly for various inputs")
@MethodSource("provideOrders")
void shouldProcessOrder(Order order, String expectedStatus) {
    // when
    OrderResult result = service.process(order);

    // then
    assertThat(result.getStatus()).isEqualTo(expectedStatus);
}

private static Stream<Arguments> provideOrders() {
    return Stream.of(
        Arguments.of(new Order("item1", 1), "CONFIRMED"),
        Arguments.of(new Order("item2", 0), "REJECTED")
    );
}
```

### Null and empty inputs with `@NullAndEmptySource`
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

// Boolean
assertThat(actual).isTrue();
assertThat(actual).isFalse();

// String
assertThat(actual).contains("substring");
assertThat(actual).startsWith("prefix");
assertThat(actual).isBlank();

// Collections
assertThat(list).hasSize(3);
assertThat(list).containsExactly("a", "b", "c");
assertThat(list).containsExactlyInAnyOrder("c", "a", "b");
assertThat(list).isEmpty();
assertThat(list).extracting("name").containsOnly("Alice", "Bob");

// Exceptions
assertThatThrownBy(() -> service.call())
    .isInstanceOf(RuntimeException.class)
    .hasMessageContaining("failed");

// No exception
assertThatCode(() -> service.call()).doesNotThrowAnyException();

// Optional
assertThat(optional).isPresent().hasValue(expected);
assertThat(optional).isEmpty();
```

