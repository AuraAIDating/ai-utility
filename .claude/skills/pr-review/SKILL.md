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

### Step 2: Prepare Git Worktrees for Parallel Agent Access

Multiple review agents running in parallel must never share a working directory — concurrent reads are fine, but any agent that runs tools (`./gradlew`, `grep`, `find`) against the source tree can interfere with others if they share the same checkout.

**Create one worktree per branch/commit under review before launching agents.**

```bash
# Resolve the repo root
REPO_ROOT=$(git rev-parse --show-toplevel)
WORKTREE_BASE="$REPO_ROOT/../.pr-review-worktrees"
mkdir -p "$WORKTREE_BASE"

# --- For a PR / branch diff ---
BRANCH=$(gh pr view <pr-number> --json headRefName -q .headRefName)
BASE_BRANCH=$(gh pr view <pr-number> --json baseRefName -q .baseRefName)

# Worktree for the PR head (agents read the new code here)
git worktree add "$WORKTREE_BASE/pr-head" "$BRANCH"

# Worktree for the base branch (agents compare against this)
git worktree add "$WORKTREE_BASE/pr-base" "$BASE_BRANCH"

# --- For a commit hash ---
git worktree add "$WORKTREE_BASE/commit-<hash>" <hash>

# --- For uncommitted changes (default) ---
# No worktree needed — agents read the current working tree read-only
```

**Worktree hygiene — always clean up after the review:**

```bash
git worktree remove --force "$WORKTREE_BASE/pr-head"
git worktree remove --force "$WORKTREE_BASE/pr-base"
git worktree prune
```

> **Tip:** If the worktree directory already exists (e.g., a previous review was interrupted), run `git worktree remove --force <path>` before re-adding.

Each agent receives its own `WORKTREE_PATH` so it operates on an isolated checkout:

| Agent group | Worktree |
|---|---|
| Agents reviewing new code (most agents) | `$WORKTREE_BASE/pr-head` |
| Agents doing base-vs-head comparison | both paths |
| Agents running static analysis / grep | `$WORKTREE_BASE/pr-head` (never the live working tree) |

---

### Step 3: Gather Full Context

**Diffs alone are not enough.** After getting the diff, read the entire file(s) being modified.

- Use the diff to identify which files changed and categorize them (Java source, test, config, Docker, SQL, feature files, etc.)
- Use `git status --short` to identify untracked files, then read their full contents
- Read full files (from the worktree path) to understand existing patterns, control flow, and error handling
- Identify which review agents are relevant based on the files changed

### Step 4: Launch Parallel Review Agents

The orchestrator agent MUST launch sub-agents in parallel using the Agent tool. Each sub-agent receives:
- The diff content for relevant files
- The list of changed file paths
- The **absolute path to their assigned worktree** (`WORKTREE_PATH=$WORKTREE_BASE/pr-head`) so all file reads and tool invocations use the isolated checkout
- Instructions to read full file contents (from `WORKTREE_PATH`) and the referenced skill before reviewing

**Launch ALL applicable agents in a single message with multiple Agent tool calls.** Every agent MUST use its assigned `WORKTREE_PATH` — never the live working tree — so parallel tool invocations are fully isolated.

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

#### Agent 3: Java Development Review (Conditional)

**Skill reference:** `.claude/skills/java-development/SKILL.md`

**Applies when:** Java source files (`.java`) are changed. **Skip entirely if no Java files are in the diff.**

**Checks:**
- **Coding Standards** — Naming (PascalCase/camelCase), immutability (records, final fields), Optional usage, streams, exception handling, generics, package structure
- **Spring Boot Patterns** — Layered architecture (controller -> service -> repository), constructor injection, DTOs with validation, `@RestControllerAdvice` error handling, async processing, SLF4J structured logging, pagination
- **Unit Testing** — Tests exist for new/modified classes covering happy and unhappy paths. JUnit 5, Mockito, AssertJ used correctly. `@DisplayName`, `@Nested`, given/when/then pattern, `@ParameterizedTest` for variants
- **TDD & Web Layer** — MockMvc tests for controllers, `@DataJpaTest` for repositories, test data builders
- **Code Formatting** — `./gradlew spotlessApply` run (Google Java Style via Spotless)
- **Verification** — Build succeeds, static analysis (checkstyle/pmd/spotbugs) passes, test coverage adequate (JaCoCo), no secrets in source, no `System.out.println`

---

#### Agent 4: BDD & Component Test Review

**Skill reference:** `.claude/skills/testing/SKILL.md`

**Applies when:** Feature files, step definitions, acceptance tests, or component tests are changed, or new endpoints/services are added.

