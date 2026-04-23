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

#### Agent 3: Java Development Review (Conditional)

**Skill reference:** `.claude/skills/java-development/SKILL.md`

**Applies when:** Java source files (`.java`) are changed. **Skip entirely if no Java files are in the diff.**

**Checks:**
- **Coding Standards** — Naming, immutability, Optional usage, streams, exception handling, generics, package structure
- **Spring Boot Patterns** — Layered architecture, constructor injection, DTOs with validation, error handling, async, structured logging, pagination
- **Unit Testing** — Tests exist covering happy and unhappy paths. JUnit 5, Mockito, AssertJ. `@DisplayName`, `@Nested`, given/when/then, `@ParameterizedTest`
- **TDD & Web Layer** — MockMvc tests, `@DataJpaTest`, test data builders
- **Code Formatting** — `./gradlew spotlessApply` (Google Java Style via Spotless)
- **Verification** — Build, static analysis, test coverage, no secrets in source

---

#### Agent 4: BDD & Component Test Review

**Skill reference:** `.claude/skills/testing/SKILL.md`

**Applies when:** Feature files, step definitions, acceptance tests, or component tests are changed, or new endpoints/services are added.

**Checks:**
- **BDD Acceptance Tests** — Gherkin best practices, happy and unhappy paths, reusable step definitions, `@SpringBootTest` with real Testcontainers, no mocking, business-readable
- **Component Tests** — Black-box via HTTP (RestAssured), app as Docker container, no Spring context, user-perspective assertions only
- **Common** — Awaitility for async, DataTables for input, `@smoke`/`@full` tags, containers start once, declarative Gherkin

---

#### Agent 5: Healthcare Domain Review

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

#### Agent 10: Temporal Workflow Review (Conditional)

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

#### Agent 11: Terraform Infrastructure Review (Conditional)

**Skill reference:** `.claude/skills/terraform/SKILL.md`

**Applies when:** Terraform files (`.tf`), `terraform/` directories, or backend state configs are changed. **Skip entirely if no Terraform files are in the diff.**

**Checks:**
- Project structure: per-environment directories (dev, staging, prod) with separated Terraform state; single-service repos have one state per environment; multi-component repos have per-component states
- Backend configuration: S3 bucket as remote backend with DynamoDB locking table, KMS CMK encryption with rotation enabled, unique state keys per environment
- State file naming: `terraform/aws/environments/{env}/{service-name}/terraform.tfstate` for single-service or `terraform/aws/environments/{env}/{component}/terraform.tfstate` for multi-component
- Terraform block: version constraint `>= 1.0`, backend with S3 bucket/key/region/dynamodb_table/encrypt all present; profile optional when using AWS profiles
- Provider blocks: AWS `~> 5.0` (or Azure `~> 3.0`), region/subscription/tenant variables externalized
- Variables and outputs: all have descriptions, validation rules for constraints, sensitive flags for secrets, defaults documented
- Tagging strategy: standard tags (Project, Environment, ManagedBy, Owner) applied to all resources via locals block
- IAM patterns: assume_role with correct waveum-polaris-{service}-terraform-deploy-role, least privilege, no hardcoded credentials
- KMS encryption: CMK mandatory for state files (S3 + DynamoDB) and production sensitive resources; case-by-case for other S3 buckets (dev/staging non-sensitive may use AES256); `enable_key_rotation = true` mandatory for all CMKs; Secrets Manager uses AWS-managed key only (not CMK)
- S3 encryption: Terraform state must use CMK; production data must use CMK; dev/staging non-sensitive buckets may use AES256
- Bootstrap process: self-contained backend module creates S3 bucket + DynamoDB table automatically with CMK; 3-step initialization documented
- S3 security: all four block public access settings enabled, versioning enabled, encryption type appropriate for data sensitivity
- Secrets management: AWS Secrets Manager with AWS-managed key (never kms_key_id parameter)
- Module design: single responsibility, no provider blocks in modules, variables/outputs exposed, tags passed through
- Environment separation: no shared state files, no cross-environment hardcoded values
- Documentation: README documents bootstrap steps, environment setup, how to run terraform per environment

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

#### Agent 14: Python Development Review (Conditional)

**Skill reference:** `.claude/skills/python-development/SKILL.md`

**Applies when:** Python source files (`.py`) are changed. **Skip entirely if no Python files are in the diff.**

**Checks:**
- Project structure follows src layout with layered architecture (api -> domain -> infrastructure)
- FastAPI patterns (dependency injection, lifespan, Pydantic schemas, async endpoints)
- Configuration via pydantic-settings (no hardcoded values)
- RFC 7807 error responses with domain exceptions
- Async patterns (SQLAlchemy async, proper await usage)
- Type hints on all function signatures (modern syntax)
- pytest unit tests with Arrange/Act/Assert
- Component tests using httpx AsyncClient + Testcontainers
- BDD tests using pytest-bdd with Gherkin feature files covering happy and unhappy paths
- Structured logging via structlog (no PHI in logs)
- Tooling configured in pyproject.toml (ruff, mypy, pytest)
- No bare `except`, no mutable default arguments
- Alembic migrations for database changes

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

**Be polite, constructive, and evidence-based.** Every comment MUST be backed by data — never flag an issue without concrete evidence.

1. **State the severity and agent** — e.g., `**Critical** (Healthcare):` or `**Medium** (Kafka):`
2. **Explain what is wrong** — Concisely, so the reader understands without deep reading
3. **Back it with data** — Every claim must cite evidence. Examples of data-backed arguments:
   - **Code evidence**: "This method is called 14 times across 3 services (grep confirms), so changing its signature is a breaking change"
   - **Performance data**: "This query does a full table scan on `orders` (EXPLAIN shows Seq Scan). With 500K+ rows in production, this will degrade response times beyond the 200ms SLA"
   - **Pattern evidence**: "The other 8 repositories in Polaris use `@Transactional(readOnly = true)` for read queries — this is the only service that omits it"
   - **Specification reference**: "RFC 7807 requires a `type` URI field in error responses (Section 3.1). This endpoint returns `{\"error\": \"not found\"}` instead"
   - **Test coverage**: "This service method has 4 code paths but only 1 test covering the happy path"
   - **Security evidence**: "This field contains PHI and is logged at INFO level. HIPAA §164.312(a)(1) requires access controls on PHI"
   - **Dependency data**: "This version has CVE-2022-42003 (CVSS 7.5). Fixed version is 2.13.4.2+"
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
> **Medium** (JPA): `dateOfService` is mapped as `String`, which bypasses date validation. The DB column is type `DATE` (confirmed in migration V3__create_orders.sql:12), so Hibernate will throw `DataException` on insert. The other 6 date fields in this service use `LocalDate` — this is the only one mapped as String.
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
