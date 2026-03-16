# CLAUDE.md — RootReady Project

This file tells Claude how to work with the RootReady project.
Read this before making any changes to the codebase.

---

## Project Overview

**RootReady** is a Hugo static site hosted on GitHub Pages.
It is a progressive learning resource for non-technical people who want to
learn Linux server operations — from installing Rocky Linux to running
rootless Podman containers secured with SELinux and fail2ban.

- **Hugo version:** 0.157.0 Extended
- **Theme:** Hextra (git submodule at `themes/hextra`) — **lowercase `hextra`**
- **Hosting:** GitHub Pages (https://hanamichisakuragiking.github.io/rootready/)
- **GitHub repo:** https://github.com/hanamichisakuragiking/rootready
- **CI/CD:** GitHub Actions (`deploy.yml` builds Hugo and deploys to Pages)
- **Quality gates:** github/spec-kit (frontmatter + content validation)
- **Target audience:** Complete beginners — zero Linux or sysadmin background assumed

---

## Repository Structure

```
rootready/
├── .github/
│   ├── workflows/
│   │   ├── deploy.yml          # Build Hugo → deploy to GitHub Pages
│   │   └── spec.yml            # Run spec-kit checks on every PR
│   └── specs/                  # spec-kit spec files
│       ├── frontmatter.yml     # Every lesson must have title, description, date, weight
│       └── structure.yml       # Every lesson must have a Summary section
├── content/
│   ├── _index.md               # Site homepage
│   └── lessons/
│       ├── _index.md           # Lessons index page
│       └── 01-install-rocky/
│           └── index.md        # Lesson 1
├── static/
│   └── img/                    # Images referenced in lessons
├── themes/
│   └── hextra/                 # Git submodule — DO NOT edit files here
├── hugo.toml                   # Hugo site configuration
└── CLAUDE.md                   # This file
```

---

## Hugo Commands

Always run Hugo commands from the project root.
Use Git Bash on Windows — not PowerShell — for Hugo and git commands.

```bash
# Start local dev server (overrides baseURL for localhost)
hugo server -b http://localhost:1313

# Build the site to public/
hugo

# Create a new lesson
hugo new content/lessons/02-first-steps/index.md

# Check Hugo version
hugo version
```

> **Important:** The `baseURL` in `hugo.toml` is set to the GitHub Pages URL
> (`https://hanamichisakuragiking.github.io/rootready/`).
> Without `-b http://localhost:1313`, the local server serves everything
> under `/rootready/` and the root path returns 404.

---

## Content Guidelines

### Lesson Frontmatter (required fields)

Every lesson `index.md` MUST start with this frontmatter block.
spec-kit will fail the pipeline if any field is missing.

```yaml
---
title: "Lesson N — Title Here"
description: "One sentence. What the reader will walk away knowing."
date: YYYY-MM-DD
weight: N        # Controls sidebar order — increment by 1 per lesson
tags: []         # e.g. ["rocky linux", "beginner", "selinux"]
---
```

### Lesson Structure (required sections)

Every lesson MUST contain these sections in this order:

```markdown
## What You Will Learn
## [Main content sections]
## Troubleshooting
## Summary
```

spec-kit will block deployment if `## Summary` is missing.

### Writing Style Rules

- **Audience:** Non-technical. No assumed Linux or sysadmin knowledge.
- **Tone:** Friendly, encouraging, direct. Never condescending.
- **Commands:** Every command block must be preceded by a plain-English
  explanation of what it does and why.
- **Jargon:** Define every technical term the first time it appears,
  inline using a blockquote:
  ```
  > **What is root?**
  > root is the all-powerful administrator account in Linux...
  ```
- **Assumptions:** State them explicitly at the top of each lesson
  under a "What You Need Before Starting" section.
- **Length:** Lessons should be completable in 20–30 minutes of reading + doing.

### Lesson Progression (planned syllabus)

| Weight | Slug | Title | Status |
|--------|------|-------|--------|
| 1 | `01-install-rocky` | Installing Rocky Linux in a VM | ✅ Done |
| 2 | `02-first-steps` | Users, SSH, and firewalld | Planned |
| 3 | `03-selinux` | Understanding SELinux | Planned |
| 4 | `04-nginx` | Installing and Hardening Nginx | Planned |
| 5 | `05-podman-basics` | Podman and Rootless Containers | Planned |
| 6 | `06-quadlet` | Running Containers with Quadlet + systemd | Planned |
| 7 | `07-reverse-proxy` | Nginx as a Reverse Proxy | Planned |
| 8 | `08-logging` | Logging with journald | Planned |
| 9 | `09-fail2ban` | Preventing DoS with fail2ban | Planned |
| 10 | `10-cicd` | CI/CD with GitHub Actions | Planned |

---

## GitHub Actions Workflows

### deploy.yml — triggered on push to `main`

Pipeline steps:
1. Checkout repo (with submodules)
2. Install Hugo Extended 0.157.0 from tar.gz release
3. Build site (`hugo build --gc --minify`)
4. Upload artifact and deploy to GitHub Pages

The workflow installs Hugo by downloading the `.tar.gz` from GitHub releases
(not the `.deb` — that format does not exist for Hugo).
Uses `actions/checkout@v6` (not v4) to avoid Node.js 20 deprecation warnings.

### spec.yml — triggered on every PR

Pipeline steps:
1. Checkout repo
2. Run spec-kit against `content/lessons/**/*.md`
3. Fail PR if any spec does not pass

**A PR cannot be merged if spec-kit fails.**
This is enforced via branch protection rules on `main`.

---

## spec-kit Specs

Specs live in `.github/specs/`.

### frontmatter.yml — validates required frontmatter fields

Checks that every lesson has: `title`, `description`, `date`, `weight`

### structure.yml — validates required content sections

Checks that every lesson contains:
- `## What You Will Learn`
- `## Summary`

---

## Git Conventions

```
Branch naming:
  lesson/02-first-steps
  fix/broken-link-lesson-1
  chore/update-theme

Commit messages (conventional commits):
  feat: add lesson 2 first steps
  fix: correct nginx config example in lesson 4
  chore: update hextra submodule
  ci: add spec-kit frontmatter check
  content: draft lesson 3 selinux intro

Never commit directly to main.
Always open a PR — this triggers the spec-kit checks.
```

---

## Theme Notes

- Theme is Hextra, loaded as a git submodule at `themes/hextra`
- **`hugo.toml` must use `theme = 'hextra'` (lowercase)** — the submodule
  folder is `themes/hextra`. Windows is case-insensitive but the Linux CI
  runner is case-sensitive. Using `Hextra` will break the GitHub Actions build.
- **Never edit files inside `themes/hextra/`** — changes will be lost on submodule update
- To override theme layouts, copy the file to the matching path under `layouts/`
  e.g. `themes/hextra/layouts/_default/single.html` → `layouts/_default/single.html`
- Sidebar order is controlled by `weight` in frontmatter
- Hextra uses `type: docs` layout for sidebar navigation. The `lessons/_index.md`
  has `cascade: type: docs` so all lessons get the sidebar automatically.
- The navbar is configured via `[[menu.main]]` entries in `hugo.toml`.
  Currently has: Lessons link + Search.

---

## Known Issues / Environment Notes

- Use **Git Bash** on Windows for all `hugo` and `git` commands — PowerShell
  may not have Hugo on its PATH
- Hugo is available in Git Bash but **not in VS Code's default PowerShell terminal**.
  If running Hugo from VS Code terminal, make sure the terminal is set to Git Bash.
- Hugo submodule must be initialized before first build:
  `git submodule update --init --recursive`
- **`baseURL` gotcha:** `hugo.toml` sets `baseURL` to the GitHub Pages URL
  (`https://hanamichisakuragiking.github.io/rootready/`). When running
  `hugo server` locally without `-b http://localhost:1313`, the site is served
  at `http://localhost:1313/rootready/` — the root `/` returns 404.
  Always use `hugo server -b http://localhost:1313` for local dev.
- **Do not use `relref` shortcodes to link to lessons that don't exist yet.**
  Hugo will refuse to build if the target page is missing. Use plain text
  like `Lesson 2 — Title (coming soon)` until the lesson is created.
- **`public/` is gitignored.** The site is built on GitHub Actions, not locally.
  Never commit the `public/` folder.
- **`.hugo_build.lock` and `.trunk/` are gitignored.**
