---
name: review-ai-utility
description: "Review the ai-utility repo itself — validates skill/prompt/command sync, checks for broken references, ensures consistency across Claude Code, GitHub Copilot, and OpenCode. Works on local uncommitted changes or a PR link."
---

# Review AI Utility Skill

Self-review skill for the ai-utility repository. Validates that all AI skills, GitHub Copilot prompts, and OpenCode commands are consistent, correctly linked, and follow project conventions.

## When to Activate

- User asks to "review the utility repo", "check skills", or "validate ai config"
- User runs `/review-ai-utility`
- User provides a PR link for this repo
- Before committing changes to this repo

## Workflow

### Step 1: Determine Review Scope

Based on user input:

1. **No arguments (default)**: Review all uncommitted changes
   - Run: `git diff` for unstaged changes
   - Run: `git diff --cached` for staged changes
   - Run: `git status --short` to identify untracked files

2. **PR URL or number**: Review the pull request
   - Run: `gh pr view <input>` to get PR context
   - Run: `gh pr diff <input>` to get the diff

### Step 2: Read Full Context

- Read all changed files fully — diffs alone are not enough
- Read `.claude/skills/*/SKILL.md` for all skills
- Read `.github/prompts/*.prompt.md` for all prompts
- Read `.opencode/commands/*.md` for all commands
- Read `README.md` for documentation accuracy

### Step 3: Run Validation Checks

#### Check 1: Skill/Prompt/Command Sync

Every skill directory in `.claude/skills/<name>/` MUST have:
- A corresponding `.github/prompts/<name>.prompt.md`
- A corresponding `.opencode/commands/<name>.md`

And vice versa — no orphaned prompts or commands without a matching skill.

**How to check:**
```bash
# List all three
ls .claude/skills/
ls .github/prompts/
ls .opencode/commands/

# Find mismatches
for skill in .claude/skills/*/; do
  name=$(basename "$skill")
  [ ! -f ".github/prompts/${name}.prompt.md" ] && echo "MISSING PROMPT: $name"
  [ ! -f ".opencode/commands/${name}.md" ] && echo "MISSING COMMAND: $name"
done

for f in .github/prompts/*.prompt.md; do
  name=$(basename "$f" .prompt.md)
  [ ! -d ".claude/skills/${name}" ] && echo "ORPHAN PROMPT: $name"
done

for f in .opencode/commands/*.md; do
  name=$(basename "$f" .md)
  [ ! -d ".claude/skills/${name}" ] && echo "ORPHAN COMMAND: $name"
done
```

#### Check 2: Broken Skill References

GitHub prompts and OpenCode commands reference skill files via paths like `.claude/skills/<name>/SKILL.md`. Verify every referenced path exists.

```bash
for f in .github/prompts/*.prompt.md .opencode/commands/*.md; do
  ref=$(grep -oP '\.claude/skills/[^/]+/SKILL\.md' "$f" 2>/dev/null)
  if [ -n "$ref" ] && [ ! -f "$ref" ]; then
    echo "BROKEN: $f -> $ref"
  fi
done
```

#### Check 3: Internal Cross-References

Skills may reference other skills (e.g., "see the `testing` skill"). Verify all such references point to skills that actually exist.

```bash
# Find all skill name references in SKILL.md files
grep -rn 'skill' .claude/skills/*/SKILL.md | grep -oP '`[a-z-]+`\s+skill' | sort -u
```

Cross-check each referenced name against `.claude/skills/`.

#### Check 4: SKILL.md Frontmatter Validation

Every `SKILL.md` must have valid YAML frontmatter with:
- `name` — matches the directory name
- `description` — non-empty, under 300 characters

```bash
for dir in .claude/skills/*/; do
  name=$(basename "$dir")
  file="${dir}SKILL.md"
  if [ ! -f "$file" ]; then
    echo "MISSING SKILL.md: $name"
    continue
  fi
  # Check frontmatter exists
  head -1 "$file" | grep -q '^---' || echo "NO FRONTMATTER: $name"
  # Check name matches directory
  grep -q "^name: ${name}$" "$file" || echo "NAME MISMATCH: $name"
  # Check description exists
  grep -q '^description:' "$file" || echo "NO DESCRIPTION: $name"
done
```

#### Check 5: GitHub Prompt Frontmatter Validation

