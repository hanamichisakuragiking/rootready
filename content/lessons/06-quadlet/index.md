---
title: "Lesson 6 — Quadlet and systemd Integration"
description: "You will learn how to turn a Podman container into a proper systemd service using a Quadlet unit file, so your container starts automatically on boot, restarts if it crashes, and is managed exactly like any other service on the system."
date: 2026-03-19
weight: 6
tags: ["quadlet", "systemd", "podman", "containers", "beginner", "rocky-linux"]
---

## What You Will Learn

By the end of this lesson you will have:

- Understood why `podman run` alone is not enough for production
- Learned what Quadlet is and how it fits into the systemd ecosystem
- Written a `.container` Quadlet unit file from scratch
- Started, stopped, and inspected your container as a **user systemd service**
- Enabled **linger** so the service survives SSH logouts and server reboots
- Tested automatic restart by deliberately stopping the container
- Understood what to do when systemd cannot find a unit or fails to start one

This lesson is the bridge between "I can run a container manually" and "my container is a service that manages itself." Every lesson from here on assumes you have a container running as a reliable systemd service.

---

## What You Need Before Starting

- **A Rocky Linux 9 VM** with Podman installed and a working rootless container workflow, from [Lesson 5]({{< relref "/lessons/05-podman-basics" >}})
- **An SSH connection** to the VM as `student`
- **About 30 minutes** of your time

> **Podman version requirement:** Quadlet is built into Podman 4.4 and later. Rocky Linux 9 ships Podman 4.x, so you are covered. If you are on an older system, check `podman --version` before continuing.

---

## Step 1 — Why Containers Need a Service Manager

In Lesson 5 you ran containers with commands like:

```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx:latest
```

This works perfectly for experimenting. For a real server it has three critical problems.

### Problem 1 — Containers do not survive reboots

When your VM reboots, that `podman run` command is gone. Nothing will restart the container. Your application is down until someone logs in and runs the command manually.

### Problem 2 — Containers do not restart after crashes

If the containerised application panics and exits, it stays down. There is no watchdog. In production, transient failures — out-of-memory events, a momentarily unavailable database, a network blip — should not require human intervention.

### Problem 3 — There is no standard way to manage them

With bare `podman run`, there is no `systemctl status myapp`, no `journalctl -u myapp`, no dependency ordering with other services. You are outside the tooling every Linux administrator already knows.

### The solution: systemd

> **What is systemd?**
> systemd is the init system and service manager in Rocky Linux. It is PID 1 — the first process the kernel starts. Everything else on the system is a child of systemd. It is responsible for starting all services at boot, restarting them on failure, and collecting their logs. You have already used it with `systemctl start nginx` and `systemctl status nginx` in Lesson 4.

systemd manages services through **unit files** — small text files that declare what a service is, how to start it, and what should happen when it stops. If you can describe your container in a unit file, systemd takes care of the rest: starting on boot, restarting on failure, log collection, dependency ordering.

> **What is Quadlet?**
> Quadlet is a systemd generator that ships inside Podman. It reads simple `.container` files you write, and automatically generates the correct systemd unit files from them. You never have to write raw `podman run` commands inside a unit file — Quadlet translates a clean declarative description into the right invocation. It was merged into Podman in version 4.4 and is the officially recommended way to run containers as services on Red Hat-family systems.

The key insight: with Quadlet, your container *becomes* a systemd service. You start it with `systemctl --user start`, check it with `systemctl --user status`, and read its logs with `journalctl --user -u`. It is the same workflow as every other service on the system.

---

## Step 2 — Understand the Two Levels of systemd

Before writing any files, you need to understand one thing that trips up almost everyone: systemd runs at **two levels**, and they use different commands and different directories.

### System-level services (`sudo systemctl`)

These services run as root, are managed by the system, and start before any user logs in. Nginx, sshd, and auditd are system-level services. Their unit files live in `/usr/lib/systemd/system/` and `/etc/systemd/system/`.

You interact with them using `sudo systemctl start nginx`.

