---
name: code-review
description: "PR code review skill. Retrieves diffs, reads full file context, posts inline review comments at specific line numbers on GitHub PRs. Asks for user permission before posting. Tone is polite, constructive, and grounded in evidence."
---

# Code Review Skill

Review pull requests and post constructive, evidence-based inline comments at the exact lines where issues occur.

## When to Activate

- User asks to "review a PR", "review pull request", "code review", or "review changes"
- User provides a PR URL or PR number for review
- User runs a review command targeting a branch, commit, or PR

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

**Diffs alone are not enough.** After getting the diff, read the entire file(s) being modified to understand the full context. Code that looks wrong in isolation may be correct given surrounding logic — and vice versa.

- Use the diff to identify which files changed
- Use `git status --short` to identify untracked files, then read their full contents
- Read the full file to understand existing patterns, control flow, and error handling
- Check for existing style guide or conventions files (CONVENTIONS.md, AGENTS.md, .editorconfig, etc.)

### Step 3: Analyze Changes

**What to look for:**

**Bugs** — Primary focus.
- Logic errors, off-by-one mistakes, incorrect conditionals
- If-else guards: missing guards, incorrect branching, unreachable code paths
- Edge cases: null/empty/undefined inputs, error conditions, race conditions
- Security issues: injection, auth bypass, data exposure, credential leaks
- Broken error handling that swallows failures, throws unexpectedly, or returns error types that are not caught

**Structure** — Does the code fit the codebase?
- Does it follow existing patterns and conventions?
- Are there established abstractions it should use but doesn't?
- Excessive nesting that could be flattened with early returns or extraction

**Performance** — Only flag if obviously problematic.
- O(n²) on unbounded data, N+1 queries, blocking I/O on hot paths

**Behavior Changes** — If a behavioral change is introduced, raise it (especially if it's possibly unintentional).

### Step 4: Validate Findings

**Be certain before flagging.** If you are going to call something a bug, you need to be confident it actually is one.

- Only review the changes — do not review pre-existing code that wasn't modified
- Don't flag something as a bug if you're unsure — investigate first
- Don't invent hypothetical problems — if an edge case matters, explain the realistic scenario where it breaks
- If you need more context to be sure, use tools to get it (explore codebase, read related files, search for usage patterns)

**Don't be a zealot about style.**

- Verify the code is *actually* in violation of project conventions
- Some "violations" are acceptable when they're the simplest option
- Don't flag style preferences as issues unless they clearly violate established project conventions

### Step 5: Present Findings to User

**Always present findings to the user first.** Show a summary table of all issues found with:
- Severity (Critical, Medium, Low)
- File and line number
- Brief description of the issue

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
2. For new files: the hunk header line in the diff output + offset gives the file line number. Count `+` lines from the start of the hunk to find the file line.
3. For modified files: use the new file line numbers from the `+` side of the hunk header.

**Posting the review:**
Use the GitHub API to create a PR review with inline comments:

```bash
# For github.com
gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input review.json

# For GitHub Enterprise
gh api --hostname {ghe-host} repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input review.json
```

The review JSON structure:

```json
{
  "body": "",
  "event": "COMMENT",
  "comments": [
    {
      "path": "relative/file/path.java",
      "line": 42,
      "body": "Comment text here"
    }
  ]
}
```

Write the JSON to a temp file and use `--input` to avoid shell escaping issues.

**Detecting GitHub Enterprise:** If the PR URL contains a hostname other than `github.com`, use `gh api --hostname <host>` and extract owner/repo from the URL. Run `gh auth status` if unsure which hosts are configured.

## Comment Tone and Style

**Be polite, constructive, and evidence-based.** Every comment should:

1. **State the severity clearly** — Use labels: `Critical`, `Medium`, `Low` at the start of the comment.
2. **Explain *what* is wrong** — Describe the issue concisely so the reader can understand without reading too closely.
3. **Explain *why* it matters** — Provide the concrete scenario, input, or environment where the issue manifests. Don't just say "this is wrong" — show why.
4. **Suggest a fix** — Offer a specific recommendation or alternative approach when possible.
5. **Be honest about uncertainty** — If you're not sure, say "This may be intentional, but..." or "Consider whether..." rather than presenting it as a definite bug.
6. **Respect the author's intent** — Use phrasing like "Consider using...", "This could be improved by...", "It might be worth..." rather than demanding changes.

**Avoid:**
- Flattery or filler ("Great job!", "Thanks for this!")
- Accusatory tone ("You forgot to...", "You should have...")
- Overstating severity — a style nit is not a critical bug
- Vague complaints without evidence — always cite the specific code, pattern, or constraint that supports your point

**Example good comment:**
> **Medium**: `dateOfService` is mapped as `String`, which bypasses date validation and allows malformed values (e.g., `"not-a-date"`) to be persisted. If the database column is a date type, Hibernate will throw a runtime exception on insert. Consider using `LocalDate` instead, which gives you type safety and enables date range queries at the database level.

**Example bad comment:**
> This should be LocalDate not String.

## Edge Cases

- **GitHub Enterprise**: Always check `gh auth status` and use `--hostname` flag for GHE instances
- **Large PRs**: If the diff is truncated, read the full output file and use offset/limit or grep to find specific sections
- **Shell escaping**: Write review JSON to a temp file instead of trying to inline it in the command. Use `--input /tmp/review.json` to avoid quoting issues with special characters in comments.
- **Mixed file types**: Only review code files. Skip reviewing generated files, lock files, or binary assets unless they indicate a problem (e.g., committed secrets).
