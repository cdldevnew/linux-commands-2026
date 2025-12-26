# Linux Administration — Practical Guide (Step-by-step, Commands & Explanations)

Comprehensive hands-on guide covering common Linux administration tasks with clear step-by-step instructions, command examples and explanations. Designed for practicing on a VM and turning concepts into repeatable tasks.

Contents
- Introduction & prerequisites
- User & access management
- LVM, storage & filesystems
- Networking & routing
- Services, boot process & kernels
- SELinux & security
- Logs, monitoring & performance
- NFS, cron, rsync & RAID
- Troubleshooting & recovery
- Useful command reference / cheatsheet
- Appendix: resources & tips

---

## Introduction & prerequisites

This document assumes:
- You have a Linux VM (RHEL/CentOS/Fedora or similar) and root or sudo access.
- Familiarity with basic shell navigation: `cd`, `ls`, `cat`, `vi`/`vim`.
- Always test destructive commands (disk, lvremove, format) on disposable VMs.
- Keep backups/snapshots before changes that can't be trivially reversed.

How to use this guide:
- Follow the step-by-step exercises on a test VM.
- Read the explanation before running commands.
- Use `--dry-run` or list commands where available before executing.

---

## 1. User & Access Management

Goal: manage users safely, set passwords, control expiry and access, and enable sudo.

1.1 Create a user
- Command:
  - `sudo useradd alice`
  - `sudo passwd alice` (then set password)
- Explanation: `useradd` creates an account; `passwd` sets a password interactively.

1.2 Deleting a user and their home
- Command:
  - `sudo userdel -r alice`
- Explanation: `-r` removes the home directory and mail spool.

1.3 Modify user attributes
- Change shell:
  - `sudo usermod -s /bin/bash alice`
- Add to groups:
  - `sudo usermod -aG wheel,developers alice`
- Explanation: `-aG` appends supplementary groups; without `-a` you'll overwrite groups.

1.4 Password aging & expiry
- View:
  - `sudo chage -l alice`
- Set max days (never expiring common setting):
  - `sudo chage -M 99999 alice`
- Force change on next login:
  - `sudo chage -d 0 alice`
- Explanation: `chage` controls expiry, inactivity and minimum/maximum password age.

1.5 Lock/unlock accounts
- Lock: `sudo passwd -l alice`
- Unlock: `sudo passwd -u alice`
- Explanation: `-l` prefixes `!` in `/etc/shadow` to disable authentication.

1.6 Permit sudo access
- Add to wheel (RHEL) or sudo (Debian):
  - `sudo usermod -aG wheel alice`
- Or edit sudoers with `sudo visudo` and add:
  - `alice ALL=(ALL) ALL`
- Explanation: using `visudo` prevents syntax mistakes.

1.7 Restrict login times via PAM
- Edit `/etc/security/time.conf`:
  - Example deny 9PM-7AM: `login;*;alice;!Al2000-0700`
- Ensure `/etc/pam.d/sshd` has:
  - `account required pam_time.so`
- Explanation: PAM `pam_time.so` enforces time-based access control.

1.8 File permission basics
- `chmod 755 script.sh` — owner rwx, group/other rx.
- `chown bob:dev file` — set owner and group.
- Special bits: SUID, SGID, sticky:
  - SUID: `chmod 4755 /usr/bin/someprog` (runs as owner).
  - SGID on directory: `chmod 2775 /shared` (new files inherit group).
  - Sticky bit: `chmod +t /tmp` (prevents deletion by others).

1.9 ACLs for fine-grained control
- Set ACL: `setfacl -m u:john:rwx /data/file1`
- Show: `getfacl /data/file1`
- Remove: `setfacl -b /data/file1`
- Explanation: ACLs allow per-user/per-group permissions beyond the classic owner/group/other bits.

---

## 2. LVM, Storage & Filesystems

Goal: create, grow, shrink (careful), and manage logical volumes and filesystems.

2.1 LVM components & basic commands
- PV (physical volume): `sudo pvcreate /dev/sdb`
- VG (volume group): `sudo vgcreate vgdata /dev/sdb`
- LV (logical volume): `sudo lvcreate -n lvdata -L 5G vgdata`
- Show status: `pvs`, `vgs`, `lvs`, or `pvdisplay`, `vgdisplay`, `lvdisplay`

