---
agent: speckit.implement
---

## Pre-flight

Before writing any code or content, read `CLAUDE.md` in the workspace root.
It contains project-specific information you must know before acting:

- How to invoke Hugo on this machine (full executable path, required flags)
- Repository structure and content conventions
- Lesson frontmatter requirements and section order
- Writing style rules

Do not skip this step. Commands like `hugo build` will fail if you use the
wrong invocation.

## Task

Implement the next incomplete task from the active spec's `tasks.md`.

1. Read `specs/<feature>/tasks.md` to find the first unchecked task group.
2. Read `specs/<feature>/spec.md` and `specs/<feature>/plan.md` for full context.
3. Implement all items in that task group.
4. Run `hugo build` (using the path from `CLAUDE.md`) and confirm zero errors.
5. Mark every completed checkbox in `tasks.md`.
6. Commit with a descriptive message referencing the lesson and key changes.