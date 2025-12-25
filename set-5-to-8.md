# Sets 5–8 & Bonus — Detailed Solutions, Explanations, Verification and Recovery

This document contains complete, step‑by‑step answers for Sets 5, 6, 7, 8 and the Shell Scripting bonus tasks you provided. For each task you will find:

- Objective
- Prerequisites / assumptions
- Exact commands (copy‑paste friendly)
- Verification steps
- Recovery / undo steps
- Notes, pitfalls and exam tips

Use these on a disposable VM or with loopback devices for any disk/LVM operations.

---

SET 5 — ADVANCED PRACTICE (solutions)

Task 1 — Configure sudo access for a user
- Objective
  - Create user `opsuser` and allow them to run only systemctl commands without being prompted for a password. Deny all other root operations.

- Prereqs
  - root or sudo on the system
  - visudo available (always use visudo to edit sudoers)

- Commands
  ```bash
  # 1. Create the user with a home
  sudo useradd -m -s /bin/bash opsuser
  sudo passwd opsuser

  # 2. Create a sudoers file snippet for opsuser using visudo (safe)
  # Use the visudo -f to create a dedicated file under /etc/sudoers.d
  sudo visudo -f /etc/sudoers.d/opsuser
  ```
  In the editor, add the following lines (exact):
  ```
  # Allow opsuser to run only systemctl (and nothing else) without password
  Cmnd_Alias SYSTEMCTL_CMDS = /bin/systemctl, /usr/bin/systemctl
  opsuser ALL=(root) NOPASSWD: SYSTEMCTL_CMDS

  # Deny escalation for everything else explicitly (best-effort; sudo denies anything not allowed)
  ```
  Save and exit.

- Verification
  ```bash
  # As opsuser attempt to run systemctl without password:
  sudo -l -U opsuser      # lists sudo privileges for opsuser
  sudo -u opsuser sudo systemctl status --no-pager sshd.service
  # Try a command not allowed:
  sudo -u opsuser sudo /bin/ls /root || echo "not allowed"
  ```

- Recovery
  ```bash
  # remove the sudoers file
  sudo rm -f /etc/sudoers.d/opsuser
  sudo visudo -c  # sanity check
  sudo userdel -r opsuser
  ```

- Notes / Pitfalls
  - sudo denies any command not explicitly allowed; you don't need an explicit deny entry.
  - Always use visudo to avoid locking out sudo.
  - Limiting by full path is important; include both `/bin/systemctl` and `/usr/bin/systemctl` depending on distro.

---

Task 2 — Create and manage archived log backups
- Objective
  - Find files in /var/log older than 7 days, compress into /backup/logs-<date>.tar.gz, delete originals safely, and verify extraction.

- Prereqs
  - /backup exists and has sufficient space (create if absent)
  - run as root (logs often owned by root)

- Commands
  ```bash
  # Prepare
  sudo mkdir -p /backup
  DATE=$(date +%F)                       # e.g., 2025-12-25
  ARCHIVE="/backup/logs-${DATE}.tar.gz"
  TMPFILE=$(mktemp /tmp/logfilelist.XXXX)

  # Find regular files older than 7 days and list them to a file (exclude directories)
  sudo find /var/log -type f -mtime +7 -print > "$TMPFILE"

  # Inspect list before doing anything
  echo "Files to archive:"
  sudo sed -n '1,200p' "$TMPFILE"

  # Create the tar.gz archive from the list (safe) WITHOUT removing files yet
  sudo tar -czf "$ARCHIVE" -T "$TMPFILE" --warning=no-file-changed

  # Verify the archive contents
  sudo tar -tzf "$ARCHIVE" | sed -n '1,30p'

  # After verifying archive is good, remove original files safely
  # Option A: Remove only files explicitly archived
  sudo xargs -a "$TMPFILE" -r -d '\n' rm -f --

  # Option B (single command approach): tar --remove-files (use with care)
  # sudo tar -czf "$ARCHIVE" --remove-files -T "$TMPFILE"
  ```

- Verification (extraction test)
  ```bash
  # Create a temporary dir to test extraction
  TMPDIR=$(mktemp -d /tmp/logs-extract.XXXX)
  sudo tar -xzf "$ARCHIVE" -C "$TMPDIR"
  ls -al "$TMPDIR" | head -n 40
  # when satisfied remove test
  sudo rm -rf "$TMPDIR"
  ```

- Recovery
  - If you accidentally deleted files without a backup, stop and restore from snapshot. That's why the two-step approach (create archive first, validate, THEN delete originals) is recommended.

- Notes / Pitfalls
  - Some log files are actively written by services — compressing and deleting may cause services to recreate empty files; rotate logs using logrotate as a better long term solution.
  - Use `lsof` or `fuser` to check if any files are held open (if you need to rotate/truncate instead).

---

Task 3 — Create LVM with extension
- Objective
  - Create VG `vgdata` across two disks, create LV `lvapps` 4GB, extend it to 8GB and extend the filesystem online.