2.2 Format and mount
- Format (XFS): `sudo mkfs.xfs /dev/vgdata/lvdata`
- Make mount point & mount:
  - `sudo mkdir -p /data`
  - `sudo mount /dev/vgdata/lvdata /data`
- Persist in `/etc/fstab` by UUID:
  - `sudo blkid /dev/vgdata/lvdata`
  - Add: `UUID=<uuid> /data xfs defaults 0 0`
- Explanation: XFS can't shrink; ext4 can be shrunk with steps (see below).

2.3 Extend LV and filesystem (online)
- Extend LV: `sudo lvextend -L +5G /dev/vgdata/lvdata`
- If XFS: `sudo xfs_growfs /data`
- If ext4: `sudo resize2fs /dev/vgdata/lvdata`
- Explanation: extend LV first, then grow the filesystem to use new space.

2.4 Reduce LV (dangerous — backup first)
- Steps for ext4:
  1. Backup data.
  2. Unmount: `sudo umount /data`
  3. fsck: `sudo e2fsck -f /dev/vgdata/lvdata`
  4. shrink filesystem: `sudo resize2fs /dev/vgdata/lvdata 5G`
  5. lvreduce: `sudo lvreduce -L 5G /dev/vgdata/lvdata`
  6. mount and verify.
- Explanation: XFS cannot be shrunk, so plan LVM sizes carefully.

2.5 Move data between PVs
- Add new disk to VG: `sudo vgextend vgdata /dev/sdc`
- Migrate: `sudo pvmove /dev/sdb /dev/sdc`
- Remove old PV: `sudo vgreduce vgdata /dev/sdb`

2.6 Partitioning & detection without reboot
- `sudo fdisk /dev/sdb` (interactive)
- `lsblk` or `fdisk -l` to list
- Rescan SCSI: `echo "- - -" | sudo tee /sys/class/scsi_host/host0/scan` or use `partprobe`.

---

## 3. Networking & Routing

Goal: check and configure network, routes and troubleshoot connectivity.

3.1 Check addresses and routes
- Show IP addresses: `ip addr show` or `ip a`
- Show routes: `ip route` or `route -n`
- Test connectivity: `ping 8.8.8.8`, `curl -I https://example.com`

3.2 Listening ports and services
- `ss -tulnp` or `netstat -tulnp`
- Filter: `ss -ltnp | grep 8080`
- Test port reachability from remote: `nc -zv host 8080` or `telnet host 8080`

3.3 Static IP configuration (RHEL-style)
- Edit `/etc/sysconfig/network-scripts/ifcfg-ens33`:
  ```
  BOOTPROTO=none
  IPADDR=192.168.1.10
  PREFIX=24
  GATEWAY=192.168.1.1
  DNS1=8.8.8.8
  ONBOOT=yes
  ```
- Restart: `sudo systemctl restart NetworkManager`

3.4 DNS & hosts
- Local overrides: edit `/etc/hosts`.
- For debugging: `dig example.com`, `nslookup`, `host`.

3.5 Firewall (firewalld)
- Open port: `sudo firewall-cmd --add-port=1521/tcp --permanent && sudo firewall-cmd --reload`
- Add service: `sudo firewall-cmd --add-service=http --permanent && sudo firewall-cmd --reload`
- Rich rule example:
  - `sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.1" port protocol="tcp" port="22" accept'`

---

## 4. Services, Boot Process & Kernel Management

Goal: manage systemd services, control targets, and handle kernel installations.

4.1 systemd basics
- Start/stop: `sudo systemctl start httpd` / `sudo systemctl stop httpd`
- Enable/disable on boot: `sudo systemctl enable httpd`
- Check status & logs: `sudo systemctl status httpd` and `journalctl -u httpd -f`

4.2 Switch/run targets
- Isolate a target: `sudo systemctl isolate multi-user.target`
- Set default (persist): `sudo systemctl set-default multi-user.target`

