# Tasks: RootReady Lesson Series

**Spec**: `specs/001-lesson-series/spec.md` | **Plan**: `specs/001-lesson-series/plan.md`

## Phase 1 — Lesson 1: Installing Rocky Linux in a VM ✅

- [x] Create `content/lessons/01-install-rocky/index.md`
- [x] Write frontmatter (title, description, date, weight:1, tags)
- [x] Write all sections (What You Will Learn → Summary)
- [x] Verify `hugo build` succeeds with zero errors
- [x] Commit and deploy

**Delivered**: Commit `1c55504`

---

## Phase 2 — Lesson 2: First Steps After Install ✅

- [x] Create `content/lessons/02-first-steps/index.md`
- [x] Write frontmatter (weight: 2, tags: users, ssh, firewalld)
- [x] Section: What You Will Learn
- [x] Section: What You Need Before Starting (reference Lesson 1)
- [x] Section: Step — Create a non-root user with sudo
- [x] Section: Step — Generate and install SSH keys
- [x] Section: Step — Disable root login and password auth
- [x] Section: Step — Configure firewalld basics
- [x] Section: Troubleshooting (locked out via SSH, firewall blocking)
- [x] Section: Summary
- [x] Update Lesson 1 "Next up" to real `relref` link
- [x] Verify `hugo build` succeeds
- [x] Commit

---

## Phase 3 — Lesson 3: Understanding SELinux

