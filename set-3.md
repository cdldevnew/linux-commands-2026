# Set 3 — Services, Runlevels, Processes, Tar (10 Tasks)

Overview: Services and systemctl, runlevels (targets), processes, managing process lifecycles, and tar backup/restore.

---

Task 1 — Create `userA` & `userB` sharing same group
- Commands:
  ```bash
  sudo useradd -m userA
  sudo useradd -m userB
  sudo groupadd sharedgrp
  sudo usermod -g sharedgrp userA
  sudo usermod -g sharedgrp userB
  ```
- Verification:
  ```bash
  id userA
  id userB
  getent group sharedgrp
  ```
- Recovery:
  ```bash
  sudo userdel -r userA userB
  sudo groupdel sharedgrp
  ```
- Time: 5 minutes | Difficulty: Easy

---

Task 2 — Create `/secure` accessible only to root
- Commands:
  ```bash
  sudo mkdir -p /secure
  sudo chown root:root /secure
  sudo chmod 700 /secure
  ```
- Verification:
  ```bash
  ls -ld /secure
  sudo -u userA ls /secure || echo "permission denied"
  ```
- Recovery:
  ```bash
  sudo rm -rf /secure
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 3 — Set default runlevel to multi-user
- Commands:
  ```bash
  sudo systemctl set-default multi-user.target
  sudo systemctl get-default
  ```
- Verification:
  ```bash
  systemctl get-default
  ```
- Recovery:
  ```bash
  sudo systemctl set-default graphical.target
  ```
- Time: 2 minutes | Difficulty: Easy
- Tip: Changing default target does not change current target; use `systemctl isolate` only if needed.

---

Task 4 — List enabled boot services
- Commands:
  ```bash
  systemctl list-unit-files --type=service --state=enabled
  ```
- Verification: visually inspect the output for expected services
- Time: 2 minutes | Difficulty: Easy

---

Task 5 — Change hostname
- Commands:
  ```bash
  sudo hostnamectl set-hostname server1.example.com
  hostnamectl status
  ```
- Verification:
  ```bash
  hostnamectl
  hostname -f
  ```
- Recovery:
  ```bash
  sudo hostnamectl set-hostname previous-hostname
  ```
- Time: 3 minutes | Difficulty: Easy

---

Task 6 — Set timezone Asia/Kolkata
- Commands:
  ```bash
  sudo timedatectl set-timezone Asia/Kolkata
  timedatectl status
  ```
- Verification:
  ```bash
  timedatectl | grep "Time zone"
  ```
- Recovery:
  ```bash
  sudo timedatectl set-timezone UTC
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 7 — Top 5 memory processes
- Commands:
  ```bash
  ps aux --sort=-%mem | head -n 6
  ```
- Verification: See the table printed by the command
- Time: 2 minutes | Difficulty: Easy

---

Task 8 — Kill process by name
- Commands:
  ```bash
  pkill httpd
  # or more careful:
  pgrep httpd
  sudo kill -TERM <pid>
  ```
- Verification:
  ```bash
  ps -C httpd
  ss -tulnp | grep httpd
  ```
- Recovery:
  ```bash
  sudo systemctl start httpd
  ```
- Time: 3–5 minutes | Difficulty: Medium
- Tip: `pkill -f` matches full command lines; be cautious.

---

Task 9 — Tar backup of `/etc`
- Commands:
  ```bash
  sudo mkdir -p /backup
  sudo tar -czvf /backup/etc.tar.gz /etc
  # verify file exists
  ls -lh /backup/etc.tar.gz
  ```
- Verification:
  ```bash
  tar -tzf /backup/etc.tar.gz | head -n 20
  ```
- Recovery:
  ```bash
  sudo rm -f /backup/etc.tar.gz
  ```
- Time: 5–10 minutes | Difficulty: Easy

---

Task 10 — Extract archive to `/restore`
- Commands:
  ```bash
  sudo mkdir -p /restore
  sudo tar -xzvf /backup/etc.tar.gz -C /restore
  ls -l /restore/etc | head -n 20
  ```
- Verification:
  ```bash
  ls -l /restore/etc
  ```
- Recovery:
  ```bash
  sudo rm -rf /restore
  ```
- Time: 5 minutes | Difficulty: Easy

---
