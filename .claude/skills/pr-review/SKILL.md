---
name: pr-review
description: "Orchestrated PR review that launches parallel sub-agents to check API design, backend patterns, unit/component/BDD tests, healthcare compliance, Kafka, Docker, JPA, Spring Boot, Temporal, and technical design alignment. Collects findings, presents summary, and posts inline PR comments after user approval."
---

# PR Review Skill (Orchestrated)

Multi-agent PR review that runs specialized review agents in parallel for fast, comprehensive feedback.

## When to Activate

- User asks to "review a PR", "review pull request", "pr review", or "review changes"
- User provides a PR URL, PR number, commit hash, or branch name
- User runs `/pr-review`

## Workflow

### Step 1: Determine Review Scope

Based on user input, determine which type of review to perform:

1. **No arguments (default)**: Review all uncommitted changes
   - Run: `git diff` for unstaged changes
   - Run: `git diff --cached` for staged changes
   - Run: `git status --short` to identify untracked (net new) files

2. **Commit hash** (40-char SHA or short hash): Review that specific commit
   - Run: `git show <commit-hash>`

3. **Branch name**: Compare current branch to the specified branch
   - Run: `git diff <branch>...HEAD`

4. **PR URL or number** (contains "github.com" or "pull" or looks like a PR number): Review the pull request
   - Run: `gh pr view <input>` to get PR context
   - Run: `gh pr diff <input>` to get the diff

### Step 2: Gather Full Context

**Diffs alone are not enough.** After getting the diff, read the entire file(s) being modified.

- Use the diff to identify which files changed and categorize them (Java source, test, config, Docker, SQL, feature files, etc.)
- Use `git status --short` to identify untracked files, then read their full contents
- Read full files to understand existing patterns, control flow, and error handling
- Identify which review agents are relevant based on the files changed

### Step 3: Launch Parallel Review Agents

The orchestrator agent MUST launch sub-agents in parallel using the Agent tool. Each sub-agent receives:
- The diff content for relevant files
- The list of changed file paths
- Instructions to read full file contents and the referenced skill before reviewing

**Launch ALL applicable agents in a single message with multiple Agent tool calls.**

Only launch agents that are relevant to the files changed. For example, skip the Kafka agent if no Kafka-related files are in the diff.

---

#### Agent 1: API Design Review

**Skill reference:** `.claude/skills/api-design/SKILL.md`

**Applies when:** Controllers, DTOs, OpenAPI specs, or REST endpoint files are changed.

**Checks:**
- OpenAPI-first workflow is followed (spec exists before implementation)
- URL naming conventions (kebab-case, plural nouns, no verbs)
- Correct HTTP status codes for each operation
- RFC 7807 problem detail error responses
- Tracing headers (X-Request-Id, X-Correlation-Id) propagated
- SLA contracts defined for new endpoints
- Pagination, filtering, and versioning patterns
- Rate limiting considerations

---

#### Agent 2: Backend Patterns Review

**Skill reference:** `.claude/skills/backend-patterns/SKILL.md`

**Applies when:** Any Java/backend source files are changed.

**Checks:**
- Layered architecture respected (controller -> service -> repository)
- Proper separation of concerns
- Error handling patterns
- Database query optimization
- Caching strategy where applicable
- Async processing patterns
- Logging standards (structured, no PII)

---

#### Agent 3: Unit Test Review

**Skill reference:** `.claude/skills/java-unit-testing/SKILL.md`

**Applies when:** Java source files are added or modified.

**Checks:**
- Unit tests exist for new/modified classes
- Tests cover both happy and unhappy paths
- JUnit 5, Mockito, and AssertJ used correctly
- Test naming conventions followed
- Edge cases covered (null, empty, boundary values)
- No logic in tests; tests are readable and focused
- Mocking is appropriate (not over-mocking)

---

#### Agent 4: Component Test Review

**Skill reference:** `.claude/skills/component-tests/SKILL.md`

**Applies when:** New endpoints, services, or integrations are added.

**Checks:**
- Component tests exist for new features
- Black-box testing via HTTP (RestAssured)
- Testcontainers used (app runs as Docker container)
- No Spring context in test JVM
- Scenarios simulate real user behavior
- No direct database or Kafka inspection in tests

---

#### Agent 5: BDD/Cucumber Review

**Skill reference:** `.claude/skills/bdd-cucumber/SKILL.md`

