# Java SDK — Agentic Workflow Patterns

This document provides Java-specific implementation guidance for building agentic workflows with Temporal. See `references/core/ai-patterns.md` for conceptual background.

## Project Structure

```
com.polaris.order
├── model/
│   ├── AgentGoal.java              # Goal definition record
│   ├── ToolDefinition.java         # Tool metadata record
│   └── ToolArgument.java           # Tool argument schema record
│
├── goals/
│   └── DummyOrderGoals.java        # Static goal constants
│
├── tools/
│   ├── ToolRegistry.java           # Central registry of all ToolDefinitions
│   └── dummyorder/
│       ├── ValidateOrderTool.java  # Fast tool (pure logic)
│       ├── PlaceOrderTool.java     # Fast tool (pure logic)
│       ├── CheckChildSuitabilityTool.java   # LLM-powered tool
│       └── SuggestAlternativeTool.java      # LLM-powered tool
│
├── prompts/
│   └── PromptGenerator.java        # Builds LLM prompts from AgentGoal metadata
│
├── workflow/
│   └── dummyorder/
│       ├── OrderWorkflow.java      # @WorkflowInterface
│       └── OrderWorkflowImpl.java  # Agent loop implementation
│
├── activities/
│   └── dummyorder/
│       ├── ToolActivities.java     # @ActivityInterface for fast tools
│       ├── OrderActivitiesImpl.java # Fast tool activity impl
│       ├── LlmActivities.java     # @ActivityInterface for LLM tools
│       └── LlmActivitiesImpl.java  # LLM tool activity impl
│
├── worker/
│   ├── DummyOrderWorker.java       # Registers workflow + activities
│   └── WorkerBootstrap.java        # Starts WorkerFactory
│
└── config/
    └── WorkflowProperties.java     # Static config for workflow classes
```

## AgentGoal Record

Define goals as Java records. Each goal is a self-contained unit bundling the objective, available tools, starter prompt, and few-shot example.

```java
public record AgentGoal(
    String id,
    String name,
    String description,
    List<ToolDefinition> tools,
    String starterPrompt,
    String exampleConversationHistory) {
}
```

## ToolDefinition and ToolArgument Records

Tool metadata is separate from tool implementation. Use `@JsonProperty` annotations for serialization.

```java
public record ToolDefinition(
    @JsonProperty("name") String name,
    @JsonProperty("description") String description,
    @JsonProperty("arguments") List<ToolArgument> arguments) {
}

public record ToolArgument(
    @JsonProperty("name") String name,
    @JsonProperty("type") String type,
    @JsonProperty("description") String description) {
}
```

## ToolRegistry

Define all tools as `static final` constants in a single registry class. This is the single source of truth consumed by goals, prompts, and dispatch logic.

```java
public final class ToolRegistry {

    private ToolRegistry() {}

    public static final ToolDefinition VALIDATE_ORDER = new ToolDefinition(
        "ValidateOrder",
        "Validate the incoming order structure (ID, customer name, items, quantities).",
        List.of(
            new ToolArgument("orderId", "string", "The unique order identifier"),
            new ToolArgument("customerName", "string", "Name of the customer"),
            new ToolArgument("items", "array", "List of order items")));

    public static final ToolDefinition CHECK_CHILD_SUITABILITY = new ToolDefinition(
        "CheckChildSuitability",
        "Determine whether a product is appropriate for children under 16.",
        List.of(
            new ToolArgument("itemName", "string", "The name of the product to evaluate")));

    // ... more tools ...
}
```

**Adding a new tool requires:**
1. Add the `ToolDefinition` constant to `ToolRegistry`
2. Create the tool implementation class (in `tools/`)
3. Add the activity method to the appropriate `@ActivityInterface`
4. Add the activity implementation
5. Add a `case` to the workflow's `dispatch()` method
6. Include the tool in the relevant `AgentGoal`

## Goal Definitions

Define goals as `static final` constants in dedicated goal classes. Each goal references tools from the `ToolRegistry`.

