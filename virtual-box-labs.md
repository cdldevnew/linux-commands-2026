# VirtualBox Advanced Linux Lab — Complete Workbook (Labs 1–20)

A complete, step‑by‑step VirtualBox lab practice plan that you can perform entirely inside VirtualBox. Each lab includes: objective, prerequisites, exact commands (copy‑paste friendly), verification, recovery/undo, explanatory notes and best practices.

Use snapshots and disposable VMs for any destructive or boot-critical tasks.

---

## Table of Contents

- Lab Prep — VirtualBox baseline and recommended actions
- Section 1 — User, Group & Sudo Advanced Labs (Lab 1–2)
- Section 2 — systemd, Boot & Recovery Labs (Lab 3–4)
- Section 3 — Storage & LVM Advanced Labs (Lab 5–6)
- Section 4 — Networking Real-World Labs (Lab 7–8)
- Section 5 — NFS / Samba / FTP Labs (Lab 9–10)
- Section 6 — Security & SELinux Advanced Labs (Lab 11–12)
- Section 7 — Kernel & Performance Labs (Lab 13–14)
- Section 8 — VirtualBox-specific Labs (Lab 15–16)
- Section 9 — Automation & Scripting Labs (Lab 17–18)
- Section 10 — Troubleshooting Scenario Labs (Lab 19–20)
- Final practice checklist & tips

---

## LAB PREP — VirtualBox baseline and recommended actions

Objective
- Build a repeatable VirtualBox lab with 3 VMs (server1, server2, client1), snapshots, host-only networking, NAT for internet.

Recommended VM config (each)
- OS: RHEL / Rocky / Alma / Ubuntu (consistent or mixed)
- RAM: 2 GB
- Disk: 20–40 GB
- CPUs: 1–2
- NICs: Adapter1 = NAT, Adapter2 = Host-Only
- Enable snapshots, cloning, shared folders

Key VirtualBox operations
- Create VM (GUI or CLI):
  ```bash
  VBoxManage createvm --name server1 --register
  VBoxManage modifyvm server1 --memory 2048 --cpus 1 --nic1 nat --nic2 hostonly --hostonlyadapter2 vboxnet0
  ```
- Take snapshot:
  ```bash
  VBoxManage snapshot server1 take "pre-lab" --description "Baseline snapshot"
  ```
- Linked clone (space-efficient):
  ```bash
  VBoxManage clonevm base --name server1_clone --register --options link
  ```

Host-only network
- Create host network in VirtualBox GUI: File → Host Network Manager → Create vboxnet0
- Configure DHCP or use static host-only addresses.

Tips
- Always snapshot before risky operations (fstab, LVM, kernel)
- Use loopback devices (truncate + losetup) to practice disk ops safely

---

## SECTION 1 — USER, GROUP & SUDO ADVANCED LABS

### Lab 1 — Create project-based access environments

Objective
- Users: dev1, dev2, qa1
- Groups: developers, qa
- Directories: `/apps/dev`, `/apps/qa`
- Requirements: developers have full access to `/apps/dev`, qa have full access to `/apps/qa`. No cross access.

Prereqs
- root / sudo

Commands
```bash
# Create groups and users
sudo groupadd developers
sudo groupadd qa
sudo useradd -m -s /bin/bash -G developers dev1
sudo useradd -m -s /bin/bash -G developers dev2
sudo useradd -m -s /bin/bash -G qa qa1
sudo passwd dev1 && sudo passwd dev2 && sudo passwd qa1

# Create directories and apply SGID + perms
sudo mkdir -p /apps/dev /apps/qa
sudo chown root:developers /apps/dev
sudo chmod 2770 /apps/dev
sudo chown root:qa /apps/qa
sudo chmod 2770 /apps/qa

# Optional per-user ACLs
sudo setfacl -m u:dev1:rwx /apps/dev
sudo setfacl -m u:dev2:rwx /apps/dev
sudo setfacl -m u:qa1:rwx /apps/qa

# Ensure no cross-access (remove ACL entries if present)
sudo setfacl -x u:qa1 /apps/dev || true
sudo setfacl -x u:dev1 /apps/qa || true
```

Verification
```bash
ls -ld /apps/dev /apps/qa
getfacl /apps/dev /apps/qa
# Test as dev1 and qa1 (in separate sessions)
sudo -u dev1 touch /apps/dev/dev1file
sudo -u qa1 ls /apps/dev || echo "qa1 cannot access /apps/dev"
```

Recovery
```bash
sudo rm -rf /apps/dev /apps/qa
sudo userdel -r dev1 dev2 qa1
sudo groupdel developers qa
```

