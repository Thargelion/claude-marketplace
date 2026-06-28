---
name: test
description: Create, fix, or run unit tests. Use whenever the user wants to add tests, fix failing tests, check test coverage, run tests for specific files, or mentions 'test', 'tests', 'spec', 'jest', 'coverage', 'add tests', 'fix tests', 'run tests', or 'test coverage'. Accepts optional file paths or test names as argument.
allowed-tools: Agent
argument-hint: <file-paths or test-name>
model: haiku
---

# Test


Delegate the testing workflow to the `test-engineer-agent` agent, which specializes in Jest and React Testing Library.



## Process

### Step 1: Parse arguments

Determine the action based on what the user provided:

- **File paths** (e.g., `src/components/Button.tsx`) — create or update tests for those files
- **"fix"** or **failing test output** — fix failing tests
- **"run"** or **"coverage"** — run tests or check coverage
- **No arguments** — ask the user what they need

### Step 2: Build context summary


Before spawning the agent, check if the current conversation has context about the code that needs testing. If you have been writing or modifying code in this conversation, build a **context summary**:

```
Context: {what was changed and why}
Source files: {list of files modified}
Test files: {existing test files for those sources, if known}
What needs testing: {specific behaviors, branches, or lines that need coverage}
Coverage gaps: {uncovered lines from coverage report, if available}
```

This lets the agent skip discovery (globbing, grepping, reading unrelated files) and go straight to writing tests.


### Step 3: Launch test-engineer-agent


Use the Agent tool to spawn the `test-engineer-agent` with the appropriate prompt:

**Create tests — with context summary (preferred):**

```
{context summary}

Create or update unit tests for the source files listed above. Use the context to target the specific behaviors and coverage gaps described. Do not glob or grep for test patterns — go directly to the test files listed. Run the tests after creating them to verify they pass.
```

**Create tests — without context:**

```
Create or update unit tests for these files: {file list}.
Read each source file, then create or improve its test file following Jest and React Testing Library conventions from rule-jest-testing.
Keep tests focused: 3-5 tests per component/function covering the main behavior and edge cases.
Run the tests after creating them to verify they pass.
```

**Fix failing tests:**

```
These tests are failing: {test output or file paths}.
Read the failing test file and its source file. Diagnose whether the test or the source is wrong.
Fix the minimal amount needed to make tests pass. Prefer fixing the test over weakening assertions.
Run the fixed tests to verify.
```

**Run tests or coverage:**

```
Run the test suite: {specific command or npm test}.
Report results concisely: total passed/failed, and list any failures with file:line.
If coverage was requested, report the coverage summary.
```

Always include `subagent_type: "test-engineer-agent"` when spawning the agent.




### Step 4: Report result

When the agent completes, summarize:

- What tests were created, fixed, or run
- Pass/fail results
- Any remaining issues

## Important

- Do not rewrite the entire test suite — focus on the scope requested
- Prefer semantic queries (getByRole, getByText) over data-testid
- Prefer userEvent over fireEvent
- If a test failure suggests broken production code, fix the source instead of weakening the test