```java
public final class DummyOrderGoals {

    private DummyOrderGoals() {}

    public static final AgentGoal CHILD_SAFE_ORDER = new AgentGoal(
        "goal_child_safe_order",
        "Child-Safe Order Processing",

        // Step-by-step description for the LLM
        "The user wants to place an order that must be safe for children under 16. "
            + "To achieve this goal, follow these steps IN ORDER: "
            + "1. ValidateOrder: validate the order structure. "
            + "2. CheckChildSuitability: check EVERY item for child-safety. "
            + "3. SuggestAlternativeItem: for each unsuitable item, suggest a replacement. "
            + "4. PlaceOrder: place the finalised order with any replacements applied. "
            + "Do NOT skip steps. Do NOT call PlaceOrder until ALL items have been checked.",

        // Tools available for this goal (from ToolRegistry)
        List.of(ToolRegistry.VALIDATE_ORDER, ToolRegistry.CHECK_CHILD_SUITABILITY,
            ToolRegistry.SUGGEST_ALTERNATIVE, ToolRegistry.PLACE_ORDER),

        // Starter prompt
        "Process the order: validate, check child-safety of each item, "
            + "replace unsafe items, then place the order.",

        // Few-shot example conversation (multiline string)
        String.join("\n",
            "user: Process order ORD-1234 for 'Alice' with items: [Hunting Knife x1, Teddy Bear x2]",
            "agent: {\"next\":\"confirm\",\"tool\":\"ValidateOrder\",...}",
            "tool_result: Order ORD-1234 is valid",
            // ... complete example showing all tool calls and results
            "agent: {\"next\":\"done\",\"tool\":null,...}"));
}
```

## Activity Interfaces — Dual Task Queue Pattern

Split activities into two interfaces based on latency profile:

### Fast Local Activities (ToolActivities)

Short timeouts, main task queue, high concurrency.

```java
@ActivityInterface
public interface ToolActivities {

    @ActivityMethod
    String ValidateOrder(Order order);

    @ActivityMethod
    String PlaceOrder(Order finalOrder, List<OrderChange> changes);
}
```

Implementation delegates to static tool classes:

```java
@Component
public class OrderActivitiesImpl implements ToolActivities {

    @Override
    public String ValidateOrder(Order order) {
        return ValidateOrderTool.execute(order);
    }

    @Override
    public String PlaceOrder(Order finalOrder, List<OrderChange> changes) {
        return PlaceOrderTool.execute(finalOrder, changes);
    }
}
```

### LLM Activities (LlmActivities)

Longer timeouts, dedicated task queue, lower concurrency.

```java
@ActivityInterface
public interface LlmActivities {

    @ActivityMethod
    String PlanNextTool(AgentGoal goal, List<Map<String, String>> conversationHistory);

    @ActivityMethod
    String CheckChildSuitability(String itemName);

    @ActivityMethod
    String SuggestAlternativeItem(String itemName, String reason, String suggestedAlternative);
}
```

Implementation injects the LLM client:

```java
@Component
public class LlmActivitiesImpl implements LlmActivities {

    private final GeminiClient gemini;

    public LlmActivitiesImpl(GeminiClient gemini) {
        this.gemini = gemini;
    }

    @Override
    public String PlanNextTool(AgentGoal goal, List<Map<String, String>> conversationHistory) {
        String systemPrompt = PromptGenerator.generateSystemPrompt(goal, conversationHistory);
        return gemini.call(systemPrompt);
    }

    @Override
    public String CheckChildSuitability(String itemName) {
        return CheckChildSuitabilityTool.execute(itemName, gemini);
    }

    @Override
    public String SuggestAlternativeItem(String itemName, String reason, String alt) {
        return SuggestAlternativeTool.execute(itemName, reason, alt, gemini);
    }
}
```

## Tool Implementations

### Fast Tool (Pure Logic)

No LLM calls, no I/O. Throws on invalid input.