**Applies when:** Feature files, step definitions, or acceptance test files are changed.

**Checks:**
- Feature files follow Gherkin best practices
- Scenarios cover happy and unhappy paths
- Step definitions are reusable and composable
- @SpringBootTest with real Testcontainers (PostgreSQL + Kafka)
- No mocking in acceptance tests
- Scenarios are business-readable

---

#### Agent 6: Healthcare Domain Review

**Skill reference:** `.claude/skills/healthcare-domain/SKILL.md`

**Applies when:** Any files related to healthcare data, PHI, NPI, ICD-10, HCPCS, Medicaid, EDI, or compliance.

**Checks:**
- HIPAA PHI safeguards (encryption, access controls, audit logging)
- NPI validation (Luhn algorithm)
- PECOS status verification
- ICD-10/HCPCS code validation with Levenshtein distance correction
- Medicaid/Title XIX requirements
- X12 EDI 270/271/278 structure validation
- No PHI in logs, error messages, or exception details
- Data retention and de-identification policies

---

#### Agent 7: Kafka Patterns Review

**Skill reference:** `.claude/skills/kafka-patterns/SKILL.md`

**Applies when:** Kafka producers, consumers, topic configs, or messaging code is changed.

**Checks:**
- KafkaTopics constants used (no magic strings)
- Idempotent consumer patterns
- DefaultErrorHandler with Dead Letter Topic (DLT)
- Producer completion callbacks
- @EmbeddedKafka in tests with Awaitility assertions
- Serialization/deserialization configured correctly

---

#### Agent 8: Docker Patterns Review

**Skill reference:** `.claude/skills/docker-patterns/SKILL.md`

**Applies when:** Dockerfiles, docker-compose files, or container configs are changed.

**Checks:**
- Multi-stage builds for minimal image size
- Non-root user in container
- Health checks defined
- Secrets not baked into images
- Volume strategies appropriate
- Networking configuration correct
- Docker Compose service orchestration

---

#### Agent 9: JPA Patterns Review

**Skill reference:** `.claude/skills/jpa-patterns/SKILL.md`

**Applies when:** Entity classes, repositories, or database-related code is changed.

**Checks:**
- Entity design (proper use of @Entity, @Table, @Id)
- Relationship mappings (OneToMany, ManyToOne, cascading)
- Query optimization (N+1 detection, fetch strategies)
- Transaction management
- Auditing fields (createdAt, updatedAt)
- Indexing strategy
- Pagination patterns
- Connection pooling configuration

---

#### Agent 10: Spring Boot Patterns Review

**Skill reference:** `.claude/skills/springboot-patterns/SKILL.md`

**Applies when:** Any Spring Boot source files are changed.

**Checks:**
- Architecture patterns (layered services, dependency injection)
- REST API design within Spring context
- Data access patterns
- Caching configuration and usage
- Async processing (@Async, CompletableFuture)
- Logging (SLF4J, structured logging, MDC)
- Configuration management (profiles, externalized config)

---

#### Agent 11: Spring Boot Verification

**Skill reference:** `.claude/skills/springboot-verification/SKILL.md`

**Applies when:** Any Spring Boot source or config files are changed.

**Checks:**
- Build succeeds (`./gradlew build`)
- Static analysis passes
- Tests pass with adequate coverage
- Security scan findings
- Diff review for unintended changes

---

#### Agent 12: Temporal Workflow Review (Conditional)

**Skill reference:** `.claude/skills/temporal-developer/SKILL.md`

**Applies when:** Files import Temporal SDK classes, or workflow/activity files are changed. **Skip this agent entirely if no Temporal usage is detected.**

**Checks:**
- Workflow determinism (no non-deterministic calls in workflow code)
- Activity idempotency
- Retry policies configured
- Signal/query handlers correct
- Error handling and compensation logic
- Versioning for running workflows

---

#### Agent 13: Technical Design & NFR Alignment

**Applies when:** Always — every PR should align with technical design and meet NFRs.

**Checks:**

**Technical Design Alignment:**
- Fetch and review the Technical Design at https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/technical-design/overview.md
- Fetch and review Decision Records at https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/decision-records/overview.md
- Verify the PR changes are consistent with documented architecture decisions
- Flag any deviations from the technical design

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
- Naming conventions (PascalCase classes, camelCase methods)
- Immutability (records, final fields, unmodifiable collections)
- Optional usage (no `.get()` without `.isPresent()`, no Optional fields)
- Stream usage (prefer streams over imperative loops where readable)
- Exception handling (custom exceptions, no catching generic Exception)
- Generics usage
- Package structure and project layout

