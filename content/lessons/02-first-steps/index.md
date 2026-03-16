---
title: "Lesson 2 — First Steps After Install: Users, SSH, and firewalld"
description: "You will create a regular user account, set up SSH so you can connect from your host machine, and configure the firewall to keep your server secure."
date: 2026-03-16
weight: 2
tags: ["users", "ssh", "firewalld", "beginner", "security"]
---

## What You Will Learn

By the end of this lesson you will have:

- Created a regular (non-root) user account with `sudo` privileges
- Generated an SSH key pair on your host machine
- Copied the public key to your server so you can log in without a password
- Disabled root login and password authentication over SSH
- Configured `firewalld` to allow only SSH traffic

These are the first things a real system administrator does on a new server. When you finish, you will be connecting to your VM the same way professionals connect to production servers — securely, over SSH, as a normal user.

---

## What You Need Before Starting

- **A running Rocky Linux 9 VM** from [Lesson 1]({{< relref "/lessons/01-install-rocky" >}})
- **Root access** to the VM (the root password you set during installation)
- **A terminal on your host machine**:
  - **Windows:** Open **Git Bash** (installed with Git for Windows) or **PowerShell**
  - **macOS / Linux:** Open the built-in **Terminal** app
- **About 25 minutes** of your time

> **What is SSH?**
> SSH (Secure Shell) is a way to connect to a remote computer over the network and type commands on it — as if you were sitting in front of it. Everything you type and everything the server sends back is encrypted, so nobody can spy on the connection. SSH is how the entire IT industry manages servers.

---

## Step 1 — Find Your VM's IP Address

Before you can connect to the VM from your host machine, you need to know the VM's IP address.

Log in to the VM as `root` (directly in the VirtualBox window) and run:

```bash
ip addr show
```

Look for a section called `enp0s3` (or similar). You will see a line that looks like:

```text
inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
```

The number after `inet` — in this example, `10.0.2.15` — is your VM's IP address.

> **Why 10.0.2.15?**
> VirtualBox uses a network mode called **NAT** by default. In NAT mode, every VM gets the address `10.0.2.15`. This address is not directly reachable from your host machine — we will fix that in a moment with **port forwarding**.

### Set Up VirtualBox Port Forwarding

Because NAT hides the VM behind a private network, we need to tell VirtualBox to forward a port from your host to the VM:

1. **Power off** the VM (run `shutdown now` inside it).
2. In VirtualBox Manager, select your VM and click **Settings → Network**.
3. Make sure **Adapter 1** is set to **NAT**.
4. Click **Advanced** (the little arrow to expand the section), then click **Port Forwarding**.
5. Click the green **+** icon to add a new rule:

   | Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
   |------|----------|---------|-----------|----------|------------|
   | SSH  | TCP      |         | 2222      |          | 22         |

   Leave **Host IP** and **Guest IP** blank (they default to all interfaces and the VM's IP).

6. Click **OK** twice to save.
7. Start the VM again.

Now you can reach the VM's SSH port (22) by connecting to **localhost port 2222** on your host.

---

## Step 2 — Create a Regular User Account

Running everything as `root` is dangerous. One mistyped command and you can destroy the entire system. The safe practice is to use a regular user account for daily work and only escalate to root privileges when you actually need them.

Log in to the VM as `root` and create a new user:

```bash
useradd -m -G wheel student
```

Here is what each part means:

- `useradd` — the command to create a new user
- `-m` — create a **home directory** for the user (at `/home/student`)
- `-G wheel` — add the user to the **wheel** group
- `student` — the username (you can pick any name you like)

> **What is the wheel group?**
> The `wheel` group is a special group on Red Hat-based systems (like Rocky Linux). Members of this group are allowed to use `sudo` — a command that lets you temporarily act as root. If your user is not in the `wheel` group, `sudo` will refuse to work.

Now set a password for the new user:

```bash
passwd student
```

Type a password and confirm it. You will not see any characters on screen as you type — this is normal.

### Test the New Account

Switch to the new user account without logging out:

```bash
su - student
```

Your prompt changes from `#` (root) to `$` (normal user):

```text
[student@rocky-server ~]$
```

Now verify that `sudo` works:

```bash
sudo whoami
```

It will ask for **your** password (the one you set for `student`, not the root password). After you enter it, the output should be:

```text
root
```

This proves your user can escalate to root when needed. Type `exit` to switch back to the root account.

---

## Step 3 — Enable and Start the SSH Service

Rocky Linux minimal install comes with the SSH server (`sshd`) already installed and running. Let's confirm:

```bash
systemctl status sshd
```

> **What is systemctl?**
> `systemctl` is the command you use to manage **services** — programs that run in the background. Think of services like apps that start automatically. `sshd` is the SSH service (the **d** stands for **daemon**, which is Linux's word for a background service).

You should see output that includes:

```text
Active: active (running)
```

If for some reason it is not running, enable and start it:

```bash
systemctl enable --now sshd
```

The `--now` flag means "enable it to start on every boot **and** start it right now." Two actions in one command.

---

## Step 4 — Connect via SSH From Your Host Machine

Open a terminal on your **host** machine (not inside the VM). This is the Git Bash, PowerShell, or Terminal app on your real computer.

Connect to the VM through the port-forwarding rule you set up earlier:

```bash
ssh student@localhost -p 2222
```

Here is what each part means:

- `ssh` — the SSH client command
- `student` — the username you created on the VM
- `localhost` — your own computer (VirtualBox is forwarding the connection)
- `-p 2222` — connect to port 2222 (which VirtualBox forwards to the VM's port 22)

The first time you connect, SSH asks you to verify the server's identity:

```text
The authenticity of host '[localhost]:2222 (...)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes` and press **Enter**. This is normal — SSH is telling you it has never seen this server before and wants you to confirm it is the right one.

Enter the password for `student` when prompted. You should land in a shell on the VM:

```text
[student@rocky-server ~]$
```

You are now connected to your server from your host machine — exactly like a real sysadmin. You can close the VirtualBox window (the VM keeps running in the background) and do all your work from this terminal.

---

## Step 5 — Generate an SSH Key Pair

Typing a password every time you connect is tedious and less secure than using SSH keys. An SSH key pair consists of two files:

- A **private key** — stays on your host machine, never leaves it. Think of it as your house key.
- A **public key** — gets copied to the server. Think of it as a lock you install on the door.

When you connect, SSH proves you have the private key without ever sending it over the network. This is far more secure than passwords.

**On your host machine** (not the VM), run:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

- `-t ed25519` — use the Ed25519 algorithm (modern and secure)
- `-C "your-email@example.com"` — a label for the key (use your actual email or any identifier)

When prompted:

1. **File location:** Press **Enter** to accept the default (`~/.ssh/id_ed25519`)
2. **Passphrase:** You can set one for extra security, or press **Enter** twice for no passphrase (fine for a learning VM)

This creates two files:

- `~/.ssh/id_ed25519` — your private key (**never share this**)
- `~/.ssh/id_ed25519.pub` — your public key (safe to share)

---

## Step 6 — Copy Your Public Key to the Server

Now install the public key on the VM so SSH recognizes your host machine.

**On your host machine**, run:

```bash
ssh-copy-id -p 2222 student@localhost
```

This command:

1. Reads your public key from `~/.ssh/id_ed25519.pub`
2. Connects to the server as `student`
3. Appends the public key to `~/.ssh/authorized_keys` on the server

Enter the `student` password one last time when prompted. You should see:

```text
Number of key(s) added: 1
```

### Test Key-Based Login

Disconnect from the VM (type `exit` or press **Ctrl+D**) and reconnect:

```bash
ssh student@localhost -p 2222
```

This time, **you should not be asked for a password**. SSH uses your key automatically. If it works — congratulations, you have key-based authentication.

> **What if `ssh-copy-id` is not available?**
> On Windows (PowerShell without Git Bash), `ssh-copy-id` may not exist. You can do it manually:
>
> ```powershell
> type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 student@localhost "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
> ```
>
> This pipes the contents of your public key file into the server and appends it to the authorized keys.

---

## Step 7 — Disable Root Login and Password Authentication

Now that you can log in with a key, you should lock down SSH so that:

1. **Nobody can log in as root over SSH** — root access should only happen via `sudo`
2. **Nobody can use a password to log in** — keys only

On the VM (you should be logged in as `student` via SSH), edit the SSH configuration file:

```bash
sudo vi /etc/ssh/sshd_config.d/99-custom.conf
```

> **Why a file in `sshd_config.d/`?**
> Rocky Linux 9's SSH config includes everything in the `/etc/ssh/sshd_config.d/` directory. By putting your changes in a separate file instead of editing the main `sshd_config`, you keep things clean and avoid merge conflicts when the system updates the default file.

> **Quick vi survival guide:**
> `vi` is a text editor that comes installed on every Linux system. Press **i** to enter insert mode (you can now type). Press **Esc** to leave insert mode. Type `:wq` and press **Enter** to save and quit. Type `:q!` and press **Enter** to quit without saving.

Type the following content into the file (press **i** first to enter insert mode):

```text
PermitRootLogin no
PasswordAuthentication no
```

Press **Esc**, then type `:wq` and press **Enter** to save.

Now restart the SSH service so it picks up the changes:

```bash
sudo systemctl restart sshd
```

### Test the Lockdown

**Important:** Do not close your current SSH session yet. Open a **second** terminal on your host and test that you can still connect:

```bash
ssh student@localhost -p 2222
```

If this works (you get in without a password prompt), the lockdown is successful. You can now safely close the first session.

To verify root login is blocked, try:

```bash
ssh root@localhost -p 2222
```

You should see:

```text
root@localhost: Permission denied (publickey).
```

Root cannot log in over SSH. Exactly what we want.

---

## Step 8 — Configure firewalld

Your server is currently running a firewall called `firewalld`. A firewall controls which network traffic is allowed in and out of the server. We want to make sure only SSH traffic is permitted.

> **What is a firewall?**
> A firewall is a filter that sits between your server and the network. It looks at every incoming connection and decides — based on rules you set — whether to allow it or block it. Think of it as a bouncer at a door: only people on the list get in.

First, check that `firewalld` is running:

```bash
sudo systemctl status firewalld
```

You should see `active (running)`. If it is not running:

```bash
sudo systemctl enable --now firewalld
```

### Check the Current Rules

```bash
sudo firewall-cmd --list-all
```

You will see output like:

```text
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  ...
```

The important line is `services:`. By default, Rocky Linux allows `cockpit`, `dhcpv6-client`, and `ssh`.

> **What is cockpit?**
> Cockpit is a web-based server management dashboard. We will not use it in this series — we are learning the command line. It is safe to remove.

### Remove Unnecessary Services

Remove `cockpit` since we do not need it:

```bash
sudo firewall-cmd --permanent --remove-service=cockpit
```

The `--permanent` flag means the change persists after a reboot. Without it, the rule would be lost when `firewalld` restarts.

Now reload the firewall to apply the changes:

```bash
sudo firewall-cmd --reload
```

### Verify the Final Rules

```bash
sudo firewall-cmd --list-all
```

The `services:` line should now show only:

```text
services: dhcpv6-client ssh
```

> **What about dhcpv6-client?**
> DHCP (Dynamic Host Configuration Protocol) is how your VM gets its IP address automatically. Removing this could break networking. Leave it in.

SSH is the only remote-access service allowed. Any attempt to connect on a port or service not in this list will be silently dropped by the firewall.

### Test the Firewall

To prove the firewall is working, try to access a port that is not open. From your **host machine**, run:

```bash
curl -m 5 http://localhost:8080
```

This should time out or refuse the connection — because port 8080 is not forwarded and not allowed through the firewall. That is the correct behavior.

---

## Troubleshooting

### I am locked out of SSH after disabling password authentication

This is the most common mistake. It happens when you disable password authentication before your SSH key is properly installed.

**Fix:** You still have direct console access through VirtualBox.

1. Open the VirtualBox window for your VM.
2. Log in as `root` using the password you set during installation.
3. Re-enable password authentication temporarily:

   ```bash
   vi /etc/ssh/sshd_config.d/99-custom.conf
   ```

   Change `PasswordAuthentication no` to `PasswordAuthentication yes`. Save and exit.

4. Restart SSH:

   ```bash
   systemctl restart sshd
   ```

5. From your host, re-copy your SSH key with `ssh-copy-id -p 2222 student@localhost`.
6. Test key login, then disable password authentication again.

### firewall-cmd says "FirewallD is not running"

Enable and start the service:

```bash
sudo systemctl enable --now firewalld
```

Then try your `firewall-cmd` command again.

### SSH connection is refused

Check the following:

1. **Is the VM running?** Look in VirtualBox Manager — it should say "Running."
2. **Is the port-forwarding rule correct?** Go to VM Settings → Network → Advanced → Port Forwarding. Make sure Host Port is `2222` and Guest Port is `22`.
3. **Is sshd running inside the VM?** Log in via the VirtualBox console and run:

   ```bash
   systemctl status sshd
   ```

4. **Is the firewall allowing SSH?** Run:

   ```bash
   sudo firewall-cmd --list-services
   ```

   `ssh` must be in the list.

### "sudo: student is not in the sudoers file"

This means the user was not added to the `wheel` group. As root, run:

```bash
usermod -aG wheel student
```

Then log out and log back in as `student` for the change to take effect.

---

## Summary

In this lesson you:

- **Created a regular user** (`student`) with sudo privileges via the `wheel` group — so you no longer need to run everything as root
- **Set up VirtualBox port forwarding** — so your host machine can reach the VM's SSH port
- **Enabled SSH** and connected to the VM from your host terminal — the professional way to manage a server
- **Generated an SSH key pair** and installed the public key on the server — replacing password-based login with key-based authentication
- **Hardened SSH** by disabling root login and password authentication in a drop-in config file
- **Configured firewalld** to allow only SSH traffic — blocking everything else at the network level

Your server is now secured at two layers: SSH only accepts key-based login from a non-root user, and the firewall only permits SSH traffic. This is a solid security baseline for everything that follows.

---

**Next up:** Lesson 3 — Understanding SELinux (coming soon)