**Checks:**
- **BDD Acceptance Tests** — Feature files follow Gherkin best practices, scenarios cover happy and unhappy paths, step definitions are reusable, `@SpringBootTest` with real Testcontainers (PostgreSQL + Kafka), no mocking, business-readable scenarios
- **Component Tests** — Black-box testing via HTTP (RestAssured), app runs as Docker container via Testcontainers, no Spring context in test JVM, scenarios simulate real user behavior, no direct database or Kafka inspection
- **Common** — Awaitility for async (no `Thread.sleep`), DataTables for structured input, `@smoke`/`@full` tags, containers start once, declarative Gherkin

---

#### Agent 5: Healthcare Domain Review

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

#### Agent 10: Temporal Workflow Review (Conditional)

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

#### Agent 14: Python Development Review (Conditional)

**Skill reference:** `.claude/skills/python-development/SKILL.md`

**Applies when:** Python source files (`.py`) are changed. **Skip entirely if no Python files are in the diff.**

**Checks:**
- Project structure follows src layout with layered architecture (api -> domain -> infrastructure)
- FastAPI patterns (dependency injection, lifespan, Pydantic schemas, async endpoints)
- Configuration via pydantic-settings (no hardcoded values)
- RFC 7807 error responses with domain exceptions
- Async patterns (SQLAlchemy async sessions, proper await usage)
- Type hints on all function signatures (modern syntax: `list[str]`, `str | None`)
- pytest unit tests exist with proper Arrange/Act/Assert structure
- Component tests using httpx AsyncClient + Testcontainers
- BDD tests using pytest-bdd with Gherkin feature files covering happy and unhappy paths
- Structured logging via structlog (no PHI in logs)
- Tooling configured in pyproject.toml (ruff, mypy, pytest)
- No bare `except`, no mutable default arguments
- Alembic migrations for database changes

---

### Step 5: Collect and Deduplicate Results

As sub-agents complete, the orchestrator:

1. Collects all findings from each agent
2. Deduplicates overlapping findings (e.g., same issue flagged by both JPA and Spring Boot agents)
3. Assigns final severity: **Critical**, **Medium**, **Low**
4. Groups findings by file for coherent inline comments

### Step 6: Present Consolidated Findings

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

### Step 7: Ask Permission Before Posting

**CRITICAL: Never post review comments without explicit user permission.**

After presenting findings, ask the user:
- Whether they want to post the comments on the PR
- Whether they want to modify, remove, or add any comments
- Whether they want the comments posted as inline (file-specific) or general comments

Only proceed to post after receiving explicit confirmation.

### Step 8: Post Inline Comments at Specific Lines

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

### Step 9: Clean Up Worktrees

After all comments are posted (or if the review is abandoned), always remove the worktrees:

```bash
git worktree remove --force "$WORKTREE_BASE/pr-head"
git worktree remove --force "$WORKTREE_BASE/pr-base"   # if created
git worktree prune
rm -rf "$WORKTREE_BASE"
```

> **Never leave stale worktrees.** They hold references to the checked-out commits and prevent garbage collection. If the process is interrupted, the user can clean up manually with `git worktree list` and `git worktree remove --force <path>`.

## Comment Tone and Style

**Be polite, constructive, and evidence-based.** Every comment MUST be backed by data — never flag an issue without concrete evidence.

1. **State the severity and agent** — e.g., `**Critical** (Healthcare):` or `**Medium** (Kafka):`
2. **Explain what is wrong** — Concisely, so the reader understands without deep reading
3. **Back it with data** — Every claim must cite evidence. Examples of data-backed arguments:
   - **Code evidence**: "This method is called 14 times across 3 services (grep confirms), so changing its signature is a breaking change"
   - **Performance data**: "This query does a full table scan on `orders` (EXPLAIN shows Seq Scan). With 500K+ rows in production, this will degrade response times beyond the 200ms SLA"
   - **Pattern evidence**: "The other 8 repositories in Polaris use `@Transactional(readOnly = true)` for read queries — this is the only service that omits it"
   - **Specification reference**: "RFC 7807 requires a `type` URI field in error responses (Section 3.1). This endpoint returns `{\"error\": \"not found\"}` instead"
   - **Test coverage**: "This service method has 4 code paths (2 if-branches x 2 states) but only 1 test covering the happy path. The null-input and error-state paths are untested"
   - **Security evidence**: "This field contains PHI (patient date of birth) and is logged at INFO level on line 42. HIPAA §164.312(a)(1) requires access controls on PHI"
   - **Dependency data**: "This version of jackson-databind (2.13.x) has CVE-2022-42003 (CVSS 7.5). The fixed version is 2.13.4.2+"
4. **Explain why it matters** — Concrete scenario where the issue manifests, not hypothetical
5. **Suggest a fix** — Specific recommendation or code example
6. **Reference the skill** — e.g., *Ref: jpa-patterns skill - entity design*
7. **Be honest about uncertainty** — "This may be intentional, but..." or "Consider whether..."
8. **Respect the author's intent** — "Consider using...", "This could be improved by..."

