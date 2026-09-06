*This project has been created as part of the 42 curriculum by nkerstin.*

# Born2beRoot

## 📖 Description

**Born2beRoot** is a System Administration project from the 42 curriculum. The goal is to set up a virtual machine from scratch and configure it as a secure, minimal server by following a strict set of rules — without ever relying on a graphical interface.

The project covers:

- Installing **Debian** inside **VirtualBox**
- Partitioning the disk with **LVM** on top of **encrypted (LUKS)** partitions
- Configuring a strong **password policy** for all users, including root
- Setting up **sudo** with logging, restricted paths, a limited number of attempts and a custom error message
- Configuring **SSH** to run on port `4242`, with root login disabled
- Configuring the **UFW** firewall to only allow traffic on port `4242`
- Enabling **AppArmor** at startup
- Creating a mandatory user (`nkerstin`) belonging to the `sudo` and `user42` groups
- Writing a custom **monitoring.sh** bash script that reports the server's status every 10 minutes via `wall`, scheduled through a **cron** job

The point of the exercise isn't the final server itself, but understanding *why* each of these components (LVM, LUKS, sudo, SSH, UFW, AppArmor, cron) exists and how they fit together on a real system.

## ⚙️ Instructions

This project doesn't require compilation. It consists of a configured virtual machine plus one script.

