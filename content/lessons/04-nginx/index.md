---
title: "Lesson 4 — Installing and Hardening Nginx"
description: "You will install Nginx, serve a custom static page through the firewall, set the correct SELinux contexts, and harden the server with security headers so its version is never revealed to visitors."
date: 2026-03-17
weight: 4
tags: ["nginx", "web-server", "security", "selinux", "beginner", "rocky-linux"]
---

## What You Will Learn

By the end of this lesson you will have:

- Installed Nginx using the `dnf` package manager
- Started and enabled Nginx as a systemd service so it survives reboots
- Opened ports 80 and 443 in firewalld so browsers can reach your server
- Set the correct SELinux file context on your web content
- Replaced the default Nginx page with your own custom HTML page
- Hardened Nginx by hiding its version number and adding security headers

The result is a running web server that serves your content to any browser on your network, locked down with firewall rules, SELinux confinement, and secure response headers.

---

## What You Need Before Starting

- **A Rocky Linux 9 VM** with a non-root `student` user, SSH access, and SELinux enforcing, from [Lesson 3]({{< relref "/lessons/03-selinux" >}})
- **An SSH connection** to the VM as `student`
- **About 30 minutes** of your time

---

## Step 1 — Install Nginx via dnf

> **What is dnf?**
> `dnf` (Dandified YUM) is Rocky Linux's package manager. It downloads and installs software from official repositories, similar to an app store for your server. Every package it installs is signed by the Rocky Linux project so you know it has not been tampered with.

First, make sure your package index is up to date, then install Nginx:

```bash
sudo dnf install -y nginx
```

The `-y` flag tells `dnf` to answer "yes" to all prompts so the install runs without interruption. You will see output similar to this as packages are downloaded and installed:

```text
Dependencies resolved.
================================================================================
 Package          Architecture  Version               Repository           Size
================================================================================
Installing:
 nginx            x86_64        1:1.20.1-16.el9       appstream           586 k
...
Installed:
  nginx-1:1.20.1-16.el9.x86_64
Complete!
```

Verify the installation succeeded by checking the Nginx version:

```bash
nginx -v
```

Expected output:

```text
nginx version: nginx/1.20.1
```

---

## Step 2 — Enable and Start Nginx with systemctl

> **What is systemctl?**
> `systemctl` is the tool for managing *services* — programs that run in the background on your server. You use it to start, stop, restart, enable (start on boot), and check the status of any service.

Start Nginx immediately:

```bash
sudo systemctl start nginx
```

Enable it so it starts automatically every time the server reboots:

```bash
sudo systemctl enable nginx
```

You will see:

```text
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.
```

This symlink is how systemd knows to launch Nginx at boot. Now confirm it is running:

```bash
sudo systemctl status nginx
```

Look for the `active (running)` line:

```text
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Mon 2026-03-17 10:00:00 UTC; 5s ago
   Main PID: 1234 (nginx)
```

If the status says `failed` or `inactive`, jump to the **Troubleshooting** section at the bottom of this lesson.

---

## Step 3 — Open Firewall Ports 80 and 443

Right now, even though Nginx is running, your firewall is blocking all traffic except SSH. You need to open the standard web ports:

- **Port 80** — HTTP (unencrypted web traffic)
- **Port 443** — HTTPS (encrypted web traffic — we are not setting up TLS today, but opening the port now is good practice)

> **What is firewalld?**
> `firewalld` is the firewall manager in Rocky Linux. It controls which network traffic is allowed in and out of your server. By default it blocks everything except services you explicitly allow.