- Prereqs / safety
  - Use two spare block devices. If you don't have physical disks, create two loopback files (recommended).
  - Commands run as root.

- Commands (using loopback safe example)
  ```bash
  # Create two 6GB loopback files
  sudo truncate -s 6G /tmp/lvm-disk1.img
  sudo truncate -s 6G /tmp/lvm-disk2.img
  sudo losetup -fP /tmp/lvm-disk1.img
  sudo losetup -fP /tmp/lvm-disk2.img
  LOOP1=$(losetup -j /tmp/lvm-disk1.img | cut -d: -f1)
  LOOP2=$(losetup -j /tmp/lvm-disk2.img | cut -d: -f1)

  # Create PVs on the loops
  sudo pvcreate "$LOOP1" "$LOOP2"

  # Create VG across both PVs
  sudo vgcreate vgdata "$LOOP1" "$LOOP2"

  # Create LV of 4G
  sudo lvcreate -L 4G -n lvapps vgdata

  # Create filesystem (example: ext4)
  sudo mkfs.ext4 /dev/vgdata/lvapps

  # Mount it
  sudo mkdir -p /mnt/lvapps
  sudo mount /dev/vgdata/lvapps /mnt/lvapps

  # Verify initial size
  df -h /mnt/lvapps
  ```

- Extend LV to 8GB and grow filesystem online
  ```bash
  # Extend logical volume by additional 4G (4G -> 8G)
  sudo lvextend -L +4G /dev/vgdata/lvapps

  # If ext4 use resize2fs (works online)
  sudo resize2fs /dev/vgdata/lvapps

  # If XFS was used, use xfs_growfs on the mount point:
  # sudo xfs_growfs /mnt/lvapps

  # Verify new size
  df -h /mnt/lvapps
  sudo lvs /dev/vgdata/lvapps
  ```

- Recovery (remove created LVM and loop devices)
  ```bash
  sudo umount /mnt/lvapps
  sudo lvremove -f /dev/vgdata/lvapps
  sudo vgremove -f vgdata
  sudo pvremove -f "$LOOP1" "$LOOP2"
  sudo losetup -d "$LOOP1" "$LOOP2"
  rm -f /tmp/lvm-disk1.img /tmp/lvm-disk2.img
  ```

- Notes
  - `resize2fs` for ext4 can run while the filesystem is mounted.
  - If PVs are real disks, double-check device names to avoid data loss.
  - Use `pvs`, `vgs`, `lvs` to inspect state.

---

Task 4 — Apache / Nginx administration task
- Objective
  - Install a web server, enable at boot, set custom index, change default port 80 → 8080, open firewall, validate via curl.

- Commands (example uses nginx; alternate apache steps provided)
  ```bash
  # Install nginx (RHEL family)
  sudo dnf install -y nginx         # or: sudo apt install -y nginx

  # Enable and start
  sudo systemctl enable --now nginx

  # Default index file location: /usr/share/nginx/html/index.html (RHEL) or /var/www/html/index.html (apache)
  # Replace content with custom page:
  echo "<html><body><h1>Welcome from nginx on port 8080</h1></body></html>" | sudo tee /usr/share/nginx/html/index.html

  # Change listen port to 8080
  # Edit /etc/nginx/nginx.conf or /etc/nginx/conf.d/default.conf (server block)
  # Safe sed approach (edit server block listen directive)
  sudo sed -i 's/listen\s\+80;/listen 8080;/' /etc/nginx/nginx.conf 2>/dev/null || \
    sudo sed -i 's/listen 80;/listen 8080;/' /etc/nginx/conf.d/default.conf 2>/dev/null

  # If SELinux is enforcing, allow http port 8080:
  sudo yum install -y policycoreutils-python-utils 2>/dev/null || true
  sudo semanage port -a -t http_port_t -p tcp 8080 2>/dev/null || \
    sudo semanage port -m -t http_port_t -p tcp 8080

  # Reload nginx
  sudo nginx -t
  sudo systemctl restart nginx

  # Open firewall port 8080
  sudo firewall-cmd --permanent --add-port=8080/tcp
  sudo firewall-cmd --reload

  # Validate locally
  curl -i http://localhost:8080
  ```

- Apache alternative (if you prefer apache/httpd)
  ```bash
  sudo dnf install -y httpd
  sudo systemctl enable --now httpd
  echo "<h1>Hello Apache on 8080</h1>" | sudo tee /var/www/html/index.html
  sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf
  sudo firewall-cmd --permanent --add-port=8080/tcp
  sudo firewall-cmd --reload
  # SELinux: semanage port -a -t http_port_t -p tcp 8080
  sudo systemctl restart httpd
  curl -i http://localhost:8080
  ```

- Verification
  ```bash
  # Service listening
  ss -tulnp | grep 8080
  # HTTP response
  curl -sS http://127.0.0.1:8080 | head -n 5
  # firewall rules
  sudo firewall-cmd --list-ports
  # SELinux port mapping
  sudo semanage port -l | grep http | grep 8080 || true
  ```

