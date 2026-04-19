---
name: review
description: "Local code review of uncommitted changes using parallel sub-agents. Checks API design, backend patterns, tests, healthcare compliance, Kafka, Docker, JPA, Spring Boot, Temporal, and coding standards. After review, offers to create a PR with a generated description."
---

# Local Code Review Skill (Orchestrated)

Multi-agent review of uncommitted local changes. No PR required — works on your working tree.

## When to Activate

- User asks to "review my code", "review changes", "review locally", or "check my work"
- User runs `/review`

## Workflow

### Step 1: Gather Local Changes

Collect all uncommitted changes:

```bash
# Unstaged changes
git diff

# Staged changes
git diff --cached

# Untracked (net new) files
git status --short
```

Read the full contents of every changed and untracked file. Categorize them:
- Java source, test files, config, Docker, SQL, feature files, OpenAPI specs, etc.

If there are no changes, inform the user and stop.

### Step 2: Identify Current Branch Context

```bash
# Current branch name
git branch --show-current

# Commits ahead of main (to understand the full scope of work)
git log main..HEAD --oneline

# Combined diff from main (all commits + uncommitted)
git diff main
```

Use this to understand the full picture — uncommitted changes plus any commits on the branch that haven't been merged.

### Step 3: Launch Parallel Review Agents

The orchestrator MUST launch sub-agents in parallel using the Agent tool. Each sub-agent receives:
- The diff content for relevant files
- The list of changed file paths
- Instructions to read full file contents and the referenced skill before reviewing

**Launch ALL applicable agents in a single message with multiple Agent tool calls.**

Only launch agents relevant to the files changed. Skip agents whose domain has no matching files.

---

#### Agent 1: API Design Review

**Skill reference:** `.claude/skills/api-design/SKILL.md`

**Applies when:** Controllers, DTOs, OpenAPI specs, or REST endpoint files are changed.

**Checks:**
- OpenAPI-first workflow followed (spec exists before implementation)
- URL naming conventions (kebab-case, plural nouns, no verbs)
- Correct HTTP status codes
- RFC 7807 problem detail error responses
- Tracing headers propagated
- SLA contracts for new endpoints
- Pagination, filtering, versioning patterns

---

#### Agent 2: Backend Patterns Review

**Skill reference:** `.claude/skills/backend-patterns/SKILL.md`

**Applies when:** Any Java/backend source files are changed.

**Checks:**
- Layered architecture (controller -> service -> repository)
- Separation of concerns
- Error handling patterns
- Database query optimization
- Caching strategy
- Async processing patterns
- Logging standards (structured, no PII)

---

#### Agent 3: Unit Test Review

**Skill reference:** `.claude/skills/java-unit-testing/SKILL.md`

**Applies when:** Java source files are added or modified.

**Checks:**
- Unit tests exist for new/modified classes
- Happy and unhappy paths covered
- JUnit 5, Mockito, AssertJ used correctly
- Test naming conventions
- Edge cases (null, empty, boundary values)
- No logic in tests
- Appropriate mocking

---

#### Agent 4: Component Test Review

**Skill reference:** `.claude/skills/component-tests/SKILL.md`

**Applies when:** New endpoints, services, or integrations are added.

**Checks:**
- Component tests exist for new features
- Black-box testing via HTTP (RestAssured)
- Testcontainers used (app as Docker container)
- No Spring context in test JVM
- Scenarios simulate real user behavior

---

#### Agent 5: BDD/Cucumber Review

**Skill reference:** `.claude/skills/bdd-cucumber/SKILL.md`

**Applies when:** Feature files, step definitions, or acceptance test files are changed.

**Checks:**
- Gherkin best practices
- Happy and unhappy path scenarios
- Reusable step definitions
- @SpringBootTest with real Testcontainers
- No mocking in acceptance tests
- Business-readable scenarios

---

#### Agent 6: Healthcare Domain Review

