# Feature Specification: RootReady Lesson Series

**Feature Branch**: `001-lesson-series`
**Created**: 2026-03-16
**Status**: In Progress
**Input**: User description: "A progressive 10-lesson series taking complete beginners from installing Rocky Linux to running rootless Podman containers with CI/CD."

## User Scenarios & Testing

### ✅ User Story 1 — Installing Rocky Linux in a VM (Priority: P1) — DELIVERED

A reader with no Linux experience downloads Rocky Linux, installs VirtualBox, creates a VM, completes the Rocky Linux installation, and logs in for the first time.

**Why this priority**: This is the foundation — nothing else works without a running Linux server.

**Independent Test**: Reader can run `cat /etc/rocky-release` and see Rocky Linux version output.

**Acceptance Scenarios**:

1. **Given** a Windows/macOS/Linux computer with 4 GB RAM, **When** reader follows all steps, **Then** they have a Rocky Linux 9 VM they can log into as root.
2. **Given** a completed install, **When** reader runs `cat /etc/rocky-release`, **Then** output shows `Rocky Linux release 9.x`.
3. **Given** a running VM, **When** reader runs `shutdown now`, **Then** the VM powers off gracefully.

**Status**: ✅ Complete — `content/lessons/01-install-rocky/index.md`

---

### ✅ User Story 2 — Users, SSH, and firewalld (Priority: P1) — DELIVERED

A reader learns why running everything as root is dangerous, creates a regular user account with sudo access, enables SSH for remote access, and configures firewalld to allow only SSH traffic.

**Why this priority**: Security fundamentals — every subsequent lesson assumes a non-root user with sudo and SSH access.

**Independent Test**: Reader can SSH into the VM from their host machine as the new user and run a sudo command.

**Acceptance Scenarios**:

1. **Given** a Rocky Linux VM with only root, **When** reader creates a user and grants sudo, **Then** they can run `sudo whoami` and see `root`.
2. **Given** SSH is enabled, **When** reader connects from host terminal via `ssh user@ip`, **Then** they land in a shell on the VM.
3. **Given** firewalld is active, **When** reader runs `firewall-cmd --list-all`, **Then** only `ssh` service is listed.
4. **Given** firewalld is active, **When** reader tries to access an unlisted port, **Then** the connection is refused.

**Status**: ✅ Complete — `content/lessons/02-first-steps/index.md`

---

### User Story 3 — Understanding SELinux (Priority: P2)

A reader understands what SELinux is, why it exists, and how to read and fix SELinux denials. They learn the difference between enforcing, permissive, and disabled modes.

**Why this priority**: SELinux is on by default in Rocky Linux and causes confusion. Understanding it now prevents debugging pain in every future lesson.

**Independent Test**: Reader can trigger an SELinux denial, find it in the audit log, and resolve it.

**Acceptance Scenarios**:

1. **Given** SELinux is enforcing, **When** reader runs `getenforce`, **Then** output is `Enforcing`.
2. **Given** a deliberate SELinux violation, **When** reader checks `ausearch -m AVC`, **Then** they see the denial and understand the output.
3. **Given** an SELinux denial, **When** reader applies the correct fix (context/boolean), **Then** the operation succeeds.

---

### User Story 4 — Installing and Hardening Nginx (Priority: P2)

A reader installs Nginx, serves a basic static page, hardens the configuration (disable server tokens, set security headers), and confirms it works through the firewall.

**Why this priority**: Nginx is the web server and reverse proxy used in later lessons. Reader needs hands-on web server experience.

**Independent Test**: Reader can open a browser on the host and see their custom page served by Nginx on the VM.

**Acceptance Scenarios**:

1. **Given** Nginx is installed, **When** reader runs `systemctl status nginx`, **Then** it shows active/running.
2. **Given** firewalld allows HTTP, **When** reader opens `http://vm-ip` in a browser, **Then** they see their custom page.
3. **Given** hardened config, **When** reader checks response headers, **Then** `Server` header does not reveal Nginx version.

