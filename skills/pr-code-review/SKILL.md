---
name: pr-code-review
description: Multi-perspective PR review for Godot 4 / C# projects. Runs a simplification pass, then a .NET, mathematician, and Godot specialist in parallel, then adjudicates their conflicting findings into one report with an ordered resolution plan. Use whenever the user wants a deep or multi-agent review of a branch or PR, or says 'pr review', 'review this PR', 'review my branch', 'deep review', 'review before merge', or '/pr-code-review'. Accepts an optional PR number or file paths.
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob, Agent
argument-hint: "[PR# | file paths] [--all] [--base <ref>]"
context: fork
---

# PR Code Review

Five reviewers, three waves. Ponytail proposes deletions; three specialists review the diff and may veto those deletions from their domain; Commander White adjudicates and produces the plan.

## Step 1 — Discovery

Resolve the scope from the argument:

| Argument | Command |
|---|---|
| PR number (`#123` / `123`) | `gh pr diff <n>` and `gh pr diff <n> --name-only` |
| File paths | `git diff <base>...HEAD -- <paths>` |
| None | base = `--base <ref>` if given, else `main`; `git diff --name-only <base>...HEAD` |

If the current branch **is** the base, fall back to `git diff --name-only HEAD` (uncommitted) and say so in the report scope line.

### Filter

Exclude entirely — never counted, never diffed:

```
.godot/**  **/bin/**  **/obj/**  graphify-out/**  Windows/**
*.uid  *.import  *.DS_Store
*.png *.jpg *.jpeg *.svg *.wav *.ogg *.mp3 *.ttf *.res *.glb *.blend
```

### Classify into buckets

| Bucket | Glob |
|---|---|
| `cs_kernel` | `Kernel/**/*.cs` |
| `cs_domain` | `Domain/**/*.cs` |
| `cs_scripts` | `Scripts/**/*.cs`, root-level `*.cs` |
| `cs_tests` | `Tests/**/*.cs` |
| `godot_res` | `*.tscn`, `*.tres`, `*.gdshader`, `project.godot`, `export_presets.cfg`, `android/**` |
| `data` | `Stages/*.json`, `Weapons/*.json`, `Rewards/**/*.json`, `Docs/stage.schema.json`, `Localization/**` |
| `build` | `*.csproj`, `*.sln`, `.editorconfig`, `.github/**` |
| `docs` | `*.md` |

`cs_any` = union of the four `cs_*` buckets.

### Content triggers — over added/modified `+` lines only

- **MATH-A** — 1 hit fires:
  `MathF?\.|\b(Sin|Cos|Tan|Asin|Acos|Atan2|Sqrt|Pow|Exp|Log|Lerp|Slerp|SmoothStep)\b|Normalized|LengthSquared|\.Dot\(|\.Cross\(|DegToRad|RadToDeg|\bTau\b|\bRandom\b|Next(Double|Single|Int64)?\(|Shuffle|Weighted|Pity`
- **MATH-B** — 3 *distinct* hits fire:
  `angle|velocity|speed|radius|amplitude|frequency|interval|offset|probability|weight|chance|threshold|epsilon|ratio|percent|clamp|delta`
- **GODOT-C** — 1 hit fires:
  `using Godot|GD\.|GetNode|GetTree|QueueFree|\[Export|\[Signal|EmitSignal|_Ready|_Process|_PhysicsProcess|_Draw|_Input|Callable\.From|PackedScene|ResourceLoader|Tween|Area2D|Node2D|SubViewport`

### Orchestrator's own checks — always run, regardless of gating

1. `grep -rn "using Godot" Kernel/ Domain/` — any hit is a **pre-declared P1 CRITICAL** (breaks the CI dependency rule). Pass it to Commander White as an orchestrator finding.
2. If `data` files changed and `Docs/stage.schema.json` exists, check the changed stage JSON against it and report any conformance break the same way.

These run even when every agent is gated out, so a data-only diff is never reviewed by nobody.

## Step 2 — Gate

| Changed set | 1 Pony | 2 .NET | 3 Math | 4 Godot |
|---|---|---|---|---|
| `Kernel/**/*.cs` | Y | Y | **Y** | if GODOT-C |
| `Domain/{Boss,Stages,Rewards,Battery,Shield,Combat}/**` | Y | Y | **Y** | if GODOT-C |
| `Domain/{Input,Localization,SaveState,Devices,Scoring,Story}/**` | Y | Y | if MATH-A or 3×MATH-B | if GODOT-C |
| `Scripts/**/*.cs` | Y | Y | if MATH-A or 3×MATH-B | **Y** |
| `Tests/**` only | Y | Y | if MATH-A or 3×MATH-B | N |
| `*.tscn` / `*.tres` / `*.gdshader` only | Y | N | N | **Y** |
| `project.godot` / `export_presets.cfg` / `android/**` only | Y | N | N | **Y** |
| `*.csproj` / `*.sln` only | Y | **Y** | N | **Y** |
| `data` only | Y | N | N | N (schema check covers it) |
| `docs` only, or empty after filtering | — | — | — | — |
| `--all` flag | Y | Y | Y | Y |

