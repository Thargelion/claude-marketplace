---
name: check-deps
description: Check dependencies for security vulnerabilities, deprecations, and issues. Uses the codeguard MCP when available, falls back to manual checks. Use whenever the user mentions 'check dependencies', 'audit deps', 'dependency vulnerabilities', 'check vulns', 'scan dependencies', 'are my deps safe', 'check packages', or 'dependency audit'. Accepts optional package names as argument.
allowed-tools: Bash(git *), Bash(npm *), Bash(npx *), Bash(go *), Bash(pip *), Read, Grep, Glob, Task
argument-hint: <package names or empty for full audit>
context: fork
---

# Check Dependencies

Validate project dependencies for security vulnerabilities and issues.

## Process

### Step 1: Detect stack and gather dependencies

Determine the project stack and extract dependencies to check:

- **Specific packages provided as argument**: check only those
- **No argument**: detect manifest and extract all direct dependencies

| Manifest | Stack | Ecosystem |
|----------|-------|-----------|
| `package.json` | node | npm |
| `go.mod` | go | go |
| `requirements.txt` / `pyproject.toml` | python | pypi |
| `pom.xml` | java | maven |
| `build.gradle` / `build.gradle.kts` | kotlin | maven |

Read the manifest file and extract dependencies with their versions.

For **JS/TS**, extract both `dependencies` and `devDependencies` from `package.json`.
For **Go**, extract `require` entries from `go.mod`.
For **Python**, extract entries from `requirements.txt` or `[project.dependencies]` in `pyproject.toml`.
For **Java/Kotlin**, extract `<dependency>` from `pom.xml` or `implementation`/`api` from `build.gradle`.

Also extract `name_user` and `name_repository` from `git remote get-url origin` (parse org/repo from the URL). If not available, use "unknown".

### Step 2: Build context summary

Before spawning the agent, prepare a context summary so it can skip discovery and go straight to validation:

```
## Context summary
- Stack: [node/go/python/java/kotlin]
- Ecosystem: [npm/go/pypi/maven]
- Repository: [user/repo]
- Manifest: [path to manifest file]
- Total dependencies: [N]
- Scope: [full audit / specific packages / new deps from diff]

## Dependencies to check
- [name]@[version]
- [name]@[version]
...
```

If you already know from the current conversation which dependencies were recently added or changed, include only those and note `Scope: new deps from conversation context` — this avoids re-checking the entire manifest.

### Step 3: Check with dependency-checker agent


Launch the `dependency-checker-agent` (`subagent_type: "dependency-checker-agent"`):

```
Check the following dependencies for security vulnerabilities.

[paste context summary from Step 2]

Use the codeguard MCP (`mcp__meli_appsec_codeguard__safe_add_dependency`) if available — pass all dependencies in a single call. If the MCP is not available, fall back to manual checks. Report all vulnerabilities, deprecations, and concerns.
```

The context summary gives the agent everything it needs. It should **not** need to read manifest files or run git commands — go straight to validation.




### Step 4: Report results

Present the agent's findings directly to the user. If all dependencies are safe, say so clearly.

## Important

- Prefer the codeguard MCP over manual checks — it has the most complete vulnerability database
- When checking specific packages, still detect the stack to use the right ecosystem
- Do not modify any files — this is a read-only audit
- For large projects (>50 deps), consider checking only direct dependencies, not transitive
- The context summary is key: it lets the agent skip all discovery and use 1-2 tool calls instead of 5-6