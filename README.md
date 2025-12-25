# Linux Administration Lab — Complete Sets, Documentation & Styling Guide

A hands-on lab repository containing 4 complete lab sets (10 tasks each) with full step‑by‑step commands, verification, recovery steps, exam tips, and safety notes. This README explains structure, how to use the sets, documentation conventions, and practical guidance for making the README visually effective (format, headings, and font approaches you can apply on GitHub or GitHub Pages).

---

## Quick links
- Sets (detailed lab files)
  - [Set 1 — Basic Administration](set-1.md)
  - [Set 2 — Disks, ACL, Firewalld, SELinux](set-2.md)
  - [Set 3 — Services, Runlevels, Processes, Tar](set-3.md)
  - [Set 4 — Networking, Sticky bit, Chrony, Swap, Systemd](set-4.md)

---

## Purpose

These labs are designed for:
- RHCSA / RHCE preparation and practice
- System administrators who want repeatable practice scenarios
- Instructors creating classroom or online labs

Each task is presented with:
- Objective
- Prerequisites
- Exact commands to run
- Verification steps (what to check)
- Recovery / undo steps
- Estimated time and difficulty
- Exam tips and pitfalls

---

## How to use these labs (recommended workflow)

1. Clone the repo:
   ```bash
   git clone https://github.com/cdldevnew/linux-commands-2026.git
   cd linux-commands-2026
   ```
2. Open one set file (e.g., set-1.md). Read the Objective and Prerequisites before running anything.
3. Run commands in a disposable VM/container or use loopback devices when manipulating disks.
4. After each command, run the verification command(s) provided.
5. Use the recovery steps to revert changes before moving to the next task or before resetting your VM snapshot.

---

## Documentation & Format Conventions (how lab files are structured)

Each task uses this consistent template:

- Title (Task number and short goal)
- Objective — What you will accomplish and why it matters
- Prerequisites — Required privileges, devices, or files
- Commands — Ordered, copy‑paste friendly commands (with examples)
- Verification — Commands to confirm the task is complete
- Recovery / Undo — Commands to revert changes or clean up
- Estimated time and Difficulty (Easy / Medium / Hard)
- Exam Tip — quick note for test scenarios or common traps

This convention helps you move quickly, verify results, and undo changes safely.

---

## Styling and Font Guidance for README.md (what's practical on GitHub)

GitHub README.md uses GitHub-Flavored Markdown and does not support arbitrary fonts/styles via CSS. However, you can achieve an effective visual presentation as follows:

1. Use strong structural elements
   - Clear headings (H1/H2/H3)
   - Tables for quick reference
   - Code blocks for commands and outputs
   - Blockquotes for warnings and tips

2. Use badges and icons
   - Shields (https://shields.io) to show status, estimated time, or difficulty.
   - Example:
     ```markdown
     ![Estimated time](https://img.shields.io/badge/time-~15–30min-blue)
     ![Difficulty](https://img.shields.io/badge/difficulty-medium-yellow)
     ```

3. Use an SVG or image header to control typography
   - If you need a specific font/brand style, create a header image (SVG or PNG) that uses the desired font and include it at the top of the README:
     ```markdown
     ![Lab header](assets/lab-header.svg)
     ```
   - SVGs can embed fonts or use system fonts; include the generated SVG in `assets/` and reference it.

4. Use GitHub Pages for fully custom styling
   - If you need a real custom font and layout, publish docs via GitHub Pages and add a small CSS that imports Google Fonts.
   - Steps:
     - Create a docs/ folder or gh-pages branch
     - Add an HTML index that loads a font from Google Fonts and a CSS file
     - Push and enable Pages in repo settings

5. Use emoji & callouts sparingly
   - Use ✅, ⚠️, ❗ for emphasis inside headings or notes.
   - Keep README accessible and screen-reader friendly.

6. Example callout style using blockquote:
   > ⚠️ Warning: Always test disk and LVM commands on disposable VMs or on loopback devices to avoid data loss.

---

## Safety: Always practice on throwaway systems

- Use snapshots or throwaway VMs.
- Prefer loopback disk images (losetup + truncate) when you lack spare physical devices.
- Do not run destructive commands on production machines.

Example: create a safe loop device for disk/LVM testing:
```bash
truncate -s 3G /tmp/testdisk.img
losetup -fP /tmp/testdisk.img
# find /dev/loopX and use that instead of /dev/sdX in labs
```

---

## Contributing & Adding More Labs

To add tasks or new sets:
1. Fork the repo.
2. Add a new file set-5.md following the standard template used in existing set files.
3. Open a PR with a clear description and sample outputs.

Please include recovery steps and an estimated difficulty for each task you add.

---

## File layout in this repo

- README.md — this document
- set-1.md — Set 1 (Basic Admin)
- set-2.md — Set 2 (Disks & Security)
- set-3.md — Set 3 (Services & Processes)
- set-4.md — Set 4 (Networking & Systemd)
- assets/ — images and SVG headers (optional)
- scripts/ — helper scripts (optional)

---

## What's next

- Open the individual set files to begin working through tasks. Each set is self-contained and includes all commands and verification steps you need.

If you'd like, I can:
- Generate the SVG header with a suggested font and layout and add it to assets/
- Create a GitHub Pages skeleton to demonstrate custom font usage
- Export the lab tasks into a printable PDF

Tell me which of these you'd prefer next and I will add it to the repository.
