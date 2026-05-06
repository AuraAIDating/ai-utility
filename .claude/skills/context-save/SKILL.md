---
name: context-save
description: "Save working session context for Project Services. Captures git state, decisions made, remaining work, and open questions so any future Claude Code session can resume without losing a beat. Stores context files in .my-context/ within the repo. Pair with /context-restore. Adapted from community patterns."
origin: community-adapted
---

# Save Working Context

You are a **Staff Engineer keeping meticulous session notes**. Capture the full working context - what's being done, what decisions were made, what's left - so any future session can resume without losing a beat.

**HARD GATE:** Do NOT make code changes. This skill captures state only. Only write to `.my-context/` and update `.gitignore` to include it if missing.

---

## When to Activate

- "save progress"
- "save state"
- "context save"
- "save my work"
- End of a coding session on a long-running feature
- Before switching branches or tasks
- Before a long `./gradlew` build you want to walk away from

---

## Parse command

- `/context-save` -> save with inferred title
- `/context-save <title>` -> save with provided title
- `/context-save list` -> list saved contexts (see List flow below)

---

## Save Flow

### Step 1: Gather git state

```bash
echo "=== BRANCH ==="
git rev-parse --abbrev-ref HEAD 2>/dev/null

echo "=== STATUS ==="
git status --short 2>/dev/null

echo "=== DIFF STAT ==="
git diff --stat 2>/dev/null

echo "=== STAGED DIFF STAT ==="
git diff --cached --stat 2>/dev/null

echo "=== RECENT LOG ==="
git log --oneline -10 2>/dev/null
```

### Step 2: Capture Spring Boot project context (Java 21+)

```bash
echo "=== SPRING BOOT VERSION ==="
grep "version =" build.gradle.kts | head -1

echo "=== KEY DEPENDENCIES ==="
grep -E "kafka|temporal|jpa|spring-security" build.gradle.kts | head -10

echo "=== ACTIVE PROFILES ==="
grep -r "spring.profiles.active" src/main/resources/ .env 2>/dev/null

echo "=== TEST COMMAND FOR THIS FEATURE ==="
find . -name "*Test.java" -newer .git/ 2>/dev/null | head -3 | sed 's|src/test/java/||; s|\.java||; s|/|.|g'
```

### Step 3: PHI safety scan (before confirming save)

Run a quick scan against the new context file. If any hits appear, redact before confirming.

```bash
echo "=== PHI SCAN: Check for common patterns ==="
echo "Patient data (MRN, SSN, patient email):"
grep -rn "patient\|mrn\|ssn\|member_id" .my-context/ 2>/dev/null | wc -l

echo "Credentials (API keys, passwords):"
grep -rn "password\|api.key\|secret\|token" .my-context/ 2>/dev/null | wc -l

echo "HIPAA-sensitive fields (NPI, email in cleartext):"
grep -rE "[0-9]{10}|[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" .my-context/ | wc -l
```

If any count is non-zero, redact the context before confirming the save.

### Step 4: Summarize context (structured)

Using the gathered state and conversation history, produce a summary covering:

1. **What is being worked on** - the high-level goal or feature (1-3 sentences)
2. **Decisions made** - include rationale, trade-offs, and alternatives considered
3. **Remaining work** - concrete next steps in priority order with effort estimates
4. **Open questions** - unresolved decisions with blocker flag and assignee
5. **Notes** - gotchas, things tried, useful commands
6. **File changes summary** - categorize modified files by layer (Controllers, Services, Repositories, Entities, Tests, Config, Resources)
7. **Metrics** - lines added/removed, files modified, test coverage delta (if known)
8. **JVM context** - Spring Boot version, Java version, key dependencies
9. **PHI scan status** - passed, flagged, or manual_review_required

If the user provided a title, use it. Otherwise infer a concise title (3-6 words) from the work.
Keep the overall context file under 5KB by staying concise.

### Step 5: Write context file

```bash
mkdir -p .my-context

# Ensure .my-context/ is gitignored before writing any context files.
# PHI leakage risk: context summaries can inadvertently contain patient-adjacent data.
grep -qxF ".my-context/" .gitignore 2>/dev/null || echo ".my-context/" >> .gitignore

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
echo "TIMESTAMP=$TIMESTAMP"
echo "BRANCH=$BRANCH"
```

Write to `.my-context/${TIMESTAMP}-${TITLE_SLUG}.md` using this structure:

```markdown
---
status: in-progress
branch: {current branch}
timestamp: {ISO-8601 timestamp}
service: {my-<service-name> or multi-service}
feature_area: {api-design | jpa-patterns | kafka-patterns | testing | infrastructure | other}
jvm_context:
  spring_boot_version: "3.2+"
  java_version: "21+"
  key_dependencies: ["kafka", "temporal", "jpa"]
metrics:
  lines_added: {number}
  lines_removed: {number}
  files_modified: {number}
  test_coverage_change: "85% -> 87%"
phi_scan:
  scan_status: passed
  flagged_terms: []
---

# {Title}

## Context Metadata (JSON)

```json
{
  "title": "{title}",
  "branch": "{current branch}",
  "timestamp": "{ISO-8601 timestamp}",
  "status": "in-progress",
  "service": "my-<service-name>",
  "feature_area": "kafka-patterns",
  "what_being_worked_on": "{1-3 sentences}",
  "decisions_made": [
    {
      "decision": "{decision}",
      "rationale": "{why}",
      "trade_offs": "{trade-offs}"
    }
  ],
  "remaining_work": [
    {"priority": 1, "task": "{task}", "estimated_effort": "1h"}
  ],
  "open_questions": [
    {"question": "{question}", "blocker": true, "assignee": "team"}
  ],
  "notes": [
    {"type": "useful_command", "content": "./gradlew test --tests \"com.MyProject.order.*\""}
  ],
  "git_state": {
    "current_branch": "{branch}",
    "uncommitted_changes_count": {number},
    "staged_changes_count": {number},
    "recent_commits": ["{hash} {message}"]
  },
  "metrics": {
    "lines_added": {number},
    "lines_removed": {number},
    "files_modified": {number},
    "test_coverage_change": "85% -> 87%"
  },
  "jvm_context": {
    "spring_boot_version": "3.2+",
    "java_version": "21+",
    "key_dependencies": ["kafka", "temporal", "jpa"]
  },
  "phi_scan": {
    "scan_status": "passed",
    "flagged_terms": []
  }
}
```

## What is being worked on

{High-level goal - 1-3 sentences}

## File Changes Summary

**Controllers** (API contract changes):
- {ControllerName.java}

**Services** (Business logic):
- {ServiceName.java}

**Repositories** (Data access):
- {RepositoryName.java}

**Entities** (Domain model):
- {EntityName.java}

**Tests** (Test coverage):
- {TestName.java}

**Config** (Infrastructure):
- {ConfigName.java}

**Resources** (Application configuration):
- {application.yaml}

## Decisions made

1. **{Decision title}**
   - Decision: {what was chosen}
   - Rationale: {why it was chosen}
   - Trade-off: {what you give up or risk}
   - Alternative considered: {what was rejected and why}

## Remaining work

1. {Priority 1} (effort: 1h)
2. {Priority 2} (effort: 2h)
3. {Priority 3} (effort: 1d+)

## Open questions / blockers

- Q1: {question}
  | Blocker: YES
  | Assignee: {username or team}

## Notes

- {Gotcha or important context}
- {Command or snippet that is useful}

## Test Coverage Delta

**Before**: {line%} line coverage, {branch%} branch coverage
**After (expected)**: {line%} line coverage, {branch%} branch coverage
**Gap**: {coverage gap or missing tests}
**Command to verify**: `./gradlew jacocoTestReport && open build/reports/jacoco/test/html/index.html`

## Useful Commands

- Run this feature's tests: `./gradlew test --tests "com.MyProject.order.*"`
- Build without tests (quick): `./gradlew clean assemble -x test`
- Check what changed since last save: `git diff --stat`
- Resume investigation if blocked: `/investigate`
- Save again with progress: `/context-save "progress update: <what changed>"`

## Multi-Service Context (if applicable)

**Primary service**: {my-<service-name>}
**Related services affected**:
- {service-name} (reason)

**Contract changes**:
- {event/class change}

**Deployment order**:
1. {service A}
2. {service B}

## Investigation Notes (if applicable)

**Last blocker encountered**: {issue}
**Investigation started**: /investigate
**Root cause found**: {root cause}
**Fix applied**: {fix}
**Verification**: {verification result}

## Git state at save

Branch: {branch}
Modified files:
{git status --short output}

Recent commits:
{git log --oneline -5 output}
```

### Step 6: Confirm save

Tell the user:
```
Context saved to .my-context/{filename}.md
Branch: {branch}
Title: "{title}"
Resume with: /context-restore
```

---

## Structured JSON Schema (reference)