1. **Create the VM**
   - Install [VirtualBox](https://www.virtualbox.org/) (or UTM on Apple Silicon)
   - Create a new VM (min. recommended: 1 vCPU, 1024 MB RAM, ~8–10 GB disk)
   - Attach the latest stable **Debian** netinst ISO

2. **Install Debian**
   - Choose the manual/expert partitioning method
   - Set up **LVM on top of an encrypted (LUKS) physical volume**
   - Create at least two logical volumes (e.g. `root` and `home`, plus `swap`)
   - Do **not** install a desktop environment / graphical server (no X.org, no Wayland)

3. **Post-install configuration** (as root, via the console — no GUI)
   - Set the hostname to `nkerstin42`
   - Install and configure `sudo`, `ufw`, `libpam-pwquality` (password policy)
   - Edit `/etc/ssh/sshd_config` to run SSH on port `4242` and disable root login
   - Configure `ufw` to allow only port `4242`
   - Create the user `nkerstin`, add it to the `sudo` and `user42` groups
   - Copy `monitoring.sh` to `/usr/local/bin/` (or similar), make it executable, and schedule it via a cron job (`@reboot` + every 10 minutes) so it broadcasts the status through `wall`

4. **Verify**
   - `sudo systemctl status ssh` → SSH running on port 4242
   - `sudo ufw status` → only 4242 open
   - `sudo aa-status` → AppArmor enabled
   - `lsblk` → LVM on top of an encrypted partition
   - `sudo chage -l nkerstin` → password policy applied

5. **Submission**
   - Only `README.md` and `signature.txt` (SHA1 of the `.vdi`/`.qcow2` file) are pushed to the Git repository
   - The virtual machine itself is **never** committed

## 🖥️ Project Description – Technical Choices

### Why Debian?

I chose **Debian** over Rocky Linux mainly for its simplicity and because it's the operating system most widely documented and used across the 42 network, which made troubleshooting easier as a first system administration project.

| | Debian | Rocky Linux |
|---|---|---|
| **Origin** | Community-driven, one of the oldest Linux distributions | RHEL-compatible, community continuation of CentOS |
| **Package manager** | `apt` / `dpkg` (`.deb`) | `dnf` / `yum` (`.rpm`) |
| **Security module** | AppArmor | SELinux |
| **Firewall tool** | UFW | firewalld |
| **Release cycle** | Slower, very stable ("stable" branch) | Follows RHEL's enterprise release cycle |
| **Learning curve** | Beginner-friendly | Steeper (SELinux policies, RHEL conventions) |
| **Typical use** | Servers, desktops, general purpose | Enterprise servers, environments needing RHEL compatibility |

### AppArmor vs SELinux

Both are **Mandatory Access Control (MAC)** systems that restrict what a process can do, going beyond standard Unix permissions.

- **AppArmor** (used on Debian) works with **path-based profiles**: it restricts what specific applications can access based on file paths. It's simpler to write and read, but slightly less granular.
- **SELinux** (used on Rocky) works with **labels/contexts** attached to every file and process, offering finer-grained control, but with a much steeper learning curve and more complex policy management.

In short: AppArmor trades some flexibility for simplicity, SELinux is more powerful but harder to configure correctly.

### UFW vs firewalld

- **UFW (Uncomplicated Firewall)**, used on Debian, is a simple front-end for `iptables`/`nftables`. Rules are added with short commands (e.g. `ufw allow 4242`), making it very approachable for beginners.
- **firewalld**, used on Rocky, works with the concept of **zones** (e.g. `public`, `internal`) that group interfaces and rules together, and it can reload rules without dropping active connections. It's more flexible for complex network setups but has more concepts to learn.

For this project, both accomplish the same goal: leaving only port `4242` open.

### VirtualBox vs UTM

- **VirtualBox** is a free, cross-platform (Windows/Linux/macOS-Intel) type-2 hypervisor from Oracle. It's the recommended tool for this project and has broad documentation and community support.
- **UTM** is a virtualization/emulation tool for macOS, built on top of QEMU, and is used as an alternative on **Apple Silicon (M1/M2/M3...)** Macs, where VirtualBox support is limited. It achieves the same result but with a different interface and disk format (`.qcow2` instead of `.vdi`).

I used **VirtualBox**, as I'm on an x86/AMD64 machine.

### Partitioning

The disk is protected with **LUKS encryption**, on top of which **LVM** manages the logical volumes. This setup was chosen because it satisfies the subject's mandatory requirement of at least two *encrypted* partitions, while LVM keeps the layout flexible (volumes can be resized without repartitioning the disk).

Logical volumes created:
- `root` (`/`)
- `swap`
- `home` (`/home`)

### Security policies

- **Password policy** (via `libpam-pwquality` and `/etc/login.defs`): minimum 10 characters, at least one uppercase, one lowercase and one digit, no more than 3 consecutive identical characters, no username in the password, password expires every 30 days, minimum 2 days between changes, 7-day warning before expiry, and at least 7 new characters compared to the previous password (root excluded from this last rule).
- **sudo policy**: max 3 password attempts, custom warning message on failure, full logging of input/output to `/var/log/sudo/`, TTY mode enabled, restricted `secure_path`.
- **SSH**: listens on port `4242`, root login disabled (`PermitRootLogin no`).
- **Firewall (UFW)**: default deny, only port `4242/tcp` allowed.
- **AppArmor**: enabled and running at startup.

### User management

Besides `root`, the mandatory user `nkerstin` was created and added to two groups:
- `sudo` — grants administrative privileges
- `user42` — a custom group required by the subject

### Services installed

Only the minimal set of services required by the project:
- `openssh-server` (SSH on port 4242)
- `sudo`
- `ufw`
- `libpam-pwquality` / `cracklib`
- `apparmor` + `apparmor-utils`
- `cron` (to schedule `monitoring.sh`)

No graphical server (X.org/Wayland) was installed, as required.

## 📊 Monitoring Script

`monitoring.sh` is a bash script that gathers and broadcasts (via `wall`) the following information every 10 minutes and at every boot (via `cron`):

- OS architecture & kernel version
- Number of physical CPUs / vCPUs
- RAM usage (used/total + percentage)
- Disk usage (used/total + percentage)
- CPU load percentage
- Last boot date/time
- LVM status (active or not)
- Number of active TCP connections
- Number of logged-in users
- Server's IPv4 and MAC address
- Number of commands executed via `sudo`

## 📚 Resources

- [Born2beRoot subject (42 School)](https://cdn.intra.42.fr/pdf/pdf/) — official project subject
- [Born2beRoot tutorial (GitBook)](https://noreply.gitbook.io/born2beroot) — step-by-step guide used for the installation and configuration (VirtualBox setup, Debian install, LVM/LUKS partitioning, SSH/UFW/sudo/AppArmor configuration, monitoring script and cron)
- [Debian Wiki](https://wiki.debian.org/)
- [Debian AppArmor documentation](https://wiki.debian.org/AppArmor)
- [ArchWiki – LVM](https://wiki.archlinux.org/title/LVM)
- [man pages](https://man7.org/): `sudo(8)`, `sshd_config(5)`, `ufw(8)`, `crontab(5)`, `chage(1)`

### AI usage

An AI assistant (Claude) was used **only** to help draft and format this `README.md` file (structuring the sections, writing the comparison tables, and phrasing the explanations) based on the actual configuration performed on the virtual machine, as described in the [Born2beRoot GitBook tutorial](https://noreply.gitbook.io/born2beroot). No AI was used to generate the server configuration itself, the `monitoring.sh` script, or to complete the actual system administration tasks — those were done manually while following the guide, per the project's AI-usage rules.
