---
name: code-review
description: Run a comprehensive code review using specialized agents (security, quality, architecture) in parallel. Use this skill whenever the user asks to review code, check a PR, audit changes, look over files before merging, give feedback on code quality, or sanity check changes. Also use when the user mentions 'code review', 'review my changes', 'check my code', 'is this ready to merge', 'look at my changes', 'give me feedback', or 'audit this'. Accepts optional file paths or PR number as argument.
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob, Task
argument-hint: <file-paths or PR#>
context: fork
---

# Code Review


Run a multi-agent code review on specified files or changes. Specialized agents (security, quality, architecture, and optionally dependency-checker) analyze the code in parallel, and their findings are consolidated into a single prioritized report.




## Process

### Step 1: Determine scope, detect stacks, and collect context

Identify what to review based on the argument provided:

- **PR number** (e.g., `#123`): run `gh pr diff <number>` to get the diff and `gh pr diff <number> --name-only` for the file list
- **File paths**: use those directly, run `git diff HEAD -- <files>` if they are tracked to get the diff context
- **No argument**: detect changes automatically:
  1. If there are uncommitted changes: `git diff --name-only HEAD` (staged + unstaged)
  2. Otherwise: `git diff --name-only HEAD~1` (last commit)

**Filter to code files only** (exclude images, lock files, generated files, etc.):

| Stack | Extensions |
|-------|-----------|
| JS/TS | `.js`, `.jsx`, `.ts`, `.tsx`, `.mjs`, `.cjs` |
| Go | `.go` |
| Python | `.py` |
| Java/Kotlin | `.java`, `.kt`, `.kts` |
| Styles | `.scss`, `.css` |
| Config | `.json`, `.yaml`, `.yml`, `.toml` (only when they appear alongside code changes) |

If no code files are found after filtering, tell the user and stop.

**Detect stacks present in the file list.** Classify each file by its extension and build a list of detected stacks (e.g., `["js/ts", "go"]`). This determines which security rules the security reviewer should focus on.

**Check for dependency changes.** If any of these files are in the diff, flag for dependency review: `package.json`, `go.mod`, `go.sum`, `requirements.txt`, `pyproject.toml`, `setup.py`, `pom.xml`, `build.gradle`, `build.gradle.kts`. Extract the dependency-specific diff separately.

**Collect the diff only for the filtered code files.** Do not pass a repo-wide or PR-wide diff that includes non-code files. Prefer per-file diffs over one huge combined diff when the review set is large.

**Default heuristic:** if the review scope is 10 files or fewer, collect the patch for each filtered file and review that set. If it is larger than 10 files or roughly larger than 400 changed lines, split the review into logical batches (group by directory or functional area) and summarize by batch instead of dumping one giant diff into every reviewer.

### Step 2: Launch review agents in parallel


Use the Task tool to spawn agents **in parallel** (all in a single message with multiple tool calls). Each agent is a custom agent definition installed in `.claude/agents/`.

Pass each agent the **file list**, the **diff output**, and the **detected stacks** so they can focus their review on the actual changes rather than reviewing entire files blindly.

Always launch these **3 core agents**:

1. **security-reviewer** (`subagent_type: "security-reviewer-agent"`)

   Prompt:
   ```
   Review the following files for security vulnerabilities. Focus your analysis on the changed lines shown in the diff.

   Detected stacks: [list of stacks, e.g., "js/ts, go"]

   Files to review: [file list]

   Diff of changes:
   ```
   [paste diff]
   ```

   Apply security rules for each detected stack. Review the diff first. Open a full file only if a specific finding needs extra local context. Do not scan unrelated files or the whole repository.
   ```

2. **code-quality-reviewer** (`subagent_type: "code-quality-reviewer-agent"`)

   Prompt:
   ```
   Review the following files for code quality issues. Focus your analysis on the changed lines shown in the diff.

   Detected stacks: [list of stacks]

   Files to review: [file list]

   Diff of changes:
   ```
   [paste diff]
   ```

   Review the diff first. Open a full file only if a specific finding needs extra local context. Do not scan unrelated files or the whole repository.
   ```

3. **architecture-reviewer** (`subagent_type: "architecture-reviewer-agent"`)

   Prompt:
   ```
   Review the following files for architectural issues. Focus your analysis on the changed lines shown in the diff, but also consider how the changes fit within the broader codebase structure.

   Detected stacks: [list of stacks]

   Files to review: [file list]

   Diff of changes:
   ```
   [paste diff]
   ```

   Review the diff first. Open a full file only if a specific finding needs extra local context. Do not scan unrelated files or the whole repository.
   ```

**Conditionally launch** a 4th agent when dependency manifest files changed:

4. **dependency-checker** (`subagent_type: "dependency-checker-agent"`) — only when Step 1 flagged dependency changes

   Prompt:
   ```
   Check the following dependency changes for security vulnerabilities and issues.

   Detected stack: [stack]

   Dependency diff:
   ```
   [paste only the dependency manifest diff, e.g., package.json changes]
   ```

   Use the codeguard MCP if available. Report vulnerabilities, deprecations, and concerns.
   ```




### Step 3: Consolidate results

Once all review passes complete, synthesize findings into a single report. Deduplicate findings that were flagged by multiple passes (keep the most detailed version and note overlap).

**Output format:**

```markdown
## Code Review Report

**Scope:** X files reviewed | Y lines changed | Stacks: [detected stacks]

### Critical & High Priority

- **[severity]** `file:line` — description of finding (reviewer: security/quality/architecture/dependencies)
  - **Fix:** concrete recommendation

### Medium Priority

- **[severity]** `file:line` — description of finding (reviewer)
  - **Fix:** concrete recommendation

### Low Priority / Suggestions

- `file:line` — suggestion (reviewer)

### Dependencies (if applicable)

- [package@version] — [status: safe / vulnerable / deprecated / warning]

### Strengths

- What the code does well (be specific, reference actual patterns seen)

### Summary

| Reviewer | Findings | Critical | High | Medium | Low |
|----------|----------|----------|------|--------|-----|
| Security | X | - | - | - | - |
| Quality | X | - | - | - | - |
| Architecture | X | - | - | - | - |
| Dependencies | X | - | - | - | - |

**Overall verdict:** Ready to merge / Needs minor changes / Needs significant rework
```

## Important

- Focus on real defects and regression risks, not cosmetic comments
- When deduplicating, prefer the finding with the most actionable fix recommendation
- Prioritize real issues over stylistic nitpicks — if the code is clean, say so clearly
- If the diff is very large (>500 lines), split the review by logical areas and summarize per area
- Prefer findings tied to changed lines. Only cite untouched code when it is directly implicated by the change


- Always launch the 3 core review agents in parallel — this is the key performance optimization
- The dependency-checker is conditional: only launch when manifest files changed
- Each agent only reads files, never modifies them