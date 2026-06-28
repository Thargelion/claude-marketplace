---
name: explain
description: Quick explanation of code, concepts, or tools without modifying files. Use whenever the user asks 'what is', 'how does', 'explain', 'what does this do', 'why does', or has a knowledge question about code, libraries, patterns, or tools. Accepts optional file path or topic as argument.
allowed-tools: Agent
argument-hint: <file-path or topic>
model: haiku
---

# Explain


Delegate the explanation to the `quick-explainer-agent` agent, which runs on a fast model to answer quickly without consuming main context.



## Process

### Step 1: Parse arguments

Determine what the user wants explained:

- **File path** — explain what that file/function does
- **Topic or concept** — explain the concept
- **Code snippet in conversation** — explain the referenced code
- **No arguments** — ask the user what they want to understand

### Step 2: Launch quick-explainer-agent


Use the Agent tool to spawn the `quick-explainer-agent` with the appropriate prompt:

**Explain a file or function:**

```
Explain what {file path} does. Read the file, then provide a concise explanation covering:
- Purpose and responsibility
- Key inputs/outputs or props
- How it fits in the broader codebase
Keep it to 2-4 paragraphs. Use code snippets only when they help clarify.
```

**Explain a concept or tool:**

```
Explain {topic}. Keep the answer concise (2-5 paragraphs), use code examples when they help, and relate it to practical usage in a JavaScript/TypeScript/React project when relevant.
```

Always include `subagent_type: "quick-explainer-agent"` and `model: "haiku"` when spawning the agent.




### Step 3: Report result

When the agent completes, relay the explanation to the user. Do not add to it unless the user asks follow-up questions.

## Important

- Never create, edit, or delete files — this is read-only
- Keep answers concise and focused
- If the question requires implementation work, say so and suggest using the appropriate skill instead