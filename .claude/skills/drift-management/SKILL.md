---
name: drift-management
description: "Detect drift between code/implementation and technical documentation (Technical Design, Decision Records). Identifies where the codebase has diverged from documented architecture, ADRs, or design decisions, and flags undocumented decisions that need new records."
---

# Drift Management Skill

Detects drift between the codebase and its governing technical documentation. Works across two axes:

1. **Doc → Code**: Documented decisions that the code no longer honours
2. **Code → Doc**: Code patterns that deviate from or contradict the Technical Design and Decision Records, with no corresponding record justifying the deviation

Multi-agent execution: documentation fetching and domain drift analysis run in parallel to minimise wall-clock time.

## When to Activate

- User asks to "check drift", "detect drift", "find drift", "validate against tech design", "check decision records", "are we aligned with the architecture", or "does the code match the docs"
- User runs `/drift-management`
- After a large refactor to confirm alignment
- Before creating an ADR to check whether the decision is already recorded

## Inputs

The skill accepts an optional scope argument:

| Argument | Behaviour |
|----------|-----------|
| _(none)_ | Analyse all uncommitted local changes (working tree + staged) against documentation |
| `--branch` | Analyse all commits on the current branch that diverge from `main` |
| `--pr <url-or-number>` | Analyse a specific pull request |
| `--file <path>` | Analyse a single file or directory |
| `--full` | Full codebase scan — slower, use when onboarding or after a major redesign |

## Documentation Sources

### Technical Design
- **Overview index**: `https://github.example.com/your-org/my-documentation/blob/main/docs/technical-design/overview.md`
- Fetch the overview first, then follow links to component-level design docs scoped to the **current repo**.

### Decision Records (ADRs / DRs)
- **Overview index**: `https://github.example.com/your-org/my-documentation/blob/main/docs/decision-records/overview.md`
- Fetch the index, then fetch individual records relevant to the changed code or platform-wide decisions.

**Always fetch fresh content** — never rely on cached or assumed content. If a URL is unreachable, note it in findings and continue with available sources.

---

## Workflow

### Step 0: Detect Repo Context (Orchestrator)

Before launching any agents, identify which service/repo the skill is running in and where its source code lives.

```bash
# Primary: git remote — gives the full repo URL (used later to match tech design pages)
git remote get-url origin
# e.g. https://github.example.com/your-org/my-order-connector
#   or git@github.example.com:your-org/my-order-connector.git

# Fallback: directory name
basename $(git rev-parse --show-toplevel)

# Locate the source root — check common conventions in order:
# 1. Java/Kotlin (Maven/Gradle)
ls src/main/java 2>/dev/null && echo "src/main/java"
ls src/main/kotlin 2>/dev/null && echo "src/main/kotlin"
# 2. Python src layout
ls src 2>/dev/null && echo "src"
# 3. Flat layout (Python, Node, Go)
ls app 2>/dev/null && echo "app"
# 4. Ultimate fallback — repo root (only if none of the above exist)
git rev-parse --show-toplevel
```

From this, extract and record:
- **Repo URL** — the full remote URL (e.g., `https://github.example.com/your-org/my-order-connector`). Used in Step 2 to match tech design pages by repo URL.
- **Repo name** — the last path segment (e.g., `my-order-connector`). Used as a human-readable label.
- **Source root** — the first directory that exists from the list above (e.g., `src/main/java`). **All file scanning and reading in Steps 1 and 3 is restricted to this directory.** Never read outside it (no root-level scripts, no `build/`, no `target/`, no `.github/`, no `terraform/` unless the source root IS the repo root and no `src/` variant exists).

If the repo name cannot be determined, ask the user:
> Which service is this? (e.g., `my-order-connector`) — used to scope the technical design lookup.

---

### Step 1: Gather Scope (Orchestrator)

Collect the changeset based on the argument. **All file paths must be under the detected source root — ignore any changed files outside it** (e.g., root-level `build.gradle`, `Dockerfile`, `.github/` are out of scope for drift analysis).