```java
public final class ValidateOrderTool {

    private ValidateOrderTool() {}

    public static String execute(Order order) {
        if (order.orderId() == null || order.orderId().isBlank())
            throw new IllegalArgumentException("Order ID must not be blank");
        if (order.customerName() == null || order.customerName().isBlank())
            throw new IllegalArgumentException("Customer name must not be blank");
        if (order.items() == null || order.items().isEmpty())
            throw new IllegalArgumentException("Order must contain at least one item");

        for (OrderItem item : order.items()) {
            if (item.itemName() == null || item.itemName().isBlank())
                throw new IllegalArgumentException("Every order item must have a name");
            if (item.quantity() <= 0)
                throw new IllegalArgumentException(
                    "Item '" + item.itemName() + "' has invalid quantity: " + item.quantity());
        }

        return "Order " + order.orderId() + " is valid";
    }
}
```

### LLM-Powered Tool

Calls the LLM with a domain-specific prompt. Returns structured JSON.

```java
public final class CheckChildSuitabilityTool {

    private CheckChildSuitabilityTool() {}

    public static String execute(String itemName, GeminiClient gemini) {
        String prompt = """
            You are a child-safety product advisor.
            Determine whether the following product is appropriate for children under 16.

            Product: "%s"

            Respond ONLY with a valid JSON object using exactly this schema:
            {
              "suitable": true | false,
              "reason": "<brief explanation>",
              "suggestedAlternative": "<child-safe alternative, or null if suitable>"
            }
            """.formatted(itemName);

        return gemini.call(prompt);
    }
}
```

## Workflow Implementation — The Agent Loop

The workflow is the heart of the agentic pattern. It maintains conversation history, calls the LLM planner, dispatches tools, and tracks state.

```java
public class OrderWorkflowImpl implements OrderWorkflow {

    private static final Logger log = Workflow.getLogger(OrderWorkflowImpl.class);
    private final ObjectMapper objectMapper = new ObjectMapper();

    // The goal driving this workflow
    private final AgentGoal goal = DummyOrderGoals.CHILD_SAFE_ORDER;

    // Fast local activities — short timeout, main task queue
    private final ToolActivities localActivities = Workflow.newActivityStub(
        ToolActivities.class,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(
                Duration.ofSeconds(WorkflowProperties.localActivityTimeoutSeconds()))
            .setRetryOptions(RetryOptions.newBuilder()
                .setMaximumAttempts(WorkflowProperties.maxRetryAttempts()).build())
            .build());

    // LLM-bound activities — longer timeout, dedicated task queue
    private final LlmActivities llmActivities = Workflow.newActivityStub(
        LlmActivities.class,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(
                Duration.ofSeconds(WorkflowProperties.llmActivityTimeoutSeconds()))
            .setTaskQueue(WorkflowProperties.llmTaskQueue())
            .setRetryOptions(RetryOptions.newBuilder()
                .setMaximumAttempts(WorkflowProperties.maxRetryAttempts()).build())
            .build());

    // Conversation memory
    private final List<Map<String, String>> conversationHistory = new ArrayList<>();

    // State tracking
    private final Map<String, String> itemReplacements = new LinkedHashMap<>();
    private final Map<String, String> itemReasons = new LinkedHashMap<>();

    @Override
    public void processOrder(Order order) {
        // Seed conversation with system prompt and user request
        addMessage("system", goal.starterPrompt());
        addMessage("user", "Process order " + order.orderId() + " for '"
            + order.customerName() + "' with items: " + order.items());

        int maxTurns = WorkflowProperties.maxAgentTurns();
        int turns = 0;

        while (turns < maxTurns) {
            turns++;

            // 1. ASK LLM: "What tool should I call next?"
            String plannerResponse = llmActivities.PlanNextTool(goal, conversationHistory);
            addMessage("agent", plannerResponse);

            // 2. PARSE the LLM's JSON response
            JsonNode plan = parsePlan(plannerResponse);
            String next = plan.path("next").asText("done");
            String tool = plan.path("tool").asText(null);

            // 3. CHECK for termination
            if ("done".equals(next) || tool == null) break;
            if ("PlaceOrder".equals(tool)) break;

            // 4. DISPATCH the tool
            String result;
            try {
                result = dispatch(tool, plan.path("args"), order);
            } catch (Exception e) {
                result = "ERROR: " + e.getMessage();
            }

            // 5. FEED RESULT back into conversation
            addMessage("tool_result", "Tool " + tool + " returned: " + result);

            // 6. TRACK state changes
            if ("CheckChildSuitability".equals(tool)) handleSuitability(result, plan.path("args"));
            if ("SuggestAlternativeItem".equals(tool)) trackReplacement(plan.path("args"), result);
        }

        // Final action: build order with replacements and place it
        Order finalOrder = buildFinalOrder(order);
        localActivities.PlaceOrder(finalOrder, buildChanges());
    }

    // Route tool names to activity calls
    private String dispatch(String tool, JsonNode args, Order order) {
        return switch (tool) {
            case "ValidateOrder" -> localActivities.ValidateOrder(order);
            case "CheckChildSuitability" ->
                llmActivities.CheckChildSuitability(args.path("itemName").asText());
            case "SuggestAlternativeItem" ->
                llmActivities.SuggestAlternativeItem(
                    args.path("itemName").asText(),
                    args.path("reason").asText(""),
                    args.path("suggestedAlternative").asText(null));
            default -> throw new IllegalArgumentException("Unknown tool: " + tool);
        };
    }

    private void addMessage(String actor, String message) {
        conversationHistory.add(Map.of("actor", actor, "response", message));
    }

    // ... JSON parsing, state tracking, final order assembly helpers ...
}
```