Notes
- SGID (2xxx) ensures files inherit group ownership.
- ACLs allow fine-grained exceptions.
- Set umask (umask 002) for group writable behavior as needed.

---

### Lab 2 — Configure restricted sudo privileges

Objective
- User `ops` allowed to run only reboot and systemctl restart * (no password), everything else denied.

Commands
```bash
sudo useradd -m -s /bin/bash ops
sudo passwd ops

# Create sudoers snippet
sudo visudo -f /etc/sudoers.d/ops
# Add the following lines (inside visudo):
# Cmnd_Alias OPS_CMDS = /sbin/reboot, /bin/systemctl restart *, /bin/systemctl status *
# ops ALL=(root) NOPASSWD: OPS_CMDS
```

Verification
```bash
sudo -l -U ops
sudo -u ops sudo systemctl restart sshd
sudo -u ops sudo /usr/bin/passwd root || echo "not allowed"
```

Recovery
```bash
sudo rm -f /etc/sudoers.d/ops
sudo userdel -r ops
```

Notes
- Use full paths in sudoers.
- Sudo denies commands not explicitly allowed.

---

## SECTION 2 — systemd, BOOT & RECOVERY LABS

### Lab 3 — Recover root password

Objective
- Reset forgotten root password using GRUB boot edit (rd.break) and relabel SELinux if enforced.

Steps (RHEL-family)
1. Reboot and edit GRUB: press `e`, append `rd.break` to linux line.
2. Boot, then:
```bash
mount -o remount,rw /sysroot
chroot /sysroot /bin/bash
passwd root
touch /.autorelabel   # if SELinux enforcing
exit
reboot
```

Ubuntu (init=/bin/bash)
```bash
# At GRUB edit append: init=/bin/bash
mount -o remount,rw /
passwd root
sync
reboot -f
```

Verification
- Reboot normally and login as root with new password:
```bash
su - root
getenforce   # SELinux check
```

Notes
- `rd.break` drops into dracut; `chroot /sysroot` required for correct environment.

---

### Lab 4 — Create custom systemd service

Objective
- `/usr/local/bin/checkdisk.sh` runs at boot, logs `/var/log/diskreport.log`, restarts on failure; enable as `customcheck.service`.

Commands
```bash
sudo tee /usr/local/bin/checkdisk.sh > /dev/null <<'SH'
#!/bin/bash
LOG=/var/log/diskreport.log
echo "Disk report at $(date)" >> "$LOG"
df -h >> "$LOG"
exit 0
SH
sudo chmod +x /usr/local/bin/checkdisk.sh
sudo touch /var/log/diskreport.log
sudo chown root:root /var/log/diskreport.log

sudo tee /etc/systemd/system/customcheck.service > /dev/null <<'UNIT'
[Unit]
Description=Custom Disk Check
After=local-fs.target network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/checkdisk.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now customcheck.service
```

Verification
```bash
sudo systemctl status customcheck.service
tail -n 20 /var/log/diskreport.log
```

Recovery
```bash
sudo systemctl disable --now customcheck.service
sudo rm -f /etc/systemd/system/customcheck.service /usr/local/bin/checkdisk.sh /var/log/diskreport.log
sudo systemctl daemon-reload
```

Notes
- Use `Type=simple` for long-running services; `oneshot` is for scripts that run and exit.

---

## SECTION 3 — STORAGE & LVM ADVANCED LABS

### Lab 5 — LVM snapshot and restore

Objective
- Create VG `vgdata`, LV `lvdb` 5GB, snapshot `snap1`, corrupt original, restore from snapshot using `lvconvert --merge`.

Safe loopback example
```bash
# Create loop file and setup loop device
sudo truncate -s 10G /tmp/lvmbase.img
sudo losetup -fP /tmp/lvmbase.img
LOOP=$(losetup -j /tmp/lvmbase.img | cut -d: -f1)

# LVM setup
sudo pvcreate "$LOOP"
sudo vgcreate vgdata "$LOOP"
sudo lvcreate -L 5G -n lvdb vgdata
sudo mkfs.ext4 /dev/vgdata/lvdb
sudo mkdir -p /mnt/lvdb
sudo mount /dev/vgdata/lvdb /mnt/lvdb

# Add data
sudo dd if=/dev/urandom of=/mnt/lvdb/file1 bs=1M count=50; sync

# Snapshot
sudo lvcreate -L 1G -s -n snap1 /dev/vgdata/lvdb
sudo lvs -a

# Corrupt original
sudo rm -rf /mnt/lvdb/*; sync
sudo umount /mnt/lvdb

# Merge snapshot back
sudo lvconvert --merge /dev/vgdata/snap1
sudo lvchange -ay /dev/vgdata/lvdb
sudo mount /dev/vgdata/lvdb /mnt/lvdb
ls -l /mnt/lvdb
```

