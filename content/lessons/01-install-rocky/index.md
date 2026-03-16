---
title: "Lesson 1 — Installing Rocky Linux in a Virtual Machine"
description: "You will download Rocky Linux, set up a free virtual machine, and complete a full install — giving you your own Linux server to learn on."
date: 2026-03-16
weight: 1
tags: ["rocky linux", "beginner", "virtualbox", "installation"]
---

## What You Will Learn

By the end of this lesson you will have:

- Downloaded a Rocky Linux ISO (the installer file)
- Installed VirtualBox (a free app that runs a virtual computer inside your real one)
- Created a virtual machine and booted it from the Rocky Linux installer
- Completed the full Rocky Linux installation
- Logged in to your brand-new Linux server for the first time

No prior Linux knowledge is needed. We will explain every step and every click.

---

## What You Need Before Starting

- **A computer** running Windows 10/11, macOS, or Linux with at least 4 GB of RAM and 25 GB of free disk space
- **An internet connection** to download the software (about 2 GB total)
- **About 30 minutes** of your time

> **What is a virtual machine (VM)?**
> A virtual machine is a pretend computer that runs inside your real computer. It behaves exactly like a separate machine — with its own operating system, its own files, and its own network — but it is just a window on your desktop. If you break something inside the VM, your real computer is completely unaffected. This makes it the perfect place to learn.

> **What is Rocky Linux?**
> Rocky Linux is a free, community-driven version of Linux designed for servers. It is built from the same source code as Red Hat Enterprise Linux (RHEL), which is what many companies run in production. Learning Rocky means you are learning real, industry-standard skills.

---

## Step 1 — Download the Rocky Linux ISO

An **ISO file** is a disk image — think of it as a virtual DVD that contains the installer.

1. Open your web browser and go to the official Rocky Linux download page:
   **https://rockylinux.org/download**

2. Under **Rocky Linux 9**, find the **x86_64** architecture row (this is the standard option for most computers).

3. Click the **Minimal** ISO link. This gives you a small, server-focused install — no desktop environment, just the command line. That is exactly what we want.

4. Save the file somewhere easy to find, like your **Downloads** folder. The file will be roughly 1.5–2 GB.

While that downloads, move on to the next step.

---

## Step 2 — Download and Install VirtualBox

VirtualBox is a free, open-source application by Oracle that lets you run virtual machines.

1. Go to the VirtualBox download page:
   **https://www.virtualbox.org/wiki/Downloads**

2. Click the link for your operating system:
   - **Windows hosts** if you are on Windows
   - **macOS / Intel hosts** or **macOS / Apple Silicon hosts** if you are on a Mac
   - **Linux distributions** if you are already on Linux

3. Run the downloaded installer and follow the prompts. Accept all the defaults — they are fine for our purposes.

4. When the install finishes, open VirtualBox. You should see the **VirtualBox Manager** window with an empty list of machines.

> **Tip:** If Windows asks "Do you want to allow this app to make changes to your device?" click **Yes**. VirtualBox needs permission to set up virtual network adapters.

---

## Step 3 — Create a New Virtual Machine

Now we tell VirtualBox to create a blank virtual computer that we will install Rocky Linux onto.

1. In VirtualBox Manager, click the **New** button (or go to **Machine → New**).

2. Fill in the details:
   - **Name:** `rocky-server` (or anything you like)
   - **Folder:** Leave the default, or pick a folder with enough space
   - **ISO Image:** Click the dropdown and select **Other…**, then browse to the Rocky Linux ISO you downloaded in Step 1
   - **Type:** Linux
   - **Subtype:** Red Hat
   - **Version:** Red Hat 9.x (64-bit)

3. Check the box that says **Skip Unattended Installation**. We want to walk through the installer manually so you understand every choice.

4. Click **Next**.

### Memory and Processors

5. Set **Base Memory** to at least **2048 MB** (2 GB). If your computer has 8 GB or more of RAM, you can safely give the VM 4096 MB (4 GB).

6. Set **Processors** to **2** if your computer has 4 or more CPU cores. Otherwise, leave it at 1.

7. Click **Next**.

### Virtual Hard Disk

8. Select **Create a Virtual Hard Disk Now**.

9. Set the disk size to **20 GB**. This is more than enough for a learning server. The file on your real hard drive will only grow as the VM actually uses space — it will not immediately take up 20 GB.

10. Click **Next**, review the summary, and click **Finish**.

You now have a virtual machine listed in the VirtualBox Manager. It is powered off and has a blank hard disk — ready for Rocky Linux.

---

## Step 4 — Install Rocky Linux

Time to boot the virtual machine and run the installer.

1. Select your `rocky-server` VM in the list and click **Start** (the green arrow).

2. A new window opens. The VM boots from the ISO and you will see a menu. Use your arrow keys to select:

   ```text
   Install Rocky Linux 9
   ```

   Press **Enter**.

3. The graphical installer (called **Anaconda**) loads. First, select your **language** (English is fine) and click **Continue**.