- Recovery
  ```bash
  sudo firewall-cmd --permanent --remove-port=8080/tcp
  sudo firewall-cmd --reload
  # revert nginx/apaches config edits and restart or uninstall:
  sudo systemctl stop nginx
  sudo dnf remove -y nginx
  sudo semanage port -d -t http_port_t -p tcp 8080 2>/dev/null || true
  ```

- Notes
  - If you edit `nginx.conf`, ensure you modify the correct server block; use `nginx -t` to test config.
  - SELinux must be updated when using non-standard http ports.

---

Task 5 — User resource limits
- Objective
  - Limit user `student` to max 30 processes and max 100 open files via /etc/security/limits.conf and validate.

- Prereqs
  - student account exists (create if not)

- Commands
  ```bash
  # Create user if missing
  sudo id -u student >/dev/null 2>&1 || sudo useradd -m student

  # Add limits to /etc/security/limits.conf
  sudo cp /etc/security/limits.conf /etc/security/limits.conf.bak
  cat <<'EOF' | sudo tee -a /etc/security/limits.conf
  # Custom limits for student
  student hard nproc 30
  student soft nproc 30
  student hard nofile 100
  student soft nofile 100
  EOF

  # Ensure pam_limits is enabled in PAM session (RHEL/CentOS use system-auth)
  # On many distributions this is already configured. Check:
  grep -E "pam_limits.so" /etc/pam.d/* || true
  ```

  If pam_limits is not present in session, add it to the proper PAM file (example for Debian/Ubuntu: /etc/pam.d/common-session):
  ```bash
  # Debian/Ubuntu example (only if missing)
  sudo sed -i '/session\s\+required\s\+pam_permit.so/ a session required pam_limits.so' /etc/pam.d/common-session
  ```

- Verification
  ```bash
  # Start a new login shell as student to get updated limits
  sudo su - student -s /bin/bash -c 'ulimit -u; ulimit -n'

  # To test process limit, try to spawn many sleep processes (as student)
  sudo su - student -s /bin/bash <<'BASH'
  for i in $(seq 1 35); do sleep 1000 & echo "started $!"; done 2>/dev/null
  ps -u student | wc -l
  ulimit -u
  pkill -u student
  BASH
  ```

- Recovery
  ```bash
  sudo cp /etc/security/limits.conf.bak /etc/security/limits.conf
  ```

- Notes
  - Changes take effect at login (open a new session).
  - System-wide process limits can also be affected by systemd or PAM.
  - Some distros set `nproc` in limits.d files — check `/etc/security/limits.d/`.

---

SET 6 — TROUBLESHOOTING

Task 6 — Fix a system boot failure due to /etc/fstab
- Objective
  - Boot into rescue/emergency mode, mount root, correct /etc/fstab and reboot.

- Prereqs / assumptions
  - You have console/VM access to the host's boot loader (GRUB)
  - Basic familiarity with grub menu options to pass kernel parameters

- Steps (high-level then commands)
  1. At boot, interrupt GRUB and edit the kernel line:
     - Append systemd.unit=rescue.target or emergency to the kernel command line.
     - Examples: press 'e' in GRUB, find the linux line and append: systemd.unit=rescue.target

  2. Boot with the edited entry. You will be in rescue or emergency shell.

  3. If you are in an initramfs shell or dracut shell, find root:
     - For RHEL/CentOS with dracut: `switch_root:/# /bin/sh` or use `chroot /sysroot`.
  4. Mount root rw and edit /etc/fstab.

- Commands (once you have rescue shell)
  ```bash
  # Remount root as read-write (if read-only)
  mount -o remount,rw /

  # If in dracut, you may need:
  # chroot /sysroot
  chroot /sysroot /bin/bash || true

  # Inspect fstab
  cat /etc/fstab

  # If a bad entry exists (example: a missing device or wrong UUID), comment it out:
  sed -i.bak -E 's|^([^#].*bad-device.*)$|#\1|' /etc/fstab || true

  # Better: identify the offending line; e.g., run `systemd-analyze verify /etc/fstab`
  systemd-analyze verify /etc/fstab || true

  # Use blkid/lsblk to verify available devices and UUIDs
  blkid
  lsblk -f

  # Edit /etc/fstab with vi/nano to correct UUIDs or comment problematic lines
  vi /etc/fstab

  # After saving, exit chroot and reboot
  exit
  reboot
  ```

- Verification
  - System should boot normally. Check `systemctl --failed` and `journalctl -b` for messages.

- Recovery
  - If changes make boot worse, re-enter rescue and restore /etc/fstab from `/etc/fstab.bak`.

- Notes / Pitfalls
  - Use UUIDs for fstab to avoid device name changes.
  - For network mounts (NFS), add `_netdev` option to avoid blocking boot.
  - For critical production, always have a rescue ISO or console access.

---

