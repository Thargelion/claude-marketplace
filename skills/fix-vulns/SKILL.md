---
name: fix-vulns
description: Find and fix JavaScript/TypeScript security vulnerabilities and dependency issues. Accepts an optional GitHub Dependabot Alert URL as argument. If no URL is provided, asks the user whether to audit specific packages or do a full project vulnerability scan. Use when the user says "fix vulnerabilities", "fix npm audit", "patch security issues", "fix CVEs", "secure my dependencies", "dependabot alert", or "audit vulns".
allowed-tools: Bash(gh *), Bash(npm audit *), Bash(git *), AskUserQuestion, Task
argument-hint: <GitHub Dependabot Alert URL or empty>
context: fork
---

# Fix Vulnerabilities

Orchestrate the `js-security-fixer-agent` to find and fix JS/TS security issues.

## Process

### Step 1: Determine mode

**If a GitHub Dependabot Alert URL was provided as argument:**
→ Go to Step 2 (Dependabot mode).

**If no argument was provided:**
→ Ask the user:

```
AskUserQuestion:
  question: "What would you like to do?"
  options:
    - label: "Audit & fix specific packages"
      description: "Provide one or more package names to check and fix (targeted, fast)"
    - label: "Full project vulnerability scan"
      description: "Run npm audit and scan all source files for insecure patterns (thorough)"
```

- If user chooses **specific packages**: ask for the package names, then go to Step 3b.
- If user chooses **full project scan**: go to Step 3c.

---

### Step 2: Dependabot mode — fetch alert details

Parse the URL to extract the repo and alert number.

Dependabot alert URL format: `https://github.com/<owner>/<repo>/security/dependabot/<alert-number>`

Fetch alert details:
```bash
gh api repos/<owner>/<repo>/dependabot/alerts/<alert-number>
```

Extract from the response:
- `dependency.package.name` — vulnerable package
- `dependency.manifest_path` — which manifest (e.g. `package.json`)
- `security_advisory.severity` — critical / high / moderate / low
- `security_advisory.cve_id` — CVE identifier
- `security_advisory.summary` — short description
- `security_vulnerability.vulnerable_version_range` — affected range
- `security_vulnerability.first_patched_version.identifier` — safe version to upgrade to

Build context summary:
```
## Dependabot Alert Context
- Repo: <owner>/<repo>
- Alert: #<number>
- Package: <name>
- Severity: <severity>
- CVE: <cve_id>
- Issue: <summary>
- Vulnerable range: <vulnerable_version_range>
- Patched version: <first_patched_version>
- Manifest: <manifest_path>
```

→ Go to Step 3a.

---

### Step 3: Launch js-security-fixer-agent

#### 3a — Dependabot targeted fix

Spawn `js-security-fixer-agent` with:

```
Fix the following Dependabot security alert in this JS/TS project.

[paste Dependabot Alert Context from Step 2]

Focus only on this package. Run `npm audit` to confirm the advisory is present, apply
the minimum safe upgrade (to the patched version listed above or later), then verify
`npm audit` no longer reports this advisory. Also scan the source for any direct usage
of the vulnerable package API that may need patching after the upgrade.
```

#### 3b — Targeted package audit + fix

Spawn `js-security-fixer-agent` with:

```
Audit and fix the following packages in this JS/TS project: <package names>.

Run `npm audit` filtered to these packages, check for vulnerabilities, and apply safe
upgrades where available. Also grep the source for insecure usage patterns related to
these packages. Report what was fixed and what was skipped.
```

#### 3c — Full project scan

Spawn `js-security-fixer-agent` with:

```
Run a full vulnerability scan and fix pass on this JS/TS project.

1. Run `npm audit --json` and process all advisories (Critical and High first, then Moderate).
2. Apply `npm audit fix` for all non-breaking fixes.
3. For remaining advisories, attempt targeted upgrades one at a time.
4. Grep `src/` for insecure code patterns: eval, innerHTML, dangerouslySetInnerHTML,
   hardcoded secrets, Math.random in security contexts, prototype pollution, child_process.exec
   with template literals.
5. Patch all found code issues with the minimum safe change.
6. Verify TypeScript still compiles after each code edit.
7. Report everything fixed, everything skipped, and anything that needs manual intervention.
```

---

### Step 4: Present results

Relay the agent's Fix Report to the user. If any issues were skipped, highlight them clearly so the user knows what still needs attention.