### Key Implementation Details

**Activity stub configuration:**
- `localActivities` uses the default task queue with a short timeout (10s)
- `llmActivities` uses a dedicated `llm-processing` task queue with a long timeout (180s)
- Both use `RetryOptions` with a bounded `maxRetryAttempts`

**Conversation history format:**
```java
// Each message is a Map with "actor" and "response" keys
conversationHistory.add(Map.of("actor", "system", "response", "..."));
```

**JSON extraction from LLM responses:**
LLMs sometimes wrap JSON in markdown code blocks. The `extractJson()` helper strips `` ```json ... ``` `` wrappers before parsing.

**Graceful error handling in dispatch:**
Tool failures are caught and fed back as `"ERROR: ..."` messages, allowing the LLM to adapt or retry with different arguments.

## Prompt Generator

Build LLM system prompts from structured `AgentGoal` metadata. Never hardcode prompt strings.

```java
public final class PromptGenerator {

    private PromptGenerator() {}

    public static String generateSystemPrompt(AgentGoal goal,
            List<Map<String, String>> conversationHistory) {

        StringBuilder sb = new StringBuilder();

        // 1. Goal description
        sb.append("=== Agent Goal ===\n");
        sb.append("Goal: ").append(goal.name()).append("\n");
        sb.append(goal.description()).append("\n\n");

        // 2. Tool definitions (auto-generated from ToolDefinition records)
        sb.append("=== Tools Definitions ===\n");
        for (ToolDefinition tool : goal.tools()) {
            sb.append("Tool name: ").append(tool.name()).append("\n");
            sb.append("  Description: ").append(tool.description()).append("\n");
            sb.append("  Required args:\n");
            for (ToolArgument arg : tool.arguments()) {
                sb.append("    - ").append(arg.name())
                    .append(" (").append(arg.type()).append("): ")
                    .append(arg.description()).append("\n");
            }
            sb.append("\n");
        }

        // 3. Conversation history
        sb.append("=== Conversation History ===\n");
        for (Map<String, String> msg : conversationHistory) {
            sb.append(msg.get("actor")).append(": ").append(msg.get("response")).append("\n");
        }

        // 4. Example conversation (optional)
        if (goal.exampleConversationHistory() != null
                && !goal.exampleConversationHistory().isBlank()) {
            sb.append("=== Example Conversation ===\n");
            sb.append(goal.exampleConversationHistory()).append("\n\n");
        }

        // 5. JSON response format
        sb.append("=== CRITICAL: JSON-ONLY RESPONSE FORMAT ===\n");
        sb.append("""
            Your response must be ONLY valid JSON:
            {
              "next": "<confirm|done>",
              "tool": "<tool_name or null>",
              "args": { "<arg1>": "<value1>", ... },
              "response": "<brief explanation>"
            }
            """);

        return sb.toString();
    }
}
```