Task 7 — Package management broken dependency issue
- Objective
  - Install a package with missing dependencies, resolve dependencies manually, clear caches and rebuild rpm/dnf cache, verify rpm/db.

- Steps & Commands
  ```bash
  # Example: attempt to install a package
  sudo dnf install -y somepackage || sudo yum install -y somepackage

  # If dependencies fail, find missing dependencies
  sudo dnf provides somepackage || true
  # Or use: sudo dnf repoquery --requires somepackage

  # Enable required repos (example enabling extras or epel)
  # Example: enable EPEL (RHEL/Alma)
  sudo dnf install -y epel-release || true

  # Clear caches and rebuild metadata
  sudo dnf clean all
  sudo dnf makecache

  # Retry install
  sudo dnf install -y somepackage

  # If database issues appear, rebuild rpm db
  sudo rpm --rebuilddb

  # For corrupted rpmdb, consider:
  sudo rm -f /var/lib/rpm/__db* && sudo rpm --rebuilddb

  # Verify package is installed
  rpm -q somepackage
  dnf repoquery --installed somepackage || rpm -qi somepackage
  ```

- Verification
  - `rpm -qa | grep somepackage`
  - `dnf check` to see package dependency issues.

- Notes
  - Avoid forcing installs (`--nodeps`, `--force`) on production systems unless you know consequences.
  - Use `dnf history` and `dnf repoquery` to inspect.

---

Task 8 — SELinux troubleshooting scenario (webserver cannot access /app/html)
- Objective
  - Do not disable SELinux. Check AVC denials, apply correct file context, restore contexts and persist.

- Commands & steps
  ```bash
  # 1. Check AVC denials
  sudo ausearch -m AVC -ts today | sudo aureport -a || sudo grep AVC /var/log/audit/audit.log | tail -n 50

  # Or get latest AVC lines:
  sudo ausearch -m AVC -ts recent --raw | audit2allow -a

  # Example showing that httpd is denied r/o access to /app/html
  sudo grep -i "httpd" /var/log/audit/audit.log | tail -n 50

  # 2. Inspect current SELinux context for the directory
  ls -ldZ /app/html
  ls -Z /app/html/*

  # 3. Apply correct context for web content (persist with semanage + restorecon)
  sudo semanage fcontext -a -t httpd_sys_content_t "/app/html(/.*)?"
  sudo restorecon -Rv /app/html

  # 4. If webserver needs write access (uploads/tmp), set httpd_sys_rw_content_t for specific subdir
  # sudo semanage fcontext -a -t httpd_sys_rw_content_t "/app/html/uploads(/.*)?"
  # sudo restorecon -Rv /app/html/uploads

  # 5. Test (no SELinux disable)
  sudo systemctl restart httpd nginx
  curl -I http://localhost:8080/path-to-app # as appropriate
  ```

- Verification
  - Re-run `ausearch -m AVC -ts recent` to confirm no new denials.
  - Webserver can read files and serves content.

- Notes / Pitfalls
  - `semanage` is in policycoreutils-python-utils on RHEL.
  - Use `audit2allow` to create temporary local allow rules if absolutely necessary — but prefer correct contexts.

---

Task 9 — SSH login problem troubleshooting
- Objective
  - Investigate when a user reports password correct but cannot SSH. Check logs, PAM, shell validity, home permissions, and apply fix.

- Steps & commands
  ```bash
  # 1. Inspect sshd logs (RHEL: /var/log/secure, Debian: /var/log/auth.log)
  sudo tail -n 200 /var/log/secure | grep -i sshd
  sudo tail -n 200 /var/log/auth.log | grep -i sshd

  # 2. Check sshd config
  sudo grep -E "PermitRootLogin|PasswordAuthentication|UsePAM" /etc/ssh/sshd_config
  # If PasswordAuthentication no, enable it for password-based tests:
  # Edit and set: PasswordAuthentication yes
  # Then reload:
  sudo systemctl reload sshd

  # 3. Check PAM config for sshd
  sudo sed -n '1,200p' /etc/pam.d/sshd

  # 4. Check user's shell is valid
  grep '^student:' /etc/passwd
  grep "$(awk -F: '/^student:/{print $7}' /etc/passwd)" /etc/shells || echo "shell invalid"

  # 5. Check home and authorized_keys permissions
  ls -ld /home/student
  ls -l /home/student/.ssh || true
  # Ensure home is not group/world writable
  sudo chmod 700 /home/student
  sudo chown student:student /home/student

  # 6. Check account locked/expired
  sudo passwd -S student
  sudo chage -l student

  # 7. Check SELinux contexts on home (if using ssh keys)
  ls -Z /home/student/.ssh || true

  # 8. Test a manual login:
  sudo su - student -c 'echo "local login OK"'

  # 9. If PAM limits applied (nproc/limits), ensure not hitting maxlogins; check /var/log/secure for "authentication failure" types.
  ```