---

### Step 4: Collect and Deduplicate Results

As sub-agents complete, the orchestrator:

1. Collects all findings from each agent
2. Deduplicates overlapping findings (e.g., same issue flagged by both JPA and Spring Boot agents)
3. Assigns final severity: **Critical**, **Medium**, **Low**
4. Groups findings by file for coherent inline comments

### Step 5: Present Consolidated Findings

Present a summary table to the user:

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
- Any agents that were skipped (and why)

### Step 6: Ask Permission Before Posting

**CRITICAL: Never post review comments without explicit user permission.**

After presenting findings, ask the user:
- Whether they want to post the comments on the PR
- Whether they want to modify, remove, or add any comments
- Whether they want the comments posted as inline (file-specific) or general comments

Only proceed to post after receiving explicit confirmation.

### Step 7: Post Inline Comments at Specific Lines

When posting comments on a PR, **always use inline comments at the exact line numbers** — never post a single wall-of-text general comment.

**Determining line numbers:**
1. Parse the diff to find the file path and hunk headers (`@@ -old,count +new,count @@`)
2. For new files: count `+` lines from the start of the hunk to find the file line
3. For modified files: use the new file line numbers from the `+` side of the hunk header

**Posting the review:**

```bash
# For github.com
gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input review.json

# For GitHub Enterprise
gh api --hostname {ghe-host} repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input review.json
```

The review JSON structure:

```json
{
  "body": "Polaris PR Review - 14 agents, X issues found (Y critical, Z medium, W low)",
  "event": "COMMENT",
  "comments": [
    {
      "path": "relative/file/path.java",
      "line": 42,
      "body": "**Critical** (Healthcare): `dateOfBirth` is logged in the error message at line 42. This is PHI under HIPAA and must not appear in logs. Remove the field from the log statement or mask it.\n\n*Ref: healthcare-domain skill - HIPAA PHI safeguards*"
    }
  ]
}
```

Write the JSON to a temp file and use `--input` to avoid shell escaping issues.

**Detecting GitHub Enterprise:** If the PR URL contains a hostname other than `github.com`, use `gh api --hostname <host>` and extract owner/repo from the URL. Run `gh auth status` if unsure which hosts are configured.

## Comment Tone and Style

**Be polite, constructive, and evidence-based.** Every comment should:

1. **State the severity and agent** — e.g., `**Critical** (Healthcare):` or `**Medium** (Kafka):`
2. **Explain what is wrong** — Concisely, so the reader understands without deep reading
3. **Explain why it matters** — Concrete scenario where the issue manifests
4. **Suggest a fix** — Specific recommendation or code example
5. **Reference the skill** — e.g., *Ref: kafka-patterns skill - idempotent consumers*
6. **Be honest about uncertainty** — "This may be intentional, but..." or "Consider whether..."
7. **Respect the author's intent** — "Consider using...", "This could be improved by..."

**Avoid:**
- Flattery or filler ("Great job!", "Thanks for this!")
- Accusatory tone ("You forgot to...", "You should have...")
- Overstating severity — a style nit is not a critical bug
- Vague complaints without evidence

**Example good comment:**
> **Medium** (JPA): `dateOfService` is mapped as `String`, which bypasses date validation and allows malformed values (e.g., `"not-a-date"`) to persist. If the DB column is a date type, Hibernate throws a runtime exception on insert. Consider `LocalDate` for type safety and date range queries.
>
> *Ref: jpa-patterns skill - entity design*

## Edge Cases

- **GitHub Enterprise**: Always check `gh auth status` and use `--hostname` flag for GHE instances
- **Large PRs**: If the diff is truncated, read the full output and use offset/limit or grep
- **Shell escaping**: Write review JSON to a temp file; use `--input /tmp/review.json`
- **Mixed file types**: Only review code files. Skip generated files, lock files, or binary assets unless they indicate a problem (e.g., committed secrets)
- **Monorepo**: If files span multiple services, launch separate agent groups per service
- **No tests found**: Flag as Critical if new functionality has zero test coverage
