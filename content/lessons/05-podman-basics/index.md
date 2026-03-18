---
title: "Lesson 5 — Podman and Rootless Containers"
description: "You will learn what containers are, install Podman, pull and run images as a non-root user, map ports and volumes, and manage the full container lifecycle — all without ever needing sudo."
date: 2026-03-18
weight: 5
tags: ["podman", "containers", "security", "beginner", "rocky-linux"]
---

## What You Will Learn

By the end of this lesson you will have:

- Understood what containers are and why they are useful (in plain English)
- Installed Podman on your Rocky Linux server
- Pulled a container image from a public registry
- Run a container as the non-root `student` user — no `sudo` required
- Mapped a container's port to your server so it is reachable from a browser
- Mounted a directory from your server into a container (a *volume*)
- Stopped and removed containers cleanly

Containers are the building block for every lesson that follows. By the end of this lesson you will be comfortable running, inspecting, and cleaning up containers from the command line.

---

## What You Need Before Starting

- **A Rocky Linux 9 VM** with a non-root `student` user, SSH access, SELinux enforcing, and Nginx installed, from [Lesson 4]({{< relref "/lessons/04-nginx" >}})
- **An SSH connection** to the VM as `student`
- **About 30 minutes** of your time

---

## Step 1 — What Containers Are

Before running any commands, it is worth understanding what you are working with.

> **What is a container?**
> A container is a lightweight, self-contained package that includes an application and everything it needs to run — its code, runtime, libraries, and configuration. It runs in an isolated environment on your server but shares the host's operating system kernel. Think of it as a sealed lunchbox: everything the app needs is inside, nothing leaks out, and you can carry the same lunchbox to any server and it will behave identically.

Containers solve a common problem in software development: *"it works on my machine."* Because the container bundles its own dependencies, it runs the same way everywhere — on your laptop, on your server, in CI/CD.

### Containers vs. Virtual Machines

You already have a virtual machine — Rocky Linux running inside VirtualBox on your host computer. A VM virtualises an entire computer, including its own operating system kernel. A container is much lighter: it shares the host kernel and only packages the application and its dependencies. This makes containers start in milliseconds rather than minutes and use far less RAM and disk space.

| | Virtual Machine | Container |
|---|---|---|
| Starts in | Minutes | Milliseconds |
| Disk footprint | Gigabytes | Megabytes |
| OS kernel | Its own copy | Shared with host |
| Isolation | Full hardware emulation | Process + filesystem namespace |

### Why rootless?

Traditional Docker requires a background daemon running as root. If a container is compromised, an attacker may be able to escape to the host as root. **Rootless containers** run entirely as your regular user — no root daemon, no root privileges. Even if an attacker breaks out of the container, they land as `student`, not root. This is the safer default and the approach we use throughout this course.

> **What is Podman?**
> Podman (Pod Manager) is a container tool developed by Red Hat. Unlike Docker, Podman has no daemon — each `podman` command runs directly as the calling user. It is compatible with Docker's command syntax, so most Docker knowledge transfers directly. Rocky Linux ships Podman in its official repositories.

---

## Step 2 — Install Podman

Connect to your VM as `student` and install Podman:

```bash
sudo dnf install -y podman
```

Verify the installation:

```bash
podman --version
```

Expected output:

```text
podman version 4.9.4
```

(The exact version may differ — that is fine.)

Now confirm that Podman works without `sudo`. Run the built-in smoke-test image as `student`:

```bash
podman run --rm hello-world
```

> **What does `--rm` do?**
> The `--rm` flag tells Podman to automatically delete the container after it finishes running. Without it, stopped containers accumulate on disk. Use `--rm` for one-off commands you do not need to keep.

You will see Podman pull the image and then run it:

```text
Resolved "hello-world" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/hello-world:latest...
Getting image source signatures
...

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

The container ran, printed its message, and was deleted — all without `sudo`. Podman is working.

---

## Step 3 — Pull and Run a Container Image

The `hello-world` image is a toy. Let us run something more useful: **Nginx inside a container**.

> **What is a container image?**
> An image is a read-only template used to create containers. It contains the filesystem, binaries, and configuration for an application. Images are stored in *registries* — public servers you can pull from. The most widely used public registry is `docker.io` (Docker Hub). When you run `podman pull nginx`, Podman downloads the Nginx image from Docker Hub.

Pull the official Nginx image explicitly (this downloads it without running it):

```bash
podman pull docker.io/library/nginx:latest
```

List the images you now have locally:

```bash
podman images
```

Expected output:

```text
REPOSITORY                TAG         IMAGE ID      CREATED       SIZE
docker.io/library/nginx   latest      a72860cb95fd  2 weeks ago   192 MB
docker.io/library/hello-world  latest  d2c94e258dcb  11 months ago  13.3 kB
```

Run the Nginx container in the foreground so you can see its logs:

```bash
podman run --rm --name my-nginx docker.io/library/nginx:latest
```

> **What does `--name` do?**
> The `--name` flag gives the container a human-readable name. Without it, Podman assigns a random name like `admiring_hopper`. A named container is easier to reference in follow-up commands like `podman stop my-nginx`.

You will see Nginx start up and print access log lines. Press `Ctrl+C` to stop it.

The container stops and, because of `--rm`, is deleted. The image stays on disk — you do not need to pull it again.

---

## Step 4 — Run Rootless as a Non-Root User

Everything so far has run as `student` — you are already doing rootless containers. Let us make this explicit and understand why it matters.

Check which user is running the container process:

```bash
podman run --rm docker.io/library/nginx:latest whoami
```

Output:

```text
root
```

Wait — does this mean the container is running as root? Inside the container, the Nginx process thinks it is root. But this is an illusion created by Linux *user namespaces*.

> **What are user namespaces?**
> A user namespace maps container UIDs to unprivileged host UIDs. Inside the container, UID 0 (root) maps to your `student` UID (e.g. 1000) on the host. The process has no real root privileges on the host — it is fully contained. This is the core mechanism that makes rootless containers safe.

Check what the container process looks like from the host's perspective. Open a second SSH session (or run the container in the background with `-d` first) and inspect:

```bash
# Run the container in detached (background) mode
podman run -d --name rootless-test docker.io/library/nginx:latest

# Check the process on the host
ps aux | grep nginx | grep -v grep
```

You will see the Nginx processes owned by `student` (UID ~1000), not `root` (UID 0). The container thinks it is root; the host knows it is `student`.

Clean up:

```bash
podman stop rootless-test
podman rm rootless-test
```

---

## Step 5 — Map Ports and Volumes

A container is isolated by default — nothing can reach it from outside, and it cannot see your server's files. Port mapping and volume mounts break those barriers in a controlled way.

### Port mapping

You want to serve the Nginx container on port **8080** of your VM (we leave port 80 for the system Nginx from Lesson 4). Map container port 80 to host port 8080 with the `-p` flag:

```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx:latest
```

> **What does `-p 8080:80` mean?**
> The format is `-p HOST_PORT:CONTAINER_PORT`. Any request that arrives at port 8080 on your VM is forwarded into the container on port 80. The container itself always listens on 80 — you decide which host port it appears on.

> **What does `-d` do?**
> The `-d` (detached) flag runs the container in the background so your terminal is freed up. The container keeps running after the command returns.

Open the firewall for port 8080:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

Test from the VM itself:

```bash
curl http://localhost:8080
```

You should see the default Nginx HTML page. You can also open `http://YOUR-VM-IP:8080` in a browser on your host.

### Volume mounts

A volume mount lets the container read files from your server's filesystem. This is how you give the containerised Nginx your own HTML content without rebuilding the image.

Create a directory for your content:

```bash
mkdir -p ~/www
echo '<h1>Hello from a rootless container!</h1>' > ~/www/index.html
```

Stop and remove the current container, then start a new one that mounts your directory:

```bash
podman stop web
podman rm web

podman run -d --name web \
  -p 8080:80 \
  -v ~/www:/usr/share/nginx/html:Z \
  docker.io/library/nginx:latest
```

> **What does the `:Z` flag do on the volume mount?**
> The `:Z` suffix tells Podman to relabel the mounted directory with an SELinux context that the container process is allowed to read. Without `:Z`, SELinux will block the container from reading files in your home directory. This is the rootless-container equivalent of `restorecon` — Podman handles it automatically when you add `:Z`.

Test again:

```bash
curl http://localhost:8080
```

Expected output:

```text
<h1>Hello from a rootless container!</h1>
```

Your custom page is now being served by Nginx running inside a rootless container.

