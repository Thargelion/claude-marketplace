---
name: commit
description: Commit, push, and create PRs using conventional commits. Use whenever the user wants to commit changes, push code, create a pull request, or says things like 'commit this', 'push my changes', 'make a PR', 'save my work', 'commit and push'. Accepts optional arguments like 'and push', 'and create PR', or a custom commit message.
allowed-tools: Agent
argument-hint: [and push] [and create PR] [custom message]
model: haiku
---

# Commit



Delegate the git commit workflow to the `git-committer-agent` agent (commit + push), and optionally chain the `pr-summary-agent` for PR creation.




## Process

### Step 1: Parse arguments

Check if the user provided additional instructions as arguments:

- **"and push"** or **"y pushea"** — commit and push
- **"and create PR"** or **"y crea PR"** — commit, push, and create PR
- **A custom message** — use it as the commit message (skip message generation)
- **No arguments** — just commit, ask about push/PR

### Step 2: Build context summary



Before spawning the agent, check if the current conversation has context about what was changed. If you have been working on changes in this conversation (editing files, writing code, fixing bugs), build a **context summary** to pass to the agent. This avoids redundant git discovery.

**Context summary format:**

```
Context: {brief description of what was done}
Files changed: {list of files modified in this session}
Suggested commit type: {feat|fix|refactor|test|chore|docs|style|ci|perf|a11y}
Suggested scope: {optional scope}
```

If you do NOT have context (e.g., user just said `/commit` without prior work in the conversation), omit the context summary and let the agent do full discovery.



### Step 3: Launch git-committer-agent



Use the Agent tool to spawn the `git-committer-agent` agent with the appropriate prompt based on the parsed arguments.

**With context summary (preferred — saves 4-5 tool calls):**

```
{context summary}

Stage all changed files, generate a conventional commit message based on the context above, and commit. Run only git status --short to confirm files. Do not run git diff --stat, git log, or guardrails checks. Do not push or create a PR.

CRITICAL: If the pre-commit hook fails due to lint errors, stop immediately. Do NOT try to fix lint errors yourself. Do NOT use sed, awk, Edit, or Write tools to modify files. Report the exact hook error output and stop.
```

**Without context (cold start — no context summary triggers Full Discovery Mode with user confirmation):**

```
Analyze the current changes with git status and git diff --stat / git diff --cached --stat, generate a conventional commit message, and commit. Prefer a single commit unless the stat output clearly shows unrelated work. Do not inspect README or unrelated files. Do not push or create a PR.

CRITICAL: If the pre-commit hook fails due to lint errors, stop immediately. Do NOT try to fix lint errors yourself. Do NOT use sed, awk, Edit, or Write tools to modify files. Report the exact hook error output and stop.
```

**With custom message:**

```
Stage and commit the current changes with this message: "{message}". Run only git status --short to confirm files. Do not run git diff, git log, or guardrails checks. Then push to remote if requested.

CRITICAL: If the pre-commit hook fails due to lint errors, stop immediately. Do NOT try to fix lint errors yourself. Do NOT use sed, awk, Edit, or Write tools to modify files. Report the exact hook error output and stop.
```

For **push** variants, append: `After committing, push to remote.`

Always include `subagent_type: "git-committer-agent"` and `model: "haiku"` when spawning the agent.

### Step 3b: Pre-commit hook failure recovery (lint errors)

If the `git-committer-agent` reports that the commit failed due to a **pre-commit hook / lint error**:

1. **First: spawn the `linter-agent`** to auto-fix the issues:

   ```
   Run `npx prettier --write` and `npx eslint --fix` on these specific files to fix lint errors:
   {list from context summary or git status}

   Do NOT read files, analyze code, or make manual edits. Only run the auto-fix commands on the listed files, then stage them with `git add`. Stop after staging.
   ```

   Always include `subagent_type: "linter-agent"` and `model: "haiku"`.

2. **If the linter-agent succeeds**, re-run `git-committer-agent` with the same original prompt + append: `Files were already lint-fixed. Re-stage and commit directly without running lint again.`

3. **If the linter-agent fails or reports unfixable errors**, spawn the `general-purpose` agent:
   ```
   The pre-commit hook failed and auto-lint could not fix it. Investigate and fix the root cause of the following lint/build errors, then re-stage the files.
   Errors: {error output from linter-agent}
   Files: {list of files}
   NEVER use sed, awk, cat > /tmp, or /tmp scripts to edit files. Use the Edit tool instead.
   ```
   After the general-purpose agent fixes the issues, re-run `git-committer-agent` to complete the commit.

> Note: Step 3b is a recovery step — only trigger it if the git-committer-agent explicitly reported a failure.

### Step 3c: Launch pr-summary-agent (only if PR was requested)

After the git-committer-agent completes successfully (commit + push done), spawn the `pr-summary-agent`:

**With context summary (preferred):**

```
Create a pull request for the current branch against {base branch, default: master}.

Context: {same context summary from the commit step}
Commits: {commit messages from the git-committer result}

Use this context to draft the PR title and body. Run only git branch --show-current and git status --short. Do not run git log or git diff --stat.
This is a chained invocation — create the PR directly without asking for confirmation.
```

**Without context:**

```
Create a pull request for the current branch against {base branch, default: master}.
Use git log and git diff --stat against the base branch to generate the PR title and body.
Do not inspect README or unrelated files.
This is a chained invocation — create the PR directly without asking for confirmation.
```

Always include `subagent_type: "pr-summary-agent"` and `model: "haiku"` when spawning the agent.





### Step 4: Report result

When the agent completes, summarize what happened:

- What was committed (files, message)
- Whether it was pushed (and to which branch)
- PR URL if one was created