- Common fixes
  - If shell not present in /etc/shells, change to /bin/bash: `sudo usermod -s /bin/bash student`
  - If account locked: `sudo passwd -u student`
  - Fix permissions: `sudo chmod 700 /home/student; sudo chmod 600 /home/student/.ssh/authorized_keys`
  - Re-enable PasswordAuthentication if intentionally disabled for tests (but prefer key auth in production)

- Verification
  - Test `ssh student@hostname` from client or `ssh -v` to get debug.

- Notes
  - Use `sshd -T` to print effective sshd configuration.
  - For banned IPs, check fail2ban or firewall rules.

---

Task 10 — Crashed process auto recovery
- Objective
  - Create a script to start `httpd` and configure systemd (or cron + watchdog) to ensure it restarts if stopped. Demonstrate killing the process and auto-recovery.

- Approach (preferred): systemd unit for a wrapper is best and standard.

- Commands (systemd unit)
  ```bash
  # Example: ensure httpd has Restart=always (httpd already managed by systemd)
  sudo systemctl edit --full httpd.service <<'EOF'
  [Service]
  Restart=always
  RestartSec=5
  EOF

  # Reload systemd and restart service
  sudo systemctl daemon-reload
  sudo systemctl restart httpd
  sudo systemctl status httpd --no-pager

  # Kill the main PID and watch auto-restart
  sudo systemctl show -p MainPID httpd
  # Find PID:
  PID=$(pidof httpd || pidof apache2)
  sudo kill -9 "$PID"

  # Wait a few seconds then check status; systemd should have restarted it
  sudo systemctl status httpd --no-pager
  ```

- Alternative (monitoring script + cron)
  ```bash
  # /usr/local/bin/check_httpd.sh
  sudo tee /usr/local/bin/check_httpd.sh > /dev/null <<'SH'
  #!/bin/bash
  if ! systemctl is-active --quiet httpd; then
    systemctl start httpd
  fi
  SH
  sudo chmod +x /usr/local/bin/check_httpd.sh

  # Cron: run every minute (root crontab)
  (sudo crontab -l 2>/dev/null; echo "* * * * * /usr/local/bin/check_httpd.sh") | sudo crontab -
  ```

- Verification
  - After killing, `systemctl is-active httpd` becomes active quickly.
  - `journalctl -u httpd -f` shows restart events.

- Recovery
  - Remove systemd override or cron job:
  ```bash
  sudo systemctl revert httpd
  sudo crontab -l | grep -v check_httpd.sh | sudo crontab -
  sudo rm -f /usr/local/bin/check_httpd.sh
  ```

- Notes
  - Use systemd native restart features rather than cron when possible.

---

SET 7 — NETWORKING

Task 1 — Static IP configuration manual method (ens33)
- Objective
  - Assign a permanent IP to `ens33` without GUI or nmtui, with DNS and gateway, then restart networking safely.

- Approach (RHEL/CentOS using nmcli — recommended for NetworkManager systems)
  ```bash
  # Example values (replace with desired addresses)
  IPADDR=192.168.10.50/24
  GATEWAY=192.168.10.1
  DNS1=8.8.8.8

  # Create or modify connection for ens33
  sudo nmcli con modify ens33 ipv4.addresses "$IPADDR"
  sudo nmcli con modify ens33 ipv4.gateway "$GATEWAY"
  sudo nmcli con modify ens33 ipv4.dns "$DNS1"
  sudo nmcli con modify ens33 ipv4.method manual

  # Bring connection down and up
  sudo nmcli con down ens33
  sudo nmcli con up ens33

  # Verify
  ip addr show dev ens33
  ip route show
  nmcli dev show ens33 | grep IP4.DNS
  ```

- Debian/Ubuntu (netplan method)
  ```yaml
  # create /etc/netplan/01-static.yaml
  network:
    version: 2
    ethernets:
      ens33:
        addresses: [192.168.10.50/24]
        gateway4: 192.168.10.1
        nameservers:
          addresses: [8.8.8.8,1.1.1.1]
  ```
  Then:
  ```bash
  sudo netplan apply
  ip addr show ens33
  ```

- Verification
  - Ping gateway and DNS: `ping -c3 192.168.10.1` and `ping -c3 8.8.8.8`
  - `curl -I https://www.google.com` to verify DNS+internet

- Notes
  - Editing files by hand is OK, but NetworkManager may overwrite if it manages the interface; use nmcli or update-connections.

---

Task 2 — Firewalld zone management (publicweb)
- Objective
  - Create zone `publicweb`, assign interface, allow ports 80 & 443, and restrict other ports.

- Commands
  ```bash
  # Create the zone
  sudo firewall-cmd --permanent --new-zone=publicweb

  # Assign interface ens33 to this zone
  sudo firewall-cmd --permanent --zone=publicweb --change-interface=ens33

  # Allow ports 80 and 443 in that zone
  sudo firewall-cmd --permanent --zone=publicweb --add-service=http
  sudo firewall-cmd --permanent --zone=publicweb --add-service=https

  # Optionally set default target to drop incoming in this zone (reject other traffic)
  sudo firewall-cmd --permanent --zone=publicweb --set-target=DROP

  # Reload firewalld
  sudo firewall-cmd --reload

  # Show zone config
  sudo firewall-cmd --zone=publicweb --list-all
  ```

