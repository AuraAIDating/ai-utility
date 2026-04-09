# polaris-ai-utility

A shared repository of AI skills, commands, and configuration for use across all Polaris project repos. This repo contains no application source code -- it serves as the single source of truth for AI coding assistant knowledge used during development.

## Supported AI Assistants

| Assistant | Config Directory | Assets |
|-----------|-----------------|--------|
| **Claude Code** | `.claude/skills/` | 19 skill definitions (`SKILL.md` files) |
| **OpenCode** | `.opencode/commands/` | 19 command wrappers that delegate to Claude skills |

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

## Usage

### With Claude Code

Skills are automatically available when this repo is configured as a dependency. Claude Code detects and loads skills from `.claude/skills/` based on task context.

### With OpenCode

Commands in `.opencode/commands/` act as thin wrappers that delegate to the corresponding Claude skill files. Each command accepts `$ARGUMENTS` and passes them to the skill.

### Distribution to Polaris Repos

Skills and commands are automatically synced to all Polaris project repos via a scheduled GitHub Actions workflow. The workflow compares the contents of this repository against each target repo and copies any new or updated files. No manual setup is required in consuming repos -- changes pushed here will propagate automatically on the next sync cycle.

## Project Structure

```
polaris-ai-utility/
├── .claude/
│   └── skills/                  # 19 Claude Code skill definitions
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
│       ├── springboot-patterns/
│       ├── springboot-tdd/
│       ├── springboot-verification/
│       └── temporal-developer/
│           ├── SKILL.md
│           └── references/      # 14 reference docs for Temporal
├── .opencode/
│   ├── commands/                # 19 OpenCode command wrappers
│   └── package.json             # @opencode-ai/plugin dependency
├── .gitignore
└── README.md
```

## Contributing

To add a new skill:

1. Create a new directory under `.claude/skills/<skill-name>/` with a `SKILL.md` file.
2. Create a matching command wrapper under `.opencode/commands/<skill-name>.md`.
3. Update this README with the new skill's description.
4. Raise a PR against this repository for review.