- [x] Create `content/lessons/03-selinux/index.md`
- [x] Write frontmatter (weight: 3, tags: selinux, security)
- [x] Section: What You Will Learn
- [x] Section: What You Need Before Starting (reference Lesson 2)
- [x] Section: Step — What SELinux is and why it matters
- [x] Section: Step — Check SELinux status with `getenforce` / `sestatus`
- [x] Section: Step — Understanding enforcing, permissive, disabled
- [x] Section: Step — Read and fix AVC denials with `audit2why`
- [x] Section: Step — Set file contexts and booleans
- [x] Section: Troubleshooting (service won't start due to SELinux)
- [x] Section: Summary
- [x] Update Lesson 2 "Next up" link
- [x] Verify `hugo build` succeeds
- [x] Commit

---

## Phase 4 — Lesson 4: Installing and Hardening Nginx ✅

- [x] Create `content/lessons/04-nginx/index.md`
- [x] Write frontmatter (weight: 4, tags: nginx, web-server)
- [x] Section: What You Will Learn
- [x] Section: What You Need Before Starting (reference Lesson 3)
- [x] Section: Step — Install Nginx via dnf
- [x] Section: Step — Enable and start with systemctl
- [x] Section: Step — Open firewall port 80/443
- [x] Section: Step — Set correct SELinux contexts
- [x] Section: Step — Serve a custom HTML page
- [x] Section: Step — Harden with security headers
- [x] Section: Troubleshooting (403 Forbidden, SELinux AVC on content)
- [x] Section: Summary
- [x] Update Lesson 3 "Next up" link
- [x] Verify `hugo build` succeeds
- [x] Commit

---

## Phase 5 — Lesson 5: Podman and Rootless Containers ✅

- [x] Create `content/lessons/05-podman-basics/index.md`
- [x] Write frontmatter (weight: 5, tags: podman, containers)
- [x] Section: What You Will Learn
- [x] Section: What You Need Before Starting (reference Lesson 4)
- [x] Section: Step — What containers are (beginner explanation)
- [x] Section: Step — Install Podman
- [x] Section: Step — Pull and run a container image
- [x] Section: Step — Run rootless as non-root user
- [x] Section: Step — Map ports and volumes
- [x] Section: Step — Stop and remove containers
- [x] Section: Troubleshooting (permission denied, port conflicts)
- [x] Section: Summary
- [x] Update Lesson 4 "Next up" link
- [x] Verify `hugo build` succeeds
- [x] Commit

---

## Phase 6 — Lesson 6: Quadlet and systemd Integration

- [x] Create `content/lessons/06-quadlet/index.md`
- [x] Write frontmatter (weight: 6, tags: quadlet, systemd, podman)
- [x] Section: What You Will Learn
- [x] Section: What You Need Before Starting (reference Lesson 5)
- [x] Section: Step — Why containers need a service manager
- [x] Section: Step — Write a `.container` Quadlet file
- [x] Section: Step — Enable and start with `systemctl --user`
- [x] Section: Step — Verify auto-restart behavior
- [x] Section: Troubleshooting (unit not found, ExecStart failure)
- [x] Section: Summary
- [x] Update Lesson 5 "Next up" link
- [x] Verify `hugo build` succeeds
- [x] Commit
- [x] Section: Step — `podman generate systemd` (legacy) vs Quadlet — why it was replaced
- [x] Section: Step — Inspect all container services with `systemctl --user list-units`
- [x] Section: Step — Configure `podman auto-update` with `AutoUpdate=registry` in the unit file
- [x] Troubleshooting: SELinux denial on the container image itself (not the volume mount)
- [x] Verify `hugo build` succeeds after additions
- [x] Commit additions

---

## Phase 7 — Lesson 7: Nginx as a Reverse Proxy

- [ ] Create `content/lessons/07-reverse-proxy/index.md`
- [ ] Write frontmatter (weight: 7, tags: nginx, reverse-proxy, podman)
- [ ] Section: What You Will Learn
- [ ] Section: What You Need Before Starting (reference Lesson 6)
- [ ] Section: Step — What a reverse proxy does (diagram/explanation)
- [ ] Section: Step — Configure Nginx upstream and proxy_pass
- [ ] Section: Step — Set proxy headers (Host, X-Real-IP, etc.)
- [ ] Section: Step — Fix SELinux for network connections (`httpd_can_network_connect`)
- [ ] Section: Step — Test with curl
- [ ] Section: Troubleshooting (502 Bad Gateway, SELinux boolean)
- [ ] Section: Summary
- [ ] Update Lesson 6 "Next up" link
- [ ] Verify `hugo build` succeeds
- [ ] Commit

---

## Phase 8 — Lesson 8: Logging with journald

- [ ] Create `content/lessons/08-logging/index.md`
- [ ] Write frontmatter (weight: 8, tags: journald, logging, systemd)
- [ ] Section: What You Will Learn
- [ ] Section: What You Need Before Starting (reference Lesson 7)
- [ ] Section: Step — What journald is and how it relates to systemd
- [ ] Section: Step — Use `journalctl` to read logs
- [ ] Section: Step — Filter by unit, priority, time range
- [ ] Section: Step — Configure persistent storage
- [ ] Section: Step — Read Podman container logs via journal
- [ ] Section: Troubleshooting (logs not persisting after reboot)
- [ ] Section: Summary
- [ ] Update Lesson 7 "Next up" link
- [ ] Verify `hugo build` succeeds
- [ ] Commit

---

## Phase 9 — Lesson 9: Preventing DoS with fail2ban

- [ ] Create `content/lessons/09-fail2ban/index.md`
- [ ] Write frontmatter (weight: 9, tags: fail2ban, security, firewalld)
- [ ] Section: What You Will Learn
- [ ] Section: What You Need Before Starting (reference Lesson 8)
- [ ] Section: Step — What fail2ban does (brute-force protection)
- [ ] Section: Step — Install from EPEL
- [ ] Section: Step — Configure jail.local for SSH
- [ ] Section: Step — Configure jail for Nginx
- [ ] Section: Step — Verify bans with `fail2ban-client status`
- [ ] Section: Troubleshooting (legitimate IP banned, jail not active)
- [ ] Section: Summary
- [ ] Update Lesson 8 "Next up" link
- [ ] Verify `hugo build` succeeds
- [ ] Commit

---

## Phase 10 — Lesson 10: CI/CD with GitHub Actions

- [ ] Create `content/lessons/10-cicd/index.md`
- [ ] Write frontmatter (weight: 10, tags: ci-cd, github-actions, deployment)
- [ ] Section: What You Will Learn
- [ ] Section: What You Need Before Starting (reference Lesson 9)
- [ ] Section: Step — What CI/CD means (beginner explanation)
- [ ] Section: Step — Create a workflow file (.github/workflows/)
- [ ] Section: Step — Build and test on push
- [ ] Section: Step — Deploy to the server (conceptual / example)
- [ ] Section: Step — Add a status badge to the repo
- [ ] Section: Troubleshooting (workflow syntax error, secrets not set)
- [ ] Section: Summary
- [ ] Update Lesson 9 "Next up" link
- [ ] Verify `hugo build` succeeds
- [ ] Commit

---

## Post-Completion

- [ ] All 10 "Next up" links are real `relref` links
- [ ] Full `hugo build --gc --minify` succeeds with zero errors
- [ ] All lessons render correctly on GitHub Pages
- [ ] `CLAUDE.md` updated with final syllabus status
- [ ] Feature spec stories updated to ✅ DELIVERED
