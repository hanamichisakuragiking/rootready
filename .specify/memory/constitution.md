# RootReady Constitution

## Core Principles

### I. Beginner-First Writing (NON-NEGOTIABLE)

Every word is written for someone who has never touched a Linux terminal. This is the foundational principle — all other principles serve it.

- **Zero assumed knowledge.** Never assume the reader knows any Linux, networking, or sysadmin concepts. If a term hasn't been explained in a prior lesson, explain it now.
- **Define every technical term** the first time it appears, inline using a blockquote callout:
  ```
  > **What is SSH?**
  > SSH (Secure Shell) is a way to connect to a remote computer...
  ```
- **Explain before showing.** Every command block must be preceded by a plain-English explanation of *what* it does and *why* the reader is running it. Never drop a command without context.
- **Tone is friendly, encouraging, and direct.** Never condescending. Never assume something is "easy" or "obvious." If it were obvious, the reader wouldn't need the lesson.
- **Use concrete language.** Say "click the green Start button" not "initiate the VM." Say "type this command" not "execute the following."

### II. Content Accuracy (NON-NEGOTIABLE)

Every command, configuration snippet, and technical claim must be correct and reproducible.

- **Commands must work.** Every command shown must produce the described output on a clean Rocky Linux 9 minimal install. No hypothetical or untested commands.
- **Version-pin where it matters.** Specify the Rocky Linux version, software versions, and config file paths. If a path or option changed between versions, note it.
- **Show expected output.** After significant commands, show what the reader should see. This builds confidence and helps with troubleshooting.
- **Troubleshooting is mandatory.** Every lesson must include a `## Troubleshooting` section covering the most common errors a beginner will hit at each step.

### III. Progressive Lesson Structure

Lessons follow a strict order. Each lesson builds exactly on what came before — no skipping, no forward references to unexplained concepts.

- **Fixed section order** for every lesson:
  1. `## What You Will Learn` — bullet list of outcomes
  2. `## What You Need Before Starting` — explicit prerequisites
  3. Main content sections (numbered steps)
  4. `## Troubleshooting` — common errors and fixes
  5. `## Summary` — bullet recap of what was accomplished
- **Required frontmatter:** `title`, `description`, `date`, `weight`, `tags`. Missing fields break the pipeline.
- **Weight controls order.** Lessons are numbered sequentially. Weight must match the lesson number.
- **No forward references.** Never link to or mention concepts from a future lesson without marking it clearly as "(covered in Lesson N)."
- **Each lesson is 20–30 minutes** of reading + doing. If it's longer, split it.

### IV. Accessibility and Inclusion

The site must be usable by the widest possible audience.

- **Alt text on every image.** Describe what the image shows, not just "screenshot."
- **Code blocks must have a language tag** for syntax highlighting and screen readers (e.g., ` ```bash `, ` ```text `, ` ```yaml `).
- **No color-only information.** Don't say "the green button" without also saying "the Start button."
- **Keep paragraphs short.** 2–4 sentences max. Walls of text lose beginners.
- **Use numbered steps** for procedures. Bullet lists for non-sequential information.

### V. Technical Stack Discipline

The project uses a specific, minimal stack. Don't expand it without justification.

- **Hugo 0.157.0 Extended** — static site generator. No other SSG.
- **Hextra theme** — loaded as a git submodule at `themes/hextra` (lowercase). Never edit theme files directly.
- **GitHub Pages** — deployment target via GitHub Actions. No other hosting.
- **Markdown only** — all content is Markdown. No custom JavaScript, no interactive widgets, no embedded apps.
- **Theme name is case-sensitive.** `hugo.toml` must use `theme = 'hextra'` (lowercase) or the Linux CI build fails.

## Content Quality Gates

Every lesson must pass these checks before merging:

| Gate | Requirement |
|---|---|
| Frontmatter | Has `title`, `description`, `date`, `weight`, `tags` |
| Structure | Contains `## What You Will Learn` and `## Summary` |
| Troubleshooting | Contains `## Troubleshooting` with at least one issue |
| Commands explained | No command block without a preceding explanation |
| Jargon defined | No undefined technical term on first use |
| Code fence language | Every fenced code block has a language identifier |
| No broken links | No `relref` to pages that don't exist yet — use plain text instead |
| Build passes | `hugo build` completes with zero errors |

## Development Workflow

1. **Branch from `main`** using the naming convention: `lesson/NN-slug`, `fix/description`, `chore/description`
2. **Write content** following the principles above
3. **Test locally** with `hugo server -b http://localhost:1313` in Git Bash
4. **Commit** using conventional commits: `content:`, `fix:`, `chore:`, `ci:`
5. **Open a PR** — this triggers quality gate checks
6. **Merge only when all checks pass**

## Governance

This constitution is the single source of truth for how RootReady content is created and maintained. It supersedes any conflicting guidance.

- All PRs must comply with these principles
- Amendments to this constitution require a dedicated PR with `chore: amend constitution` as the commit message
- The `CLAUDE.md` file contains operational details (commands, paths, environment notes) and should stay aligned with this constitution

**Version**: 1.0.0 | **Ratified**: 2026-03-16 | **Last Amended**: 2026-03-16