## Worker Registration — Dual Task Queue Setup

Register two separate Temporal workers: one for the workflow + fast activities, another for LLM activities.

```java
@Component
public class DummyOrderWorker {

    public DummyOrderWorker(
            WorkerFactory workerFactory,
            OrderActivitiesImpl orderActivities,
            LlmActivitiesImpl llmActivities,
            @Value("${temporal.task-queue}") String taskQueue,
            @Value("${temporal.llm-task-queue}") String llmTaskQueue,
            @Value("${temporal.order-worker.max-concurrent-workflow-task-executors}") int workflowConcurrency,
            @Value("${temporal.order-worker.max-concurrent-activity-executors}") int activityConcurrency,
            @Value("${temporal.llm-worker.max-concurrent-activity-executors}") int llmConcurrency) {

        // Worker 1: Workflow + fast local activities
        Worker orderWorker = workerFactory.newWorker(taskQueue,
            WorkerOptions.newBuilder()
                .setMaxConcurrentWorkflowTaskExecutionSize(workflowConcurrency)
                .setMaxConcurrentActivityExecutionSize(activityConcurrency)
                .build());
        orderWorker.registerWorkflowImplementationTypes(OrderWorkflowImpl.class);
        orderWorker.registerActivitiesImplementations(orderActivities);

        // Worker 2: LLM-bound activities on dedicated queue
        Worker llmWorker = workerFactory.newWorker(llmTaskQueue,
            WorkerOptions.newBuilder()
                .setMaxConcurrentActivityExecutionSize(llmConcurrency)
                .build());
        llmWorker.registerActivitiesImplementations(llmActivities);
    }
}
```

**Typical concurrency settings:**
| Worker | Workflow Tasks | Activity Tasks |
|--------|---------------|----------------|
| `order-processing` | 200 | 200 |
| `llm-processing` | N/A | 50 |

## WorkflowProperties — Spring-to-Temporal Bridge

Temporal instantiates workflow classes directly (not via Spring DI). Use a static accessor pattern to bridge Spring configuration into workflow code.

```java
@Component
public class WorkflowProperties {

    private static String llmTaskQueue;
    private static int localActivityTimeout;
    private static int llmActivityTimeout;
    private static int maxRetryAttempts;
    private static int maxAgentTurns;

    @Value("${temporal.llm-task-queue}")
    private String llmTaskQueueValue;

    @Value("${temporal.local-activity-timeout}")
    private int localActivityTimeoutValue;

    // ... other @Value fields ...

    @PostConstruct
    void init() {
        llmTaskQueue = llmTaskQueueValue;
        localActivityTimeout = localActivityTimeoutValue;
        // ... assign all static fields ...
    }

    public static String llmTaskQueue() { return llmTaskQueue; }
    public static int localActivityTimeoutSeconds() { return localActivityTimeout; }
    public static int llmActivityTimeoutSeconds() { return llmActivityTimeout; }
    public static int maxRetryAttempts() { return maxRetryAttempts; }
    public static int maxAgentTurns() { return maxAgentTurns; }
}
```

**Typical configuration values (application.yml):**
```yaml
temporal:
  task-queue: order-processing
  llm-task-queue: llm-processing
  local-activity-timeout: 10      # seconds
  llm-activity-timeout: 180       # seconds
  max-retry-attempts: 3
  max-agent-turns: 20
```

## Determinism Considerations

Agentic workflows are **fully deterministic** despite using an LLM, because:
- All LLM calls happen inside activities (non-deterministic by nature)
- On replay, Temporal uses stored activity results rather than re-calling the LLM
- The workflow loop itself only does deterministic operations (parsing JSON, building maps, calling activities)

