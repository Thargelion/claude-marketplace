---
name: create-branch
description: Create a branch and push to origin. Supports KMSQ ticket branches (feature/KMSQ-123-desc) and custom branches (fix/, hotfix/, chore/, feature/). Use whenever the user wants to create a branch, start working on a ticket, begin a new feature, make a fix branch, or mentions a KMSQ ticket number.
allowed-tools: Bash(git *), AskUserQuestion
model: haiku
---

# Create Branch

Creates a branch and pushes it to origin. Supports two modes:

- **Ticket branch**: `feature/KMSQ-{ticket}-{description}` (from Jira ticket)
- **Custom branch**: `{type}/{description}` (for work without a ticket)

## Process

### Step 1: Determine mode

Check the user's arguments to decide which mode to use:

**Direct branch format** — If the argument starts with `fix/`, `hotfix/`, `chore/`, `feature/`, `refactor/`, or `release/`, use it as-is (skip to Step 3). Just sanitize the description part (lowercase, hyphens only).

**Ticket reference** — If the argument looks like a ticket number (e.g. `KMSQ-435`, `435`), use Ticket mode.

**No arguments** — Ask the user:

```
How do you want to create the branch?
1. From ticket (KMSQ-1234)
2. Custom branch (type/description)
```

### Step 2: Gather information

#### Ticket mode

1. **Ticket number** — If not provided, ask: "What is the KMSQ ticket number? (just the number, e.g. 435)"
2. **Description** — If not provided, suggest one based on the conversation context. Ask the user to confirm or provide their own.
3. **Format**: `feature/KMSQ-{ticket}-{description}`

#### Custom mode

1. **Type** — Ask the user to pick:
   - `feature` — new feature or capability
   - `fix` — bug fix
   - `hotfix` — urgent production fix
   - `chore` — maintenance, config, dependencies
   - `refactor` — code restructuring
   - `release` — release branch
2. **Description** — Ask for a short description (e.g. "update-auth-flow")
3. **Format**: `{type}/{description}`

### Step 3: Validate

- Ticket number (if applicable) must be numeric
- Description must be lowercase, using only `a-z`, `0-9`, and hyphens. If the user provides something else, auto-convert it (lowercase, replace spaces/underscores with hyphens, strip invalid characters)
- Branch name must not be empty after sanitization

### Step 4: Create Branch

```bash
git checkout -b {branch-name}
```

- If the branch already exists, inform the user and ask how to proceed

### Step 5: Push to Origin

```bash
git push --set-upstream origin {branch-name}
```

- If `origin` is not configured, inform the user and skip this step

### Step 6: Confirm

Report the result:

```
Branch created and pushed: {branch-name}
```

## Rules enforced

- Ticket branches follow `feature/KMSQ-{ticket}-{description}` convention
- Custom branches follow `{type}/{description}` convention
- Auto-sanitization of description (lowercase, hyphens only)
- User confirmation before push — never push without consent