---

### User Story 5 — Podman and Rootless Containers (Priority: P2)

A reader understands what containers are, why rootless is better for security, pulls an image, runs a container, and manages its lifecycle — all without root.

**Why this priority**: Core skill — Podman is the container runtime used for all subsequent deployment lessons.

**Independent Test**: Reader can run `podman run` as a non-root user and see the container responding.

**Acceptance Scenarios**:

1. **Given** Podman is installed, **When** reader runs `podman run hello` as non-root, **Then** the container executes and produces output.
2. **Given** a running container, **When** reader runs `podman ps`, **Then** the container is listed.
3. **Given** a container, **When** reader stops and removes it, **Then** `podman ps -a` shows no containers.

---

### User Story 6 — Running Containers with Quadlet + systemd (Priority: P3)

A reader learns how to make containers start automatically on boot using Quadlet unit files instead of manual `podman run` commands. They also understand the history of container service management, how to inspect all running container services, how to keep images up to date automatically, and what to do when SELinux blocks the container image itself.

**Why this priority**: Production readiness — containers must survive reboots.

**Independent Test**: Reader reboots the VM and the container starts automatically.

**Acceptance Scenarios**:

1. **Given** a Quadlet `.container` file, **When** reader runs `systemctl --user start myapp`, **Then** the container starts.
2. **Given** linger is enabled, **When** the VM reboots, **Then** the container is running without manual intervention.
3. **Given** a running Quadlet service, **When** reader runs `systemctl --user status myapp`, **Then** status shows active.
4. **Given** `podman generate systemd` is mentioned, **When** reader reads the explanation, **Then** they understand it is the legacy approach and why Quadlet replaces it.
5. **Given** multiple Quadlet services are running, **When** reader runs `systemctl --user list-units 'pod*'`, **Then** they see all container units and their states at a glance.
6. **Given** `podman-auto-update` is configured with `AutoUpdate=registry` in the `.container` file, **When** reader runs `podman auto-update --dry-run`, **Then** they see which images have updates available without pulling them.
7. **Given** SELinux is enforcing and a container is denied despite a correctly labelled volume, **When** reader checks `ausearch -m AVC`, **Then** they can identify and fix a denial caused by the container image's own label rather than the volume mount.

---

### User Story 7 — Nginx as a Reverse Proxy (Priority: P3)

A reader configures Nginx to forward incoming HTTP requests to a container running on a high port, so the application is accessible on port 80.

**Why this priority**: Connects the web server (Lesson 4) to containers (Lessons 5–6) — the standard production pattern.

**Independent Test**: Reader opens `http://vm-ip` and sees the container application served through Nginx.

**Acceptance Scenarios**:

1. **Given** Nginx proxy config, **When** reader accesses `http://vm-ip`, **Then** they see the container app.
2. **Given** a reverse proxy, **When** the container restarts, **Then** Nginx begins serving it again automatically.
3. **Given** SELinux is enforcing, **When** Nginx proxies to the container port, **Then** no SELinux denials occur (correct booleans set).

---

### User Story 8 — Logging with journald (Priority: P3)

A reader learns how to read system and service logs using journalctl, filter by service/time/priority, and understand what the logs mean.

**Why this priority**: Observability — when something breaks, logs are the first place to look.

**Independent Test**: Reader can find and interpret logs from Nginx and their container using journalctl.

**Acceptance Scenarios**:

1. **Given** a running system, **When** reader runs `journalctl -u nginx --since "5 minutes ago"`, **Then** they see recent Nginx logs.
2. **Given** a container managed by Quadlet, **When** reader filters by the container unit, **Then** container stdout/stderr appears.
3. **Given** a log entry, **When** reader reads the priority/timestamp/message fields, **Then** they understand what happened.

---

### User Story 9 — Preventing DoS with fail2ban (Priority: P3)