```bash
# No args — uncommitted changes, filtered to source root
git diff -- <source-root>
git diff --cached -- <source-root>
git status --short -- <source-root>

# Branch scope
git log main..HEAD --oneline
git diff main -- <source-root>

# PR scope
gh pr view <number>
gh pr diff <number>   # then filter paths to source root only
```

Read the full content of every changed file within the source root. For `--full`, read all source files under the source root:
- Controllers, services, repositories (all layers)
- Configuration classes (`@Configuration`, `application.yml`, `application.properties`) — only if inside the source root
- Kafka producers/consumers
- Temporal workflows and activities

If the scope is empty after filtering to the source root, inform the user and stop.

Categorise changed files by domain:
- **API layer** — controllers, DTOs, OpenAPI specs
- **Service / domain layer** — services, domain models, business logic
- **Persistence layer** — entities, repositories, migrations
- **Messaging** — Kafka producers/consumers, event types
- **Workflow** — Temporal workflows and activities
- **Infrastructure** — Terraform, Docker, Compose, CI/CD
- **Observability** — logging config, metrics, tracing
- **Security / compliance** — auth, PHI handling, encryption

This categorisation determines which drift agents to launch in Step 3.

---

### Step 2: Fetch Documentation (Parallel — launch both agents simultaneously)

**Launch both documentation agents at the same time in a single message.**

Each agent receives the detected **repo URL**, repo name, and source root, and must scope its work accordingly.

---

#### Doc Agent A: Technical Design

1. Fetch `https://github.example.com/your-org/my-documentation/blob/main/docs/technical-design/overview.md`
2. From the index, find all tech design pages that contain the **repo URL** (e.g., `https://github.example.com/your-org/my-order-connector`) anywhere in their content or metadata. These are the service-specific design docs for this repo.
3. Also identify pages covering **platform-wide concerns** (auth, observability, inter-service communication, data residency, logging) — these apply to all services regardless of repo URL.
4. Fetch each matched page in parallel.
5. Skip pages whose repo URL points to a different service.
6. Return: a structured list of `{ section, constraint, scope: "service" | "platform" }` entries extracted from all fetched pages.

---

#### Doc Agent B: Decision Records

1. Fetch `https://github.example.com/your-org/my-documentation/blob/main/docs/decision-records/overview.md`
2. From the index, identify DRs where:
   - The scope/context mentions the current repo, or
   - The decision is platform-wide (applies to all Project Services)
3. Fetch each relevant DR in parallel
4. Return: a structured list of `{ dr-id, title, status, decision, scope: "service" | "platform" }` entries

---

Wait for both Doc Agents to complete before launching Step 3. The combined output — all extracted constraints and decisions — is passed to every drift agent in Step 3.

---

### Step 3: Domain Drift Analysis (Parallel — launch all applicable agents simultaneously)

**Launch ALL applicable drift agents in a single message.** Each agent receives:
- The diff / changed file contents for its domain (scoped to the source root)
- The full list of constraints from Doc Agent A
- The full list of decisions from Doc Agent B
- The detected repo URL, repo name, and source root

Only launch agents whose domain has matching files in the changeset (within the source root). Skip agents with no relevant files and note the skip reason in the final report.

---

#### Drift Agent 1: API & Contract Drift

**Applies when:** Controllers, DTOs, OpenAPI specs, or REST endpoint files are changed.

**Checks against docs:**
- URL naming, versioning, and pagination constraints from Technical Design
- DRs mandating OpenAPI-first workflow
- DRs on error response format (RFC 7807)
- DRs on SLA contracts and rate limiting
- Any DR or TD section prescribing HTTP status code usage

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 2: Service & Domain Logic Drift

**Applies when:** Service classes, domain models, or business logic files are changed.

**Checks against docs:**
- Layered architecture constraints (controller → service → repository; no skipping layers)
- DRs on separation of concerns or domain model boundaries
- TD sections on business rules or domain invariants
- DRs on transaction boundaries
- Any DR mandating or prohibiting specific patterns (e.g., no business logic in controllers)

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 3: Persistence & Data Drift