If you want strict validation, create `.my-context/schema.json` (gitignored) with this schema and keep the JSON metadata block aligned.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": [
    "title",
    "branch",
    "timestamp",
    "status",
    "service",
    "feature_area",
    "what_being_worked_on",
    "decisions_made",
    "remaining_work",
    "open_questions",
    "notes",
    "git_state",
    "metrics",
    "jvm_context",
    "phi_scan"
  ],
  "properties": {
    "title": {"type": "string", "minLength": 3, "maxLength": 50},
    "branch": {"type": "string"},
    "timestamp": {"type": "string", "format": "date-time"},
    "status": {"enum": ["in-progress", "blocked", "ready-for-review", "merged"]},
    "service": {"type": "string"},
    "feature_area": {"enum": ["api-design", "jpa-patterns", "kafka-patterns", "testing", "infrastructure", "other"]},
    "what_being_worked_on": {"type": "string"},
    "decisions_made": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["decision", "rationale", "trade_offs"],
        "properties": {
          "decision": {"type": "string"},
          "rationale": {"type": "string"},
          "trade_offs": {"type": "string"}
        }
      }
    },
    "remaining_work": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["priority", "task", "estimated_effort"],
        "properties": {
          "priority": {"type": "integer", "minimum": 1, "maximum": 10},
          "task": {"type": "string"},
          "estimated_effort": {"enum": ["30min", "1h", "2h", "1d", "1d+"]}
        }
      }
    },
    "open_questions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["question", "blocker", "assignee"],
        "properties": {
          "question": {"type": "string"},
          "blocker": {"type": "boolean"},
          "assignee": {"type": "string"}
        }
      }
    },
    "notes": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["type", "content"],
        "properties": {
          "type": {"enum": ["gotcha", "useful_command", "warning"]},
          "content": {"type": "string"}
        }
      }
    },
    "git_state": {
      "type": "object",
      "required": ["current_branch", "uncommitted_changes_count", "staged_changes_count", "recent_commits"],
      "properties": {
        "current_branch": {"type": "string"},
        "uncommitted_changes_count": {"type": "number"},
        "staged_changes_count": {"type": "number"},
        "recent_commits": {"type": "array", "items": {"type": "string"}}
      }
    },
    "metrics": {
      "type": "object",
      "required": ["lines_added", "lines_removed", "files_modified", "test_coverage_change"],
      "properties": {
        "lines_added": {"type": "number"},
        "lines_removed": {"type": "number"},
        "files_modified": {"type": "number"},
        "test_coverage_change": {"type": "string"}
      }
    },
    "jvm_context": {
      "type": "object",
      "required": ["spring_boot_version", "java_version", "key_dependencies"],
      "properties": {
        "spring_boot_version": {"type": "string"},
        "java_version": {"type": "string"},
        "key_dependencies": {"type": "array", "items": {"type": "string"}}
      }
    },
    "phi_scan": {
      "type": "object",
      "required": ["scan_status", "flagged_terms"],
      "properties": {
        "scan_status": {"enum": ["passed", "flagged", "manual_review_required"]},
        "flagged_terms": {"type": "array", "items": {"type": "string"}}
      }
    }
  }
}
```

---

## Example Context File (Spring Boot - order-consumer)

```markdown
---
status: in-progress
branch: feat/order-consumer-virtual-threads
timestamp: 2026-04-30T14:15:22Z
service: my-order-consumer
feature_area: kafka-patterns
jvm_context:
  spring_boot_version: "3.2.4"
  java_version: "21+"
  key_dependencies: ["kafka", "jpa"]
metrics:
  lines_added: 214
  lines_removed: 42
  files_modified: 6
  test_coverage_change: "82% -> 85%"
phi_scan:
  scan_status: passed
  flagged_terms: []
---

# Order consumer idempotency

## Context Metadata (JSON)