### User-level services (`systemctl --user`)

These services run as your own user — in this case `student`. They start when that user's session begins and are managed entirely without root. Their unit files live in `~/.config/systemd/user/`. Quadlet `.container` files live in `~/.config/containers/systemd/`.

You interact with them using `systemctl --user start myapp` — **no sudo**.

Rootless Podman containers are always user-level services. This is correct and intentional — it means a compromised container cannot affect system services running as root.

> **The `--user` flag is not optional.** Running `sudo systemctl start myapp` will look for a system-level service named `myapp` and fail. You must use `systemctl --user` for anything managed by Quadlet.

---

## Step 3 — Write a Quadlet `.container` File

Quadlet files live in a specific directory that Podman's systemd generator watches:

```bash
mkdir -p ~/.config/containers/systemd
```

Create your first Quadlet unit file. We will run the same Nginx container from Lesson 5, but this time as a managed service:

```bash
vi ~/.config/containers/systemd/web.container
```

Enter the following content exactly:

```ini
[Unit]
Description=My web application (Nginx container)
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/library/nginx:latest
PublishPort=8080:80
Volume=%h/www:/usr/share/nginx/html:Z
Environment=NGINX_ENTRYPOINT_QUIET_LOGS=1

[Service]
Restart=always
RestartSec=5s

[Install]
WantedBy=default.target
```

Save and close the file (`:wq` in vi).

Now let us go through every line:

### `[Unit]` section

```ini
Description=My web application (Nginx container)
```
A human-readable label that appears in `systemctl status` output and in log entries.

```ini
After=network-online.target
Wants=network-online.target
```
This tells systemd: do not start this service until the network is up. Without this, the container might try to pull the image before the network interface is active on boot.

> **What is a target?**
> In systemd, a *target* is a named synchronisation point — a way to say "only start after these things are ready." `network-online.target` means "the network is configured and reachable." `default.target` is what the system reaches when it has fully booted into normal operation.

### `[Container]` section

This is the Quadlet-specific section. There is no equivalent in plain systemd — Quadlet translates these fields into the correct `podman run` arguments.

```ini
Image=docker.io/library/nginx:latest
```
The full image reference. Always use the full registry path (`docker.io/library/`) to avoid ambiguity — short names like `nginx` can resolve differently depending on registry configuration.

```ini
PublishPort=8080:80
```
Maps host port 8080 to container port 80. This is the Quadlet equivalent of `-p 8080:80`.

```ini
Volume=%h/www:/usr/share/nginx/html:Z
```
Mounts your `~/www` directory into the container. `%h` is a systemd specifier that expands to the home directory of the service user — so for `student`, `%h` becomes `/home/student`. The `:Z` suffix tells Podman to apply the correct SELinux label, exactly as in Lesson 5.

```ini
Environment=NGINX_ENTRYPOINT_QUIET_LOGS=1
```
Sets an environment variable inside the container. This particular one suppresses Nginx's startup banners to keep the journal tidy. You can set any variable your application needs here, one `Environment=` line per variable.

### `[Service]` section

This is a standard systemd `[Service]` section. Quadlet passes it through unchanged.

```ini
Restart=always
```
If the container stops for any reason — crash, OOM kill, or a manual `podman stop` — systemd will restart it. This is the auto-restart policy.

```ini
RestartSec=5s
```
Wait 5 seconds before restarting. Without a delay, a container that crashes immediately on startup will restart in a tight loop and overwhelm the system. 5 seconds gives the problem time to clear (e.g. a network resource becoming available) and gives you time to run `systemctl --user stop web` if you need to intervene.

### `[Install]` section

```ini
WantedBy=default.target
```
This makes the service start automatically when the system reaches normal operation. Without this line, the service exists but will never start on its own — you would have to start it manually every time.

---

## Step 4 — Enable and Start the Service

### Make sure the content directory exists

The volume mount expects `~/www` to exist. If you completed Lesson 5 it already does; if not, create it now:

```bash
mkdir -p ~/www
echo '<h1>Served by a Quadlet container service</h1>' > ~/www/index.html
```

### Reload the systemd daemon

Before systemd can see your new unit file, you need to tell it to re-read all unit files:

```bash
systemctl --user daemon-reload
```

> **Why is this required?**
> systemd reads unit files at startup and caches them. When you add a new file, the running daemon does not see it automatically. `daemon-reload` flushes the cache and re-reads everything in the unit file directories.

### Check that systemd found the unit

```bash
systemctl --user status web
```

If Quadlet parsed the file successfully you will see:

```text
○ web.service - My web application (Nginx container)
     Loaded: loaded (/home/student/.config/containers/systemd/web.container; generated)
     Active: inactive (dead)
```

The key word is `generated` — systemd found the `.container` file and Quadlet generated a `.service` unit from it. If you see `Unit web.service could not be found`, go to the **Troubleshooting** section.

### Enable the service

```bash
systemctl --user enable web
```

Enabling a service creates the symlink that causes it to start at login. You will see:

```text
Created symlink /home/student/.config/systemd/user/default.target.wants/web.service
  → /run/user/1000/systemd/generator/web.service
```

### Start the service

```bash
systemctl --user start web
```

### Verify it is running

```bash
systemctl --user status web
```

You should see:

```text
● web.service - My web application (Nginx container)
     Loaded: loaded (/home/student/.config/containers/systemd/web.container; generated)
     Active: active (running) since Wed 2026-03-19 10:00:00 UTC; 3s ago
   Main PID: 5678 (conmon)
      Tasks: 11
     CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/web.service
             ├─5678 /usr/bin/conmon ...
             └─5690 nginx: master process nginx ...
```

`active (running)` confirms the container is up. Test it:

```bash
curl http://localhost:8080
```

Expected output:

```text
<h1>Served by a Quadlet container service</h1>
```

---

## Step 5 — Enable Linger So the Service Survives Logouts and Reboots

Right now there is a critical gap. Log out of your SSH session and log back in, then check:

```bash
systemctl --user status web
```

You may find the service has stopped. This is because by default, **user systemd instances only run while the user has an active login session**. The moment your last SSH connection closes, the user systemd instance shuts down and takes all user services with it.

The fix is a single command called **linger**.

> **What is linger?**
> Linger is a systemd feature that keeps a user's systemd instance running permanently — even when the user is not logged in. When linger is enabled for a user, their services start at boot (before any login) and keep running after logout. Without linger, user services are tied to login sessions.

Enable linger for `student`:

```bash
sudo loginctl enable-linger student
```

Verify it is set:

```bash
loginctl show-user student | grep Linger
```

Expected output:

```text
Linger=yes
```

Now test the full boot cycle. First, start the service if it is not already running:

```bash
systemctl --user start web
```

Then reboot the VM:

```bash
sudo reboot
```

Wait about 30 seconds, then SSH back in and check:

```bash
systemctl --user status web
```

The service should show `active (running)` — started automatically at boot, with no manual intervention. This is the independent test from the spec: **reboot → container auto-starts**.

---

## Step 6 — Verify Auto-Restart Behaviour

The `Restart=always` policy means systemd will restart the container if it stops unexpectedly. Let us prove it works.

First, find the container ID using Podman:

```bash
podman ps
```

Expected output:

```text
CONTAINER ID  IMAGE                           COMMAND               CREATED      STATUS      PORTS                 NAMES
a1b2c3d4e5f6  docker.io/library/nginx:latest  nginx -g daemon o...  5 min ago    Up 5 min    0.0.0.0:8080->80/tcp  systemd-web
```

Notice the container is named `systemd-web` — Quadlet prefixes the service name with `systemd-` when creating containers.

Stop the container directly with Podman (simulating a crash):

```bash
podman stop systemd-web
```

Now watch what happens. Run these two commands quickly one after the other:

```bash
systemctl --user status web
```