Add the HTTP and HTTPS services to the firewall:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
```

The `--permanent` flag writes the rule to disk so it survives a reboot. Apply the changes immediately without a full restart:

```bash
sudo firewall-cmd --reload
```

Confirm both services are now listed:

```bash
sudo firewall-cmd --list-services
```

Expected output:

```text
cockpit dhcpv6-client http https ssh
```

You should now be able to open a browser on your host machine, type your VM's IP address into the address bar, and see the default Nginx welcome page. To find your VM's IP:

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

The IP will look something like `192.168.56.101`.

---

## Step 4 — Set the Correct SELinux Context on Web Content

> **Why does SELinux care about web files?**
> SELinux confines the Nginx process to a specific *type* called `httpd_t`. That process is only allowed to read files that carry the `httpd_sys_content_t` label. If your content files carry the wrong label — for example because you copied them from your home directory — SELinux will silently block Nginx from reading them, and visitors will see a 403 Forbidden error.

The web root directory is `/var/www/html/`. Files placed here by `dnf` already have the correct context, but if you create or copy files manually, you need to verify and restore the label.

Check the current context on the web root:

```bash
ls -Zd /var/www/html/
```

Expected output:

```text
system_u:object_r:httpd_sys_content_t:s0 /var/www/html/
```

The important part is `httpd_sys_content_t`. Any file you create inside this directory will inherit that type automatically. But if you *copy* a file from, say, your home directory, the original label is preserved — and that label will be wrong.

After creating or copying any content, run `restorecon` to fix the labels:

```bash
sudo restorecon -Rv /var/www/html/
```

The `-R` flag means recursive (fix all files inside the directory), and `-v` means verbose (show what it changed). If the labels were already correct, the command outputs nothing — that is fine.

---

## Step 5 — Serve a Custom HTML Page

The default Nginx page tells the world you have just installed Nginx. Replace it with a page of your own.

Create a simple HTML file in the web root:

```bash
sudo tee /var/www/html/index.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>RootReady — My First Server</title>
</head>
<body>
  <h1>Hello from Rocky Linux!</h1>
  <p>This page is being served by Nginx. SELinux is enforcing. The firewall is on.</p>
  <p>You built this. Good work.</p>
