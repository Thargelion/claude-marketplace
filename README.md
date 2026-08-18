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

### `pr-code-review` panel

These seven are the waves of the [`/pr-code-review`](#skills) skill. They can be invoked individually, but they are designed to run as a pipeline — the reviewers are read-only, and only the executors can modify files.

| Agent | Description | Model · effort |
|-------|-------------|----------------|
| `ponytail-reviewer-agent` | Over-engineering pass. Reports what to delete; its output becomes the deletion set the others adjudicate. Complexity only — correctness is out of scope by contract. | sonnet · high |
| `dotnet-reviewer-agent` | C#/.NET correctness and quality — logic defects, edge cases, async/await, nullable flow, exception paths. Owns general correctness on the panel. | sonnet · high |
| `mathematician-agent` | Algorithm correctness then efficiency — geometry, trigonometry, probability, numerical stability, degenerate inputs. Requires a named trigger for any P0. | sonnet · xhigh |
| `godot-specialist-agent` | Godot 4 engine usage, weighted to per-frame performance — allocations, node lookup in hot paths, lifecycle traps, physics layers. | sonnet · high |
| `commander-white-agent` | Adjudicates the reviewers' conflicting findings via a burden-of-proof cascade, then writes an ordered resolution plan with execution briefs. | sonnet · xhigh |
| `2e-agent` | Applies one approved **simple** fix from an execution brief, runs the verify command, reports real output. Fails loudly rather than improvising. | sonnet · low |
| `2e-complex-agent` | Same contract for **complex** fixes — multi-file, new logic, or tests. | sonnet · medium |

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
| `pr-code-review` | `/pr-code-review [PR# or files] [--all]` | Five-perspective review for **Godot 4 / C#** — simplification pass, then .NET + mathematician + Godot specialists in parallel, adjudicated into one plan, then optionally applied. See below. |
| `commit` | `/commit [message] [and push] [and create PR]` | Conventional commit with optional push and PR creation. |
| `create-branch` | `/create-branch` | Create and push a feature/fix/hotfix/chore branch. |
| `explain` | `/explain [file or topic]` | Explain code, concepts, or tools without modifying files. |
| `lint` | `/lint [fix] [files]` | Run linter; pass `fix` to auto-correct. |
| `pr-summary` | `/pr-summary [base-branch]` | Generate a structured PR description from branch diff. |
| `test` | `/test [files or test-name]` | Create, fix, or run unit tests. |

---

## `/pr-code-review` — how it works

Aimed at Godot 4 / C# repos with a layered architecture. Reviews the current branch against `main` by default; accepts a PR number or file paths.

```
Wave A   ponytail                          → proposes deletions (blocking)
Wave B   .NET · mathematician · Godot      → review in parallel, may VETO a deletion
Wave C   Commander White                   → adjudicates, writes the plan
Gate     you approve                       → nothing is written before this
Wave D   2E / 2E-complex                   → applies the approved fixes
```

**Gating.** Reviewers whose domain the diff never touches are skipped — a stage-JSON-only change spawns no Godot reviewer, a docs-only diff spawns nothing at all. Skips are always reported: *a SKIPPED reviewer means unexamined, not clean.*

**Adjudication.** Specialists genuinely disagree (ponytail says "delete the abstraction", .NET says "extract an interface"). Commander White resolves with a burden-of-proof cascade rather than agent seniority — a correctness claim only outranks a simplicity claim if it names a concrete failing input; an unmet burden gets demoted and competes on line count. She must also attempt a third option that dissolves the conflict before declaring a winner, and may return `UNRESOLVED` rather than invent certainty.

**Execution.** Commander White writes each step as a complete brief — exact file, exact anchor, exact replacement, rationale, scope fence, verify command — so the executor can run at low effort. The expensive reasoning happens once, in the adjudicator. 2E fails loudly (`BLOCKED`, `FAILED`) rather than improvising: an unapplyable brief is a plan defect, not an executor problem.

**Safety.** The five reviewers hold no write tools at all. The two executors have `Edit`/`Write` plus `Bash` scoped to `dotnet` and read-only `git` — no commit, push, checkout, reset, or `rm`. Nothing runs before explicit approval, and nothing is ever committed. `UNRESOLVED` items, `HUMAN ONLY` steps, and anything at `risk: high` are never auto-applied.

**Note on effort.** `effort:` is static agent frontmatter and cannot be set at dispatch, which is why the executor ships as two agents (`2e-agent` at low, `2e-complex-agent` at medium) with Commander White tagging each step `SIMPLE` or `COMPLEX` to route it.

---

## Dependabot Integration

Both `fix-php-vulns` and `fix-vulns` accept a GitHub Dependabot alert URL:

```
/fix-php-vulns https://github.com/owner/repo/security/dependabot/42
/fix-vulns https://github.com/owner/repo/security/dependabot/42
```

The skill fetches the alert via `gh api`, extracts the CVE, vulnerable range, and patched version, then delegates a targeted fix to the appropriate agent.