- Verification
  ```bash
  # From remote/test machine:
  curl -I http://<server-ip> --connect-timeout 5
  nmap -p1-1024 <server-ip>    # to see which ports are allowed/blocked
  ```

- Recovery
  ```bash
  sudo firewall-cmd --permanent --zone=publicweb --remove-interface=ens33
  sudo firewall-cmd --permanent --delete-zone=publicweb
  sudo firewall-cmd --reload
  ```

- Notes
  - Zone setting `DROP` causes silent drop. If you prefer explicit rejection, use `REJECT`.
  - Be careful assigning an interface to a zone — it affects all traffic on that interface.

---

Task 3 — TCP connection monitoring
- Objective
  - Show only ESTABLISHED connections, show programs using each port, show realtime updates and save report to `/root/conn.txt`.

- Commands
  ```bash
  # Show only ESTABLISHED (with program)
  sudo ss -tanp state established

  # Show programs using each port and write to /root/conn.txt (snapshot)
  sudo ss -tanp > /root/conn.txt

  # Real-time updates (every 2 seconds) and append to file (Ctrl-C to stop)
  sudo watch -n 2 'ss -tanp | head -n 200'   # interactive view

  # For a continuous logging to a file (add timestamp)
  while true; do echo "==== $(date) ====" >> /root/conn.txt; ss -tanp state established >> /root/conn.txt; sleep 5; done &
  ```

- Verification
  - Tail the report: `sudo tail -n 100 /root/conn.txt`

- Notes
  - `netstat` is deprecated; `ss` is preferred. Use `-p` to show PID/program.

---

Task 4 — DNS client troubleshooting
- Objective
  - Fix /etc/resolv.conf misconfiguration, persist changes and test resolution with dig/nslookup/ping.

- Diagnosing & Fix
  ```bash
  # Inspect current resolv.conf
  cat /etc/resolv.conf

  # If it's a symlink to systemd-resolved, update via systemd-resolved or NetworkManager:
  ls -l /etc/resolv.conf

  # Quick fix: update /etc/resolv.conf directly (temporary; will be overwritten by NM)
  sudo tee /etc/resolv.conf > /dev/null <<'EOF'
  nameserver 8.8.8.8
  nameserver 1.1.1.1
  search example.com
  EOF

  # Make persistent via NetworkManager (recommended)
  sudo nmcli con modify ens33 ipv4.dns "8.8.8.8 1.1.1.1"
  sudo nmcli con modify ens33 ipv4.ignore-auto-dns yes
  sudo nmcli con up ens33

  # If using systemd-resolved:
  sudo systemd-resolve --set-dns=8.8.8.8 --interface=ens33
  sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
  ```

- Verification
  ```bash
  dig +short google.com
  nslookup example.com
  ping -c3 google.com
  ```

- Notes
  - Always check whether `resolv.conf` is managed by NetworkManager or systemd-resolved and configure via the appropriate tool for persistence.

---

SET 8 — SECURITY & AUDITING

Task 1 — Configure password policy
- Objective
  - Set min length 10, at least 1 number, 1 special char, lock user after 3 failed attempts using /etc/login.defs and PAM.

- Steps (RHEL/CentOS example using pwquality and pam_faillock)
  ```bash
  # 1. Configure password complexity via /etc/security/pwquality.conf (or /etc/pam.d configuration)
  sudo cp /etc/security/pwquality.conf /etc/security/pwquality.conf.bak
  sudo sed -i '/^#/!b;s/.*/&/' /etc/security/pwquality.conf || true

  # Update or add:
  sudo bash -c 'cat >> /etc/security/pwquality.conf <<EOF
  minlen = 10
  dcredit = -1   # require at least one digit
  ocredit = -1   # require at least one special char
  EOF'

  # 2. Ensure PAM uses pam_pwquality (system-auth or password-auth)
  grep -E "pam_pwquality.so" /etc/pam.d/system-auth || sudo sed -i '/password\s+substack\s+pam_pwquality.so/! s/^/password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3\n/' /etc/pam.d/system-auth

  # 3. Configure pam_faillock for lockout after 3 attempts
  # Add lines to /etc/pam.d/system-auth and /etc/pam.d/password-auth (RHEL)
  sudo authselect select sssd --force    # if using authselect (RHEL8+), use correct method to manage PAM templates
  # Manual edits example (careful; backup file first)
  sudo cp /etc/pam.d/system-auth /etc/pam.d/system-auth.bak
  sudo sed -i '/pam_faillock.so/ d' /etc/pam.d/system-auth
  sudo sed -i '1i auth required pam_faillock.so preauth silent deny=3 unlock_time=900' /etc/pam.d/system-auth
  sudo sed -i '1i auth [default=die] pam_faillock.so authfail deny=3 unlock_time=900' /etc/pam.d/system-auth
  sudo sed -i '/pam_faillock.so/ d' /etc/pam.d/password-auth
  sudo sed -i '1i auth required pam_faillock.so preauth silent deny=3 unlock_time=900' /etc/pam.d/password-auth
  ```

