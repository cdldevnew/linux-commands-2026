# Set 2 — Disks, ACL, Firewalld, SELinux (10 Tasks)

Overview: Disk/partitioning, ACLs, SGID, permanent mounts, firewalld and SELinux topics. Safety is emphasized for partitioning tasks.

---

Task 1 — Create user `john` without home directory
- Objective: Create a system or test user without a home directory.
- Commands:
  ```bash
  sudo useradd -M john
  sudo passwd john
  ```
- Verification:
  ```bash
  getent passwd john
  ls -ld /home/john || echo "No home directory"
  ```
- Recovery:
  ```bash
  sudo userdel john
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 2 — Lock user account and verify
- Commands:
  ```bash
  sudo passwd -l john
  sudo passwd -S john
  ```
- Verify output shows "L" or "locked".
- Recovery:
  ```bash
  sudo passwd -u john
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 3 — `/shared` inherits group ownership (SGID)
- Objective: Create a directory where files inherit the group's owner.
- Commands:
  ```bash
  sudo mkdir -p /shared
  sudo chown root:developers /shared
  sudo chmod 2775 /shared
  ```
- Verification:
  ```bash
  ls -ld /shared
  sudo -u someuser bash -c "touch /shared/testfile; ls -l /shared/testfile"
  ```
- Recovery:
  ```bash
  sudo rm -rf /shared
  ```
- Time: 5 minutes | Difficulty: Easy

---

Task 4 — ACL for `anna` RW on `/shared/report.txt`
- Commands:
  ```bash
  sudo touch /shared/report.txt
  sudo setfacl -m u:anna:rw /shared/report.txt
  sudo getfacl /shared/report.txt
  ```
- Verification:
  ```bash
  getfacl /shared/report.txt
  sudo -u anna test -w /shared/report.txt && echo "write ok"
  ```
- Recovery:
  ```bash
  sudo setfacl -x u:anna /shared/report.txt
  sudo rm -f /shared/report.txt
  ```
- Time: 5 minutes | Difficulty: Medium
- Tip: ACLs will appear alongside standard unix perms; use `getfacl`.

---

Task 5 — List disks & partitions
- Commands:
  ```bash
  lsblk -f
  sudo fdisk -l
  ```
- Verification:
  ```bash
  lsblk
  ```
- Time: 2–3 minutes | Difficulty: Easy

---

Task 6 — Create primary partition on `/dev/sdc` (fdisk)
- Safety: Only run on test devices or loopback.
- Commands (fdisk interactive):
  ```bash
  sudo fdisk /dev/sdc
  # inside fdisk:
  # n → p → 1 → enter → enter → w
  ```
- Verification:
  ```bash
  lsblk /dev/sdc
  sudo parted -l
  ```
- Recovery:
  ```bash
  # If using loopback, delete file/device. If real device, restore from snapshot.
  ```
- Time: 10–20 minutes | Difficulty: Medium

---

Task 7 — Format `/dev/sdc1` with XFS
- Commands:
  ```bash
  sudo mkfs.xfs /dev/sdc1
  ```
- Verification:
  ```bash
  lsblk -f /dev/sdc1
  sudo blkid /dev/sdc1
  ```
- Recovery:
  ```bash
  # mkfs is destructive; restore from backup or use a new loop file.
  ```
- Time: 5 minutes | Difficulty: Medium

---

Task 8 — Permanent mount `/dev/sdc1` on `/mnt/data`
- Commands:
  ```bash
  sudo mkdir -p /mnt/data
  sudo blkid /dev/sdc1
  # add a line to /etc/fstab using UUID=...
  # Example fstab line:
  # UUID=<uuid> /mnt/data xfs defaults 0 0
  sudo mount -a
  ```
- Verification:
  ```bash
  mount | grep /mnt/data
  df -h /mnt/data
  ```
- Recovery:
  ```bash
  sudo sed -i '\|/mnt/data|d' /etc/fstab
  sudo umount /mnt/data
  ```
- Time: 5–10 minutes | Difficulty: Medium

---

Task 9 — Open port 443/tcp in firewalld
- Commands:
  ```bash
  sudo firewall-cmd --permanent --add-port=443/tcp
  sudo firewall-cmd --reload
  sudo firewall-cmd --list-ports
  ```
- Verification:
  ```bash
  sudo firewall-cmd --list-all
  ss -tuln | grep 443 || echo "port open in firewall but no service listening"
  ```
- Recovery:
  ```bash
  sudo firewall-cmd --permanent --remove-port=443/tcp
  sudo firewall-cmd --reload
  ```
- Time: 3 minutes | Difficulty: Easy

---

Task 10 — Check & set SELinux permissive temporarily
- Commands:
  ```bash
  getenforce
  sudo setenforce 0
  getenforce
  ```
- Verification:
  ```bash
  sestatus
  # revert:
  sudo setenforce 1
  ```
- Time: 2 minutes | Difficulty: Easy
- Caution: `setenforce 0` is temporary and will revert on reboot unless config files are changed.

---