**Do NOT make claims without evidence.** If you cannot find data to support a concern, investigate further (read more code, check git history, search for usage) before raising it. If you still cannot find evidence, either drop the concern or explicitly state it as an observation rather than a finding.

**Avoid:**
- Flattery or filler ("Great job!", "Thanks for this!")
- Accusatory tone ("You forgot to...", "You should have...")
- Overstating severity — a style nit is not a critical bug
- Vague complaints without evidence — "this could be a problem" without showing why
- Opinions disguised as facts — "this is wrong" without citing the standard or pattern it violates

**Example good comment:**
> **Medium** (JPA): `dateOfService` is mapped as `String`, which bypasses date validation and allows malformed values (e.g., `"not-a-date"`) to persist. The database column `orders.date_of_service` is type `DATE` (confirmed in migration V3__create_orders.sql:12), so Hibernate will throw `DataException` at runtime on insert. The other 6 date fields in this service use `LocalDate` — this is the only one mapped as String.
>
> Consider `LocalDate` for type safety and date range queries.
>
> *Ref: jpa-patterns skill - entity design*

**Example bad comment:**
> This should be LocalDate not String.

## Severity Categories

Every finding MUST be categorized. Group and present findings by severity in the summary.

| Severity | When to use | Action required |
|----------|-------------|-----------------|
| **Critical** | Security vulnerabilities, PII/PHI exposure, data loss, broken functionality, missing auth | Must fix before merge |
| **High** | Missing tests for critical paths, NFR violations, broken error handling, incorrect business logic | Should fix before merge |
| **Medium** | Missing tests for edge cases, pattern violations, performance concerns, stale references | Fix recommended, merge at author's discretion |
| **Low** | Style inconsistencies, minor naming issues, documentation gaps, cosmetic improvements | Optional, nice to have |
| **Info** | Observations, suggestions, alternative approaches, knowledge sharing | No action required |

## PII & PHI Data Detection

**CRITICAL: Every review MUST scan for PII/PHI exposure.** Flag any data — or combination of data — that could identify a customer, patient, or individual.

### What to scan for

**Direct identifiers (flag immediately):**
- Full names, first + last name combinations
- Social Security Numbers (SSN), Medicare Beneficiary Identifiers (MBI)
- Dates of birth, date of death
- Phone numbers, email addresses, physical addresses
- Medical record numbers, health plan beneficiary numbers
- NPI associated with patient context
- Device identifiers, serial numbers tied to patients
- Biometric identifiers, facial photographs
- IP addresses, URLs in patient context

**Quasi-identifiers (flag when combined — 2+ together can re-identify):**
- Age, gender, zip code (any 2 of 3 can identify 87% of US population per Sweeney 2000)
- Diagnosis codes (ICD-10) + date of service + provider
- HCPCS codes + quantity + date + zip code
- Admission/discharge dates + facility
- Race/ethnicity + geographic subdivision + age range

### Where to check

- **Log statements** — `log.info()`, `log.debug()`, `log.error()` — PII must never appear in logs
- **Error messages** — Exception messages, API error responses, problem detail bodies
- **API responses** — Response DTOs, JSON serialization — check what's exposed to clients
- **Kafka messages** — Event payloads — check if PII is in message body or headers
- **Database queries in logs** — SQL with parameter values that contain PII
- **Test fixtures** — Hardcoded realistic-looking PII in test data (use synthetic data instead)
- **Comments and TODOs** — Developers sometimes paste real data in comments
- **Configuration files** — Connection strings, API keys, tokens with embedded PII

### How to flag

PII/PHI findings are always **Critical** severity:

> **Critical** (PII): Patient date of birth `patientDob` is included in the error response at `OrderController.java:87`. This field is PHI under HIPAA §164.514(b)(2)(i) and must not be exposed in API responses. The other error responses in this controller (lines 45, 62, 78) correctly omit patient details — this is the only one that leaks.
>
> Suggested fix: Remove `patientDob` from the error response. Return only the order ID and error type.

## Edge Cases

- **GitHub Enterprise**: Always check `gh auth status` and use `--hostname` flag for GHE instances
- **Large PRs**: If the diff is truncated, read the full output and use offset/limit or grep
- **Shell escaping**: Write review JSON to a temp file; use `--input /tmp/review.json`
- **Mixed file types**: Only review code files. Skip generated files, lock files, or binary assets unless they indicate a problem (e.g., committed secrets)
- **Monorepo**: If files span multiple services, launch separate agent groups per service
- **No tests found**: Flag as Critical if new functionality has zero test coverage