- Verification
  - Test by failing login 3 times (on a test account) and see account locked.
  - Check `/var/log/secure` for faillock messages.
  - `pam_faillock --user testuser --reset` to reset counters.

- Notes / Pitfalls
  - Many distros use authselect/installation-specific PAM templates. Prefer using distro tools (authconfig/authselect) to edit PAM safely.
  - Always backup PAM files before modifying. A bad change can lock everyone out.

---

Task 2 — File integrity monitoring (AIDE)
- Objective
  - Install `aide`, generate baseline DB, modify a system file, run check to detect change, and restore file.

- Commands
  ```bash
  sudo dnf install -y aide || sudo apt install -y aide

  # Initialize AIDE DB (first-time)
  sudo aide --init
  # Move DB to proper location
  sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

  # Run check to produce report
  sudo aide --check | tee /var/log/aide-check.log

  # Modify a system file for test
  echo "TEST_CHANGE" | sudo tee -a /etc/motd

  # Run check again - it should detect change
  sudo aide --check | tee /var/log/aide-check2.log
  grep -i "changed" /var/log/aide-check2.log || true

  # To restore, replace from backup or revert manual change; AIDE itself does not auto-restore:
  sudo sed -i '$d' /etc/motd  # remove last line if that's the change
  ```

- Verification
  - AIDE reports changed files in its output.
  - Compare DBs and timestamps in `/var/log/aide-*`.

- Notes
  - AIDE is not an automatic restoration tool; use backups to restore files flagged by AIDE.
  - Regularly schedule `aide --check` via cron or systemd timer.

---

Task 3 — Auditd configuration (track modifications to /etc/passwd)
- Objective
  - Track modifications to /etc/passwd, generate events, and search who/when/how.

- Commands
  ```bash
  # Install auditd
  sudo dnf install -y audit || sudo apt install -y auditd

  # Add rule to watch /etc/passwd for write and attribute changes
  sudo auditctl -w /etc/passwd -p wa -k passwd_watch

  # Make persistent by adding to /etc/audit/rules.d/local.rules (RHEL) or /etc/audit/audit.rules
  echo "-w /etc/passwd -p wa -k passwd_watch" | sudo tee /etc/audit/rules.d/local.rules

  # Restart auditd
  sudo systemctl restart auditd

  # Generate an event: change /etc/passwd (do NOT alter root or break system; use safe test)
  sudo cp /etc/passwd /tmp/passwd.bak
  sudo sed -n '1,10p' /etc/passwd | sudo tee -a /etc/passwd

  # Wait a moment, then search audit log
  sudo ausearch -k passwd_watch --format text | less
  # Or a prettier audit report
  sudo aureport -f --input /var/log/audit/audit.log | grep passwd
  ```

- To answer the questions
  - who modified the file?
    ```bash
    sudo ausearch -k passwd_watch | aureport -f | head -n 50
    # Or search events for syscall=open or write and print uid
    sudo ausearch -k passwd_watch -i | awk '/uid/ || /auid/ {print}'
    ```
  - when?
    - Timestamps are in the ausearch output.
  - how?
    - The syscall and execve info in the audit record show the command used.

- Recovery
  - Restore `/etc/passwd` from `/tmp/passwd.bak` if you made an unwanted change:
  ```bash
  sudo cp /tmp/passwd.bak /etc/passwd
  ```

- Notes
  - Audit logs can be large; use `ausearch` and `aureport` tools to filter.
  - Don't leave a test modification in place; revert after testing.

---

Task 4 — Disable USB storage devices
- Objective
  - Prevent automatic mounting/usage of USB storage devices; persist across reboots; optionally allow only root.

- Approaches
  1. Blacklist usb-storage kernel module (prevents mass storage driver from loading)
     ```bash
     echo "install usb-storage /bin/true" | sudo tee /etc/modprobe.d/disable-usb-storage.conf
     sudo dracut -f  # if using initramfs (RHEL) to ensure change at next boot
     sudo modprobe -r usb-storage || true  # remove if currently loaded
     ```

  2. Create a udev rule to ignore block devices from USB
     ```bash
     sudo tee /etc/udev/rules.d/99-no-usb-storage.rules > /dev/null <<'EOF'
     # Ignore USB storage block devices
     ACTION=="add", SUBSYSTEM=="block", ENV{ID_BUS}=="usb", ENV{ID_TYPE}=="disk", OPTIONS+="ignore_device"
     EOF
     sudo udevadm control --reload
     ```

  3. Use mount policy or polkit rules to restrict automounting to root or admin group (distro-dependent).

- Verification
  - Insert a USB mass storage device — it should not create /dev/sdX or should not auto-mount.
  - `dmesg | tail` should show blocked module or no mount.

