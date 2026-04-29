# Polaris AI Utility

A shared repository of AI skills, prompts, and commands for use across all Polaris project repos. This repo contains no application source code -- it serves as the **single source of truth** for AI coding assistant knowledge used during development.

Skills are authored once as Claude Code skill files (`.claude/skills/`) and are automatically available to **three AI assistants**: Claude Code, GitHub Copilot, and OpenCode. GitHub Copilot prompts and OpenCode commands are thin wrappers that delegate to the Claude skill files, so there is only one place to maintain content.

## Supported AI Assistants

| Assistant | Config Directory | Format | How skills are loaded |
|-----------|-----------------|--------|----------------------|
| **Claude Code** | `.claude/skills/<name>/SKILL.md` | Markdown with YAML frontmatter | Auto-detected by context |
| **GitHub Copilot** | `.github/prompts/<name>.prompt.md` | Prompt file with frontmatter | `@workspace /<name>` or `#<name>` in Copilot Chat |
| **OpenCode** | `.opencode/commands/<name>.md` | Plain markdown | `/command <name>` in OpenCode CLI |

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
| `terraform` | Terraform infrastructure patterns for Polaris services on AWS and Azure. **All projects (service or shared infra) must use S3+DynamoDB remote state.** Covers module structure, remote state backends, IAM role assumption, environment separation (dev/staging/prod), KMS encryption, security best practices, HIPAA compliance, naming conventions, and multi-cloud (EKS, RDS, VPC, SES, Lambda, AKS, APIM) |

### Domain
| Skill | Description |
|-------|-------------|
| `healthcare-domain` | Healthcare compliance: HIPAA PHI safeguards, NPI validation, ICD-10/HCPCS codes, X12 EDI, Medicaid requirements |
| `graphify` | Any input (code, docs, papers, images) to knowledge graph with clustered communities, HTML + JSON + audit report. HIPAA-safe with PHI pre-flight check |

### Orchestrated Reviews
| Skill | Description |
|-------|-------------|
| `review` | Local code review of uncommitted changes using parallel sub-agents. Checks Java/Python code, API design, tests, healthcare compliance, Kafka, Docker, JPA, Temporal, NFRs, PII/PHI exposure, and technical design alignment. After review, offers to create a PR with an auto-generated description |
| `pr-review` | Same multi-agent review as `review` but for pull requests. Posts inline comments at specific line numbers on the PR after user approval |
| `drift-management` | Detect drift between the current repo's code and its governing Technical Design docs and Decision Records. Auto-detects the repo from git context, scopes doc fetching to that service, and classifies findings as implementation drift, undocumented decisions, or stale documentation |
| `review-ai-utility` | Self-review for this repo: validates skill/prompt/command sync, checks broken references, ensures consistency across all three AI tools |