</body>
</html>
EOF
```

> **What does `tee` do?**
> `sudo tee file` reads from standard input and writes to a file using root privileges. The `<< 'EOF'` syntax is called a *heredoc* — everything between `<< 'EOF'` and the closing `EOF` on its own line is treated as input. This lets you write a multi-line file without a text editor.

Restore the SELinux context on the new file:

```bash
sudo restorecon -v /var/www/html/index.html
```

Now reload the page in your browser. You should see your custom content instead of the Nginx default page.

If you see a 403 Forbidden error instead, jump to the **Troubleshooting** section — this is almost always an SELinux label problem.

---

## Step 6 — Harden Nginx with Security Headers

A default Nginx installation leaks information that attackers can use. In this step you will:

1. Hide the Nginx version number from response headers
2. Add security headers that tell browsers to behave more safely

### Why this matters

When a browser requests a page, the server sends back HTTP headers along with the content. By default, one of those headers is:

```text
Server: nginx/1.20.1
```

This tells anyone watching network traffic — or using browser developer tools — exactly which version of Nginx you are running. If a vulnerability is discovered in that version, attackers can target your server by version number.

### Edit the main Nginx configuration

Open the main Nginx configuration file:

```bash
sudo vi /etc/nginx/nginx.conf
```

> **What is vi?**
> `vi` is a text editor that runs inside the terminal. Press `i` to enter *insert mode* (where you can type), and press `Escape` then type `:wq` and press `Enter` to save and quit. If you make a mistake, press `Escape` then type `:q!` and press `Enter` to quit without saving.

Inside the `http { }` block, find the line that says:

```nginx
    include /etc/nginx/conf.d/*.conf;
```

**Add the following lines directly above it:**

```nginx
    # --- Hardening ---
    server_tokens off;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header X-XSS-Protection "1; mode=block" always;
```

Here is what each directive does:

| Directive | What it does |
|---|---|
| `server_tokens off` | Removes the Nginx version from the `Server` response header |
| `X-Frame-Options: SAMEORIGIN` | Prevents your page from being embedded in an `<iframe>` on another site (stops clickjacking attacks) |
| `X-Content-Type-Options: nosniff` | Tells the browser not to guess the content type — use only what the server declares |
| `Referrer-Policy` | Controls how much URL information is sent when a visitor follows a link away from your site |
| `X-XSS-Protection` | Enables the browser's built-in cross-site scripting filter |

After editing, test the configuration for syntax errors before reloading:

```bash
sudo nginx -t
```

Expected output:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If you see any errors, re-open the file and check your edits carefully — a missing semicolon is the most common mistake.

Reload Nginx to apply the changes (this is gentler than a restart — active connections are not dropped):

```bash
sudo systemctl reload nginx
```

### Verify the headers

Use `curl` to inspect the response headers from your server. Run this from the VM itself:

```bash
curl -I http://localhost
```

> **What is curl?**
> `curl` is a command-line tool for making HTTP requests. The `-I` flag tells it to fetch only the headers, not the page body. It is extremely useful for debugging web server behaviour.

You should see output like this:

```text
HTTP/1.1 200 OK
Server: nginx
Date: Mon, 17 Mar 2026 10:00:00 GMT
Content-Type: text/html
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer-when-downgrade
X-XSS-Protection: 1; mode=block
```

Notice that the `Server` header now shows `nginx` without a version number. The four security headers are present. Your Nginx installation is now hardened.

---

## Troubleshooting

### 403 Forbidden — browser shows an error page

A 403 error means Nginx can reach the file but is not allowed to read it. This is almost always an SELinux label problem.

Check the audit log for a denial:

```bash
sudo ausearch -m AVC -ts recent | audit2why
```

If you see a denial mentioning `httpd_t` and a context other than `httpd_sys_content_t`, restore the labels:

```bash
sudo restorecon -Rv /var/www/html/
```

Then reload the page. If the 403 persists, also check the file permissions:

```bash
ls -l /var/www/html/index.html
```

The file must be readable by others (`-rw-r--r--`). If it is not:

```bash
sudo chmod 644 /var/www/html/index.html
```

### Cannot reach the server from the host browser

If you can see the Nginx welcome page with `curl http://localhost` on the VM but cannot reach it from the host browser, the firewall rule may not have applied.

Check the active firewall rules:

```bash
sudo firewall-cmd --list-services
```

If `http` is missing, add it again and reload:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Also confirm the VM's network adapter is set to **Host-only** or **Bridged** in VirtualBox — a NAT-only adapter will not be reachable from the host.

### SELinux AVC on `/var/www/html/` content

If `audit2why` reports a denial like this:

```text
scontext=system_u:system_r:httpd_t:s0
tcontext=unconfined_u:object_r:user_home_t:s0
```

It means a file was created or copied with the wrong context. Fix it:

```bash
sudo restorecon -Rv /var/www/html/
```

If the denial is about a *boolean* (for example `httpd_read_user_content`), enable it permanently:

```bash
sudo setsebool -P httpd_read_user_content on
```

But prefer fixing the file context — boolean changes are broader and grant more access than necessary.

### `nginx -t` reports a syntax error

Open the config file and look at the line number reported in the error message:

```bash
sudo vi /etc/nginx/nginx.conf
```

Common mistakes:
- Missing semicolon at the end of a directive
- A directive placed outside of the correct block (e.g., `server_tokens` must be inside `http { }`)
- A stray character from a copy-paste

After fixing, run `nginx -t` again before reloading.

### Nginx fails to start after install

Check the full error log:

```bash
sudo journalctl -u nginx --no-pager | tail -30
```

Also check if something else is already using port 80:

```bash
sudo ss -tlnp | grep :80
```

If another process is listed, you need to either stop it or configure Nginx to listen on a different port.

---

## Summary

In this lesson you:

- **Installed Nginx** with `dnf install nginx` — Rocky Linux's package manager fetched and installed the web server from the official repository
- **Started and enabled Nginx** with `systemctl enable --now nginx` so it runs immediately and restarts on every boot
- **Opened the firewall** with `firewall-cmd --permanent --add-service=http` (and https) so browsers on the network can reach port 80 and 443
- **Set SELinux file contexts** with `restorecon -Rv /var/www/html/` — ensuring Nginx's `httpd_t` process can read your content files labelled `httpd_sys_content_t`
- **Served a custom HTML page** by replacing the default index file in `/var/www/html/`
- **Hardened the server** by disabling `server_tokens` (hides the Nginx version) and adding four security headers that protect visitors from common browser-based attacks

Your server is now publicly reachable, SELinux-confined, and hardened against basic information-disclosure and browser attacks.

---

**Next up:** [Lesson 5 — Podman and Rootless Containers]({{< relref "/lessons/05-podman-basics" >}})