### Installation Summary Screen

You will see a screen with several categories. Most things are already configured. Here is what you need to check:

#### Time & Date

4. Click **Time & Date**. Select your time zone from the map or the dropdown menus. Click **Done** (top left corner).

#### Installation Destination

5. Click **Installation Destination**. You should see the 20 GB virtual hard disk already selected with a checkmark. Leave everything on **Automatic** partitioning. Click **Done**.

> **What is partitioning?**
> Partitioning means dividing a hard disk into sections. The automatic option creates a sensible layout for you — a boot partition and a main storage partition. For a learning VM, automatic is perfect.

#### Root Password

6. Click **Root Password**. Type a password you will remember, and type it again to confirm. For a local learning VM, something simple is fine — but in a real server you would use a strong, unique password.

   Make sure **Lock root account** is _unchecked_ for now (we need root access to learn).

> **What is root?**
> root is the all-powerful administrator account in Linux. It can do anything — install software, change any file, even destroy the entire system. Later we will learn how to create a normal user account and use root only when needed.

#### User Creation (optional for now)

7. You can optionally click **User Creation** to create a normal user account. If you do:
   - Enter a **Full name** and **Username** (e.g., `student`)
   - Set a password
   - Check **Make this user administrator** so you can use `sudo` later

   If you skip this, that is fine — we will create a user in Lesson 2.

#### Network & Host Name

8. Click **Network & Host Name**. You should see an Ethernet adapter. Toggle the switch to **ON** so the VM can access the internet. You can optionally change the hostname to `rocky-server`. Click **Done**.

### Begin Installation

9. Once every category shows a green checkmark (no orange warnings), click **Begin Installation** at the bottom of the screen.

10. The installer will copy files, install packages, and configure the system. This takes 5–10 minutes depending on your computer. Grab a coffee.

11. When it finishes, click **Reboot System**.

---

## Step 5 — First Boot and Login

After the VM reboots, you will see a text login prompt:

```text
rocky-server login: _
```

This is your Linux server. No desktop, no icons — just a blinking cursor waiting for your command.

1. Type `root` and press **Enter**.
2. Type the root password you set during installation and press **Enter**.
   (You will not see any characters as you type the password — this is normal Linux security behavior.)

3. You should now see a prompt that looks something like:

   ```text
   [root@rocky-server ~]#
   ```

   The `#` symbol means you are logged in as root. You are in.

### Verify Your Installation

Run your very first Linux command. Type the following and press **Enter**:

```bash
cat /etc/rocky-release
```

This command asks the system to display the contents of a file that contains the version information. You should see output like:

```
Rocky Linux release 9.x (Blue Onyx)
```

Congratulations — you have a working Rocky Linux server.

---

## Step 6 — Shut Down the VM Properly

Never just close the VirtualBox window — that is like pulling the power cord on a real server. Instead, shut down gracefully:

```bash
shutdown now
```

This command tells Linux to stop all services and power off safely.

> **Tip:** You can also use `poweroff` — it does the same thing.

The VM window will close (or show a black screen). In VirtualBox Manager, the VM state changes to **Powered Off**.

To start it again later, just select it and click **Start**.

---

## Troubleshooting

### "VT-x is not available" or "AMD-V is disabled"

Your computer's virtualization feature is turned off in the BIOS/UEFI. You need to:

1. Restart your computer and enter the BIOS/UEFI setup (usually by pressing **F2**, **F10**, **Del**, or **Esc** during boot — the key varies by manufacturer).
2. Look for a setting called **Intel VT-x**, **Intel Virtualization Technology**, **AMD-V**, or **SVM Mode**.
3. Enable it, save, and exit.

### The installer screen is too small to see the buttons

In the VirtualBox menu bar, go to **View → Scaled Mode** (or press **Host+C**). This lets you resize the window and the VM display scales with it.

### I forgot my root password

Since this is a learning VM, the easiest fix is to delete the VM and start over from Step 3. It only takes a few minutes and is good practice.

### The VM has no internet

1. Power off the VM.
2. In VirtualBox Manager, select the VM and click **Settings → Network**.
3. Make sure **Adapter 1** is enabled and set to **NAT**.
4. Start the VM again and check with:

```bash
ping -c 3 google.com
```

If you see replies, you are connected. Press **Ctrl+C** to stop the ping.

---

## Summary

In this lesson you:

- **Downloaded Rocky Linux 9** — a free, enterprise-grade Linux distribution
- **Installed VirtualBox** — a free tool to run virtual machines on your computer
- **Created a virtual machine** with 2 GB RAM and a 20 GB disk
- **Installed Rocky Linux** using the Anaconda graphical installer
- **Logged in as root** and verified the installation
- **Learned how to shut down** a Linux server properly

Your virtual server is ready. In the next lesson, you will create a regular user account, set up SSH (so you can connect to the server from a terminal on your real computer), and configure the firewall.

---

**Next up:** Lesson 2 — Users, SSH, and firewalld (coming soon)