**Skill reference:** `.claude/skills/healthcare-domain/SKILL.md`

**Applies when:** Files related to healthcare data, PHI, NPI, ICD-10, HCPCS, Medicaid, EDI, or compliance.

**Checks:**
- HIPAA PHI safeguards
- NPI validation (Luhn)
- PECOS status verification
- ICD-10/HCPCS code validation
- Medicaid/Title XIX requirements
- X12 EDI structure validation
- No PHI in logs or error messages

---

#### Agent 7: Kafka Patterns Review

**Skill reference:** `.claude/skills/kafka-patterns/SKILL.md`

**Applies when:** Kafka producers, consumers, topic configs, or messaging code is changed.

**Checks:**
- KafkaTopics constants (no magic strings)
- Idempotent consumer patterns
- DefaultErrorHandler with DLT
- Producer completion callbacks
- @EmbeddedKafka tests with Awaitility

---

#### Agent 8: Docker Patterns Review

**Skill reference:** `.claude/skills/docker-patterns/SKILL.md`

**Applies when:** Dockerfiles, docker-compose files, or container configs are changed.

**Checks:**
- Multi-stage builds
- Non-root user
- Health checks
- No secrets in images
- Volume strategies
- Networking configuration

---

#### Agent 9: JPA Patterns Review

**Skill reference:** `.claude/skills/jpa-patterns/SKILL.md`

**Applies when:** Entity classes, repositories, or database code is changed.

**Checks:**
- Entity design (@Entity, @Table, @Id)
- Relationship mappings and cascading
- N+1 detection, fetch strategies
- Transaction management
- Auditing fields
- Indexing strategy
- Pagination patterns

---

#### Agent 10: Spring Boot Patterns Review

**Skill reference:** `.claude/skills/springboot-patterns/SKILL.md`

**Applies when:** Any Spring Boot source files are changed.

**Checks:**
- Architecture patterns
- Dependency injection
- Data access patterns
- Caching configuration
- Async processing
- Structured logging with MDC
- Configuration management

---

#### Agent 11: Spring Boot Verification

**Skill reference:** `.claude/skills/springboot-verification/SKILL.md`

**Applies when:** Any Spring Boot source or config files are changed.

**Checks:**
- Build succeeds (`./gradlew build`)
- Static analysis passes
- Tests pass with adequate coverage
- Security scan findings

---

#### Agent 12: Temporal Workflow Review (Conditional)

**Skill reference:** `.claude/skills/temporal-developer/SKILL.md`

**Applies when:** Files import Temporal SDK classes or workflow/activity files are changed. **Skip entirely if no Temporal usage detected.**

**Checks:**
- Workflow determinism
- Activity idempotency
- Retry policies
- Signal/query handlers
- Error handling and compensation
- Versioning for running workflows

---

#### Agent 13: Technical Design & NFR Alignment

**Applies when:** Always.

**Checks:**

**Technical Design Alignment:**
- Fetch Technical Design at https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/technical-design/overview.md
- Fetch Decision Records at https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/decision-records/overview.md
- Verify changes align with documented architecture
- Flag deviations
- Check if new ADRs are needed (ref: `.claude/skills/architecture-decision-records/SKILL.md`)

**Non-Functional Requirements (NFRs):**
Fetch NFRs from the Technical Design and verify the changes satisfy them. Common NFR categories to check:

- **Performance** — Response time SLAs met? Bulk operations paginated? No unbounded queries? Database indexes present for new query patterns? Caching used where specified?
- **Scalability** — Stateless services? Horizontal scaling possible? No in-memory state that breaks with multiple instances? Kafka partitioning strategy correct?
- **Availability** — Circuit breakers for external calls? Retry policies with backoff? Graceful degradation? Health check endpoints present?
- **Security** — Authentication/authorization enforced? Input validation at boundaries? No SQL injection, XSS, or command injection? Secrets not hardcoded? HIPAA/PHI requirements met?
- **Observability** — Structured logging with correlation IDs? Metrics exported for SLA-critical paths? Distributed tracing propagated? Alerts defined for error thresholds?
- **Reliability** — Idempotent operations? Transactional consistency? Dead letter queues for failed messages? Data validation at ingress?
- **Maintainability** — Code follows project conventions? Adequate test coverage? Documentation for complex logic? Configuration externalized?
- **Compliance** — HIPAA, Medicaid, and healthcare regulatory requirements met? Audit trails present? Data retention policies followed?