4.3 Boot troubleshooting
- View current boot logs: `sudo journalctl -b`
- View dmesg: `dmesg --ctime | less`
- If boot fails due to `/etc/fstab`, boot rescue and comment offending entries.

4.4 Manage kernels and GRUB
- List kernels in GRUB:
  - `grep ^menuentry /boot/grub2/grub.cfg`
- Set default with `grub2-set-default <N>` and regenerate config: `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`
- Reinstall grub (if /boot/grub2 deleted):
  - Boot rescue, chroot into system, `sudo grub2-install /dev/sda` and `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`

4.5 initramfs (rebuild)
- `sudo dracut -f` or for specific kernel:
  - `sudo dracut -f /boot/initramfs-$(uname -r).img $(uname -r)`

---

## 5. SELinux & Security

5.1 SELinux contexts and fixes
- View contexts: `ls -Z /path`
- Restore defaults: `sudo restorecon -Rv /path`
- Temporarily check permissive mode: `sudo setenforce 0` (not recommended in production).
- Persistent booleans: `sudo setsebool -P httpd_can_network_connect 1`

5.2 Password policy & /etc/shadow
- `/etc/passwd` is world-readable; `/etc/shadow` holds hashed passwords and is root-only.
- Never merge them; it leaks password hashes.

5.3 SSH hardening & keys
- Generate key: `ssh-keygen -t ed25519 -C "admin@host"`
- Copy to server: `ssh-copy-id user@host`
- Disable password auth in `/etc/ssh/sshd_config`:
  - `PasswordAuthentication no`, then `sudo systemctl reload sshd`

5.4 Detect brute force login attempts
- `sudo grep "Failed password" /var/log/secure` (RHEL) or `/var/log/auth.log` (Debian)

5.5 umask & default permissions
- Show: `umask`
- Set in shell profile: add `umask 022` to `/etc/profile` or `~/.bashrc`

---

## 6. Logs, Monitoring & Performance

6.1 Read system logs
- `sudo journalctl` for systemd journal
- Classic files: `/var/log/messages`, `/var/log/secure`, `/var/log/cron`, `/var/log/boot.log`

6.2 Live-follow logs
- `sudo journalctl -f` or `tail -F /var/log/httpd/access_log`

6.3 Disk, I/O and memory monitoring
- Disk usage: `df -h`, per-directory: `du -sh /var/*`
- Check open files (deleted but held): `lsof | grep deleted`
- I/O: `iostat -xz 1`, `iotop` for per-process I/O
- Memory: `free -h`, `vmstat 1`, `top`/`htop`

6.4 Load average vs CPU cores
- `uptime` shows load average (1,5,15 minutes). If load > CPU cores → potential CPU saturation.

6.5 Create SOS report for vendor support
- `sudo dnf install sos -y` then `sudo sosreport`

6.6 Automatic log rotation & cleanup
- Use `logrotate` config files under `/etc/logrotate.d/`
- Delete older logs:
  - Dry-run: `find /var/log -type f -mtime +30`
  - Delete: `find /var/log -type f -mtime +30 -delete` (use carefully)

---

## 7. NFS, Cron, Rsync & RAID

7.1 NFS server setup (basic)
- Install: `sudo dnf install nfs-utils -y`
- Enable: `sudo systemctl enable --now nfs-server`
- Create export dir:
  - `sudo mkdir -p /nfsshare && sudo chown nobody:nobody /nfsshare && sudo chmod 0777 /nfsshare`
- Export: append `/etc/exports`:
  - `/nfsshare *(rw,sync)` and `sudo exportfs -rav`
- Open firewall: `sudo firewall-cmd --add-service=nfs --permanent && sudo firewall-cmd --reload`

7.2 Client mount
- `sudo mount -t nfs server:/nfsshare /mnt`
- Persist in `/etc/fstab`: `server:/nfsshare /mnt nfs defaults 0 0`

7.3 Rsync vs scp
- `rsync -avz /src/ user@dest:/dst/`
- Explanation: `rsync` transfers deltas, preserves perms and is efficient for repeated syncs.

7.4 Cron basics
- Edit user crontab: `crontab -e`
- Example backup at 2 AM:
  - `0 2 * * * /usr/local/bin/backup.sh`