**Safe operations in the workflow:**
- `ObjectMapper.readTree()` — deterministic JSON parsing
- `Map.of()`, `new ArrayList<>()` — standard deterministic collections
- `switch` on tool name — deterministic routing
- `Workflow.getLogger()` — replay-safe logging

**Unsafe operations (must be in activities):**
- LLM API calls (Gemini, OpenAI, etc.)
- HTTP requests to external services
- Database reads/writes
- File I/O

## Testing

### Unit Testing the Agent Loop

Mock the activity interfaces to test the workflow's dispatch and state-tracking logic without calling the LLM.

```java
@ExtendWith(TestWorkflowExtension.class)
class OrderWorkflowTest {

    @RegisterExtension
    static final TestWorkflowExtension testExtension = TestWorkflowExtension.newBuilder()
        .registerWorkflowImplementationTypes(OrderWorkflowImpl.class)
        .setActivityImplementations(mockToolActivities, mockLlmActivities)
        .build();

    @Test
    void shouldDispatchToolsBasedOnLlmPlan() {
        // Arrange: mock PlanNextTool to return specific tool selections
        when(mockLlmActivities.PlanNextTool(any(), any()))
            .thenReturn("{\"next\":\"confirm\",\"tool\":\"ValidateOrder\",\"args\":{},\"response\":\"Validating\"}")
            .thenReturn("{\"next\":\"done\",\"tool\":null,\"args\":{},\"response\":\"Done\"}");
        when(mockToolActivities.ValidateOrder(any())).thenReturn("Order is valid");

        // Act
        OrderWorkflow workflow = testExtension.newWorkflowStub(OrderWorkflow.class);
        workflow.processOrder(testOrder);

        // Assert
        verify(mockToolActivities).ValidateOrder(testOrder);
        verify(mockToolActivities).PlaceOrder(any(), any());
    }
}
```

### Testing Tool Implementations

Test tool implementations directly as unit tests — they are plain static methods.

```java
@Test
void validateOrder_shouldRejectBlankOrderId() {
    Order invalid = new Order("", "Alice", List.of(new OrderItem("Book", 1)));
    assertThatThrownBy(() -> ValidateOrderTool.execute(invalid))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("Order ID must not be blank");
}
```

### Testing the Prompt Generator

Verify that generated prompts contain all required sections.

```java
@Test
void shouldIncludeAllToolDefinitionsInPrompt() {
    String prompt = PromptGenerator.generateSystemPrompt(
        DummyOrderGoals.CHILD_SAFE_ORDER, List.of());

    assertThat(prompt)
        .contains("ValidateOrder")
        .contains("CheckChildSuitability")
        .contains("SuggestAlternativeItem")
        .contains("PlaceOrder")
        .contains("=== Agent Goal ===")
        .contains("=== Tools Definitions ===");
}
```

## Best Practices

1. **Use Java records for AgentGoal, ToolDefinition, ToolArgument** — immutable, concise, serializable
2. **Keep tool implementations as static utility classes** — no state, easy to test
3. **Inject LLM clients via constructors in activity implementations** — Spring DI for activities, not workflows
4. **Use `Workflow.getLogger()` in workflows** — replay-safe logging (standard SLF4J in activities is fine)
5. **Use `WorkflowProperties` static accessor pattern** — bridges Spring config into Temporal-instantiated workflows
6. **Parse LLM JSON defensively** — always handle malformed responses with fallback to `"done"`
7. **Strip markdown code block wrappers** — LLMs often wrap JSON in `` ```json ``` `` blocks
8. **Track state with `LinkedHashMap`** — preserves insertion order for reproducible final results
9. **Separate `@ActivityInterface` per latency profile** — `ToolActivities` for fast, `LlmActivities` for slow
10. **Register workers in Spring `@Component` constructors** — let Spring DI wire everything, then `WorkerBootstrap` starts the factory
