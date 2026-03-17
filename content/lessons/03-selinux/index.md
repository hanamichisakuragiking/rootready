---
title: "Lesson 3 — Understanding SELinux"
description: "You will learn what SELinux is, how to read its status, what enforcing and permissive modes mean, and how to find and fix access denials — without ever turning SELinux off."
date: 2026-03-16
weight: 3
tags: ["selinux", "security", "beginner", "rocky-linux"]
---

## What You Will Learn

By the end of this lesson you will have:

- Understood what SELinux is and why Rocky Linux ships with it enabled
- Checked whether SELinux is running and in what mode
- Learned the difference between **enforcing**, **permissive**, and **disabled** modes
- Read an SELinux denial from the audit log and understood what it means
- Fixed a denial by restoring a file context and by toggling an SELinux boolean
- Understood why turning SELinux off is the wrong answer

These are the skills that turn confusing "permission denied" messages into solvable problems. Every subsequent lesson assumes SELinux is enforcing — so learning to work with it now saves you hours of frustration later.

---

## What You Need Before Starting

- **A Rocky Linux 9 VM** with a non-root `student` user and SSH access, from [Lesson 2]({{< relref "/lessons/02-first-steps" >}})
- **An SSH connection** to the VM as `student`
- **About 30 minutes** of your time

> **Do not disable SELinux.** A common mistake is to turn SELinux off as soon as it causes trouble. This lesson will show you how to read what SELinux is doing and fix the actual problem. Disabling SELinux removes a core security layer that protects your server.

---

## Step 1 — What SELinux Is and Why It Matters

Every Linux system has standard file permissions — the `rwx` bits you see when you run `ls -l`. These control who can read, write, or execute a file based on user and group ownership. They are a good start, but they have a weakness: if a program is compromised, the attacker inherits everything that program was allowed to do.

**SELinux** (Security-Enhanced Linux) adds a second, completely separate layer of access control called **Mandatory Access Control (MAC)**. Even if a process has the right user permissions, SELinux can still block it from doing something it was not explicitly allowed to do.

SELinux was developed by the NSA and has been part of the Linux kernel since 2003. Red Hat-based distributions — including Rocky Linux — ship with it **enabled and enforcing by default**.

Here is the key idea: SELinux assigns a **label** (called a *context*) to every file, process, port, and socket. When a process tries to access a resource, the kernel checks the SELinux policy: is this process type allowed to access this resource type? If the policy says no, the access is denied and the event is logged.

This means:
- A web server process can only read files labelled as web content — not your SSH keys, not `/etc/shadow`.
- Even if an attacker exploits a bug in your web server, SELinux confines the damage to what the web server is allowed to do.

---

## Step 2 — Check SELinux Status

Connect to your VM as `student` and run:

```bash
getenforce
```

You should see:

```text
Enforcing
```

This tells you SELinux is active and blocking denied actions. For more detail, run:

```bash
sestatus
```

Expected output:

```text
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux mount point:            /sys/fs/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     denied
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

The important lines are:

| Line | What it means |
|---|---|
| `SELinux status: enabled` | SELinux is compiled in and running |
| `Loaded policy name: targeted` | The *targeted* policy is active — only certain high-risk processes are confined |
| `Current mode: enforcing` | Denials are blocked, not just logged |
| `Mode from config file: enforcing` | This mode will survive a reboot |

> **What is the targeted policy?**
> Rocky Linux uses the `targeted` policy by default. It applies SELinux confinement to network-facing services like web servers, SSH, and databases. Most other processes run as `unconfined_t` and are not restricted by SELinux. This balances security with usability.

---

## Step 3 — Understanding the Three Modes

SELinux operates in one of three modes:

### Enforcing

SELinux is fully active. Any access that violates the policy is **blocked** and **logged** in the audit log at `/var/log/audit/audit.log`. This is the correct mode for a production server.

```bash
sudo setenforce 1
```

(This switches to enforcing if you are currently in permissive mode. The change takes effect immediately but does not survive a reboot.)

### Permissive

SELinux is active but only **logs** violations — it does not block them. This mode is useful when you are configuring a new service and want to see what SELinux would deny without actually being blocked. It is **not safe** for a server that handles real data.

```bash
sudo setenforce 0
```

(Again, this is temporary — a reboot restores whatever the config file says.)

### Disabled

SELinux is completely turned off. The kernel does not enforce the policy and does not label new files. **Do not use this.** Re-enabling SELinux after it has been disabled requires a full filesystem relabel, which can take many minutes and can break your system if interrupted.

### Making a mode change permanent

To survive reboots, edit `/etc/selinux/config`:

```bash
sudo vi /etc/selinux/config
```

The relevant line looks like this:

```text
SELINUX=enforcing
```

Valid values are `enforcing`, `permissive`, and `disabled`. Leave it set to `enforcing`.

> **Never set SELINUX=disabled** on a server. If you need to debug a problem you suspect is SELinux-related, switch to `permissive` temporarily, reproduce the issue, check the logs, fix the context or boolean, then switch back to `enforcing`.

---

## Step 4 — Read and Understand AVC Denials

When SELinux blocks something, it writes an **AVC denial** (Access Vector Cache denial) to the audit log. Let us deliberately trigger one and read it.

### Trigger a denial

First, create a simple text file and move it to a location where the file context will be wrong:

```bash
echo "hello from the wrong context" | sudo tee /var/www/html/test.txt
```

> **Note:** The directory `/var/www/html` may not exist yet (Nginx is covered in Lesson 4). Create it first if needed:
> ```bash
> sudo mkdir -p /var/www/html
> ```

Now label the file with a context that does not match what a web server expects:

```bash
sudo chcon -t user_home_t /var/www/html/test.txt
```

This forcibly sets the SELinux type on the file to `user_home_t` — the type used for files in home directories. A web server would be denied access to a file with that type.

### Search the audit log

```bash
sudo ausearch -m AVC -ts recent
```

If you do not see output immediately, try:

```bash
sudo cat /var/log/audit/audit.log | grep AVC | tail -5
```

You will see a line like this (wrapped here for readability):

```text
type=AVC msg=audit(1710600000.123:456): avc:  denied  { read } for
  pid=1234 comm="httpd" name="test.txt" dev="sda1" ino=12345
  scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:user_home_t:s0
  tclass=file permissive=0
```

Here is how to read this:

| Field | Value | Meaning |
|---|---|---|
| `denied { read }` | read | The **read** action was blocked |
| `comm="httpd"` | httpd | The process that was blocked (the web server) |
| `scontext=…httpd_t…` | httpd_t | The **source** context — the process type |
| `tcontext=…user_home_t…` | user_home_t | The **target** context — the file type |
| `tclass=file` | file | The class of object (a file) |

Plain-English translation: **The web server process (`httpd_t`) tried to read a file, but the file was labelled `user_home_t` — a type the web server is not allowed to access.**

### Use `audit2why` for a human-readable explanation

The `audit2why` tool reads AVC messages and explains them in plain language:

```bash
sudo ausearch -m AVC -ts recent | audit2why
```

Output:

```text
type=AVC msg=audit(…):  avc:  denied  { read } for …
        Was caused by:
                Missing or wrong file context.
                (default_context_error)
        If you want to fix the label, use:
        # restorecon -v /var/www/html/test.txt
```

`audit2why` even tells you the fix. Let us apply it.

---

## Step 5 — Fix Denials: File Contexts and Booleans

SELinux problems almost always have one of two causes:

1. **Wrong file context** — a file has the wrong label (the most common cause)
2. **A boolean is not set** — a policy switch that enables or disables a specific permission is off

### Fix 1 — Restore the correct file context

```bash
sudo restorecon -v /var/www/html/test.txt
```

Output:

```text
Relabeled /var/www/html/test.txt from unconfined_u:object_r:user_home_t:s0
  to unconfined_u:object_r:httpd_sys_content_t:s0