- For sub-minute tasks use loops or systemd timers (cron granularity is 1 minute).

7.5 Software RAID (mdadm)
- Create RAID1: `sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1`
- Save config: `sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf`
- Monitor: `cat /proc/mdstat`

---

## 8. Troubleshooting & Recovery

8.1 Common unmount failures
- Busy filesystem: find processes: `lsof +f -- /mountpoint` or `fuser -vm /mountpoint`
- Force unmount: `sudo umount -l /mountpoint` (lazy) or `umount -f` (careful)

8.2 Disk space seeming full after deletion
- Files deleted but held by process: `lsof | grep deleted`
- Restart the process or truncate the file: `> /path/to/file`

8.3 System not booting due to /etc/fstab
- Boot into rescue, edit `/etc/fstab` to comment problematic entries, then `mount -a` and reboot.

8.4 Recovering from kernel/initramfs issues
- Rebuild initramfs: `sudo dracut -f`
- Reinstall kernel: `sudo dnf reinstall kernel` (or `yum` on older systems)

8.5 Find who rebooted
- `last -x` or `last reboot`

8.6 Detect OOM and kernel panic logs
- `dmesg | grep -i oom` and `journalctl -k`

---

## 9. Useful Commands & Cheatsheet

- System info:
  - `uname -a`, `lsb_release -a`, `lscpu`, `lsblk`, `blkid`
- Processes:
  - `ps aux | grep <name>`, `top`, `htop`, `pidof`, `kill -15 <PID>`
- Disk:
  - `df -h`, `du -sh /path`, `lsblk`, `fdisk -l`
- Network:
  - `ip a`, `ip route`, `ss -tulnp`, `nc -zv host port`
- LVM:
  - `pvcreate`, `vgcreate`, `lvcreate`, `lvextend`, `pvmove`, `lvreduce`
- Filesystems:
  - `mkfs.ext4`, `mkfs.xfs`, `resize2fs`, `xfs_growfs`, `fsck -y`
- Logs:
  - `journalctl -b`, `journalctl -u service`, `tail -F /var/log/messages`
- Security:
  - `getenforce`, `setenforce`, `restorecon -Rv /path`, `setsebool -P <bool> 1`
- Backup/sync:
  - `rsync -avz /src/ user@dest:/dst/`, `tar -czf backup.tar.gz /etc`
- Misc:
  - `chage -l user`, `last`, `lastlog`, `who`, `w`, `uptime`

---

## 10. Best Practices & Tips

- Always backup before destructive operations (lvremove, mkfs, fdisk).
- Test changes on staging before production.
- Use configuration management (Ansible, Salt) for repeatable configuration.
- Monitor disk, memory, and CPU; set alerts for thresholds.
- Keep audit logs and rotate logs regularly.
- Use SSH keys and disable password auth for servers.
- Keep systems patched in a controlled maintenance window; have rollback plans (YUM history undo).
- Use LVM snapshots before risky operations if supported and appropriate.

---

## Appendix: Quick Problem-Resolution Recipes

- "Cannot create file though space exists": check `df -h` and `df -i` (inodes); check quotas `quota -u user`; verify filesystem not `ro` with `mount | grep ro`.
- "Port not listening": `ss -tulnp | grep <port>`; if listening locally but unreachable, check firewall `firewall-cmd --list-all`, SELinux booleans, and service config.
- "Increase LV & grow FS": `lvextend -L +XG /dev/vg/lv` then `xfs_growfs /mountpoint` (XFS) or `resize2fs` (ext4).
- "Shrink LV ext4 safely": unmount → `e2fsck -f` → `resize2fs newsize` → `lvreduce` → mount.
- "NFS mount fails": check `rpcbind`, `nfs-server` status, `exportfs -v`, firewall rules, and SELinux booleans.

---

References & further reading
- man pages: `man systemctl`, `man lvm`, `man mount`, `man sshd_config`
- RHEL/CentOS & Fedora docs for `firewalld`, `SELinux`, `dracut`
- Practice by provisioning disposable VMs and reproducing scenarios.

---

End of guide.

Practice repeatedly, automate common steps, and build a personal snippet library of tested commands for your environment.