- Recovery
  - Remove the file /etc/modprobe.d/disable-usb-storage.conf and reboot; or remove udev rule and reload.

- Notes
  - Blacklisting the module blocks all USB mass storage but may not prevent other USB devices (keyboard/mouse).
  - Provide a waiver for maintenance by root: `modprobe -r` blocked, root could manually load module if needed; for stricter policies, enforce via security policies and restricted access.

---

BONUS — SHELL SCRIPTING LAB TASKS

Bonus 1 — Script to monitor disk usage
- Requirements
  - Warn if usage > 80% for any mount, echo or email, show mount point & size.

- Script
  ```bash
  sudo tee /usr/local/bin/check_disk_usage.sh > /dev/null <<'SH'
  #!/bin/bash
  THRESHOLD=80
  EMAIL="root@localhost"   # change to desired recipient
  df -H --output=pcent,target,size | tail -n +2 | while read -r line; do
    PERCENT=\$(echo \$line | awk '{print \$1}' | tr -d '%')
    MOUNT=\$(echo \$line | awk '{print \$2}')
    SIZE=\$(echo \$line | awk '{print \$3}')
    if [ "\$PERCENT" -ge "\$THRESHOLD" ]; then
      MSG="Warning: Disk usage on \$MOUNT is \$PERCENT% (size: \$SIZE)"
      echo "\$MSG"
      # Uncomment the following if mail is configured:
      # echo "\$MSG" | mail -s "Disk usage warning: \$MOUNT" \$EMAIL
    fi
  done
  SH

  sudo chmod +x /usr/local/bin/check_disk_usage.sh
  # Example run:
  /usr/local/bin/check_disk_usage.sh
  ```

- Cron example to run daily
  ```bash
  (sudo crontab -l 2>/dev/null; echo "0 6 * * * /usr/local/bin/check_disk_usage.sh") | sudo crontab -
  ```

- Notes
  - Use `df -h` or `df -H` to format; adjust threshold as needed.

Bonus 2 — Script to bulk create users from a file
- Script (takes filename as argument)
  ```bash
  sudo tee /usr/local/bin/bulk_create_users.sh > /dev/null <<'SH'
  #!/bin/bash
  FILE="\${1:-/root/userlist.txt}"
  LOG="/var/log/bulk_user_creation.log"
  touch "\$LOG"
  while IFS= read -r user; do
    [ -z "\$user" ] && continue
    if id "\$user" &>/dev/null; then
      echo "$(date) : user \$user already exists" | tee -a "\$LOG"
      continue
    fi
    sudo useradd -m "\$user"
    # Set a temporary random password and force change at first login
    PW=\$(openssl rand -base64 12)
    echo "\$user:\$PW" | sudo chpasswd
    sudo chage -d 0 "\$user"
    echo "$(date) : created \$user with temp password \$PW" | tee -a "\$LOG"
  done < "\$FILE"
  SH

  sudo chmod +x /usr/local/bin/bulk_create_users.sh

  # Usage:
  # Create /root/userlist.txt with names (one per line) then run:
  # sudo /usr/local/bin/bulk_create_users.sh /root/userlist.txt
  ```

- Notes
  - Logging includes temp password; secure the log or write password differently (e.g., print only once to console).
  - For production, consider provisioning via LDAP/AD or user management tooling.

Bonus 3 — Script to check service status list
- Script (reads services from /root/services.txt)
  ```bash
  sudo tee /usr/local/bin/check_services.sh > /dev/null <<'SH'
  #!/bin/bash
  INFILE="\${1:-/root/services.txt}"
  OUT="/tmp/service_report.txt"
  echo "Service report generated at: $(date)" > "\$OUT"
  while IFS= read -r svc; do
    [ -z "\$svc" ] && continue
    STATUS=\$(systemctl is-active "\$svc" 2>/dev/null || echo "unknown")
    echo "\$svc : \$STATUS" >> "\$OUT"
  done < "\$INFILE"
  echo "Report saved to \$OUT"
  cat "\$OUT"
  SH

  sudo chmod +x /usr/local/bin/check_services.sh

  # Example:
  # Create /root/services.txt:
  # sshd
  # nginx
  # run:
  # sudo /usr/local/bin/check_services.sh /root/services.txt
  ```

- Notes
  - `/tmp/service_report.txt` is overwritten each run. Change location if persistence needed.

---

FINAL NOTES & BEST PRACTICES

- Always test destructive operations (disk, LVM, filesystem) on disposable VMs or using loopback devices.
- When modifying PAM, sudoers or fstab — backup original files first.
- Use `visudo` and `sshd -t` / `nginx -t` to validate configuration before reloading services.
- For SELinux changes, prefer `semanage` + `restorecon` to create persistent file context mappings.
- Use distro-specific management tools (nmcli/netplan/authselect) rather than editing config files directly, when available, for persistent changes.
- Document every change (in change log) and apply during maintenance windows on production.