**Applies when:** Entity classes, repositories, migrations, or database config are changed.

**Checks against docs:**
- DRs prescribing the persistence technology (e.g., PostgreSQL only, no in-memory stores)
- TD sections on data model design or schema conventions
- DRs on auditing, soft deletes, data retention
- DRs on indexing strategy or query performance requirements
- PHI/PII encryption requirements (DR or TD) — flag as **Critical** if violated

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 4: Messaging & Event Drift

**Applies when:** Kafka producers, consumers, event types, or topic config are changed.

**Checks against docs:**
- DRs mandating async inter-service communication via Kafka (flag sync HTTP as **High** drift)
- TD sections on Kafka topic naming, partitioning, or event schema conventions
- DRs on idempotency, at-least-once delivery handling
- DRs on dead-letter queue patterns
- Event schema evolution rules documented in TD

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 5: Workflow & Orchestration Drift

**Applies when:** Temporal workflows, activities, or worker config files are changed.

**Checks against docs:**
- DRs prescribing Temporal for long-running or durable operations
- TD sections on workflow boundaries and which operations must be durable
- DRs on saga/compensation patterns
- TD sections on workflow versioning requirements
- Any DR restricting what logic may live inside a workflow vs an activity

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 6: Infrastructure & Deployment Drift

**Applies when:** Terraform files, Dockerfiles, Compose files, or CI/CD config are changed.

**Checks against docs:**
- DRs mandating S3+DynamoDB remote Terraform state
- TD sections on environment separation (dev/staging/prod)
- DRs on KMS encryption, IAM role patterns, naming conventions
- TD sections on container security (non-root user, health checks, no secrets in images)
- DRs on required infrastructure tags or compliance controls

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 7: Observability & Logging Drift

**Applies when:** Logging config, metrics setup, tracing config, or any file containing log statements are changed. For `--full`, scan all source files.

**Checks against docs:**
- DRs mandating structured JSON logging
- TD sections on required log fields (correlation ID, service name, trace ID)
- DRs prohibiting PHI/PII in log output — flag violations as **Critical**
- TD sections on required metrics (SLA-critical paths must export metrics)
- DRs on distributed tracing propagation

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 8: Security & Compliance Drift

**Applies when:** Auth config, PHI/PII handling, encryption, or HIPAA-relevant files are changed. Always runs if healthcare-related files are detected.

**Checks against docs:**
- DRs mandating authentication/authorisation at all endpoints
- DRs on PHI encryption at rest and in transit
- TD sections on HIPAA safeguards
- DRs on input validation at service boundaries
- Any compliance control documented in TD (audit trails, data retention, access logging)

**Returns:** list of drift findings with category, severity, doc reference, and file:line.

---

#### Drift Agent 9: Stale Documentation Scanner

**Applies when:** Always — runs against the full documentation fetched in Step 2.

This agent works in the **reverse direction**: it reads all fetched design docs and DRs, then checks the codebase for evidence that documented components, patterns, or integrations still exist.

**Checks:**
- Design doc references a service, component, or integration → verify it exists in code
- A DR with status "Accepted" prescribes a pattern → verify the pattern is present somewhere in the repo
- A DR with status "Superseded" or "Deprecated" → verify the old pattern has been removed

**Returns:** list of stale doc findings (Category C) with doc reference and explanation of what is missing from the code.

---

### Step 4: Collect and Deduplicate (Orchestrator)

As agents complete:

1. Collect all findings from all drift agents
2. Deduplicate: if two agents flag the same file:line against the same DR, merge into one finding (note both agents)
3. Assign final severity using the table below
4. Group findings by category, then by severity within each category

---

### Step 5: Classify Findings

Each drift item falls into one of three categories:

