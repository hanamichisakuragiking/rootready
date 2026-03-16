# Implementation Plan: RootReady Lesson Series

**Branch**: `001-lesson-series` | **Date**: 2026-03-16 | **Spec**: `specs/001-lesson-series/spec.md`
**Input**: Feature specification covering Lessons 1–10 of the RootReady syllabus.

## Summary

Build a 10-lesson progressive tutorial series teaching complete beginners how to set up and secure a Linux server. The implementation is pure content authoring in Markdown within a Hugo static site using the Hextra theme. No application code — only lesson Markdown files, frontmatter, and supporting images.

## Technical Context

**Language/Version**: Markdown (Hugo-flavored)
**Primary Dependencies**: Hugo 0.157.0 Extended, Hextra theme (git submodule)
**Storage**: Git repository, GitHub Pages
**Testing**: Hugo build (zero errors), manual content review against constitution
**Target Platform**: Static website — GitHub Pages at `hanamichisakuragiking.github.io/rootready/`
**Project Type**: Content / documentation site
**Performance Goals**: N/A (static site)
**Constraints**: Must follow constitution principles (beginner-first, accuracy, progressive structure, accessibility, stack discipline)
**Scale/Scope**: 10 lessons, ~250–300 lines each, ~2,500–3,000 lines of Markdown total

## Constitution Check

| Principle                         | Status  | Notes                                                                                                  |
| --------------------------------- | ------- | ------------------------------------------------------------------------------------------------------ |
| I. Beginner-First Writing         | ✅ Gate | Every lesson must be reviewed for zero assumed knowledge, term definitions, and explain-before-showing  |
| II. Content Accuracy              | ✅ Gate | Every command must be verified on Rocky Linux 9 minimal                                                |
| III. Progressive Lesson Structure | ✅ Gate | Fixed section order, required frontmatter, no forward references                                       |
| IV. Accessibility                 | ✅ Gate | Language tags on code blocks, alt text on images, short paragraphs                                     |
| V. Stack Discipline               | ✅ Gate | Hugo + Hextra + Markdown only                                                                         |

## Project Structure

### Documentation (this feature)

```text
specs/001-lesson-series/
├── spec.md              # Feature specification (lessons as user stories)
├── plan.md              # This file
└── tasks.md             # Task breakdown by lesson
```

### Content (repository root)

```text
content/
├── _index.md                           # Site homepage
└── lessons/
    ├── _index.md                       # Lessons index (cascade: type: docs)
    ├── 01-install-rocky/
    │   └── index.md                    # ✅ DELIVERED
    ├── 02-first-steps/
    │   └── index.md                    # Users, SSH, firewalld
    ├── 03-selinux/
    │   └── index.md                    # Understanding SELinux
    ├── 04-nginx/
    │   └── index.md                    # Installing and Hardening Nginx
    ├── 05-podman-basics/
    │   └── index.md                    # Podman and Rootless Containers
    ├── 06-quadlet/
    │   └── index.md                    # Quadlet + systemd
    ├── 07-reverse-proxy/
    │   └── index.md                    # Nginx as a Reverse Proxy
    ├── 08-logging/
    │   └── index.md                    # Logging with journald
    ├── 09-fail2ban/
    │   └── index.md                    # Preventing DoS with fail2ban
    └── 10-cicd/
        └── index.md                    # CI/CD with GitHub Actions
```

## Lesson Template

Every lesson follows this exact structure:

```yaml
---
title: "Lesson N — Title Here"
description: "One sentence. What the reader will walk away knowing."
date: YYYY-MM-DD
weight: N
tags: ["tag1", "tag2"]
---
```

```markdown
## What You Will Learn
- Bullet list of outcomes

## What You Need Before Starting
- Prerequisites (reference prior lessons)

## Step 1 — [Action]
[Explain → Command → Expected output]

## Step N — [Action]
[...]

## Troubleshooting
### [Common problem 1]
[Solution]

### [Common problem 2]
[Solution]

## Summary
- Bullet recap

**Next up:** Lesson N+1 — Title (coming soon)
```

## Dependency Chain

Lessons must be written in order because each builds on the previous:

```text
L1 (VM) → L2 (Users/SSH) → L3 (SELinux) → L4 (Nginx) → L5 (Podman)
                                                              ↓
          L10 (CI/CD) ← L9 (fail2ban) ← L8 (Logging) ← L7 (Reverse Proxy) ← L6 (Quadlet)
```

After each lesson is complete:
1. Update the previous lesson's "Next up" line to be a real link
2. Update the syllabus status in `CLAUDE.md`
3. Verify `hugo build` passes with zero errors
