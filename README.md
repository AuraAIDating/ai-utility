# AI Utility

A shared repository of AI skills, prompts, and commands for use across all repos in the [AuraAIDating](https://github.com/AuraAIDating) organization. This repo contains no application source code — it serves as the **single source of truth** for AI coding assistant knowledge used during development.

Skills are authored once as Claude Code skill files (`.claude/skills/`) and are automatically available to **three AI assistants**: Claude Code, GitHub Copilot, and OpenCode. GitHub Copilot prompts and OpenCode commands are thin wrappers that delegate to the Claude skill files, so there is only one place to maintain content.

## Supported AI Assistants

| Assistant | Config Directory | Format | How skills are loaded |
|-----------|-----------------|--------|----------------------|
| **Claude Code** | `.claude/skills/<name>/SKILL.md` | Markdown with YAML frontmatter | Auto-detected by context |
| **GitHub Copilot** | `.github/prompts/<name>.prompt.md` | Prompt file with frontmatter | `@workspace /<name>` in Copilot Chat |
| **OpenCode** | `.opencode/commands/<name>.md` | Plain markdown | `/<name>` in OpenCode CLI |

All three tools share the same skill content. The Copilot and OpenCode files simply reference the Claude skill:

```markdown
Use the <skill-name> skill from .claude/skills/<skill-name>/SKILL.md to help with the following request:

$ARGUMENTS
```

## Skills

### Development
| Skill | Description |
|-------|-------------|
| `java-development` | Complete Java 21+ / Spring Boot skill: coding standards, naming, immutability, Spring Boot patterns (REST, services, JPA, caching, async), unit testing (JUnit 5, Mockito, AssertJ, MockMvc), TDD workflow, code formatting (Spotless), and verification loop (build, static analysis, coverage, security scans) |
| `python-development` | Python 3.11+ patterns: project structure, FastAPI HTTP APIs, async patterns, dependency injection, Pydantic models, pytest unit/component testing, BDD with pytest-bdd, type hints, ruff/mypy, structured logging |
| `api-design` | REST API design for Spring Boot services: OpenAPI-first workflow, RFC 7807 errors, SLA contracts, tracing headers, pagination, versioning, rate limiting |
| `jpa-patterns` | JPA/Hibernate entity design, relationships, query optimization, transactions, auditing, indexing, pagination, connection pooling |
| `kafka-patterns` | Kafka producer/consumer patterns, idempotent consumers, DLT error handling, `@EmbeddedKafka` tests, Awaitility assertions |
| `temporal-developer` | Temporal durable execution: workflows, activities, signals, queries, saga patterns, versioning, agentic patterns |

### Testing
| Skill | Description |
|-------|-------------|
| `testing` | BDD acceptance tests (Cucumber + `@SpringBootTest` + Testcontainers) and black-box component tests (Cucumber + Docker container + RestAssured). Covers Gherkin feature files, step definitions, DataTables, async polling, WireMock |

### Infrastructure
| Skill | Description |
|-------|-------------|
| `docker-patterns` | Docker/Docker Compose patterns for local development, security, networking, and multi-service orchestration |
| `terraform` | Terraform infrastructure patterns on AWS and Azure. **All projects must use S3+DynamoDB remote state.** Covers module structure, IAM role assumption, environment separation (dev/staging/prod), KMS encryption, security best practices, naming conventions, and multi-cloud (EKS, RDS, VPC, SES, Lambda, AKS, APIM) |

### Tooling
| Skill | Description |
|-------|-------------|
| `graphify` | Any input (code, docs, papers, images) to knowledge graph with clustered communities, HTML + JSON + audit report |
| `context-save` | Save working session context so any future AI session can resume without losing a beat. Stores state in `.my-context/` |
| `context-restore` | Restore the most recent saved session context from `.my-context/` and resume work immediately |
| `drift-management` | Detect drift between the current repo's code and its governing Technical Design docs and Decision Records |
| `investigate` | Systematic debugging: four-phase root cause investigation (investigate → analyze → hypothesize → implement). Iron Law: no fixes without root cause |

### Orchestrated Reviews
| Skill | Description |
|-------|-------------|
| `review` | Local code review of uncommitted changes using parallel sub-agents. Checks Java/Python code, API design, tests, Kafka, Docker, JPA, Temporal, NFRs, and PII exposure. Offers to create a PR with an auto-generated description after review |
| `pr-review` | Same multi-agent review as `review` but for pull requests. Posts inline comments at specific line numbers on the PR after user approval |
| `review-ai-utility` | Self-review for this repo: validates skill/prompt/command sync, checks broken references, ensures consistency across all three AI tools |

Both `review` and `pr-review` skills:
- Launch parallel sub-agents for each review domain (only agents relevant to the changed files)
- Check NFRs: performance, scalability, availability, security, observability, reliability, maintainability
- Scan for PII exposure (direct identifiers and quasi-identifier combinations)
- Categorize findings by severity: Critical, High, Medium, Low, Info
- Require data-backed evidence for every comment — no opinions without proof
- Always ask permission before committing or posting PR comments

## How Skills Work

### Claude Code Skills

**Location:** `.claude/skills/<name>/SKILL.md`

Skills are Markdown files with YAML frontmatter (`name`, `description`) and rich content (patterns, code examples, checklists, rules). They are the **source of truth** — all other formats delegate to them.

Claude Code scans `.claude/skills/` at **session start** and indexes all skills by their `description` field. Skills are **not** injected into every conversation — they are loaded **on demand** when Claude determines the current task matches a skill's description.

Skills activate **automatically** based on context:

```
# Claude auto-activates java-development because you're working on .java files
> Add a new REST endpoint for users

# Claude activates testing because you're asking about BDD scenarios
> Write BDD scenarios for the order submission flow

# Multiple skills can activate at once — java-development + jpa-patterns + api-design
> Create a paginated GET endpoint with JPA repository for orders
```

You can also invoke skills explicitly as slash commands:

```
/review
/pr-review https://github.com/AuraAIDating/my-service/pull/42
/java-development create a service for order processing
```

### OpenCode Commands

**Location:** `.opencode/commands/<name>.md`

Slash commands for the OpenCode CLI. Registered at startup, activated by explicit invocation:

```
/java-development create a service for order processing
/review
/pr-review https://github.com/AuraAIDating/my-service/pull/42
/review-ai-utility
```

### GitHub Copilot Prompts

**Location:** `.github/prompts/<name>.prompt.md`

Prompt files registered by VS Code when the workspace opens. Activated in Copilot Chat:

```
@workspace /java-development create a service for order processing
@workspace /review
@workspace /testing write component tests for the order endpoint
@workspace /pr-review https://github.com/AuraAIDating/my-service/pull/42
```

### Summary

| Tool | File Format | Loaded at | Activated by | Auto-activates? |
|------|-------------|-----------|--------------|-----------------|
| **Claude Code Skills** | `.claude/skills/<name>/SKILL.md` | Session start (index only) | Context matching against task | Yes |
| **Claude Code Skills** | `.claude/skills/<name>/SKILL.md` | Session start | User types `/<name>` | No — explicit |
| **OpenCode Commands** | `.opencode/commands/<name>.md` | CLI startup | User types `/<name>` | No — explicit |
| **GitHub Copilot Prompts** | `.github/prompts/<name>.prompt.md` | Workspace open | User types `@workspace /<name>` | No — explicit |

### External Documentation

- [Claude Code Skills](https://code.claude.com/docs/en/skills) — Creating, configuring, and sharing skills
- [GitHub Copilot Custom Instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot) — Repository prompt files
- [OpenCode Documentation](https://opencode.ai/docs) — Getting started with OpenCode
- [OpenCode Commands](https://opencode.ai/docs/commands) — Custom command configuration

## Automatic Sync to Downstream Repos

A GitHub Actions workflow (`.github/workflows/sync-ai-config.yml`) keeps downstream repos in sync with this utility.

### How it works

1. **Trigger**: Any push to `main` that modifies `.claude/`, `.github/prompts/`, `.opencode/`, or the workflow file itself
2. **Sync**: Uses `rsync --archive --delete` to sync three directories to each target repo:
   - `.claude/` — Claude Code skills
   - `.github/prompts/` — GitHub Copilot prompts
   - `.opencode/` — OpenCode commands
3. **PR creation**: Creates a PR in each target repo with branch `chore/sync-ai-config`
4. **Merge**: A team member in each target repo reviews and merges the sync PR

The `--delete` flag in rsync ensures that skills removed from this repo are also removed from target repos.

### Adding a new repo to the sync

1. Open `.github/workflows/sync-ai-config.yml`
2. Add a new entry to the `matrix.target` array:
   ```yaml
   target:
     - { repo: "AuraAIDating/my-service" }
     - { repo: "AuraAIDating/my-new-service" }   # <-- add here
   ```
3. Ensure the `AI_BOT_APP_ID` and `AI_BOT_PRIVATE_KEY` secrets are configured
4. Merge to `main` — the workflow will trigger and create a sync PR in the new repo

## Project Structure

```
ai-utility/
├── .claude/
│   └── skills/                       # 17 Claude Code skill definitions
│       ├── api-design/
│       ├── context-restore/
│       ├── context-save/
│       ├── docker-patterns/
│       ├── drift-management/
│       ├── graphify/
│       ├── investigate/
│       ├── java-development/
│       ├── jpa-patterns/
│       ├── kafka-patterns/
│       ├── pr-review/                # Multi-agent PR review
│       ├── python-development/
│       ├── review/                   # Multi-agent local review
│       ├── review-ai-utility/        # Self-review for this repo
│       ├── temporal-developer/
│       │   ├── SKILL.md
│       │   └── references/           # Reference docs for Temporal
│       ├── terraform/
│       └── testing/
├── .github/
│   ├── prompts/                      # 17 GitHub Copilot prompt files
│   └── workflows/
│       └── sync-ai-config.yml        # Auto-sync to downstream repos
├── .opencode/
│   ├── commands/                     # 17 OpenCode command wrappers
│   └── package.json
├── .gitignore
├── CODEOWNERS
└── README.md
```

## Contributing

### Adding a new skill

Every skill requires three files — one for each AI tool. All three must be created together.

**Step 1: Create the Claude Code skill (source of truth)**

Create `.claude/skills/<skill-name>/SKILL.md`:

```markdown
---
name: <skill-name>
description: "Short description (under 300 chars) — used by Claude to decide when to activate"
---

# Skill Title

## When to Activate
- Bullet list of contexts where this skill applies

## Content
- Patterns, code examples, checklists, rules
```

The `name` field MUST match the directory name exactly.

**Step 2: Create the GitHub Copilot prompt**

Create `.github/prompts/<skill-name>.prompt.md`:

```markdown
---
mode: agent
description: "Same description as the skill"
tools: ["codebase", "githubRepo"]
---

Use the <skill-name> skill from .claude/skills/<skill-name>/SKILL.md to help with the following request:

$ARGUMENTS
```

**Step 3: Create the OpenCode command**

Create `.opencode/commands/<skill-name>.md`:

```markdown
Use the <skill-name> skill from .claude/skills/<skill-name>/SKILL.md to help with the following request:

$ARGUMENTS
```

Note: OpenCode commands have NO frontmatter — just plain markdown.

**Step 4: Update this README**

Add the new skill to the appropriate table in the Skills section.

**Step 5: Validate**

```
/review-ai-utility
```

**Step 6: Submit a PR**

Raise a PR against `main`. Once merged, the sync workflow will distribute the skill to all downstream repos automatically.

### Modifying an existing skill

Edit only the `.claude/skills/<name>/SKILL.md` file. The Copilot prompt and OpenCode command reference it by path, so they don't need updating unless the skill is renamed.

### Renaming a skill

1. Rename the directory under `.claude/skills/`
2. Delete the old prompt and command files
3. Create new prompt and command files with the new name
4. Update all references in other skills (e.g., `review` and `pr-review` agent definitions)
5. Update this README
6. Run `/review-ai-utility` to catch any broken references

### Removing a skill

1. Delete the directory under `.claude/skills/`
2. Delete the matching `.github/prompts/<name>.prompt.md`
3. Delete the matching `.opencode/commands/<name>.md`
4. Remove references from other skills
5. Update this README
6. The sync workflow's `rsync --delete` will remove it from all downstream repos on next merge

### Conventions

- **Skill names**: lowercase, kebab-case (e.g., `java-development`, `kafka-patterns`)
- **Single source of truth**: All skill content lives in `.claude/skills/`. Copilot and OpenCode files only reference it
- **No duplication**: If two skills overlap significantly, combine them
- **Trailing newlines**: All files must end with a trailing newline
- **Data-backed reviews**: The `review` and `pr-review` skills require evidence for every comment — no subjective opinions without proof
- **PII scanning**: Every review must check for personally identifiable information exposure
