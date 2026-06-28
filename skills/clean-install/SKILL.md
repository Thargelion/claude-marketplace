---
name: clean-install
description: Automated clean install, build, test and lint verification process. Accepts optional arguments: --skip-<step> to skip a step, --only <steps> to run specific steps, --legacy-peer-deps or --npm <flags> for npm install flags, --verbose for full output. Use whenever the user wants to do a fresh install, reinstall dependencies, verify the project builds, run a full check, start from scratch, or mentions 'clean install', 'fresh install', 'new version', 'reinstall node_modules', 'check if everything works', 'reinstall packages', 'rebuild the project', 'run npm install from scratch', or 'full verification'.
allowed-tools: Bash(npm *), Bash(npx *)
model: haiku
context: fork
---

# New Version / Clean Install

## Arguments

Parse arguments before executing any step. Arguments can be combined.

| Argument | Description | Example |
| --- | --- | --- |
| `--skip-<step>` | Skip a specific step | `--skip-build`, `--skip-test`, `--skip-lint`, `--skip-clean` |
| `--only <steps>` | Run only the specified steps (comma-separated) | `--only install,test` |
| `--legacy-peer-deps` | Pass `--legacy-peer-deps` to `npm install` | `/clean-install --legacy-peer-deps` |
| `--npm <flags>` | Pass extra flags to `npm install` | `--npm --force` |
| `--verbose` | Show full command output instead of summaries | |

**Parsing rules:**
- `--skip-*` and `--only` are mutually exclusive — if both are provided, `--only` takes precedence
- Valid step names: `clean`, `install`, `build`, `test`, `lint`
- Unrecognized arguments: warn the user and continue with defaults
- No arguments: run all steps with defaults

## Process

Execute the following steps **sequentially**, skipping any steps excluded by arguments. If any step fails, stop and ask the user if they want to fix the error before continuing.

### Step 1: Clean

> Skip if: `--skip-clean` or `--only` does not include `clean`

```bash
npm run clean && npm cache clean --force
```

- If `npm run clean` is not defined in package.json, fall back to `rm -rf node_modules` and `npm cache clean --force`
- If fails, report error and ask whether to continue

### Step 2: Install

> Skip if: `--skip-install` or `--only` does not include `install`

```bash
npm install [--legacy-peer-deps] [extra flags from --npm]
```

- Append `--legacy-peer-deps` if that argument was passed
- Append any flags from `--npm <flags>` if provided
- Report package count and relevant warnings
- If fails (ETARGET, ERESOLVE, etc.), ask if the user wants to attempt a fix

### Step 3: Build

> Skip if: `--skip-build` or `--only` does not include `build`

```bash
npm run build
```

- If `build` script is not defined in package.json, skip silently and note it in the summary
- If TypeScript compilation errors, list them and ask whether to fix

### Step 4: Tests

> Skip if: `--skip-test` or `--only` does not include `test`

```bash
npm run test
```

- If tests fail, list which ones and ask whether to fix

### Step 5: Lint

> Skip if: `--skip-lint` or `--only` does not include `lint`

```bash
npm run lint
```

- If lint errors, list the most critical and ask whether to fix

## Summary

After all steps, generate a summary table showing only the steps that ran:

| Step    | Status      | Details                |
| ------- | ----------- | ---------------------- |
| Clean   | ✅/❌/⏭️ skipped | ...               |
| Install | ✅/❌/⏭️ skipped | X packages, Y warnings |
| Build   | ✅/❌/⏭️ skipped | ...               |
| Test    | ✅/❌/⏭️ skipped | X passed, Y failed |
| Lint    | ✅/❌/⏭️ skipped | X errors, Y warnings |

If errors occurred, ask: "Do you want me to fix the errors in [step]?"

## Rules enforced

- Sequential execution with error gates — each step must pass before proceeding
- No verbose output unless explicitly requested — keeps output clean and focused
- User confirmation before any fix attempt — never auto-fix without consent

## Notes

- Run all commands in the project root directory
- Do not use `--verbose` in npm install (only if user explicitly requests it)
- Fix blocking errors before continuing to next steps