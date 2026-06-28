# Claude Marketplace

Personal collection of reusable [Claude Code](https://claude.ai/code) agents and skills.

## Structure

```
agents/   → Subagents invoked via the Agent tool (specialist workers)
skills/   → Slash commands invoked directly in the Claude Code CLI
```

## Installation

Copy the files you want into your global Claude config:

```bash
# Agents → ~/.claude/agents/
cp agents/<agent>.md ~/.claude/agents/

# Skills → ~/.claude/skills/<skill-name>/SKILL.md
mkdir -p ~/.claude/skills/<skill-name>
cp skills/<skill-name>/SKILL.md ~/.claude/skills/<skill-name>/
```

Then restart Claude Code or reload the session.

---

## Agents

Agents are specialist subagents. Claude invokes them autonomously when the task matches their description, or you can reference them explicitly.

| Agent | Description | Model |
|-------|-------------|-------|
| `php-security-fixer-agent` | Find and fix PHP/Composer vulnerabilities — SQL injection, XSS, CSRF, mass assignment, deserialization, secrets exposure. Runs `composer audit` + patches source. | haiku |
| `js-security-fixer-agent` | Find and fix JS/TS vulnerabilities — XSS, eval, prototype pollution, unsafe deps. Runs `npm audit` + patches source. | sonnet |
| `security-reviewer-agent` | Read-only security review for JS/TS, Go, Python, Java/Kotlin. OWASP top 10, secrets, auth patterns. | sonnet |
| `dependency-checker-agent` | Validates new/changed dependencies across npm, Go, PyPI, Maven. Flags CVEs, deprecated packages, low adoption. | haiku |
| `linter-agent` | Auto-detects stack (JS/TS, Go, Python) and fixes lint/format issues. | haiku |
| `git-committer-agent` | Stages, commits with conventional commit messages, pushes, and opens PRs. | sonnet |
| `pr-summary-agent` | Analyzes branch diff and generates a structured PR description via `gh pr create`. | sonnet |
| `test-engineer-agent` | Creates and fixes unit tests. Supports Jest + RTL (JS/TS), go test + testify, pytest. | sonnet |
| `quick-explainer-agent` | Answers knowledge questions about code, libraries, or concepts without touching files. | sonnet |
| `code-quality-reviewer-agent` | Reviews code for style violations, DRY issues, naming, testing gaps. | sonnet |
| `architecture-reviewer-agent` | Reviews component structure, state management, separation of concerns, and framework compliance. | sonnet |

---

## Skills

Skills are slash commands you invoke directly in Claude Code: `/skill-name [args]`.

| Skill | Command | Description |
|-------|---------|-------------|
| `fix-php-vulns` | `/fix-php-vulns [dependabot-url]` | Fix PHP/Composer vulnerabilities. Accepts a GitHub Dependabot alert URL for targeted fixes, or runs a full audit. |
| `fix-vulns` | `/fix-vulns [dependabot-url]` | Fix JS/TS vulnerabilities. Accepts a GitHub Dependabot alert URL or runs `npm audit` full scan. |
| `check-deps` | `/check-deps [packages]` | Audit dependencies for CVEs and deprecations across npm, Go, PyPI, Maven. |
| `clean-install` | `/clean-install [--skip-<step>]` | Clean install, build, test, and lint in one pass. |
| `code-review` | `/code-review [files or PR#]` | Parallel security + quality + architecture review via specialized subagents. |
| `commit` | `/commit [message] [and push] [and create PR]` | Conventional commit with optional push and PR creation. |
| `create-branch` | `/create-branch` | Create and push a feature/fix/hotfix/chore branch. |
| `explain` | `/explain [file or topic]` | Explain code, concepts, or tools without modifying files. |
| `lint` | `/lint [fix] [files]` | Run linter; pass `fix` to auto-correct. |
| `pr-summary` | `/pr-summary [base-branch]` | Generate a structured PR description from branch diff. |
| `test` | `/test [files or test-name]` | Create, fix, or run unit tests. |

---

## Dependabot Integration

Both `fix-php-vulns` and `fix-vulns` accept a GitHub Dependabot alert URL:

```
/fix-php-vulns https://github.com/owner/repo/security/dependabot/42
/fix-vulns https://github.com/owner/repo/security/dependabot/42
```

The skill fetches the alert via `gh api`, extracts the CVE, vulnerable range, and patched version, then delegates a targeted fix to the appropriate agent.
