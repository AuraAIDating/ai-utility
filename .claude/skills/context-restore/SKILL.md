---
name: context-restore
description: "Restore saved working context for Polaris services. Loads the most recent context saved by /context-save and presents it so you can resume work immediately. Reads from .polaris-context/ in the repo. Pair with /context-save. Adapted from GStack /context-restore."
origin: gstack-adapted
---

# Restore Working Context

You are a **Staff Engineer reading a colleague's session notes** to pick up exactly where they left off. Load the most recent saved context, validate it, orient to the current git state, and surface everything needed to resume without losing a beat.

**HARD GATE:** Do NOT make code changes. This skill only reads saved context files and runs read-only git/bash commands.

---

## When to Activate

- "resume"
- "restore context"
- "where was I"
- "pick up where I left off"
- "context restore"
- Starting a new Claude Code session on a feature branch
- Returning after hours or days away from a task

---

## Parse command

- `/context-restore` -> load the most recent saved context (any branch)
- `/context-restore <number>` -> load a specific context by number from the list
- `/context-restore <title-fragment>` -> load context matching that fragment

---

## Restore Flow

### Step 1: Find saved contexts

```bash
ls -lt .polaris-context/*.md 2>/dev/null | head -10
```

If no `.polaris-context/` directory or no files found — see **Graceful Failure Handling** below.

If multiple contexts exist, show the timeline (see **Context Timeline Visualization**) and auto-load the most recent one. Inform the user which was loaded.

---

### Step 2: Load the most recent (or requested) context

Read the target `.md` file completely.

Also extract Spring Boot specific metadata if present:

```bash
echo "=== SERVICE INFO FROM SAVED CONTEXT ==="
grep -i "service:\|spring_boot_version\|java_version" <context-file> | head -5

echo "=== FEATURE AREA ==="
grep -i "feature_area" <context-file>

echo "=== PHI SCAN STATUS ==="
grep -i "scan_status" <context-file>
```

---

### Step 3: Validate the context file

Before presenting, verify the file is well-formed:

```bash
# Check YAML frontmatter exists
head -5 <context-file> | grep -q "^---" || echo "WARNING: Invalid frontmatter"

# Check required sections are present
grep -q "## Remaining work\|## Open questions" <context-file> || echo "WARNING: Missing required sections"

# Check the file is not empty
wc -l <context-file>
```

If validation fails:
```
Context file appears corrupted or incomplete (missing frontmatter or required sections).

Options:
1. Try an older context: /context-restore <N-1>
2. Reconstruct from git: git log --oneline -20
3. Start fresh: /context-save "resuming from git state"
```

---

### Step 4: Present the context

Format the restored context clearly:

```
╔════════════════════════════════════════════╗
║           CONTEXT RESTORED                 ║
╚════════════════════════════════════════════╝

Title:        {title}
Saved:        {timestamp}
Branch:       {branch}
Status:       {status}
Service:      {service}
Feature area: {feature_area}
Java version: {jvm_context.java_version}
Spring Boot:  {jvm_context.spring_boot_version}
PHI scan:     {phi_scan.scan_status}

WHAT WAS BEING WORKED ON
─────────────────────────
{what_being_worked_on}

FILE CHANGES AT SAVE TIME
──────────────────────────
{file changes summary by layer, if present}

DECISIONS MADE
──────────────
{decisions — with rationale and trade-offs}

WHERE TO PICK UP
────────────────
{remaining work — numbered, with priority and effort}

OPEN QUESTIONS / BLOCKERS
──────────────────────────
{open questions with blocker flags and assignees}

NOTES
─────
{notes / gotchas / useful commands}

TEST COVERAGE DELTA
───────────────────
{coverage delta, if present}

GIT STATE AT SAVE TIME
───────────────────────
{git state summary}
```

---

### Step 5: Detect blocked context

If the context `status` is `blocked` OR any open question has `blocker: YES` / `"blocker": true`:

```
WARNING: BLOCKED CONTEXT DETECTED

This work is blocked on:
- {list all items with blocker: true/YES, with assignee}

Action required:
- Reach out to assignee for update
- Check if blocker was resolved while you were away
- Consider pausing and working on an unblocked item instead

See "Open questions / blockers" section above.
```

---

### Step 6: Orient to current git state

Check the current git state and compare against what was saved:

```bash
echo "=== CURRENT BRANCH ==="
git rev-parse --abbrev-ref HEAD 2>/dev/null

echo "=== CURRENT STATUS ==="
git status --short 2>/dev/null

echo "=== COMMITS SINCE SAVE ==="
git log --oneline --since="{save timestamp}" 2>/dev/null | head -10
```

**Scenario 1: Branch was deleted or no longer exists**
```bash
git rev-parse --verify $SAVED_BRANCH 2>/dev/null || echo "WARNING: Saved branch '$SAVED_BRANCH' no longer exists"
git log --oneline --all --graph | grep -i "$(echo $SAVED_BRANCH | sed 's/.*\///')" | head -5
```
-> "Branch `{saved branch}` was deleted. Your work may have been merged or rebased. Check git log --all."

**Scenario 2: New commits appeared on the saved branch**
```bash
NEW_COMMITS=$(git log --oneline $SAVED_BRANCH --not origin/main 2>/dev/null | wc -l)
# If NEW_COMMITS > 5, warn
```
-> "WARNING: {N} new commits on saved branch since context was saved. Review: git log --oneline {timestamp}..HEAD"

**Scenario 3: More uncommitted changes now than at save time**
```bash
CURRENT_CHANGES=$(git status --short | wc -l)
# Compare against uncommitted_changes_count from saved context
```
-> "WARNING: More uncommitted changes now than at save time. Did you stash something? Run: git stash list"

If current branch differs from saved branch:
-> "Note: Context was saved on `{saved branch}`. You are currently on `{current branch}`."

---

### Step 7: Check for investigation context

If the saved context contains "investigation", "root cause", "investigate", or "blocked by bug":

```
INVESTIGATION CONTEXT FOUND

It looks like you were debugging something when this was saved.

Last known issue:          {pulled from context}
Investigation phase:       {pulled from context}
Root cause hypothesis:     {pulled from context}
Fix applied (if any):      {pulled from context}

Resume debugging with: /investigate
Or jump to the fix:    {remaining work item #1}
```

---

### Step 8: Suggest test execution before resuming

If remaining work involves code changes, extract and suggest the test command:

```bash
TESTS=$(grep -i "gradlew.*test\|test --tests" <context-file> | head -3)
```

If a test command is found:
```
Before resuming, run tests to confirm current state:
  {extracted test command}

Or run the full suite:
  ./gradlew test
```

---

### Step 9: Recommend next action (smart)

Based on remaining work, current git state, and blocker status:

```
RECOMMENDED NEXT ACTION
────────────────────────

Immediate next step:
  {Priority 1 remaining work item}

Why this one?
  - Highest priority ({N}/10)
  - Estimated effort: {effort}
  - Blocked: {YES/NO}

Before you start:
  1. Run tests: {test command from saved context}
  2. Check if you need: {dependency or open question}
  3. Reference: {related skill or ADR if mentioned}
```

If all remaining work is blocked, suggest:
-> "All remaining items are blocked. Resolve open questions first, or use `/context-save` to save a blocking note and switch tasks."

---

### Step 10: Surface dependency impact

If the saved context contains a "Multi-Service Context" or "Contract changes" section:

```
DEPENDENCY & IMPACT NOTICE
───────────────────────────

This feature may affect:
- {service name}: {reason}
- {service name}: {reason}

Contract changes to coordinate:
- {event or class change}

Deployment order:
1. {service A}
2. {service B}

Action: Coordinate with affected service owners before merging.
```

---

### Step 11: Show velocity metrics (if available)

If the saved context contains a `metrics` block:

```
VELOCITY & PROGRESS METRICS
─────────────────────────────

Since last save:
  Lines added:          {N}
  Lines removed:        {N}
  Files modified:       {N}
  Test coverage change: {delta}

Estimated effort remaining:
  {sum of estimated_effort from remaining_work items}
```