```

`restorecon` looks up the correct context for that path in the SELinux policy and applies it. The file is now labelled `httpd_sys_content_t` — the type that web servers are allowed to read.

Verify the new label:

```bash
ls -Z /var/www/html/test.txt
```

Output:

```text
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/test.txt
```

> **`ls -Z` shows SELinux labels.** The `-Z` flag is available on `ls`, `ps`, `netstat`, and other common tools. It is the fastest way to inspect contexts on your system.

#### Relabel a whole directory

If you copy a lot of files into place at once, run `restorecon` recursively:

```bash
sudo restorecon -Rv /var/www/html/
```

### Fix 2 — Toggle an SELinux boolean

SELinux booleans are on/off switches built into the policy. They let you enable optional permissions without writing custom policy rules.

For example: by default, the web server is not allowed to make outbound network connections (useful if you want to prevent a compromised server from calling home). There is a boolean called `httpd_can_network_connect` that enables this.

List all booleans related to the web server:

```bash
getsebool -a | grep httpd
```

Example output (abbreviated):

```text
httpd_can_network_connect --> off
httpd_can_network_connect_db --> off
httpd_can_sendmail --> off
httpd_enable_cgi --> on
httpd_read_user_content --> off
```

To enable a boolean temporarily (until reboot):

```bash
sudo setsebool httpd_can_network_connect on
```

To enable it permanently (survives reboots):

```bash
sudo setsebool -P httpd_can_network_connect on
```

The `-P` flag writes the change to the SELinux policy store on disk so it persists. Always use `-P` for production changes.

Verify:

```bash
getsebool httpd_can_network_connect
```

Output:

```text
httpd_can_network_connect --> on
```

> **When do you need a boolean vs. `restorecon`?**
> - If the denial says the **file context is wrong**, use `restorecon`.
> - If the denial is about a **network connection, CGI execution, or sending mail**, a boolean is likely the answer.
> - `audit2why` will usually tell you which approach is needed.

---

## Troubleshooting

### A service won't start and you suspect SELinux

First, confirm SELinux is the cause. Switch to permissive mode and try starting the service:

```bash
sudo setenforce 0
sudo systemctl start myservice
```

If the service starts in permissive mode but not in enforcing mode, SELinux is blocking it. Switch back to enforcing:

```bash
sudo setenforce 1
```

Now check the audit log for the denial:

```bash
sudo ausearch -m AVC -ts recent | audit2why
```

Apply the fix (`restorecon` or `setsebool -P`) and try starting the service again in enforcing mode.

### `ausearch` returns no results

The `audit` daemon may not be running. Check:

```bash
sudo systemctl status auditd
```

If it is not active, start it:

```bash
sudo systemctl enable --now auditd
```

Alternatively, SELinux also logs to the kernel ring buffer. Try:

```bash
sudo dmesg | grep -i selinux | tail -20
```

### `restorecon` does not change the label

This can happen if the directory itself has a non-default context. Check the parent directory:

```bash
ls -Zd /var/www/html/
```

If the directory label looks wrong, run `restorecon` on it first:

```bash
sudo restorecon -v /var/www/html/
```

Then relabel the files inside it:

```bash
sudo restorecon -Rv /var/www/html/
```

### SELinux is blocking something but I cannot find an AVC in the log

If the process is running as `unconfined_t`, SELinux will not restrict it and will not log anything. Check the process context:

```bash
ps -eZ | grep myprocess
```

If it shows `unconfined_t`, SELinux is not the cause of your problem — check standard Linux permissions with `ls -l` instead.

---

## Summary

In this lesson you:

- **Learned what SELinux is** — a mandatory access control layer that confines processes to exactly what the policy allows, even if the attacker has exploited the process
- **Checked SELinux status** with `getenforce` and `sestatus` — understanding enforcing, permissive, and disabled modes and why enforcing is the correct setting for a server
- **Read an AVC denial** from the audit log — decoding the source context, target context, access class, and blocked action
- **Used `audit2why`** to get a plain-English explanation of a denial and a suggested fix
- **Fixed a file context** with `restorecon` — the most common SELinux fix
- **Toggled an SELinux boolean** with `setsebool -P` — enabling optional permissions without disabling SELinux

SELinux is no longer a black box. You can now read what it is blocking, understand why, and apply the correct fix — which makes every remaining lesson easier to debug.

---

**Next up:** [Lesson 4 — Installing and Hardening Nginx]({{< relref "/lessons/04-nginx" >}})