```json
{
  "title": "Order consumer idempotency",
  "branch": "feat/order-consumer-virtual-threads",
  "timestamp": "2026-04-30T14:15:22Z",
  "status": "in-progress",
  "service": "my-order-consumer",
  "feature_area": "kafka-patterns",
  "what_being_worked_on": "Add idempotent handling for OrderReceived events and tune virtual-thread usage in the consumer pipeline.",
  "decisions_made": [
    {
      "decision": "Use existsByOrderId() before persist",
      "rationale": "Prevent duplicate order processing while keeping the consumer simple",
      "trade_offs": "Adds one DB call per message"
    }
  ],
  "remaining_work": [
    {"priority": 1, "task": "Add DLT handler metrics", "estimated_effort": "1h"},
    {"priority": 2, "task": "Add consumer integration test", "estimated_effort": "2h"}
  ],
  "open_questions": [
    {"question": "Should DLT include failure codes?", "blocker": false, "assignee": "team"}
  ],
  "notes": [
    {"type": "useful_command", "content": "./gradlew test --tests \"com.MyProject.order.*\""}
  ],
  "git_state": {
    "current_branch": "feat/order-consumer-virtual-threads",
    "uncommitted_changes_count": 3,
    "staged_changes_count": 0,
    "recent_commits": ["a1b2c3d add idempotent consumer check"]
  },
  "metrics": {
    "lines_added": 214,
    "lines_removed": 42,
    "files_modified": 6,
    "test_coverage_change": "82% -> 85%"
  },
  "jvm_context": {
    "spring_boot_version": "3.2.4",
    "java_version": "21+",
    "key_dependencies": ["kafka", "jpa"]
  },
  "phi_scan": {
    "scan_status": "passed",
    "flagged_terms": []
  }
}
```

## What is being worked on

Implement idempotent order handling and DLT metrics for the Kafka consumer. Ensure the virtual-thread based pipeline is stable under load.

## File Changes Summary

**Services** (Business logic):
- OrderConsumerService.java

**Repositories** (Data access):
- OrderRepository.java

**Config** (Infrastructure):
- KafkaConfig.java

**Tests** (Test coverage):
- OrderConsumerServiceTest.java

## Decisions made

1. **Idempotent check before persist**
   - Decision: Use existsByOrderId() before persist
   - Rationale: Prevent duplicate processing
   - Trade-off: Adds one DB query per message
   - Alternative considered: unique index only (rejected due to error handling gaps)

## Remaining work

1. Add DLT handler metrics (effort: 1h)
2. Add consumer integration test (effort: 2h)

## Open questions / blockers

- Q1: Should DLT include failure codes?
  | Blocker: NO
  | Assignee: team

## Notes

- Virtual threads enabled via `spring.threads.virtual.enabled=true`

## Test Coverage Delta

**Before**: 82% line coverage, 76% branch coverage
**After (expected)**: 85% line, 80% branch
**Gap**: Missing tests for DLT handler
**Command to verify**: `./gradlew jacocoTestReport && open build/reports/jacoco/test/html/index.html`

## Useful Commands

- Run this feature's tests: `./gradlew test --tests "com.MyProject.order.*"`
- Build without tests (quick): `./gradlew clean assemble -x test`

## Git state at save

Branch: feat/order-consumer-virtual-threads
Modified files:
M src/main/java/com/MyProject/order/OrderConsumerService.java
M src/main/java/com/MyProject/order/OrderRepository.java
M src/test/java/com/MyProject/order/OrderConsumerServiceTest.java

Recent commits:
a1b2c3d add idempotent consumer check
```

---

## Context Lifecycle Guidance

- Contexts older than 7 days on merged branches can be deleted: `rm .my-context/<old>.md`
- Contexts on active feature branches should be kept until merging
- Before deleting, archive: `mkdir -p .archives && mv .my-context/<old>.md .archives/`
- When resuming after 3+ days: check git log since save to understand what changed

---

## Integration Notes

- Pair with `/context-restore` to resume later using the structured metadata.
- If a debugging session is in progress, include an "Investigation Notes" section and reference `/investigate`.

---

## List Flow

When user types `/context-save list`:

```bash
ls -lt .my-context/*.md 2>/dev/null | head -10
```

Display a numbered list:
```
Saved contexts (most recent first):
1. 2026-04-29-14-30 | branch: feat/new-template    | "add order confirmation template"
2. 2026-04-28-09-15 | branch: fix/smtp-retry       | "smtp transient retry fix"
3. 2026-04-27-16-00 | branch: feat/avro-v2         | "avro schema v2 migration"
```

---

## Notes on PHI Safety

Context files are stored in `.my-context/` within the repo. **Do NOT include:**

- Real patient data or email addresses in context summaries
- SMTP credentials, AWS keys, or API tokens
- PHI from Avro event payloads

`.my-context/` is **gitignored by default** (Step 5 adds it automatically). Context files stay local only. In a HIPAA environment, context summaries written during debugging sessions frequently contain patient-adjacent data (order IDs, email addresses, diagnosis codes) - committing them to git risks accidental PHI exposure in the repository history.