---

## Context Timeline Visualization

When multiple contexts exist, show a timeline before loading:

```
TIMELINE OF RECENT WORK (last 30 days)

#  TIMESTAMP            STATUS           BRANCH                          TITLE
1  2026-04-29 14:30     IN-PROGRESS      feat/order-confirmation         "add order confirmation template"
2  2026-04-28 09:15     MERGED           fix/smtp-retry                  "smtp transient retry fix"
3  2026-04-25 16:00     BLOCKED          feat/avro-v2-migration          "avro schema v2 migration"
4  2026-04-20 11:00     MERGED           feat/spring-52-upgrade          "upgrade to Spring Boot 3.2"
5  2026-04-10 08:00     IN-PROGRESS      feat/temporal-saga              "temporal saga pattern for orders"

Status key: IN-PROGRESS | MERGED | BLOCKED | READY-FOR-REVIEW

Loading #1 (most recent). Type /context-restore <N> to load a different one.
```

---

## Investigation Resume Pattern

If the restored context contains investigation notes, follow this workflow:

```
HOW TO RESUME THIS DEBUGGING SESSION
──────────────────────────────────────

1. Review what was found:
   Last issue:        {last blocker encountered}
   Root cause found:  {yes/no + description}
   Fix applied:       {yes/no + description}

2. Check if fix was committed:
   git status  (look for uncommitted changes)
   git log --oneline -5

3. Run the confirming test:
   {test command from context}

4. If test passes  -> Move to next remaining work item
   If test fails   -> Resume with /investigate
   Fix not applied -> Apply fix, then run tests

Common patterns:
- Fix applied but not committed? -> git add . && git commit -m "fix: {description}"
- Different bug found during fix? -> /context-save "investigation pivot: {new issue}" then /investigate
- Root cause still unclear?      -> /investigate (will pick up from symptoms in this context)
```

---

## Graceful Failure Handling

### No context found

```
No saved contexts found in .polaris-context/

This can happen if:
- /context-save has not been run yet
- You are on a different branch than where you saved

Current branch:
  git rev-parse --abbrev-ref HEAD

All local branches:
  git branch --list

Options:
1. Switch to your feature branch and retry: /context-restore
2. Create a fresh context: /context-save "starting: {description}"
```

### Context not found by search term

```
No context found matching "{search-term}".

Available contexts:
{numbered list of all .polaris-context/*.md files}

Options:
1. Try /context-restore <number>
2. Try /context-restore with a different title fragment
3. Create a fresh context: /context-save "resuming: {description}"
```

### Corrupted context file

```
WARNING: Context file appears corrupted (missing frontmatter or required sections).

Fallback options:
1. Try an older context:   /context-restore <N-1>
2. Reconstruct from git:   git log --oneline -20
3. Start fresh:            /context-save "resuming from git state"
```

### Branch deleted but commits exist (merged or rebased)

```
WARNING: Branch '{saved branch}' was deleted, but related commits exist.

Your work was likely merged or rebased. Check:
  git log --all --oneline | grep "{keywords from context title}"

If merged to main:
  -> Your work is done. Archive this context:
     mkdir -p .archives && mv .polaris-context/<file>.md .archives/

If rebased onto a new branch:
  -> Switch to the new branch and resume:
     git checkout {new-branch}
     /context-restore
```

---

## Context Lifecycle Reminder

- Contexts older than 7 days on merged branches can be deleted:
  `rm .polaris-context/<old>.md`
- Archive before deleting:
  `mkdir -p .archives && mv .polaris-context/<old>.md .archives/`
- When resuming after 3+ days, always check what changed:
  `git log --oneline --since="3 days ago"`

---

## Integration Notes

- Pair with `/context-save` to save session state before switching tasks.
- If restored context shows a debugging session was in progress, use `/investigate` to resume.
- PHI safety: this skill is read-only. No patient data is written or transmitted. Context files stay local.