You will briefly see `activating (start)` or even `failed` — and then, within 5 seconds (the `RestartSec` you set), it flips back to:

```text
Active: active (running)
```

systemd detected that the service stopped and restarted it automatically. The 5-second `RestartSec` delay is visible in the timestamps.

Confirm the container is back:

```bash
podman ps
curl http://localhost:8080
```

Both should succeed. Your container is now self-healing.

### What the restart policy does NOT cover

`Restart=always` restarts the service when it stops. It does **not** restart it if you explicitly stop it with `systemctl --user stop web`. A deliberate stop is treated as intentional and is not overridden. This is correct behaviour — you always have manual control.

---

## Reading the Service Logs

Because the container is a systemd service, all its output goes to the journal. You can read it with `journalctl`:

```bash
journalctl --user -u web -f
```

> **What does `-f` do?**
> The `-f` flag means "follow" — `journalctl` stays open and prints new log lines as they arrive, like `tail -f` for a file. Press `Ctrl+C` to stop following.

The `-u web` flag filters to only the `web.service` unit. You will see Nginx access log lines every time a request hits the container.

To see the last 50 lines without following:

```bash
journalctl --user -u web -n 50 --no-pager
```

To see logs since a specific time:

```bash
journalctl --user -u web --since "10 minutes ago"
```

Logging is covered in depth in Lesson 8. For now, knowing that `journalctl --user -u web` gives you your container's logs is enough.

---

## Step 7 — `podman generate systemd` (Legacy) vs Quadlet

If you search the internet for "run Podman container as service," you will find many tutorials using a command called `podman generate systemd`. This is the **old way**. It is important to understand what it was, why it was replaced, and why you should not use it for new work.

### What `podman generate systemd` did

```bash
podman generate systemd --name mycontainer --files --new
```

This command inspected a running container and generated a raw `systemd` unit file containing a long, auto-generated `ExecStart=` line — essentially a `podman run ...` command with every flag filled in:

```ini
[Service]
ExecStart=/usr/bin/podman run \
  --cidfile=/run/user/1000/mycontainer.cid \
  --cgroups=no-conmon \
  --rm \
  --sdnotify=conmon \
  -d \
  --replace \
  --name mycontainer \
  -p 8080:80 \
  docker.io/library/nginx:latest
ExecStop=/usr/bin/podman stop --ignore --cidfile /run/user/1000/mycontainer.cid
ExecStopPost=/usr/bin/podman rm -f --ignore --cidfile /run/user/1000/mycontainer.cid
```

It worked, but the generated files were long, fragile, and tightly coupled to the container that existed at generation time. If the container changed, you had to regenerate the file. If you made a mistake, you were editing raw `podman run` flags in a unit file — not a pleasant experience.

### Why `podman generate systemd` was deprecated

Red Hat deprecated `podman generate systemd` in Podman 4.4 (2022) and removed it entirely in Podman 5.0 (2024). The reasons were:

1. **Fragility** — generated files rotted when container configuration drifted.
2. **Readability** — the output was not human-friendly and was hard to review in code.
3. **No declarative source of truth** — the unit file was derived from a running container, not from a version-controlled description of intent.
4. **Quadlet already existed** — Quadlet was integrated into Podman at the same time the old command was deprecated. It solves all the problems above with a clean, declarative `.container` file format.

### Why you are using Quadlet

Quadlet gives you:

- **A short, readable file** — 15–20 lines instead of 50+
- **A declarative description** — you describe *what* you want, not *how* to invoke Podman
- **Version-control friendliness** — `.container` files are easy to diff, review, and audit
- **Automatic regeneration** — Quadlet re-generates the underlying unit file every time you run `daemon-reload`. You never touch the generated output.
- **Official support** — Quadlet is the documented, supported path on RHEL 9 and Rocky Linux 9

> **Rule of thumb:** If a tutorial tells you to run `podman generate systemd`, it is outdated. Use Quadlet instead.

Rocky Linux 9 ships with Podman 4.x where Quadlet is available and `podman generate systemd` still exists for backward compatibility, but you should treat it as off-limits for all new work.

