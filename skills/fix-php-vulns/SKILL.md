---
name: fix-php-vulns
description: Find and fix PHP security vulnerabilities and Composer dependency issues. Accepts an optional GitHub Dependabot Alert URL as argument. If no URL is provided, asks the user whether to audit specific packages or run a full project scan. Use when the user says "fix php vulnerabilities", "fix composer audit", "patch php security", "fix php CVEs", "secure my PHP", "php dependabot alert", or "audit composer deps".
allowed-tools: Bash(gh *), Bash(composer *), AskUserQuestion, Agent
argument-hint: <GitHub Dependabot Alert URL or empty>
context: fork
---

# Fix PHP Vulnerabilities

Orchestrate the `php-security-fixer-agent` to find and fix PHP/Composer security issues.

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
      description: "Provide one or more Composer package names to check and fix (targeted, fast)"
    - label: "Full project vulnerability scan"
      description: "Run composer audit and scan all PHP files for insecure patterns (thorough)"
```

- If user chooses **specific packages**: ask for the package names, then go to Step 3b.
- If user chooses **full project scan**: go to Step 3c.

---

### Step 2: Dependabot mode — fetch alert details

Dependabot alert URL format: `https://github.com/<owner>/<repo>/security/dependabot/<alert-number>`

Parse the URL to extract owner, repo, and alert number. Then fetch:

```bash
gh api repos/<owner>/<repo>/dependabot/alerts/<alert-number>
```

Extract from the response:
- `dependency.package.name` — vulnerable package (e.g. `laravel/framework`)
- `dependency.manifest_path` — which manifest (e.g. `composer.json`)
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

### Step 3: Launch php-security-fixer-agent

#### 3a — Dependabot targeted fix

Spawn `php-security-fixer-agent` with:

```
Fix the following Dependabot security alert in this PHP/Laravel project.

[paste Dependabot Alert Context from Step 2]

Focus only on this package. Run `composer audit` to confirm the advisory is present,
then apply the minimum safe upgrade (`composer require <package>:<patched-version>` or
the latest compatible patch). Verify `composer audit` no longer reports this advisory.
Also grep the source for any direct usage of the vulnerable API that may need patching
after the upgrade. Run `php artisan test --compact` if this is a Laravel project.
```

#### 3b — Targeted package audit + fix

Spawn `php-security-fixer-agent` with:

```
Audit and fix the following Composer packages in this PHP project: <package names>.

Run `composer audit` filtered to these packages, check for vulnerabilities, and apply
safe upgrades where available. Also grep the source for insecure usage patterns related
to these packages. Report what was fixed and what was skipped.
```

#### 3c — Full project scan

Spawn `php-security-fixer-agent` with:

```
Run a full vulnerability scan and fix pass on this PHP project.

1. Run `composer audit --format=json` and process all advisories (Critical and High first).
2. For each advisory, apply the minimum safe `composer require <pkg>:<safe-version>`.
3. After dependency fixes, scan PHP source for insecure patterns:
   - SQL injection (raw queries with concatenation)
   - XSS in Blade ({!! !!} on user data)
   - Mass assignment ($guarded = [])
   - Command injection (exec/shell_exec with user input)
   - Deserialization (unserialize on user-controlled data)
   - Hardcoded secrets (credentials not using env())
   - Open redirects (redirect() with user input)
4. Patch all found code issues with the minimum safe change.
5. Run `php artisan test --compact` if this is a Laravel project.
6. Report everything fixed, skipped, and anything needing manual intervention.
```

---

### Step 4: Present results

Relay the agent's Fix Report to the user. Highlight any skipped items so the user knows what still needs attention.