Recovery / cleanup
```bash
sudo umount /mnt/lvdb
sudo lvremove -f /dev/vgdata/lvdb
sudo vgremove -f vgdata
sudo pvremove -f "$LOOP"
sudo losetup -d "$LOOP"
rm -f /tmp/lvmbase.img
```

Notes
- Snapshot size must be large enough to hold changes; snapshots are copy-on-write.

---

### Lab 6 — Extend disk in VirtualBox and OS (no reboot)

Objective
- Increase VDI size in VirtualBox, grow partition, pvresize, lvextend and resize FS online.

Steps (host)
```bash
# On VirtualBox host:
VBoxManage modifymedium disk /path/to/disk.vdi --resize 40960  # size in MB (e.g., 40GB)
```

Guest (grow partition and LVM)
```bash
# Install growpart if needed
sudo apt install -y cloud-guest-utils || sudo yum install -y cloud-utils-growpart

# Identify disk & partition (e.g., /dev/sda1)
lsblk
sudo growpart /dev/sda 1
sudo partprobe /dev/sda
sudo pvresize /dev/sda2   # adjust to your partition device
sudo lvextend -L +10G /dev/vgdata/lvapps   # or -l +100%FREE
# Grow filesystem:
sudo resize2fs /dev/vgdata/lvapps      # ext4 (mounted or unmounted depending on kernel support)
# or for XFS:
sudo xfs_growfs /mnt/lvapps
df -h
```

Notes
- Keep start sector unchanged when resizing partition manually.
- `pvresize` informs LVM of the larger PV size.

---

## SECTION 4 — NETWORKING REAL-WORLD LABS

### Lab 7 — Multi-NIC configuration

Objective
- Configure NIC1 NAT (internet), NIC2 Host-only (internal). Static IP on host-only, verify routing and inter-VM connectivity.

Commands (nmcli example)
```bash
# Identify interfaces
ip link

# Configure host-only static IP on enp0s8
sudo nmcli con add type ethernet ifname enp0s8 con-name hostonly ipv4.addresses 192.168.56.10/24 ipv4.method manual
sudo nmcli con up hostonly

# Verify
ip addr show enp0s8
ping -c3 192.168.56.1  # other VM or host-only gateway
ping -c3 8.8.8.8       # internet via NAT
```

Notes
- For netplan/Ubuntu, write YAML and `sudo netplan apply`.

---

### Lab 8 — Linux router VM

Objective
- Use server1 as router: enable IP forwarding and NAT (MASQUERADE) so client1 reaches internet via server1.

Commands (iptables example)
```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
sudo tee /etc/sysctl.d/99-sysctl.conf > /dev/null <<'SYS'
net.ipv4.ip_forward = 1
SYS
sudo sysctl --system

# Set NAT (assuming enp0s3=external, enp0s8=internal)
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
# Persist rules as appropriate (iptables-save or firewall-cmd)
```

Client (set gateway)
```bash
sudo ip route add default via 192.168.56.101  # server1 host-only IP
ping -c3 8.8.8.8
traceroute 8.8.8.8
```

Verification
- `tcpdump -i enp0s8 icmp` on server1 to observe traffic
- `iptables -t nat -L -n -v`

Notes
- Use firewall-cmd `--add-masquerade` if using firewalld.

---

## SECTION 5 — NFS / SAMBA / FTP LABS

### Lab 9 — NFS shared storage lab

Server (server2)
```bash
# Install and enable NFS
sudo dnf install -y nfs-utils || sudo apt install -y nfs-kernel-server
sudo mkdir -p /nfsshare
sudo chown nfsnobody:nfsnobody /nfsshare
sudo chmod 755 /nfsshare

# Export with root_squash for host-only network
echo "/nfsshare 192.168.56.0/24(rw,sync,root_squash)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl enable --now nfs-server
sudo exportfs -v

# Firewall (RHEL)
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```

Client (server1)
```bash
sudo mkdir -p /mnt/nfsdata
sudo mount -t nfs server2:/nfsshare /mnt/nfsdata
# Make permanent in /etc/fstab:
# server2:/nfsshare /mnt/nfsdata nfs defaults,_netdev 0 0
```

Verification
- `mount | grep nfs`
- `ls -l /nfsshare` on server2 shows files created by clients (root_squash maps root to nfsnobody)

---

### Lab 10 — Samba with AD-like permissions

