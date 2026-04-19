# polaris-ai-utility

A shared repository of AI skills, commands, and configuration for use across all Polaris project repos. This repo contains no application source code -- it serves as the single source of truth for AI coding assistant knowledge used during development.

## Supported AI Assistants

| Assistant | Config Directory | Assets |
|-----------|-----------------|--------|
| **Claude Code** | `.claude/skills/` | 21 skill definitions (`SKILL.md` files) |
| **GitHub Copilot** | `.github/prompts/` | 21 prompt files that delegate to Claude skills |
| **OpenCode** | `.opencode/commands/` | 21 command wrappers that delegate to Claude skills |

## Skills

### API & Architecture
| Skill | Description |
|-------|-------------|
| `api-design` | REST API design patterns for Spring Boot services (OpenAPI-first, RFC 7807 errors, pagination, versioning) |
| `architecture-decision-records` | Capture architectural decisions as structured ADRs with context, alternatives, and rationale |
| `backend-patterns` | Backend architecture patterns, API design, and server-side best practices for Node.js, Express, and Next.js |

### Java & Spring Boot
| Skill | Description |
|-------|-------------|
| `java-coding-standards` | Naming, immutability, Optional usage, streams, exceptions, generics, and project layout |
| `java-code-formatter` | Auto-format Java source code via `./gradlew spotlessApply` (Google Java Style) |
| `springboot-patterns` | Spring Boot architecture: REST APIs, layered services, data access, caching, async processing, logging |
| `jpa-patterns` | JPA/Hibernate entity design, relationships, query optimization, transactions, auditing, indexing |
| `kafka-patterns` | Kafka producer/consumer patterns, idempotent consumers, DLT error handling, `@EmbeddedKafka` tests |
| `temporal-developer` | Temporal durable execution: workflows, activities, signals, queries, saga patterns, versioning, agentic patterns |

### Testing
| Skill | Description |
|-------|-------------|
| `java-unit-testing` | Unit tests using JUnit 5, Mockito, and AssertJ |
| `springboot-tdd` | Test-driven development for Spring Boot with JUnit 5, Mockito, MockMvc, and H2 |
| `bdd-cucumber` | Full-stack BDD acceptance tests using Cucumber, `@SpringBootTest`, and Testcontainers (PostgreSQL + Kafka) |
| `component-tests` | Black-box component tests using Cucumber, Testcontainers (app as Docker container), and RestAssured |
| `springboot-verification` | Verification loop: build, static analysis, tests with coverage, security scans, and diff review |

### Infrastructure & Data
| Skill | Description |
|-------|-------------|
| `docker-patterns` | Docker/Docker Compose patterns for local development, security, networking, and multi-service orchestration |
| `postgres-patterns` | PostgreSQL query optimization, schema design, indexing, and security |
| `database-migrations` | Migration best practices for schema changes, rollbacks, and zero-downtime deployments |

### Domain & Workflow
| Skill | Description |
|-------|-------------|
| `healthcare-domain` | Healthcare compliance: HIPAA PHI safeguards, NPI validation, ICD-10/HCPCS codes, X12 EDI, Medicaid requirements |
| `code-review` | PR code review: retrieves diffs, reads context, posts inline review comments on GitHub PRs |

### Orchestrated Reviews
| Skill | Description |
|-------|-------------|
| `pr-review` | Multi-agent PR review that launches 14 parallel sub-agents to check API design, backend patterns, tests, healthcare compliance, Kafka, Docker, JPA, Spring Boot, Temporal, coding standards, NFR alignment, and technical design conformance. Posts inline comments on PRs after user approval. |
| `review` | Local code review of uncommitted changes using the same 14 parallel sub-agents as `pr-review`. After presenting findings, offers to create a PR with an auto-generated description based on the review results. |

Both orchestrated review skills validate changes against the [Technical Design](https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/technical-design/overview.md) and [Decision Records](https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/decision-records/overview.md), including NFR checks for performance, scalability, availability, security, observability, reliability, maintainability, and compliance.

## Usage

### With Claude Code

Skills are automatically available when this repo is configured as a dependency. Claude Code detects and loads skills from `.claude/skills/` based on task context.

### With GitHub Copilot

Prompt files in `.github/prompts/` are available in Copilot Chat. Each prompt delegates to the corresponding Claude skill file, keeping a single source of truth. Reference them as `@workspace /pr-review` or `#pr-review` in Copilot Chat.

### With OpenCode

Commands in `.opencode/commands/` act as thin wrappers that delegate to the corresponding Claude skill files. Each command accepts `$ARGUMENTS` and passes them to the skill.

## Automatic Sync to Polaris Repos

A GitHub Actions workflow (`.github/workflows/sync-ai-config.yml`) automatically syncs `.claude/`, `.github/prompts/`, and `.opencode/` to all downstream Polaris service repos on merge to `main`.

**Currently synced repos:**
- [polaris-order-connector](https://waveum.ghe.com/waveum/polaris-order-connector)
- [polaris-order-workflow-agent](https://waveum.ghe.com/waveum/polaris-order-workflow-agent)

The workflow creates a PR in each target repo with title `fix(POL-00): Synced AI Skills`. To add a new repo, add an entry to the `matrix.target` array in the workflow file.

**Required secret:** `AI_UTILITY_TOKEN` -- a GHE Personal Access Token with repo write access to all target repos.

## Project Structure

```
polaris-ai-utility/
├── .claude/
│   └── skills/                    # 21 Claude Code skill definitions
│       ├── api-design/
│       ├── architecture-decision-records/
│       ├── backend-patterns/
│       ├── bdd-cucumber/
│       ├── code-review/
│       ├── component-tests/
│       ├── database-migrations/
│       ├── docker-patterns/
│       ├── healthcare-domain/
│       ├── java-code-formatter/
│       ├── java-coding-standards/
│       ├── java-unit-testing/
│       ├── jpa-patterns/
│       ├── kafka-patterns/
│       ├── postgres-patterns/
│       ├── pr-review/             # Orchestrated multi-agent PR review
│       ├── review/                # Orchestrated local code review
│       ├── springboot-patterns/
│       ├── springboot-tdd/
│       ├── springboot-verification/
│       └── temporal-developer/
│           ├── SKILL.md
│           └── references/        # 14 reference docs for Temporal
├── .github/
│   ├── prompts/                   # 21 GitHub Copilot prompt files
│   └── workflows/
│       └── sync-ai-config.yml     # Auto-sync to downstream repos
├── .opencode/
│   ├── commands/                  # 21 OpenCode command wrappers
│   └── package.json               # @opencode-ai/plugin dependency
├── .gitignore
└── README.md
```

## Contributing

To add a new skill:

1. Create a new directory under `.claude/skills/<skill-name>/` with a `SKILL.md` file.
2. Create a matching prompt file at `.github/prompts/<skill-name>.prompt.md` that delegates to the skill.
3. Create a matching command wrapper at `.opencode/commands/<skill-name>.md` that delegates to the skill.
4. Update this README with the new skill's description.
5. Raise a PR against this repository for review.

Changes merged to `main` will automatically sync to all downstream Polaris repos via the GitHub Actions workflow.

