# Linux Administration Lab — Questions & Detailed Answers

A hands-on lab repository containing Linux administration tasks with commands and verification steps. This repo is designed for learners preparing for RHCSA / RHCE, system administrators honing practical skills, and anyone wanting structured lab practice.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Who is this for?](#who-is-this-for)
- [Repository Structure](#repository-structure)
- [How to use this lab (recommended workflow)](#how-to-use-this-lab-recommended-workflow)
- [Environment & Prerequisites (important safety notes)](#environment--prerequisites-important-safety-notes)
- [Lab Sets — Summary of Content](#lab-sets--summary-of-content)
  - [Set 1 — Basic Administration](#set-1-—-basic-administration)
  - [Set 2 — Disks, ACL, Firewalld, SELinux](#set-2-—-disks-acl-firewalld-selinux)
  - [Set 3 — Services, Runlevels, Processes, Tar](#set-3-—-services-runlevels-processes-tar)
  - [Set 4 — Networking, Sticky bit, Chrony, Swap, Systemd](#set-4-—-networking-sticky-bit-chrony-swap-systemd)
- [Verification & Validation Pattern](#verification--validation-pattern)
- [Common destructive commands — use with caution](#common-destructive-commands---use-with-caution)
- [Quick cheat-sheet: frequently used commands](#quick-cheat-sheet-frequently-used-commands)
- [Best practices & exam tips](#best-practices--exam-tips)
- [Troubleshooting & debugging tips](#troubleshooting--debugging-tips)
- [Contributing](#contributing)
- [License & Contact](#license--contact)

---

## Project Overview

This repository contains 4 sets of lab tasks (10 tasks each). Each task shows the problem statement and a concise step-by-step solution with the exact commands to run, plus verification steps. The goal is to provide repeatable, practice-ready labs for core Linux administration topics.

---

## Who is this for?

- Students preparing for RHCSA / RHCE.
- Junior to intermediate sysadmins who need a hands-on checklist.
- Instructors building lab exercises.
- Anyone wanting structured, command-centered practice.

---

## Repository Structure

- README.md — (this document)
- set-1.md — Basic Administration tasks and answers (if present)
- set-2.md — Disks, ACL, Firewalld, SELinux
- set-3.md — Services, Runlevels, Processes, Tar
- set-4.md — Networking, Sticky bit, Chrony, Swap, Systemd
- scripts/ — optional helper scripts referenced by tasks (e.g., example /root/backup.sh)
- assets/ — optional screenshots, diagrams, sample files

(Note: file names above match examples in the repo; if your repo uses different filenames, adapt accordingly.)

---

## How to use this lab (recommended workflow)

1. Clone the repository:
   ```bash
   git clone https://github.com/cdldevnew/linux-commands-2026.git
   cd linux-commands-2026
   ```

2. Read one set at a time. Each task includes:
   - Commands to perform the task
   - Verification commands to confirm results
   - Notes or cautions where relevant

3. Run commands in a controlled environment (VM, container, or snapshot) — see [Environment & Prerequisites](#environment--prerequisites-important-safety-notes).

4. Practice repeating tasks from memory and then re-check with the answers in the set.

5. Use the verification commands after each step to ensure intended state.

---

## Environment & Prerequisites (important safety notes)

- Many tasks require root privileges; use a VM or a test server.
- Recommended environment:
  - A virtual machine (KVM, VirtualBox, VMware) with at least 2 GB RAM and 20 GB disk.
  - Snapshot capability so you can roll back after destructive operations.
  - Distribution: CentOS / RHEL / Rocky / AlmaLinux are closest to RHCSA/RHCE targets. Debian/Ubuntu equivalents often have different tools (apt vs dnf/rpm).
- Use caution with disk/partition, LVM, and mkfs commands — they can destroy data.
- If you don't have spare block devices, use loopback devices (losetup + truncate) for safe testing.

Example: create a safe loop device for LVM testing:
```bash
# create a 3GB file and use it as a loop device
truncate -s 3G /tmp/testdisk.img
losetup -fP /tmp/testdisk.img
# find loop device:
losetup -a
# then replace /dev/sdb in lab commands with /dev/loopX
```

---

## Lab Sets — Summary of Content

Each set includes the command(s) to accomplish a task and verification steps.

### Set 1 — Basic Administration
Covers:
- User and group creation (`useradd`, `groupadd`, `usermod`)
- Permissions (`chmod`), owners (`chown`)
- Password and password aging (`passwd`, `chage`)
- Process discovery (`pidof`, `ps`)
- Systemd service status and failed units
- Cron jobs
- LVM basics (pvcreate, vgcreate, lvcreate) and filesystem creation
- Package query (rpm/dnf)

Example task:
- Create user `student1` with home and bash:
  ```bash
  useradd -m -s /bin/bash student1
  passwd student1
  id student1
  ls -ld /home/student1
  ```

### Set 2 — Disks, ACL, Firewalld, SELinux
Covers:
- Managing users without home dirs
- Locking accounts
- SGID directories for group inheritance
- POSIX ACLs (setfacl/getfacl)
- Listing disks and partitioning (`lsblk`, `fdisk`)
- Formatting (XFS) and mounting permanently (`/etc/fstab`)
- Firewall management (`firewall-cmd`)
- SELinux mode (`getenforce`, `setenforce`)

Example:
- Permanently mount /dev/sdc1 to /mnt/data using its UUID in /etc/fstab.

### Set 3 — Services, Runlevels, Processes, Tar
Covers:
- Shared group setup and permissions
- Creating root-only directories
- Managing systemd default target
- Listing enabled services
- Hostname and timezone management
- Process listing and killing (`ps`, `pkill`)
- Tar backup and extraction

Example:
- Create tar.gz backup of /etc:
  ```bash
  mkdir -p /backup
  tar -czvf /backup/etc.tar.gz /etc
  tar -xzvf /backup/etc.tar.gz -C /restore
  ```

### Set 4 — Networking, Sticky bit, Chrony, Swap, Systemd
Covers:
- Group management and primary groups
- Password aging policy
- Sticky bit for shared directories (e.g., /tmp-like behavior)
- Networking commands (`ip addr`, `ip route`)
- Adding temporary routes
- Chrony NTP client setup and validation
- Creating swap partitions and enabling permanently
- Finding large files
- Showing listening ports (`ss`)
- Creating and enabling systemd service files

Example:
- Create systemd unit file /etc/systemd/system/customapp.service, then enable/start:
  ```bash
  systemctl daemon-reload
  systemctl enable --now customapp.service
  systemctl status customapp.service
  ```

---

## Verification & Validation Pattern

For each task, follow this pattern:
1. Run the command(s) provided by the lab.
2. Immediately run the verification commands provided.
3. If verification fails:
   - Re-check the command output and logs (`journalctl -xe` or `systemctl status` for services).
   - Use `ls -l`, `id`, `mount`, `blkid`, `getfacl`, `getenforce`, etc., depending on task.

Example: verifying an LVM and mount
```bash
# After creating LV and filesystem
blkid /dev/vgclass/lvbackup
mount | grep /backup
df -h /backup
```

---

## Common destructive commands — use with caution

The following commands can permanently destroy data. Always have backups or snapshots before running them on production systems:

- fdisk, parted (writing partition tables)
- pvcreate, vgcreate, lvcreate (when used on devices with existing data)
- mkfs.* (mkfs.ext4, mkfs.xfs)
- dd (especially without caution)
- rm -rf (dangerous when combined with root privileges)
- wipefs

When testing, prefer loopback files or disposable VMs.

---

## Quick cheat-sheet: frequently used commands

- Create user with home and shell:
  ```bash
  useradd -m -s /bin/bash username
  passwd username
  ```
- Add user to supplementary group:
  ```bash
  usermod -aG groupname username
  ```
- Set directory SGID:
  ```bash
  chmod 2775 /shared
  ```
- Set an ACL:
  ```bash
  setfacl -m u:anna:rw /shared/report.txt
  getfacl /shared/report.txt
  ```
- Create LV and mount:
  ```bash
  pvcreate /dev/sdb
  vgcreate vgclass /dev/sdb
  lvcreate -L 2G -n lvbackup vgclass
  mkfs.ext4 /dev/vgclass/lvbackup
  mount /dev/vgclass/lvbackup /backup
  ```
- Firewall (firewalld):
  ```bash
  firewall-cmd --permanent --add-port=443/tcp
  firewall-cmd --reload
  ```
- SELinux:
  ```bash
  getenforce
  setenforce 0   # not persistent
  ```
- Systemd:
  ```bash
  systemctl enable --now service
  systemctl status service
  systemctl list-unit-files --type=service --state=enabled
  ```

---

## Best practices & exam tips

- Practice inside snapshots or throwaway VMs. Roll back often.
- Memorize the verification commands — examiners expect you to show the correct system state.
- Read man pages: `man useradd`, `man chage`, `man setfacl`, `man systemctl`.
- Time management: in an exam, attempt quick tasks first (user creation, basic file perms) and save partitions/LVM for later.
- Understand the difference between ownership, permission bits, and ACLs.
- Know how to read logs: `journalctl -u <service>` and `journalctl -xe`.

---

## Troubleshooting & debugging tips

- Service not starting? `systemctl status <unit>` then `journalctl -u <unit> --no-pager`.
- Mount or fstab issue? Run `mount -a` and check `/etc/fstab` entries and `dmesg` for kernel messages.
- LVM issues? `pvs`, `vgs`, `lvs` show states; `pvdisplay`, `vgdisplay`, `lvdisplay` give details.
- SELinux denials? Check `ausearch -m AVC -ts recent` or `journalctl -t setroubleshoot` (if installed) and `sealert -a /var/log/audit/audit.log`.
- Disk/partitioning mistakes: if you accidentally overwrite a partition table, stop and restore from snapshot/backups.

---

## Contributing

Contributions are welcome. Suggested contribution workflow:
1. Fork the repository.
2. Create a branch for your changes: `git checkout -b feature/add-examples`
3. Add or improve tasks, add safety notes or scripts that create safe loop devices for testing.
4. Submit a pull request with a clear description of your changes.

Please avoid adding destructive automation without safeguards (e.g., ensure scripts check for environment or require confirmation).

---

## License & Contact

- License: Add a LICENSE file in this repository (for example, MIT or CC-BY-SA) depending on your preferred licensing.
- Contact / author: cdldevnew (GitHub)

---

Thank you for using this lab repository. Practice frequently, use snapshots, and focus on understanding the why behind each command — that will make the tasks stick and prepare you for real-world administration or certification exams.