For each NFR violation found, cite the specific NFR from the Technical Design and explain how the code fails to meet it.

---

#### Agent 14: Java Coding Standards Review

**Skill reference:** `.claude/skills/java-coding-standards/SKILL.md`

**Applies when:** Any Java source files are changed.

**Checks:**
- Naming conventions
- Immutability (records, final fields)
- Optional usage
- Stream usage
- Exception handling
- Generics
- Package structure

---

### Step 4: Collect and Deduplicate Results

As sub-agents complete, the orchestrator:

1. Collects all findings from each agent
2. Deduplicates overlapping findings
3. Assigns final severity: **Critical**, **Medium**, **Low**
4. Groups findings by file

### Step 5: Present Consolidated Findings

Present a summary table:

```
| # | Severity | Agent          | File:Line          | Finding                           |
|---|----------|----------------|--------------------|-----------------------------------|
| 1 | Critical | Healthcare     | Patient.java:42    | PHI logged in error message       |
| 2 | Critical | API Design     | -                  | No OpenAPI spec for new endpoint  |
| 3 | Medium   | Unit Tests     | OrderService.java  | No tests for error path           |
| 4 | Medium   | Kafka          | EventConsumer:18   | Missing idempotency check         |
| 5 | Low      | Java Standards | Utils.java:7       | Mutable field in record           |
```

Include:
- Total issues by severity
- Which agents found issues vs. gave a clean pass
- Which agents were skipped (and why)

### Step 6: Offer to Create a PR

After presenting the review findings, ask the user:

> Would you like to create a PR for these changes? If so, what should the PR title be?

**If the user declines**, stop here.

**If the user provides a title**, proceed to Step 7.

### Step 7: Create the PR

1. **Stage and commit** any uncommitted changes (ask the user what to include if there are both staged and unstaged changes)

2. **Push the branch**:
   ```bash
   git push -u origin $(git branch --show-current)
   ```

3. **Generate the PR description** based on the review findings:

   ```bash
   gh pr create --title "<user-provided-title>" --body "$(cat <<'EOF'
   ## Summary
   <2-4 bullet points summarizing what this PR does, derived from the diff and branch commits>

   ## Review Findings

   ### Passed
   <List agents that gave a clean pass>

   ### Issues Found
   <Summary table of findings from Step 5, if any remain after the user addressed them>

   ### Agents Skipped
   <List agents not applicable to this change set>

   ## Test Coverage
   - Unit tests: <present/missing — from Agent 3>
   - Component tests: <present/missing — from Agent 4>
   - BDD scenarios: <present/missing — from Agent 5>

   ## Checklist
   - [ ] API spec matches implementation
   - [ ] Healthcare/HIPAA compliance verified
   - [ ] Tests cover happy and unhappy paths
   - [ ] No PHI in logs or error messages
   - [ ] Technical design alignment confirmed

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```

4. **Return the PR URL** to the user.

## Comment Tone and Style

Same as pr-review skill:

- **State severity and agent** — e.g., `**Critical** (Healthcare):`
- **Explain what is wrong** — Concisely
- **Explain why it matters** — Concrete scenario
- **Suggest a fix** — Specific recommendation
- **Reference the skill** — e.g., *Ref: kafka-patterns skill*
- **Be honest about uncertainty**
- **Respect the author's intent**

**Avoid:**
- Flattery or filler
- Accusatory tone
- Overstating severity
- Vague complaints without evidence