#### Category A — Implementation Drift
The code **contradicts** a documented decision or architectural constraint.
- The decision is explicitly documented
- The code does the opposite or ignores it
- Severity: usually **Critical** or **High**

#### Category B — Documentation Lag
The code makes an architectural choice that **has no corresponding DR or design note**.
- The pattern exists in code (e.g., new caching layer, new external dependency, new async pattern)
- No DR documents why this choice was made
- Severity: usually **Medium**

#### Category C — Stale Documentation
The documentation describes something that **no longer exists in the code**.
- A design doc still references a removed component
- A DR says "we will use X" but X is absent from the codebase
- Severity: **Low** to **Medium**

---

### Step 6: Assess Severity

| Severity | Criteria |
|----------|----------|
| **Critical** | Security, PHI/PII, or compliance decision violated |
| **High** | Core architectural decision violated (wrong layer, wrong technology, wrong communication pattern) |
| **Medium** | Pattern deviation without justification; undocumented architectural choices; DR needed but missing |
| **Low** | Stale docs, minor documentation debt, naming drift |
| **Info** | Code aligns with the spirit of a decision but uses a different implementation detail |

---

### Step 7: Present Consolidated Drift Report

If there are **no findings**, report clean alignment and stop:

```
## Drift Report
Repo:   <detected repo name>
Scope:  <uncommitted changes | branch xyz | PR #42>
Result: ✓ No drift detected — code aligns with all checked Technical Design sections and Decision Records.
```

If there **are findings**, present the full report:

```
## Drift Report
Repo:        <detected repo name>
Source root: <detected source root, e.g. src/main/java>
Scope:       <uncommitted changes | branch xyz | PR #42>
Agents: <N launched, M skipped>
Docs:   <list of URLs fetched — ✓ ok | ✗ unreachable>

### Summary
Critical: N | High: N | Medium: N | Low: N | Undocumented: N | Stale Docs: N | Aligned: N

### Findings

| # | Severity | Category    | Agent        | Doc Reference              | File:Line             | Finding |
|---|----------|-------------|--------------|----------------------------|-----------------------|---------|
| 1 | Critical | Impl Drift  | Security     | DR-031: PHI encryption     | PatientEntity.java:23 | `ssn` stored as plain VARCHAR; DR-031 mandates AES-256 for PHI fields |
| 2 | High     | Impl Drift  | Messaging    | TD §4.1: Async inter-svc   | OrderService.java:42  | Sync HTTP call to patient-service; TD mandates Kafka for internal comms |
| 3 | Medium   | Undocumented| Persistence  | —                          | CacheConfig.java:15   | Redis cache introduced with no DR documenting this decision |
| 4 | Low      | Stale Doc   | Stale Docs   | TD §6.3: SES notifications | —                     | TD references SES integration; no SES code found — doc may be stale |

### Agents Launched
- API & Contract, Persistence, Messaging, Security, Stale Docs

### Agents Skipped
- Workflow (no Temporal files in changeset)
- Infrastructure (no Terraform or Docker files in changeset)
```

For each finding include:
- **Doc evidence**: exact quote or section from the fetched document
- **Code evidence**: file:line and the specific code that drifts
- **Impact**: concrete consequence if left unaddressed

Then proceed immediately to Step 8.

---

### Step 8: Ask How to Resolve Each Finding

**CRITICAL: Never make any changes — to code or documentation — without explicit user direction.**

After presenting the report, work through each finding that requires a decision. Group findings where the answer is obvious (Category C stale docs always mean "update the doc"; Category B undocumented decisions always mean "create a DR") and only ask for findings where the answer is genuinely ambiguous.

For each **Category A (Implementation Drift)** finding, ask:

> **Finding #N — <one-line summary>**
>
> The code at `File:Line` contradicts `Doc Reference`.
>
> How should this be resolved?
> 1. **Fix the code** — bring the implementation in line with the documented decision
> 2. **Update the documentation** — the code represents the correct new direction; the doc needs to be revised (and a new DR may be needed to record the change)
> 3. **Skip** — leave this finding unresolved for now

