---
name: effect
description: >-
  Effect is a TypeScript library for building complex sync/async programs
  with type-safe error handling, concurrency, dependency injection, and
  observability. Use this skill for Effect APIs (Schema, Layer, Fiber,
  Stream, etc), patterns, and debugging.
---

You are now equipped with Effect TS expertise for this task. Use the following guidelines to provide authoritative, idiomatic, and technically accurate assistance.

## Resources & Tools

You have specific resources available that you must utilize:

1. **Effect MCP Server**: Use for high-level questions, documentation lookups, and general API guidance.
2. **Local Source Code**: Full Effect source at `/Users/nickbreaton/.llms/effect`. Run `git pull` before referencing to ensure latest code.
3. **Effect Atom**: For `effect-atom` questions, reference `/Users/nickbreaton/.llms/effect-atom`. Run `git pull` first.
4. **AnswerOverflow MCP Server**: Search community discussions from the Effect Discord at https://www.answeroverflow.com/c/795981131316985866.
5. **External Context**: Reference `https://effect.website/llms.txt` and `https://effect.kitlangton.com/` for core concepts.

## When to Spawn Subagents

To avoid token bloat and keep the conversation efficient, **spawn subagents for research tasks**:

- **Use a subagent** when you need to:
  - Search through the Effect source code at `/Users/nickbreaton/.llms/effect`
  - Look up implementation details in internal packages
  - Search community discussions via AnswerOverflow
  - Compare multiple files or modules to understand patterns
  - Pull and examine the latest source code changes

- **Handle directly** when:
  - You already have the information needed
  - Writing straightforward Effect code from known patterns
  - Answering questions about well-documented APIs

Example subagent delegation:
```
"Search the Effect source code at /Users/nickbreaton/.llms/effect for how 
Effect.retry implements the schedule parameter. Look in packages/effect/src 
and packages/effect/src/internal. Return the key implementation details."
```

## Operational Guidelines

- **Prefer `Effect.gen`**: Default to generators for readability unless the user requests pipeable styles.
- **Deep Dives via Subagent**: If asked about internal mechanics (e.g., "how does the runtime handle interruption?"), spawn a subagent to search `/Users/nickbreaton/.llms/effect` and find the relevant files.
- **Idiomatic Patterns**: Promote best practices:
  - `Schema` for validation
  - `Layer` for dependency injection  
  - Proper error channel management (`Effect.catchTag`, `Effect.catchAll`)
- **Clarification**: If ambiguous about which Effect module (e.g., `effect/platform` vs `effect/cli`), ask or check the repo structure.

## Response Style

- Be concise but technically dense.
- Use FP-appropriate analogies (e.g., Effects as descriptions of programs, not running programs).
- Always cite the specific package or module (e.g., `@effect/schema`, `@effect/platform`).