A reader installs fail2ban, configures it to watch SSH and Nginx logs, and tests that repeated failed attempts result in an IP ban.

**Why this priority**: Security hardening — the last defensive layer before the server is "production ready."

**Independent Test**: Reader triggers a ban by making repeated failed SSH attempts and confirms their IP is blocked.

**Acceptance Scenarios**:

1. **Given** fail2ban is installed, **When** reader runs `fail2ban-client status sshd`, **Then** the jail is active.
2. **Given** 5 failed SSH login attempts, **When** reader checks `fail2ban-client status sshd`, **Then** their IP appears in the banned list.
3. **Given** a ban, **When** the ban duration expires (or reader unbans), **Then** the IP can connect again.

---

### User Story 10 — CI/CD with GitHub Actions (Priority: P3)

A reader creates a GitHub repo for a containerized app, writes a GitHub Actions workflow that builds the container image, and understands how to deploy updates.

**Why this priority**: Capstone — ties everything together with automation. Reader graduates from "manual server admin" to "automated deployment."

**Independent Test**: Reader pushes a code change and the GitHub Actions workflow runs successfully.

**Acceptance Scenarios**:

1. **Given** a GitHub repo with a Containerfile, **When** reader pushes to main, **Then** the Actions workflow triggers and builds the image.
2. **Given** a successful build, **When** reader checks the Actions tab, **Then** the run shows green.
3. **Given** a completed lesson series, **When** the reader reviews what they've learned, **Then** they have a full stack: VM → user/SSH → SELinux → Nginx → Podman → Quadlet → reverse proxy → logging → security → CI/CD.

---

### Edge Cases

- What if the reader's computer doesn't support virtualization (VT-x/AMD-V disabled)?
  → Covered in Lesson 1 Troubleshooting.
- What if the reader's ISP blocks SSH port 22?
  → Address in Lesson 2 with alternative port configuration.
- What if SELinux denials are too confusing for a beginner?
  → Lesson 3 starts with the "what" and "why" before any commands. Use real examples.
- What if a reader skips a lesson?
  → Every lesson lists prerequisites in "What You Need Before Starting." Sidebar weight enforces order.

## Requirements

### Functional Requirements

- **FR-001**: Every lesson MUST follow the structure defined in the constitution (What You Will Learn → content → Troubleshooting → Summary).
- **FR-002**: Every lesson MUST have valid frontmatter: `title`, `description`, `date`, `weight`, `tags`.
- **FR-003**: Every command shown MUST work on Rocky Linux 9 minimal install with the setup from prior lessons.
- **FR-004**: Every technical term MUST be defined on first use with a blockquote callout.
- **FR-005**: Every lesson MUST be completable in 20–30 minutes.
- **FR-006**: Lessons MUST build sequentially — no forward references to unexplained concepts.
- **FR-007**: Every lesson MUST include a Troubleshooting section with at least 2 common issues.
- **FR-008**: Every code fence MUST have a language identifier.
- **FR-009**: The site MUST build with `hugo build` with zero errors after each lesson is added.
- **FR-010**: Each lesson MUST link to the next lesson once it exists (plain text "coming soon" if it doesn't).

### Key Entities

- **Lesson**: A Markdown file at `content/lessons/NN-slug/index.md` with required frontmatter and sections.
- **Syllabus**: The ordered set of 10 lessons, controlled by `weight` in frontmatter.
- **VM**: The Rocky Linux 9 virtual machine that persists across all lessons — the reader's learning environment.

## Success Criteria

### Measurable Outcomes

- **SC-001**: All 10 lessons pass frontmatter and structure validation (spec-kit quality gates).
- **SC-002**: A reader with zero Linux experience can complete all 10 lessons in sequence without external help.
- **SC-003**: Every lesson builds successfully on GitHub Actions with zero Hugo errors.
- **SC-004**: By Lesson 10, the reader has: a secured VM, running a containerized app behind Nginx, with logging, fail2ban, and a CI/CD pipeline.