For findings where the answer is unambiguous, state the default action and confirm:

> **Finding #3 — Redis cache with no DR**
> This is an undocumented architectural decision. The standard resolution is to **create a DR** documenting why Redis was chosen.
> Proceed? (yes / skip)

> **Finding #4 — Stale SES reference in TD §6.3**
> The documentation references a component that no longer exists. The standard resolution is to **update the documentation** to remove or supersede the stale reference.
> Proceed? (yes / skip)

Collect all answers before moving to Step 9. Do not start making changes during the Q&A.

---

### Step 9: Present Resolution Plan

Once all decisions are collected, present a **Resolution Plan** before touching anything:

```
## Resolution Plan

| # | Finding | Decision | Action |
|---|---------|----------|--------|
| 1 | `ssn` plain VARCHAR — DR-031 violation | Fix the code | Add `@Convert(converter = AesEncryptedStringConverter.class)` to `PatientEntity.ssn`; update migration to encrypt existing rows |
| 2 | Sync HTTP to patient-service — TD §4.1 violation | Fix the code | Replace `PatientServiceClient.getPatient()` with a Kafka request-reply pattern; add `PatientRequestedEvent` and `PatientResponseEvent` |
| 3 | Redis cache — no DR | Create DR | Draft DR-XXX: "Use Redis for session caching in my-order-connector" |
| 4 | Stale SES reference in TD §6.3 | Update doc | Remove §6.3 from the Technical Design or replace with a note that SES was removed in favour of X |

Proceed with this plan? (yes / modify / cancel)
```

Wait for explicit confirmation before executing. If the user says **modify**, ask which items to change before re-presenting the plan. If **cancel**, stop.

---

### Step 10: Execute the Plan

Only after the user confirms, execute each action **in the order: Critical → High → Medium → Low**:

#### For "Fix the code" items:
- Make the minimal code change that brings the implementation in line with the documented decision
- Do not refactor unrelated code
- Summarise the change made (file:line, what was changed)
- Do **not** commit — leave that to the user unless explicitly asked

#### For "Update the documentation" items:
- If the doc is in the local repo: edit the file directly, show the diff
- If the doc is in `my-documentation` (remote repo): output the exact markdown change needed and the file path, so the user can raise a PR there. Do **not** push or create a PR without being asked.

#### For "Create a DR" items:
Draft the DR stub using this template and show it to the user before creating any file:

```markdown
# DR-XXX: <Decision Title>

**Status**: Proposed
**Date**: <today>
**Deciders**: <team / author>

## Context
<What problem was being solved? What forces were at play?>

## Decision
<What was decided?>

## Consequences
### Positive
- <benefit>
### Negative / Risks
- <trade-off>

## Alternatives Considered
- <option A>: <why rejected>
```

Ask the user to confirm the content, then write the file to the appropriate location in `my-documentation`.

#### After each action:
Report what was done:
```
✓ Finding #1 — PatientEntity.ssn encrypted (PatientEntity.java:23, V4__encrypt_ssn.sql created)
✓ Finding #3 — DR-XXX draft written to docs/decision-records/DR-XXX-redis-caching.md
⏭ Finding #2 — Skipped by user
```

---

## Evidence Standards

Every finding MUST be backed by:
1. **A direct quote or section reference** from the fetched technical document
2. **A specific file:line** from the codebase
3. **A clear explanation** of why they conflict

Do not raise a finding based on assumption. If evidence cannot be found, classify as **Info** and note that manual review is needed.

---

## Tone and Style

- State category and severity clearly: `**High** (Implementation Drift):`
- Quote the design document: _"DR-017 states: 'All domain data must be persisted in PostgreSQL...'"_
- Point to the code: `PatientRepository.java:12` uses an in-memory `HashMap`
- Explain the gap with a concrete consequence, not a hypothetical
- Suggest a specific fix
- Be honest about uncertainty: "This may be an intentional migration in progress, but if so a DR is needed"
