---
name: lint
description: Run the linter or fix linting and formatting issues. Use whenever the user asks to check lint, fix lint errors, clean up code style, run prettier, fix formatting, run eslint, fix style issues, format files, or when pre-commit hooks fail due to linting. Also use when the user says 'run lint', 'fix the lint', 'arreglá el linter', 'format the code', 'check the code style', 'corré el linter', 'eslint', 'there are style issues', or 'the pre-commit hook failed'. Accepts 'fix' as argument to auto-fix, or file paths to scope.
allowed-tools: Bash(npm run lint*), Bash(npx eslint *), Bash(npx stylelint *), Agent
argument-hint: [fix] [file paths]
model: haiku
---

# Lint


Run the linter to check for errors, or fix them automatically using a fast agent.




## Modes

This skill has two modes based on the arguments:

- **Check mode** (`/lint` or `/lint <files>`) — just run the linter and report errors. No changes.

- **Fix mode** (`/lint fix` or `/lint fix <files>`) — spawn a haiku agent to fix blocking errors first, then optionally fix safe warnings, and verify with tests.



## Check Mode

When the user wants to just see errors without fixing:

### Step 1: Run the linter

Prefer the narrowest scope available. If the user gave file paths, lint only those files. Only run project-wide lint when no narrower scope exists.

```bash
npm run lint 2>&1
```

If specific files were provided:

```bash
npx eslint {files} 2>&1
```

For SCSS files:

```bash
npx stylelint {scss-files} 2>&1
```

### Step 2: Report

Present the results clearly:

- Total errors and warnings
- Group by file for readability
- If zero issues: report success
- If there are errors: suggest running `/lint fix` to clear the blocking issues first
- If there are only warnings: ask whether the user wants to fix them, and mention if any look trivially safe to auto-fix

## Fix Mode

When the user wants to fix lint issues (argument contains "fix", "arreglá", "fixeá", or context is a pre-commit failure):

### Step 1: Launch linter-agent


Use the Agent tool with `subagent_type: "linter-agent"` and `model: "haiku"`.

**Fix all blocking lint errors first:**

```
Fix the current blocking lint errors with the smallest safe scope. Start by identifying the failing files, then run eslint --fix / prettier / stylelint only on those files. Only use a project-wide fix if no narrower scope is available. Manually fix any remaining errors. If only warnings remain, stop and report them unless they are trivially safe mechanical fixes (for example `jest/prefer-strict-equal` or import/order style cleanup). Report which files you modified and whether any errors or warnings remain.
```

**Fix specific files:**

```
Fix the blocking lint errors in these files: {file list}. Run eslint --fix and prettier / stylelint only on those files first, then manually fix any remaining errors. If only warnings remain, stop and report them unless they are trivially safe mechanical fixes. Report which files you modified and whether any errors or warnings remain.
```

**After pre-commit failure:**

```
The pre-commit hook failed with these linting errors. Fix them all:

{paste hook output}

Infer the failing files from the hook output and fix only those files. Run eslint --fix first, then manually fix remaining errors. Treat warnings as optional unless they are trivial and safe. Report which files you modified.
```

### Step 2: Run tests and fix if needed

After the linter agent completes, if it modified any files, spawn the `test-engineer-agent` to run related tests and fix any failures:

```
Run the tests related to these files that were just modified by the linter: {list of modified files}. If any tests fail because of the lint changes, fix them. Report the final state.
```

Lint fixes can accidentally change behavior — removing an "unused" variable that was actually used, changing equality semantics with `===`, or reordering code. The test-engineer-agent catches and fixes these regressions.




### Step 3: Report result

Summarize:

- How many lint errors were fixed (auto-fix vs manual)
- How many warnings were fixed automatically vs left pending
- Which files were modified by the linter
- Whether tests passed, and if the test-engineer had to fix any
- Any remaining lint errors or warnings the agent couldn't fix or intentionally left pending