---
name: pr-summary
description: Generate a pull request description from the current branch changes. Analyzes commits and diffs to create a structured PR summary following project conventions. Accepts optional base branch as argument. Use whenever the user wants to create a PR, write a PR description, prepare changes for review, or mentions 'PR summary', 'pull request', 'PR description', or 'prepare for merge'.
allowed-tools: Agent
argument-hint: <base-branch>
model: haiku
context: fork
---

# PR Summary


Delegate the PR creation workflow to the `pr-summary-agent` agent, which is specialized in analyzing branches and creating well-structured PRs.



## Process

### Step 1: Parse arguments

- If a base branch is provided as argument, use it
- Otherwise, default to `master`

### Step 2: Build context summary


Before spawning the agent, check if the current conversation has context about what was changed. If you have been working on changes and commits in this conversation, build a **context summary**:

```
Context: {brief description of all work done on this branch}
Key commits: {list of commit messages if known}
Files changed: {list of key files modified}
```

If you do NOT have context, omit the summary and let the agent do full discovery.


### Step 3: Launch pr-summary-agent


Use the Agent tool to spawn the `pr-summary-agent` agent with this prompt:

**If changes are not yet committed:**

```
There are uncommitted changes on this branch. Tell the user to commit first (suggest /commit) and stop. Do not commit on their behalf. Do not create a PR.
```

**If changes are committed — with context summary (preferred):**

```
Create a pull request for the current branch against {base branch}.

{context summary}

Use this context to draft the PR title and body. Run only git branch --show-current and git status --short to confirm state. Do not run git log or git diff --stat.
```

**If changes are committed — without context (triggers Full Discovery Mode with user confirmation):**

```
Create a pull request for the current branch against {base branch}.
Use git log and git diff --stat against the base branch to generate the PR title and body.
Do not inspect README or unrelated files.
```

Always include `subagent_type: "pr-summary-agent"` and `model: "haiku"` when spawning the agent.




### Step 4: Report result

When the agent completes, show:

- PR URL
- PR title
- Brief summary of what was included

## Important

- Never auto-create the PR without user confirmation (the agent will ask)
- If there are visual changes, remind the user to add screenshots after creation
- Keep the summary concise — avoid repeating the full diff
- Do not inspect every changed file; prefer `--stat` and targeted inspection only for ambiguous areas