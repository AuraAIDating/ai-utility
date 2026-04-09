# AI/LLM Agentic Workflow Patterns

This document covers conceptual patterns for building **agentic workflows** with Temporal — workflows where an LLM planner dynamically decides which tools to invoke at runtime to achieve a structured goal.

Language-specific implementation guidance is available at `references/{your_language}/ai-patterns.md`.

## Overview

An **agentic workflow** combines Temporal's durable execution with LLM-driven decision-making. Instead of hardcoding the sequence of steps, the workflow defines a **goal** and a set of **tools**, then enters a loop where an LLM planner selects and dispatches tools until the goal is complete.

This pattern is inspired by the [temporal-ai-agent](https://github.com/temporalio/temporal-ai-agent) reference architecture.

```
┌──────────────────────────────────────────────────────────────┐
│                   Agentic Workflow Loop                       │
│                                                              │
│  ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│  │  Goal +  │──>│  LLM     │──>│ Dispatch │──>│ Feed     │  │
│  │  History │   │ Planner  │   │  Tool    │   │ Result   │──┤
│  └─────────┘   └──────────┘   └──────────┘   └──────────┘  │
│       ▲                                            │         │
│       └────────────────────────────────────────────┘         │
│                    (repeat until done)                        │
└──────────────────────────────────────────────────────────────┘
```

## Core Concepts

### AgentGoal

An **AgentGoal** is the central abstraction that defines what the agent should achieve. It bundles:

| Field | Purpose |
|-------|---------|
| `id` | Unique identifier (e.g. `"goal_child_safe_order"`) |
| `name` | Human-readable label for logs and UI |
| `description` | Step-by-step objective the LLM must follow |
| `tools` | Ordered list of available tool definitions |
| `starterPrompt` | Initial prompt that kicks off the agent loop |
| `exampleConversationHistory` | Few-shot example showing correct tool-chaining (optional) |

**Key design principle:** The goal description tells the LLM _what_ to achieve and in what order, while the tools define _how_ it can achieve it. The LLM decides the exact sequencing at runtime.

### ToolDefinition and ToolArgument

Tools are described declaratively with metadata — they do **not** contain execution logic. This separation ensures:

- **Single source of truth** — one registry is consumed by both the prompt generator and the workflow dispatcher
- **Easy extensibility** — adding a tool requires a registry entry and a handler, nothing else
- **LLM compatibility** — tool schemas are auto-converted to prompt text

A `ToolDefinition` contains:
- `name` — the tool identifier (matched during dispatch)
- `description` — what the tool does (included in the LLM prompt)
- `arguments` — list of typed `ToolArgument` records (`name`, `type`, `description`)

### Tool Registry

A **ToolRegistry** is a static class that holds all `ToolDefinition` constants. It serves as the single source of truth for tool metadata, consumed by:
1. **Goal definitions** — to assemble the tool list for each goal
2. **Prompt generators** — to build LLM prompts with tool schemas
3. **Documentation** — tool descriptions are always accurate because they come from code

### Tool Implementations

Tool implementations are separate from tool definitions. They fall into two categories:

| Type | Characteristics | Example |
|------|----------------|---------|
| **Fast/Local tools** | Pure logic, no external I/O, sub-second | Validate input, place order |
| **LLM-powered tools** | Call an LLM with domain-specific prompts, seconds to minutes | Check suitability, suggest alternatives |

This distinction drives the **dual task queue** pattern (see below).

## The Agent Loop

The core workflow pattern is a bounded loop that:

1. **Asks the LLM planner** — "Given the goal and conversation history, what tool should I call next?"
2. **Parses the response** — extracts tool name, arguments, and continuation signal from structured JSON
3. **Checks for termination** — if the LLM signals "done" or selects a terminal tool, exit the loop
4. **Dispatches the tool** — routes the tool name to the appropriate activity call
5. **Feeds the result back** — appends the tool result to conversation history for the next turn
6. **Repeats** — until the goal is complete or the turn limit is reached

### Bounded Execution

Always enforce a **maximum turn limit** (e.g. 20 turns) to prevent runaway LLM loops. If the agent exceeds the limit, the workflow should gracefully terminate or escalate.

### Conversation Memory

The workflow maintains a `conversationHistory` — an ordered list of messages with actor/response pairs. Actors include:
- `system` — initial prompts and injected instructions
- `user` — the original request
- `agent` — LLM planner responses (JSON with tool selections)
- `tool_result` — output from executed tools

This history is passed to every LLM planner call, giving the LLM full context of what has happened so far.

### Tool Dispatch

The workflow contains a `dispatch()` method that acts as a router, mapping tool name strings from LLM output to strongly-typed Temporal activity calls. This keeps the routing logic centralized and type-safe.

### LLM Response Format

The LLM planner must return structured JSON in a consistent format:

```json
{
  "next": "<confirm|done>",
  "tool": "<tool_name or null>",
  "args": { "<arg1>": "<value1>", ... },
  "response": "<brief explanation>"
}
```

- `next: "confirm"` — the agent wants to call a tool
- `next: "done"` — the agent considers the goal complete
- `tool: null` — no more tools to call (goal complete)

## Dual Task Queue Pattern

Separate **fast local activities** from **LLM-bound activities** onto different task queues:

```
┌──────────────────────────────┐     ┌──────────────────────────────┐
│   order-processing queue     │     │     llm-processing queue     │
│                              │     │                              │
│  Workflow + Fast Activities  │     │     LLM Activities           │
│  - ValidateOrder (10s)       │     │  - PlanNextTool (180s)       │
│  - PlaceOrder (10s)          │     │  - CheckSuitability (180s)   │
│  - 200 concurrent            │     │  - SuggestAlt (180s)         │
│                              │     │  - 50 concurrent             │
└──────────────────────────────┘     └──────────────────────────────┘
```

**Why separate queues?**
- **Independent scaling** — LLM workers can scale based on LLM API capacity without affecting fast activities
- **Timeout isolation** — LLM calls need 60-180s timeouts; fast activities need 5-10s. Mixing them masks failures.
- **Backpressure** — LLM rate limits don't starve fast activity execution
- **Cost visibility** — separate queues make it easy to monitor LLM API costs

## Prompt Generation

The **PromptGenerator** builds structured system prompts from `AgentGoal` metadata. A well-structured prompt includes these sections:

1. **Agent Goal** — name and step-by-step description
2. **Tool Definitions** — all tools with argument schemas (auto-generated from `ToolDefinition` records)
3. **Conversation History** — full chat history for context
4. **Example Conversation** — few-shot example from the goal (optional but highly recommended)
5. **JSON Response Format** — strict schema the LLM must follow

**Key principle:** Generate prompts from structured data, never hardcode them. This ensures tool definitions and prompt content stay in sync.

## Few-Shot Examples

Including a complete worked example in the `AgentGoal.exampleConversationHistory` field dramatically improves LLM reliability. The example should demonstrate:

- The exact JSON response format
- Multi-step tool chaining (tool A -> result -> tool B -> result -> ...)
- How to carry forward arguments between tools
- When to signal "done"

## Durable Execution Benefits

Temporal provides critical guarantees for agentic workflows:

1. **Crash recovery** — if the JVM dies mid-turn, Temporal replays the workflow from its event history and resumes from the last completed activity
2. **LLM retry** — if an LLM API call fails, Temporal automatically retries with configurable backoff
3. **Audit trail** — every tool call and result is recorded in the workflow event history
4. **Timeout enforcement** — per-tool timeouts prevent hung LLM calls from blocking the workflow
5. **Visibility** — workflow state is queryable and searchable via the Temporal UI

## Anti-Patterns

### Do NOT hardcode tool sequences in the workflow
The workflow should only contain the agent loop and dispatch logic. The LLM decides tool ordering based on the goal description. Hardcoding sequences defeats the purpose of the agentic pattern.

### Do NOT skip the ToolRegistry
Every tool must be registered in the `ToolRegistry`. Ad-hoc tool definitions scattered across goal files lead to inconsistencies between prompts and dispatch logic.

### Do NOT mix fast and LLM activities on the same queue
LLM calls are orders of magnitude slower than local operations. Mixing them on one queue causes head-of-line blocking and makes timeout configuration impossible.

### Do NOT omit the turn limit
An unbounded agent loop with an LLM planner will eventually hallucinate infinite tool calls. Always enforce `maxAgentTurns`.

### Do NOT use raw string prompts
Always use a `PromptGenerator` that builds prompts from structured `AgentGoal` and `ToolDefinition` metadata. Raw string prompts drift out of sync with tool definitions.

## Best Practices

1. **Define goals as static constants** — each goal is a self-contained unit that can be tested and reviewed independently
2. **Use the ToolRegistry as the single source of truth** — goals, prompts, and dispatch logic all reference the same definitions
3. **Separate tool metadata from tool implementation** — `ToolDefinition` describes the tool; the implementation is a separate class/method
4. **Split activities by latency profile** — fast local activities and LLM activities on separate task queues with appropriate timeouts
5. **Include few-shot examples in goals** — they are the most effective way to improve LLM tool-calling accuracy
6. **Bound the agent loop** — enforce a maximum turn count to prevent runaway execution
7. **Log every turn** — workflow-safe logging (`Workflow.getLogger()`) at each turn aids debugging
8. **Track state changes in the workflow** — maintain maps/lists for replacements, decisions, etc. so the final action reflects all intermediate results
9. **Use structured JSON for LLM responses** — define a strict schema and include it in the prompt
10. **Test with replay** — agentic workflows are deterministic (all non-determinism is in activities), so replay testing works normally