Both `review` and `pr-review` skills:
- Launch parallel sub-agents for each review domain (only agents relevant to the changed files)
- Validate changes against the [Technical Design](https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/technical-design/overview.md) and [Decision Records](https://waveum.ghe.com/waveum/polaris-documentation/blob/main/docs/decision-records/overview.md)
- Check NFRs: performance, scalability, availability, security, observability, reliability, maintainability, compliance
- Scan for PII/PHI exposure (direct identifiers and quasi-identifier combinations)
- Categorize findings by severity: Critical, High, Medium, Low, Info
- Require data-backed evidence for every comment -- no opinions without proof
- Always ask permission before committing or posting PR comments

## Skills vs Commands vs Prompts

Understanding how each AI tool loads and uses these files is key to working with this repo.

### Skills (Claude Code)

**Location:** `.claude/skills/<name>/SKILL.md`
**Docs:** [Claude Code Skills](https://code.claude.com/docs/en/skills)

**What they are:** Markdown files with YAML frontmatter (`name`, `description`) and rich content (patterns, code examples, checklists, rules). Skills are the **source of truth** -- all other formats delegate to them.

**How they are loaded:** Claude Code scans `.claude/skills/` at **session start** and indexes all skills by their `description` field. Skills are **not** injected into every conversation -- they are loaded **on demand** when Claude determines the current task matches a skill's description. Once loaded, the skill content stays in context for the rest of the session (surviving compaction up to 5,000 tokens per skill).

**When they activate in the agent lifecycle:**

1. **Session start** -- Claude Code reads all `SKILL.md` frontmatter (name + description) and builds an index. Only descriptions are loaded at this point, not full content
2. **User prompt** -- User asks a question or gives a task
3. **Skill matching** -- Claude evaluates the user's request against skill descriptions and decides which skills are relevant
4. **Skill injection** -- Relevant skill content is loaded into context to guide Claude's response. Multiple skills can activate simultaneously
5. **Response** -- Claude uses the skill's patterns, rules, and examples to generate the answer
6. **Persistence** -- The skill stays in context for subsequent turns. Auto-compaction preserves the most recently invoked skills (25,000 token combined budget)

Skills activate **automatically** -- you don't need to invoke them. Claude matches based on context:

```
# Claude auto-activates java-development because you're working on .java files
> Add a new REST endpoint for patients

# Claude activates testing because you're asking about BDD scenarios
> Write BDD scenarios for the order submission flow

# Claude activates healthcare-domain because NPI is a healthcare concept
> Validate the NPI number before saving the provider

# Multiple skills can activate at once -- java-development + jpa-patterns + api-design
> Create a paginated GET endpoint with JPA repository for orders
```

You can also invoke skills explicitly as slash commands:

```
# Explicit invocation -- identical to commands
/review
/pr-review https://waveum.ghe.com/waveum/polaris-order-connector/pull/42
/java-development create a service for order processing
```

### Commands (Claude Code)

**Location:** `.claude/commands/<name>.md`
**Docs:** [Claude Code Skills](https://code.claude.com/docs/en/skills) (commands are merged into the skills system)

**What they are:** User-invocable slash commands that you explicitly type as `/<name>`. Commands and skills are now **unified** in Claude Code -- a file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Skills are preferred because they support additional features (supporting files, frontmatter options, auto-activation).

**When they activate:** Only when the user explicitly types `/<command-name>` in the prompt. The `$ARGUMENTS` placeholder is replaced with whatever the user types after the command name.

**Why this repo uses skills instead of commands:** Our skills are designed to auto-activate by context, which is more natural for development workflows. You don't need to remember command names -- just describe what you need and Claude picks the right skill. Skills can also be invoked as `/skill-name` when you want explicit control.

### OpenCode Commands

**Location:** `.opencode/commands/<name>.md`
**Docs:** [OpenCode Commands](https://opencode.ai/docs/commands)

**What they are:** Slash commands for the OpenCode CLI. Each file is plain markdown (no frontmatter). They delegate to the Claude skill file by referencing its path.

**How they are loaded:** OpenCode reads `.opencode/commands/` at startup and registers each `.md` file as an available slash command.

**When they activate:** Only when the user explicitly types the command:

```
# Explicit invocation -- OpenCode reads the command file, replaces $ARGUMENTS, and runs it
/java-development create a service for order processing

# Review local changes
/review

# Review a specific PR
/pr-review https://waveum.ghe.com/waveum/polaris-order-connector/pull/42

# Run the self-review
/review-ai-utility

# Use healthcare skill for compliance check
/healthcare-domain check if this endpoint handles PHI correctly
```

### GitHub Copilot Prompts

**Location:** `.github/prompts/<name>.prompt.md`
**Docs:** [GitHub Copilot Reusable Prompts](https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot)

**What they are:** Prompt files with YAML frontmatter (`mode`, `description`, `tools`) and a body that delegates to the Claude skill. They appear as reusable prompts in VS Code Copilot Chat.

**How they are loaded:** VS Code detects `.github/prompts/` in the workspace and registers each `.prompt.md` file as an available prompt. The `description` field helps Copilot understand when to suggest the prompt.

**When they activate:** When the user references them in Copilot Chat:

```
# As a slash command in Copilot Chat
@workspace /java-development create a service for order processing

# Review a PR
@workspace /pr-review https://waveum.ghe.com/waveum/polaris-order-connector/pull/42

# Review local changes
@workspace /review

# Check testing patterns
@workspace /testing write component tests for the eligibility endpoint

# Healthcare compliance
@workspace /healthcare-domain verify PHI handling in the patient API
```

### Summary: Loading and Activation

| Tool | File Format | Loaded at | Activated by | Auto-activates? |
|------|-------------|-----------|--------------|-----------------|
| **Claude Code Skills** | `.claude/skills/<name>/SKILL.md` | Session start (index only) | Context matching against task | Yes -- based on description match |
| **Claude Code Skills** | `.claude/skills/<name>/SKILL.md` | Session start | User types `/<name>` | No -- explicit invocation |
| **OpenCode Commands** | `.opencode/commands/<name>.md` | CLI startup | User types `/<name>` | No -- explicit only |
| **GitHub Copilot Prompts** | `.github/prompts/<name>.prompt.md` | Workspace open | User types `@workspace /<name>` | No -- explicit only |

**Key insight:** Claude Code skills are the only format that auto-activates by matching task context against descriptions. OpenCode and Copilot always require explicit invocation. This is why skills are the source of truth -- they contain the full knowledge that Claude uses during normal development, while the other formats are shortcuts for when you want to explicitly invoke a specific capability.

### External Documentation

- [Claude Code Skills](https://code.claude.com/docs/en/skills) -- Creating, configuring, and sharing skills
- [Claude Code Commands Reference](https://code.claude.com/docs/en/commands) -- Built-in commands and bundled skills
- [GitHub Copilot Custom Instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot) -- Repository prompt files and custom instructions
- [OpenCode Documentation](https://opencode.ai/docs) -- Getting started with OpenCode
- [OpenCode Commands](https://opencode.ai/docs/commands) -- Custom command configuration

## Automatic Sync to Polaris Repos

A GitHub Actions workflow (`.github/workflows/sync-ai-config.yml`) keeps downstream repos in sync with this utility.

### How it works

1. **Trigger**: Any push to `main` that modifies `.claude/`, `.github/prompts/`, `.opencode/`, or the workflow file itself
2. **Sync**: Uses `rsync --archive --delete` to sync three directories to each target repo:
   - `.claude/` -- Claude Code skills
   - `.github/prompts/` -- GitHub Copilot prompts
   - `.opencode/` -- OpenCode commands
3. **PR creation**: Creates a PR in each target repo with:
   - **Title**: `fix(POL-00): Synced AI Skills`
   - **Description**: `Synced AI skills from Polaris AI Utility`
   - **Branch**: `chore/sync-ai-config`
   - **Labels**: `automation`, `ai-config`
4. **Merge**: A team member in each target repo reviews and merges the sync PR

The workflow does **NOT** sync `.github/workflows/` -- only `.github/prompts/`. Each target repo keeps its own workflows.

The `--delete` flag in rsync ensures that skills removed from this repo are also removed from target repos. This keeps everything in perfect sync.

### Currently synced repos

- [polaris-order-connector](https://waveum.ghe.com/waveum/polaris-order-connector)
- [polaris-order-workflow-agent](https://waveum.ghe.com/waveum/polaris-order-workflow-agent)
- [polaris-notification-service](https://waveum.ghe.com/waveum/polaris-notification-service)

### Adding a new repo to the sync

1. Open `.github/workflows/sync-ai-config.yml`
2. Add a new entry to the `matrix.target` array:
   ```yaml
   target:
     - { repo: "waveum/polaris-order-connector" }
     - { repo: "waveum/polaris-order-workflow-agent" }
     - { repo: "waveum/polaris-notification-service" }
     - { repo: "waveum/polaris-your-new-service" }   # <-- add here
   ```
3. Ensure the `AI_UTILITY_TOKEN` secret has write access to the new repo
4. Merge to `main` -- the workflow will trigger and create a sync PR in the new repo

### Required secret

`AI_UTILITY_TOKEN` -- a GitHub Enterprise Personal Access Token with `repo` scope and write access to all target repos. This secret is configured in the repository settings of `polaris-ai-utility`.

### What happens when you merge a PR to this repo

1. GitHub detects the push to `main`
2. The workflow checks if any files in `.claude/`, `.github/prompts/`, `.opencode/`, or the workflow itself changed
3. If yes, it runs one parallel job per target repo
4. Each job checks out both repos, rsyncs the three directories, and creates a PR
5. If a `chore/sync-ai-config` branch already exists in the target (from a previous un-merged sync), the PR is updated in place
6. Team members in each target repo review and merge the sync PR at their convenience

## Project Structure

```
polaris-ai-utility/
├── .claude/
│   └── skills/                       # 15 Claude Code skill definitions
│       ├── api-design/
│       ├── docker-patterns/
│       ├── drift-management/         # Drift detection: code vs Technical Design + Decision Records
│       ├── graphify/
│       ├── healthcare-domain/
│       ├── java-development/         # Combined: coding standards + Spring Boot + testing + TDD + formatting + verification
│       ├── jpa-patterns/
│       ├── kafka-patterns/
│       ├── pr-review/                # Multi-agent PR review
│       ├── python-development/       # FastAPI, pytest, pytest-bdd
│       ├── review/                   # Multi-agent local review
│       ├── review-ai-utility/        # Self-review for this repo
│       ├── temporal-developer/
│       │   ├── SKILL.md
│       │   └── references/           # 14 reference docs for Temporal
│       ├── terraform/                # AWS + Azure infrastructure patterns
│       └── testing/                  # Combined: BDD Cucumber + component tests
├── .github/
│   ├── prompts/                      # 15 GitHub Copilot prompt files
│   └── workflows/
│       └── sync-ai-config.yml        # Auto-sync to downstream repos
├── .opencode/
│   ├── commands/                     # 15 OpenCode command wrappers
│   └── package.json                  # @opencode-ai/plugin dependency
├── .gitignore
├── CODEOWNERS
└── README.md
```

## Contributing

### Adding a new skill

Every skill requires three files -- one for each AI tool. All three must be created together.

**Step 1: Create the Claude Code skill (source of truth)**

Create `.claude/skills/<skill-name>/SKILL.md` with YAML frontmatter:

```markdown
---
name: <skill-name>
description: "Short description (under 300 chars) -- used by Claude to decide when to activate"
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

Note: OpenCode commands have NO frontmatter -- just plain markdown.

**Step 4: Update this README**

Add the new skill to the appropriate table in the Skills section.

**Step 5: Validate**

Run the `review-ai-utility` skill to check for broken links, missing files, and sync issues:

```
> /review-ai-utility
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

### Updating the review agents

Both `pr-review` and `review` define numbered agents that reference other skills. If you add a new skill that should be part of code reviews:

1. Add a new agent block in both `.claude/skills/pr-review/SKILL.md` and `.claude/skills/review/SKILL.md`
2. Specify when the agent applies (e.g., "when `.py` files are changed")
3. List the checks it performs referencing the new skill
4. Keep agent lists consistent between both review skills

### Conventions

- **Skill names**: lowercase, kebab-case (e.g., `java-development`, `kafka-patterns`)
- **Single source of truth**: All skill content lives in `.claude/skills/`. Copilot and OpenCode files only reference it
- **No duplication**: If two skills overlap significantly, combine them (e.g., `java-coding-standards` + `springboot-patterns` = `java-development`)
- **Trailing newlines**: All files must end with a trailing newline
- **Data-backed reviews**: The `review` and `pr-review` skills require evidence for every comment -- no subjective opinions without proof
- **PII/PHI scanning**: Every review must check for personally identifiable information exposure
