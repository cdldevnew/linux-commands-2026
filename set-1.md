# Set 1 — Basic Administration (10 Tasks)

Overview: This set focuses on core user/group management, file permissions, password aging, basic process discovery, systemd checks, cron, and LVM basics. Each task includes verification and recovery steps.

---

Task 1 — Create user `student1` with home directory and bash shell
- Objective: Add a normal user with a home directory and bash login shell.
- Prereqs: root or sudo
- Commands:
  ```bash
  sudo useradd -m -s /bin/bash student1
  sudo passwd student1
  ```
- Verification:
  ```bash
  id student1
  ls -ld /home/student1
  getent passwd student1
  ```
- Recovery:
  ```bash
  sudo userdel -r student1
  ```
- Time: 2–3 minutes | Difficulty: Easy
- Exam tip: Use `useradd -m` to ensure the home is created. `-s /bin/bash` sets the shell.

---

Task 2 — Create group `adminteam` & add `student1`
- Objective: Create a group and add an existing user as a supplementary member.
- Commands:
  ```bash
  sudo groupadd adminteam
  sudo usermod -aG adminteam student1
  ```
- Verification:
  ```bash
  id student1
  getent group adminteam
  ```
- Recovery:
  ```bash
  sudo gpasswd -d student1 adminteam
  sudo groupdel adminteam
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 3 — Create `/project/data` with owner-only full permissions
- Objective: Directory that only the owner can access.
- Commands:
  ```bash
  sudo mkdir -p /project/data
  sudo chown root:root /project/data
  sudo chmod 700 /project/data
  ```
- Verification:
  ```bash
  ls -ld /project/data
  sudo -u student1 ls /project/data || echo "permission denied"
  ```
- Recovery:
  ```bash
  sudo rm -rf /project/data
  ```
- Time: 3 minutes | Difficulty: Easy
- Note: Replace owner as needed.

---

Task 4 — Set password expiry 30 days for `student1`
- Objective: Configure maximum password validity period.
- Commands:
  ```bash
  sudo chage -M 30 student1
  ```
- Verification:
  ```bash
  sudo chage -l student1
  ```
- Recovery:
  ```bash
  sudo chage -M 99999 student1
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 5 — Create `/tmp/testfile` with read-only for others
- Objective: Demonstrate file permissions for "others".
- Commands:
  ```bash
  sudo touch /tmp/testfile
  sudo chmod o=r /tmp/testfile
  ls -l /tmp/testfile
  ```
- Verification:
  ```bash
  ls -l /tmp/testfile
  # as another unprivileged user:
  sudo -u nobody cat /tmp/testfile || echo "no read content"
  ```
- Recovery:
  ```bash
  sudo rm -f /tmp/testfile
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 6 — Find sshd PID (two methods)
- Objective: Learn two ways to get daemon PIDs.
- Commands:
  ```bash
  pidof sshd
  ps -C sshd -o pid,cmd
  ```
- Verification:
  ```bash
  ps -ef | grep -E 'sshd' | grep -v grep
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 7 — Show running & failed services
- Objective: See currently running and failed systemd services.
- Commands:
  ```bash
  systemctl list-units --type=service --state=running
  systemctl --failed
  ```
- Verification:
  ```bash
  systemctl status sshd
  ```
- Time: 3 minutes | Difficulty: Easy

---

Task 8 — Cron job to run `/root/backup.sh` at 3:30 AM
- Objective: Create a user root cron entry that runs daily.
- Commands:
  ```bash
  sudo crontab -e
  # add the line:
  # 30 3 * * * /root/backup.sh
  ```
- Verification:
  ```bash
  sudo crontab -l | grep '/root/backup.sh'
  # validate cron ran by checking logs (varies by distro)
  sudo grep CRON /var/log/* | tail -n 20
  ```
- Recovery:
  ```bash
  sudo crontab -l | sed '/\/root\/backup.sh/d' | sudo crontab -
  ```
- Time: 5 minutes | Difficulty: Easy
- Tip: For deterministic testing, run the script manually and inspect exit/status.

---

Task 9 — Create 2GB LV `lvbackup` in VG `vgclass`
- Objective: Create PV, VG and LV, format and mount.
- Prereqs: Spare block device (e.g., /dev/sdb). Prefer loopback device for safety.
- Commands (loopback example):
  ```bash
  # safe preparation (if no spare device)
  sudo truncate -s 3G /tmp/lvm-disk.img
  sudo losetup -fP /tmp/lvm-disk.img
  LOOP=$(losetup -j /tmp/lvm-disk.img | cut -d: -f1)
  sudo pvcreate $LOOP
  sudo vgcreate vgclass $LOOP
  sudo lvcreate -L 2G -n lvbackup vgclass
  sudo mkfs.ext4 /dev/vgclass/lvbackup
  sudo mkdir -p /backup
  sudo mount /dev/vgclass/lvbackup /backup
  ```
- Verification:
  ```bash
  lsblk
  sudo lvs
  df -h /backup
  ```
- Recovery:
  ```bash
  sudo umount /backup
  sudo lvremove -f /dev/vgclass/lvbackup
  sudo vgremove -f vgclass
  sudo pvremove -f $LOOP
  sudo losetup -d $LOOP
  rm -f /tmp/lvm-disk.img
  ```
- Time: 15–30 minutes | Difficulty: Medium
- Exam tip: Use `losetup` or temporary files on exams with no spare disks.

---

Task 10 — Find package owning `/usr/bin/ssh`
- Objective: Determine which package provides a file.
- Commands:
  ```bash
  # RPM-based systems (RHEL / CentOS / Rocky / Alma)
  rpm -qf /usr/bin/ssh
  # or use dnf
  dnf provides /usr/bin/ssh
  ```
- Verification:
  ```bash
  rpm -qf /usr/bin/ssh
  ```
- Time: 2 minutes | Difficulty: Easy

---