---

## Step 8 — Inspect All Running Container Services

When you have more than one Quadlet service — perhaps a database, a web application, and a background worker — you need a way to see all of them at once without running `systemctl --user status` on each one individually.

### List all user units matching a pattern

```bash
systemctl --user list-units 'pod*'
```

> **What does `pod*` match?**
> Quadlet-generated services are named after your `.container` file (e.g. `web.service`), but they all rely on Podman under the hood. The `pod*` pattern also matches `podman.service` and `podman-auto-update.timer` if those are active. For your own services, use a pattern that matches your naming convention — for example, if all your services start with `app-`, use `app-*`.

A more targeted approach — list only `.service` units that are currently active or failed:

```bash
systemctl --user list-units --type=service --state=active,failed
```

Expected output when `web.service` is running:

```text
  UNIT        LOAD   ACTIVE SUB     DESCRIPTION
  web.service loaded active running My web application (Nginx container)

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state.
SUB    = The low-level unit activation sub-state.

1 loaded units listed.
```

### List all unit files (including inactive services)

To see every unit Quadlet has generated — whether it is running or not:

```bash
systemctl --user list-unit-files --type=service
```

This shows every `.service` file systemd knows about, along with its enabled/disabled state:

```text
UNIT FILE    STATE    PRESET
web.service  enabled  disabled
```

`enabled` means the service has a `WantedBy=default.target` symlink and will start on boot. `disabled` means it exists but will not start automatically.

### Check a specific service quickly

```bash
systemctl --user is-active web
```

Returns `active` or `inactive` — useful in scripts where you need a one-word answer without the full status block.

```bash
systemctl --user is-failed web
```

Returns `failed` if the service hit a restart limit, `active` otherwise.

---

## Step 9 — Keep Container Images Up to Date with `podman auto-update`

Quadlet containers pull their image when the service first starts. After that, the image is cached locally and the container keeps using it — even if a newer version is pushed to the registry. To keep images current, Podman provides `podman auto-update`.

### Add `AutoUpdate=registry` to the unit file

Open your `.container` file:

```bash
vi ~/.config/containers/systemd/web.container
```

Add one line to the `[Container]` section:

```ini
[Container]
Image=docker.io/library/nginx:latest
PublishPort=8080:80
Volume=%h/www:/usr/share/nginx/html:Z
Environment=NGINX_ENTRYPOINT_QUIET_LOGS=1
AutoUpdate=registry
```

Save and reload:

```bash
systemctl --user daemon-reload
```

> **What does `AutoUpdate=registry` do?**
> It tells `podman auto-update` to check the remote registry for a newer digest of the image. If a newer version is found, Podman pulls it, restarts the service with the new image, and records the update in the journal. If the new image causes the service to fail, Podman automatically rolls back to the previous image — a safety net for production deployments.

### Run a dry-run check

Before any images are updated, preview what would change:

```bash
podman auto-update --dry-run
```

Expected output when your image is already up to date:

```text
            UNIT             CONTAINER              IMAGE                              POLICY    UPDATED
web.service  systemd-web (abc123de)  docker.io/library/nginx:latest  registry  false
```

The `UPDATED` column shows `false` — the local image matches the registry. If an update is available, it shows `pending`.

Expected output when an update is available:

```text
            UNIT             CONTAINER              IMAGE                              POLICY    UPDATED
web.service  systemd-web (abc123de)  docker.io/library/nginx:latest  registry  pending
```

### Run an actual update

```bash
podman auto-update
```

Podman checks every container with `AutoUpdate=registry`, pulls any updated images, and restarts affected services. The output confirms which services were updated.

### Automate updates with the systemd timer

Rocky Linux ships a systemd timer that runs `podman auto-update` automatically:

```bash
systemctl --user enable --now podman-auto-update.timer
```

Check when it last ran and when it will run next:

```bash
systemctl --user list-timers podman-auto-update.timer
```

Expected output:

```text
NEXT                        LEFT          LAST                        PASSED  UNIT                       ACTIVATES
Fri 2026-03-20 00:00:00 UTC 13h left      Thu 2026-03-19 00:00:00 UTC 10h ago podman-auto-update.timer   podman-auto-update.service
```

By default the timer fires once daily (at midnight). This means your containers will pick up security patches and new releases automatically — without any manual intervention — and roll back safely if the new image breaks the service.

> **Security note:** Automatic image updates are a trade-off. They protect you from known vulnerabilities in the image but introduce the risk of a registry change breaking your service. For experiments and learning, `AutoUpdate=registry` is fine. For a critical production service, consider pinning to a specific image digest and updating on a controlled schedule.

---

## Troubleshooting

### `Unit web.service could not be found`

This means systemd did not find or parse the `.container` file. Work through this checklist:

**1 — Check the file is in the right directory:**

```bash
ls -la ~/.config/containers/systemd/
```

You should see `web.container` listed. If the directory is empty, you may have saved the file in the wrong location.

**2 — Check for syntax errors in the file:**

```bash
/usr/lib/systemd/system-generators/podman-system-generator --user --dry-run 2>&1
```

This runs the Quadlet generator manually and prints any parse errors. Look for lines mentioning your file name.

**3 — Re-run daemon-reload and check again:**

```bash
systemctl --user daemon-reload
systemctl --user list-unit-files | grep web
```

If `web.service` does not appear in the list, the generator failed. Re-read the error output from step 2.

**4 — Verify Podman version:**

```bash
podman --version
```

Quadlet requires Podman 4.4 or later. If you see a version below 4.4, update Podman:

```bash
sudo dnf update podman
```

### `ExecStart` failure — service starts then immediately stops

Check the journal for the error from the container itself:

```bash
journalctl --user -u web -n 30 --no-pager
```

Common causes:

**Image not found locally and pull fails:**
The image must either be pre-pulled or the system must have internet access. Pre-pull it manually:

```bash
podman pull docker.io/library/nginx:latest
```

**Volume directory does not exist:**
If `~/www` does not exist, the mount fails silently and Nginx cannot find its content. Create it:

```bash
mkdir -p ~/www
echo '<h1>hello</h1>' > ~/www/index.html
```

**Port already in use:**
If something else is already listening on port 8080, the container fails to bind. Check:

```bash
sudo ss -tlnp | grep :8080
```

Stop the conflicting process or change the `PublishPort` in the `.container` file to a different host port (e.g. `9090:80`), then run `systemctl --user daemon-reload` and restart.

### Service is running but `curl http://localhost:8080` returns connection refused

The container is up but the port mapping is not working. Check the actual Podman port mapping:

```bash
podman port systemd-web
```

Expected output:

```text
80/tcp -> 0.0.0.0:8080
```

If the output is empty, the `PublishPort` line in the `.container` file may have a typo. Correct it, reload the daemon, and restart the service.

Also check that the firewall rule for port 8080 is still present (you added it in Lesson 5):

```bash
sudo firewall-cmd --list-ports
```

If `8080/tcp` is missing:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### Container restarts in a loop (`restart-limit` hit)

If the container crashes immediately on every start, systemd will restart it up to a limit (default 5 times in 10 seconds) and then mark the unit as `failed`:

```bash
systemctl --user status web
# shows: active (failed) — start-limit-hit
```

Reset the failure counter and investigate before restarting:

```bash
systemctl --user reset-failed web
journalctl --user -u web -n 50 --no-pager
```

Read the log carefully — the application's own error message is usually the clue. Fix the underlying problem (wrong image name, missing volume, bad environment variable) in the `.container` file, reload the daemon, and start again.

### SELinux denies the container image itself (not the volume mount)

In Lesson 5 and earlier in this lesson, you applied the `:Z` suffix to volume mounts so that SELinux labels the *host directory* correctly for container access. But sometimes SELinux denies access to the **container image layers** themselves — a different problem with a different fix.