Server
```bash
sudo dnf install -y samba || sudo apt install -y samba
sudo groupadd windowsusers
sudo useradd -m -G windowsusers alice; sudo passwd alice
sudo mkdir -p /sambadata
sudo chown root:windowsusers /sambadata
sudo chmod 2770 /sambadata

# Add smb share to /etc/samba/smb.conf:
# [sambadata]
#    path = /sambadata
#    valid users = @windowsusers
#    read only = no
#    guest ok = no
#    create mask = 0660
#    directory mask = 2770

sudo smbpasswd -a alice
sudo systemctl enable --now smb nmb  # or smbd nmbd

# Firewall
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
```

Verification
```bash
smbclient //server2/sambadata -U alice
```

Notes
- For AD integration consider winbind/sssd in production.

---

## SECTION 6 — SECURITY & SELINUX ADVANCED LABS

### Lab 11 — Configure SELinux contexts properly

Objective
- Serve content from `/myweb` while SELinux is enforcing.

Commands
```bash
sudo mkdir -p /myweb
echo "<h1>MyWeb</h1>" | sudo tee /myweb/index.html
sudo chown -R apache:apache /myweb   # use 'nginx' user for nginx

# Assign SELinux context
sudo semanage fcontext -a -t httpd_sys_content_t "/myweb(/.*)?"
sudo restorecon -Rv /myweb

# If nonstandard port 8080:
sudo semanage port -a -t http_port_t -p tcp 8080 2>/dev/null || sudo semanage port -m -t http_port_t -p tcp 8080
sudo systemctl restart httpd
```

Verification
```bash
ls -Z /myweb
curl -I http://localhost:8080/
ausearch -m AVC -ts recent
```

Notes
- `semanage` is required (package: policycoreutils-python-utils).

---

### Lab 12 — Audit system changes (auditd)

Commands
```bash
sudo dnf install -y audit || sudo apt install -y auditd
sudo systemctl enable --now auditd

# Add rules
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
sudo auditctl -w /etc/shadow -p wa -k shadow_access
# Persist by adding lines to /etc/audit/rules.d/local.rules:
# -w /etc/passwd -p wa -k passwd_changes
# -w /etc/shadow -p wa -k shadow_access

# Trigger and query
sudo useradd testuser; sudo userdel testuser
sudo ausearch -k passwd_changes -i
sudo ausearch -k shadow_access -i
```

How to determine who/when/how
- `ausearch -k passwd_changes -i` shows timestamps, auid (original user), uid, exe used.

Notes
- Keep audit rules focused to avoid excessive logs.

---

## SECTION 7 — KERNEL & PERFORMANCE LABS

### Lab 13 — Kernel tuning (sysctl)

Commands
```bash
# Temporary
sudo sysctl -w fs.file-max=200000
sudo sysctl -w net.ipv4.ip_forward=1

# Persist
sudo tee /etc/sysctl.d/99-custom.conf > /dev/null <<'SYS'
fs.file-max = 200000
net.ipv4.ip_forward = 1
SYS

sudo sysctl --system
sysctl fs.file-max
sysctl net.ipv4.ip_forward
```

Verification
- Reboot and re-run `sysctl` checks.

---

### Lab 14 — Performance tuning scenario

Commands and tools
```bash
# Install tools
sudo dnf install -y sysstat dstat || sudo apt install -y sysstat dstat
sudo dnf install -y stress || sudo apt install -y stress

# Simulate CPU load
stress --cpu 4 --timeout 120 &

# Monitor
top
vmstat 2
mpstat -P ALL 1 3
dstat -cdngy 2 5

# Simulate memory load
stress --vm 1 --vm-bytes 1G --timeout 120 &

# Identify rogue process
ps aux --sort=-%mem | head -n 10

# Graceful kill
sudo kill <pid>
# Forceful
sudo kill -9 <pid>
```

Notes
- Prefer SIGTERM before SIGKILL; use `nice`/`renice` to change priorities.

---

## SECTION 8 — VIRTUALBOX-SPECIFIC LABS

### Lab 15 — Snapshot & rollback practice

Steps
```bash
VBoxManage snapshot server1 take "baseline" --description "Before change"
# Make risky change (example: break fstab)
sudo cp /etc/fstab /etc/fstab.bak
sudo sed -i '1 s/^/#/' /etc/fstab
sudo reboot
# If boot fails, from host:
VBoxManage controlvm server1 poweroff
VBoxManage snapshot server1 restore "baseline"
VBoxManage startvm server1
```

Notes
- Snapshot rollback reverts disk & state.

---

### Lab 16 — Linked Clone & Networking