---

## Step 6 — Stop and Remove Containers

Good housekeeping prevents disk space and confusion from accumulating.

### List running containers

```bash
podman ps
```

Expected output:

```text
CONTAINER ID  IMAGE                           COMMAND               CREATED      STATUS      PORTS                 NAMES
a1b2c3d4e5f6  docker.io/library/nginx:latest  nginx -g daemon o...  2 hours ago  Up 2 hours  0.0.0.0:8080->80/tcp  web
```

### List all containers (including stopped ones)

```bash
podman ps -a
```

The `-a` flag shows stopped containers too. Stopped containers still consume disk space until you remove them.

### Stop a running container

```bash
podman stop web
```

This sends `SIGTERM` to the container's main process, giving it a chance to shut down gracefully. After a timeout (default 10 seconds), Podman sends `SIGKILL` if the process has not exited.

### Remove a stopped container

```bash
podman rm web
```

### Remove an image

If you no longer need an image:

```bash
podman rmi docker.io/library/nginx:latest
```

> **Containers vs. images:** Removing a container does not remove the image it was created from. The image stays on disk so you can create new containers from it quickly. Use `podman rmi` to explicitly delete an image.

### Remove everything at once

During development, you often want to wipe all stopped containers:

```bash
podman container prune
```

Podman will ask for confirmation before deleting. This is safe to run at any time — it only removes *stopped* containers, never running ones.

---

## Troubleshooting

### `permission denied` when running without sudo

If you see an error like:

```text
Error: OCI runtime permission denied error: ...
```

Check that your user has access to the container runtime files:

```bash
ls -la /run/user/$(id -u)/
```

If the directory is missing or has wrong permissions, log out and back in to regenerate the user session. Podman's rootless mode depends on a user *login session* being active.

Also confirm that `newuidmap` and `newgidmap` are installed (required for user namespace mapping):

```bash
rpm -q shadow-utils
```

If missing:

```bash
sudo dnf install -y shadow-utils
```

### Port already in use

If you see:

```text
Error: rootlessport cannot expose privileged port 80
```

You are trying to map to a port below 1024. Rootless containers cannot bind to privileged ports by default. Use a port above 1024 (we use 8080 in this lesson for exactly this reason).

If you see a different "port in use" error, something else is already listening on that port:

```bash
sudo ss -tlnp | grep :8080
```

Either stop the other process or choose a different host port with `-p 9090:80`.

### SELinux blocking the container from reading a volume

If `curl http://localhost:8080` returns a 403 Forbidden after adding a volume mount, and you forgot the `:Z` suffix:

```bash
podman stop web && podman rm web
podman run -d --name web \
  -p 8080:80 \
  -v ~/www:/usr/share/nginx/html:Z \
  docker.io/library/nginx:latest
```

The `:Z` relabels the host directory so the container's SELinux context can read it. Without it, you will see AVC denials in the audit log:

```bash
sudo ausearch -m AVC -ts recent | audit2why
```

### Container exits immediately

If `podman ps` shows the container is already stopped seconds after you started it:

```bash
podman logs web
```

`podman logs` prints the container's stdout and stderr — the same output you would see if you ran it in the foreground. Look for error messages from the application itself.

---

## Summary

In this lesson you:

- **Learned what containers are** — lightweight, self-contained application packages that share the host kernel but run in an isolated environment, solving the "works on my machine" problem
- **Understood rootless containers** — how user namespaces map container root to an unprivileged host UID, so a compromised container cannot escalate to real root on the host
- **Installed Podman** with `dnf install podman` — Rocky Linux's daemon-less, rootless-by-default container runtime
- **Pulled and ran a container image** with `podman pull` and `podman run` — fetching from Docker Hub and running it entirely as `student`
- **Mapped ports** with `-p 8080:80` — forwarding host port 8080 into the container's port 80 so it is reachable from a browser
- **Mounted a volume** with `-v ~/www:/usr/share/nginx/html:Z` — giving the container access to host files while letting SELinux control access via the `:Z` relabel flag
- **Managed the container lifecycle** with `podman stop`, `podman rm`, and `podman container prune` — keeping your server tidy

Your server can now run containerised applications as a regular user, with the firewall and SELinux still fully enforced.

---

**Next up:** Lesson 6 — Quadlet and systemd Integration (coming soon)