This happens when the container runtime's own file contexts are wrong, typically after a system update that changes SELinux policy without relabelling existing files.

**How to identify this denial:**

Run `ausearch` to find recent AVC denials:

```bash
ausearch -m AVC -ts recent
```

A denial caused by the container image (not a volume) looks like this:

```text
type=AVC msg=audit(1710844800.123:456): avc:  denied  { read } for
  pid=5678 comm="podman" name="overlay" dev="sda1" ino=98765
  scontext=system_u:system_r:container_runtime_t:s0
  tcontext=system_u:object_r:container_var_lib_t:s0
  tclass=dir permissive=0
```

The key is the `tcontext` — it involves `container_var_lib_t` or `container_file_t`, which are labels for Podman's storage directories, not your home directory or a mounted volume.

**Step 1 — Pass the denial through `audit2why` for a plain-English explanation:**

```bash
ausearch -m AVC -ts recent | audit2why
```

`audit2why` reads the AVC denial and prints a human-readable explanation and suggested fix. If it suggests a boolean, set it. If it says the label is wrong, continue to Step 2.

**Step 2 — Restore the correct SELinux contexts on Podman's storage:**

```bash
sudo restorecon -Rv /var/lib/containers
```

`restorecon` reapplies the correct SELinux file contexts based on the system's file context policy. The `-R` flag means recursive (all subdirectories) and `-v` means verbose (print every change). This is the standard fix when a package update changes SELinux policy without relabelling existing data.

**Step 3 — Also relabel the user's Podman storage:**

```bash
restorecon -Rv ~/.local/share/containers
```

Note: no `sudo` here — this is your home directory.

**Step 4 — Restart the service and confirm:**

```bash
systemctl --user restart web
systemctl --user status web
```

If the service starts cleanly, the context was the problem and `restorecon` fixed it. If denials persist, run `ausearch -m AVC -ts recent | audit2why` again — the new output may point to a different cause.

> **Remember:** SELinux denials on volume mounts (`:Z` is missing or wrong) and SELinux denials on container image storage are two different problems. `:Z` on the volume fixes the first. `restorecon` on Podman's storage directories fixes the second. `audit2why` tells you which one you are dealing with.

---

### Linger is set but container does not start on reboot

Verify linger is actually enabled:

```bash
loginctl show-user student | grep Linger
```

If `Linger=no`, re-run:

```bash
sudo loginctl enable-linger student
```

Also confirm the `[Install]` section of your `.container` file contains `WantedBy=default.target` and that you ran `systemctl --user enable web`. Without the enable step, the `WantedBy` line has no effect — enabling is what creates the symlink that causes autostart.

---

## Summary

In this lesson you:

- **Understood why `podman run` is not enough for production** — containers started manually do not survive reboots or crashes, and they are invisible to systemd tooling
- **Learned what Quadlet is** — Podman's built-in systemd generator that translates `.container` files into valid systemd unit files, giving containers the same management interface as any other service
- **Distinguished system-level from user-level services** — Quadlet containers are user services managed with `systemctl --user`, requiring no root, keeping containerised workloads safely unprivileged
- **Wrote a complete `.container` unit file** — declaring the image, port mapping, volume mount with SELinux `:Z`, environment variables, restart policy, and boot target
- **Managed the service with systemctl** — `daemon-reload` to pick up new files, `enable` to register autostart, `start`/`stop`/`status` to operate it
- **Enabled linger** with `loginctl enable-linger student` — the essential step that lets user services start at boot and survive SSH logouts, proven by a full reboot test
- **Verified auto-restart** — stopped the container directly with `podman stop` and watched systemd bring it back within `RestartSec` seconds
- **Read service logs** with `journalctl --user -u web` — accessing the container's stdout through the standard systemd journal interface

Your container is now a proper, self-managing service. It starts on boot, restarts on failure, and is operated with the same tools as every other service on the system.

---

**Next up:** Lesson 7 — Nginx as a Reverse Proxy (coming soon)