Steps
```bash
# Create base VM and prepare (clean machine-id & SSH host keys)
sudo rm -f /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*

# On host create linked clones
VBoxManage clonevm server-base --name child1 --register --mode machine --options link
VBoxManage clonevm server-base --name child2 --register --mode machine --options link

# Boot clones and set hostname
sudo hostnamectl set-hostname child1.example.lab
```

Notes
- Ensure guest re-generation of unique identifiers on first boot.

---

## SECTION 9 — AUTOMATION & SCRIPTING LABS

### Lab 17 — Automated user provisioning script

Script: `/usr/local/bin/provision_users.sh`
```bash
sudo tee /usr/local/bin/provision_users.sh > /dev/null <<'SH'
#!/bin/bash
IN="$1"
LOG="/var/log/user_provision.log"
while IFS=, read -r user group; do
  [ -z "$user" ] && continue
  getent group "$group" >/dev/null || groupadd "$group"
  if id "$user" &>/dev/null; then
    echo "$(date) SKIP $user exists" >> "$LOG"
    continue
  fi
  useradd -m -g "$group" "$user"
  PW=$(openssl rand -base64 12)
  echo "$user:$PW" | chpasswd
  chage -d 0 "$user"
  echo "$(date) CREATED $user in $group pwd:$PW" >> "$LOG"
done < "$IN"
SH
sudo chmod +x /usr/local/bin/provision_users.sh
```

Usage
```bash
sudo /usr/local/bin/provision_users.sh /root/users.csv
```

Security note
- Avoid logging plaintext passwords in production.

---

### Lab 18 — Service monitoring script

Script: `/usr/local/bin/service_monitor.sh`
```bash
sudo tee /usr/local/bin/service_monitor.sh > /dev/null <<'SH'
#!/bin/bash
IN=${1:-/root/services.txt}
LOG=/var/log/service_monitor.log
while read -r svc; do
  [ -z "$svc" ] && continue
  if systemctl is-active --quiet $svc; then
    echo "$(date) OK $svc" >> $LOG
  else
    echo "$(date) DOWN $svc restarting" >> $LOG
    systemctl restart $svc && echo "$(date) RESTARTED $svc" >> $LOG || echo "$(date) FAILED $svc" >> $LOG
  fi
done < $IN
SH
sudo chmod +x /usr/local/bin/service_monitor.sh
```

Cron (example)
```bash
(sudo crontab -l 2>/dev/null; echo "* * * * * /usr/local/bin/service_monitor.sh /root/services.txt") | sudo crontab -
```

Verification
- Stop a service and check `/var/log/service_monitor.log` after next run.

Notes
- Prefer native systemd watchdog for production.

---

## SECTION 10 — TROUBLESHOOTING SCENARIO LABS

### Lab 19 — Yum/DNF broken repo fix

Steps
```bash
# Simulate bad repo: create /etc/yum.repos.d/bad.repo with wrong baseurl
# Fix by disabling or correcting:
sudo sed -i 's/enabled=1/enabled=0/' /etc/yum.repos.d/bad.repo

# Clean caches and rebuild
sudo dnf clean all
sudo dnf makecache
sudo rpm --rebuilddb
sudo dnf update
```

Notes
- Import correct GPG key if GPG failures occur.

---

### Lab 20 — System boot failure / Emergency mode

Simulate broken fstab or missing grub.cfg, then recover.

Recover steps (rescue media or GRUB edit)
```bash
# Boot rescue via GRUB (append systemd.unit=rescue.target) or use rescue ISO
# From rescue shell:
mount -o remount,rw /sysroot
chroot /sysroot /bin/bash   # if dracut
# Restore fstab
mv /etc/fstab.bak /etc/fstab
# If grub.cfg missing:
grub2-mkconfig -o /boot/grub2/grub.cfg   # BIOS
# Rebuild initramfs if required:
dracut -f
exit
reboot
```

Notes
- Keep rescue ISO attached for VirtualBox VMs for easy recovery.

---

## FINAL PRACTICE CHECKLIST & BEST PRACTICES

- Snapshot before risky operations.
- Use loopback files for disk/LVM practice: `truncate -s 3G /tmp/test.img && losetup -fP /tmp/test.img`
- Always backup config files before editing:
  ```bash
  sudo cp /etc/fstab /etc/fstab.bak
  sudo cp /etc/sudoers /etc/sudoers.bak
  ```
- Use `visudo` and `sshd -t` / `nginx -t` to validate configs before reload.
- For SELinux: prefer `semanage fcontext` + `restorecon` over `chcon`.
- For services, prefer systemd's native restart/watchdog features over cron.
- Document changes in a CHANGELOG; practice rollbacks and restores regularly.

---
