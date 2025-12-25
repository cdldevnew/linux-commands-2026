# Set 4 — Networking, Sticky bit, Chrony, Swap, Systemd (10 Tasks)

Overview: Networking commands, sticky bit usage, chrony NTP client, swap creation, locating large files, port listening, and creating systemd units.

---

Task 1 — Create group `developers` and make primary group for `raj`
- Commands:
  ```bash
  sudo groupadd developers
  sudo useradd -g developers -m raj
  sudo passwd raj
  ```
- Verification:
  ```bash
  id raj
  getent group developers
  ```
- Recovery:
  ```bash
  sudo userdel -r raj
  sudo groupdel developers
  ```
- Time: 5 minutes | Difficulty: Easy

---

Task 2 — Password aging for raj
- Commands:
  ```bash
  sudo chage -M 60 -m 3 -W 7 raj
  sudo chage -l raj
  ```
- Verification: `chage -l raj`
- Recovery:
  ```bash
  sudo chage -M 99999 raj
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 3 — Sticky bit on `/audit`
- Commands:
  ```bash
  sudo mkdir -p /audit
  sudo chmod 1777 /audit
  ls -ld /audit
  ```
- Verification:
  ```bash
  touch /audit/file1
  sudo chown userA /audit/file1
  # other users cannot delete files they don't own
  ```
- Recovery:
  ```bash
  sudo rm -rf /audit
  ```
- Time: 2 minutes | Difficulty: Easy
- Tip: `/tmp` uses sticky bit so only owners can remove their files.

---

Task 4 — Show IP, MAC, and gateway
- Commands:
  ```bash
  ip addr show
  ip link show
  ip route show
  ```
- Verification: inspect outputs for addresses and default route
- Time: 2–3 minutes | Difficulty: Easy

---

Task 5 — Add temporary route
- Commands:
  ```bash
  sudo ip route add 192.168.50.0/24 via 192.168.1.1
  ip route show
  ```
- Recovery:
  ```bash
  sudo ip route del 192.168.50.0/24
  ```
- Time: 2 minutes | Difficulty: Easy

---

Task 6 — Configure chrony NTP
- Commands:
  ```bash
  sudo dnf install -y chrony  # apt: sudo apt install -y chrony
  sudo systemctl enable --now chronyd
  chronyc sources -v
  ```
- Verification:
  ```bash
  chronyc tracking
  timedatectl status
  ```
- Recovery:
  ```bash
  sudo systemctl disable --now chronyd
  sudo dnf remove -y chrony
  ```
- Time: 10 minutes | Difficulty: Medium

---

Task 7 — Create 1GB swap partition (safe loopback)
- Commands (loopback):
  ```bash
  sudo truncate -s 1G /tmp/swapfile.img
  sudo losetup -fP /tmp/swapfile.img
  LOOP=$(losetup -j /tmp/swapfile.img | cut -d: -f1)
  sudo mkswap $LOOP
  sudo swapon $LOOP
  sudo swapon --show
  # add to /etc/fstab for permanence:
  # /tmp/swapfile.img none swap defaults 0 0
  ```
- Verification:
  ```bash
  swapon --show
  free -h
  ```
- Recovery:
  ```bash
  sudo swapoff $LOOP
  sudo losetup -d $LOOP
  rm -f /tmp/swapfile.img
  ```
- Time: 10 minutes | Difficulty: Medium
- Caution: use swap files or loopback to avoid partitioning disks.

---

Task 8 — Find files > 500MB in `/var`
- Commands:
  ```bash
  sudo find /var -type f -size +500M -exec ls -lh {} \;
  ```
- Verification: Inspect output and clean safe files if required.
- Recovery: N/A (read-only search)
- Time: 5 minutes | Difficulty: Easy

---

Task 9 — Show listening ports
- Commands:
  ```bash
  ss -tulnp
  sudo ss -tulnp | head -n 40
  ```
- Verification: Look for expected services (e.g., sshd on 22)
- Time: 2 minutes | Difficulty: Easy

---

Task 10 — Create systemd service `customapp.service`
- Objective: Create and enable a simple always‑restarting service wrapper.
- Commands:
  ```bash
  # create start script
  sudo mkdir -p /opt/customapp
  sudo tee /opt/customapp/start.sh > /dev/null <<'EOF'
  #!/bin/bash
  while true; do
    echo "customapp running `date`" >> /var/log/customapp.log
    sleep 60
  done
  EOF
  sudo chmod +x /opt/customapp/start.sh

  # create unit file
  sudo tee /etc/systemd/system/customapp.service > /dev/null <<'EOF'
  [Unit]
  Description=Custom App Demo
  After=network.target

  [Service]
  Type=simple
  ExecStart=/opt/customapp/start.sh
  Restart=always
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  EOF

  sudo systemctl daemon-reload
  sudo systemctl enable --now customapp.service
  sudo systemctl status customapp.service --no-pager
  ```
- Verification:
  ```bash
  sudo systemctl is-active customapp.service
  sudo journalctl -u customapp.service -n 50
  tail -n 5 /var/log/customapp.log
  ```
- Recovery:
  ```bash
  sudo systemctl disable --now customapp.service
  sudo rm -f /etc/systemd/system/customapp.service
  sudo rm -f /opt/customapp/start.sh
  sudo rm -f /var/log/customapp.log
  sudo systemctl daemon-reload
  ```
- Time: 10–15 minutes | Difficulty: Medium
- Exam tip: Always include `Restart=always` and a simple script; test `systemctl daemon-reload`.

---