Rule 2 baseline: the .NET reviewer runs whenever `cs_any ≠ ∅`, no exceptions.

**Docs-only or empty** → print "Nothing to review — the diff contains no reviewable code files." and stop. Spawn nothing.

Record a **gate ledger**: for every agent, either `ran` or `SKIPPED — <reason>`, plus which trigger fired. This goes to Commander White and into the final Coverage table.

### Batching

- ≤10 files **and** ≤400 changed lines → one batch, per-file patches.
- Larger → batch by layer (`Kernel/`, `Domain/`, `Scripts/`, `Tests/`, resources). Each agent receives only the batches its gate matched.
- >25 files or >1200 lines → run waves A and B per batch, then a single Commander White pass over all outputs. Note the truncation explicitly.

## Step 3 — Wave A: ponytail (blocking)

One `Agent` call, `subagent_type: "ponytail-reviewer-agent"`. Wait for it.

```
Review this diff for over-engineering. Complexity only.

Files: [list]

Diff:
[diff]
```

### Parse the output into the deletion set

Ponytail emits `<path>:L<n>: <tag> <what>. <replacement>.` (tolerate a bare `L<n>:` form too). Keep only **structural** tags — `delete:`, `yagni:`, `stdlib:`, `native:`. Drop `shrink:` (cosmetic, nothing to veto). Cap at 30, sorted by structural risk; note if truncated.

**Fail soft:** if fewer than one line parses and the report is not `Lean already. Ship.`, pass ponytail's raw text through as the annex under a warning banner rather than an empty set.

## Step 4 — Wave B: specialists (parallel)

Dispatch every gated agent in **one message with multiple `Agent` calls**. This is the key performance property — never sequential.

`subagent_type`: `dotnet-reviewer-agent` · `mathematician-agent` · `godot-specialist-agent`

Each prompt:

```
PONYTAIL MODE governs code you WRITE. You are not writing code — you are reviewing it.
Do not lower the severity of, or omit, a correctness / safety / hot-path finding because
its fix would add lines. Ponytail's own charter exempts input validation, error handling
that prevents data loss, and security from simplification. Report what you find.

Review the following changes from your specialty. Work from the diff; open a full file
only when a specific candidate finding needs local context.

Files: [the batches this agent's gate matched]

Diff:
[diff]

## PENDING DELETION SET (another reviewer's CLAIMS, not decisions)

D1  path/file.cs:L88   yagni:   delete IFoo, one impl
D2  path/other.cs:L12  stdlib:  manual clamp → Math.Clamp
...

1. DO NOT re-review complexity — that scope is taken. Do not restate, agree with,
   or expand these findings.
2. DO falsify them from your specialty. If applying any Dn would break correctness,
   violate a project rule, or regress a per-frame path:
     VETO D<n> — <concrete failing input / named rule / named per-frame method> — <what breaks>
   A veto without one of those three is not a veto; stay silent instead.
3. Silence on a Dn is NOT endorsement.
```

If zero agents are gated in, skip to Step 5 and record it in the ledger.

## Step 5 — Wave C: Commander White

One `Agent` call, `subagent_type: "commander-white-agent"`.

Pass wave-B reports **verbatim and unsummarized**. Summarizing them here would make this skill an unaccountable fourth adjudicator, leaving Commander White to rule on evidence she never saw.

```
Adjudicate these review reports into one result.

Scope: N files | M lines | <head> vs <base>

## Gate ledger
[every agent: ran / SKIPPED — reason, plus which trigger fired]

## Orchestrator findings
[dependency-rule grep result; stage schema conformance result]

## Ponytail — raw report
[verbatim]

## <Specialist> — report
[verbatim, one section per agent that ran]
```

## Step 6 — Output

Print Commander White's report unchanged. Add nothing except the scope header if it is missing.

## Important

- Wave B must be a single message with N parallel `Agent` calls.
- No agent holds `Edit`, `Write`, or `Bash` — report-only is enforced by their tool lists, not by instruction. `git status` must be clean after a run.
- A `SKIPPED` reviewer means unexamined, not clean. Never let a gate read as a clean bill of health.
- Never report formatting or anything `dotnet format --verify-no-changes` catches — the repo's pre-commit hook already blocks those.