Every `.prompt.md` must have:
- `mode: agent`
- `description` — non-empty
- `tools` — includes `["codebase", "githubRepo"]`
- Body that references the correct skill path

#### Check 6: OpenCode Command Format Validation

Every `.opencode/commands/<name>.md` must:
- Reference the correct skill path: `.claude/skills/<name>/SKILL.md`
- Include `$ARGUMENTS` placeholder
- Have no YAML frontmatter (plain markdown only)

#### Check 7: Review Skill Content Quality

For each changed or new skill, check:
- **Structure** — Has "When to Activate" or "When to Use" section
- **Actionable content** — Contains code examples, patterns, or checklists (not just vague guidance)
- **No stale references** — URLs, tool versions, and dependency versions are reasonable
- **Consistent tone** — Imperative, concise, pattern-focused

#### Check 8: README Accuracy

If skills were added, removed, or renamed:
- Verify `README.md` lists the correct skill count
- Verify the skills table matches the actual skills
- Verify the project structure tree is accurate
- Verify the sync workflow target repos are up to date

#### Check 9: Workflow File Validation

If `.github/workflows/sync-ai-config.yml` was changed:
- Verify YAML syntax is valid
- Verify `SYNC_DIRS` matches the directories that should be synced
- Verify the workflow does NOT sync `.github/workflows/` to target repos
- Verify the `paths` trigger includes all synced directories

#### Check 10: pr-review and review Agent Consistency

Both `pr-review` and `review` skills should have the same set of agents. Compare agent lists and flag any drift between them.

### Step 4: Present Findings

Present a summary table:

```
| # | Severity | Check                  | Finding                                    |
|---|----------|------------------------|--------------------------------------------|
| 1 | Critical | Sync                   | Missing prompt for skill "new-skill"        |
| 2 | Critical | Broken Link            | graphify.prompt.md -> nonexistent SKILL.md  |
| 3 | Medium   | Cross-Reference        | java-development references "bdd-cucumber"  |
| 4 | Medium   | README                 | Skill count says 15, actual is 12           |
| 5 | Low      | Frontmatter            | testing skill description > 300 chars       |
```

**Severity guidelines:**
- **Critical** — Missing files, broken links, sync mismatches (will cause tool failures)
- **Medium** — Stale references, README drift, content quality issues (misleading but not broken)
- **Low** — Style inconsistencies, minor frontmatter issues (cosmetic)

For each finding:
- Cite the specific file and line number
- Explain why it matters with concrete evidence
- Suggest a specific fix

### Step 5: Suggest Changes

For each finding, propose a concrete fix:
- Missing prompt/command → provide the exact file content to create
- Broken reference → provide the corrected path
- README drift → provide the corrected section
- Stale cross-reference → provide the updated skill name

### Step 6: Ask Permission Before Acting

**CRITICAL: Never commit changes or post PR comments without explicit user permission.**

After presenting findings, ask the user:
- Whether they want to apply the suggested fixes
- Whether they want to commit the changes
- If reviewing a PR: whether they want to post inline comments

Only proceed after receiving explicit confirmation.

### Step 7: Post Inline Comments (PR only)

When posting comments on a PR, **always use inline comments at the exact line numbers**.

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input review.json
```

```json
{
  "body": "AI Utility Review - X issues found",
  "event": "COMMENT",
  "comments": [
    {
      "path": ".claude/skills/new-skill/SKILL.md",
      "line": 3,
      "body": "**Medium** (Frontmatter): The `name` field is `new_skill` but the directory is `new-skill`. These must match for Claude Code to load the skill correctly.\n\nSuggested fix: Change line 3 to `name: new-skill`"
    }
  ]
}
```

Write JSON to a temp file and use `--input` to avoid shell escaping issues.

## Comment Tone and Style

- **Be polite and constructive** — "Consider updating..." not "This is wrong"
- **Use data-backed arguments** — cite the specific file, line, and why it matters (e.g., "This reference points to `bdd-cucumber` which was merged into `testing` in commit abc123")
- **State severity clearly** — `**Critical**`, `**Medium**`, `**Low**`
- **Suggest specific fixes** — include the exact corrected content, not just "fix this"
- **Respect intent** — if a pattern is intentional, acknowledge it: "This may be intentional, but the convention elsewhere is..."
- **Reference evidence** — "The other 11 skills follow the pattern X, this one deviates by Y"
