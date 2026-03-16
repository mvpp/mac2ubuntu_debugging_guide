# Debugging Ubuntu on a Mac — a practical, field-tested guide

Based on a real troubleshooting session with a 27" Late-2015 iMac, here’s a *thorough*, step-by-step tutorial you can follow. It covers how to narrow down and fix freezes on Ubuntu (desktop or server installs) running on Apple hardware, with useful commands, notes where clarification was needed during the session, and concrete fixes that worked.

Entities referenced (one each): Ubuntu, iMac (27-inch, Late 2015)

---

# Overview / TL;DR

Most total freezes on Ubuntu running on older Macs fall into a few buckets:

* failing or slow disk I/O (HDD / Fusion Drive),
* memory errors,
* GPU / kernel stalls (very common with older AMD GPUs),
* or kernel / firmware interaction (ACPI / PCIe / DPM).

A reliable diagnosis workflow:

1. Confirm disk and RAM health (smartctl, memtest)
2. Rule out I/O stalls (lsblk, iotop, `D`-state processes)
3. Capture kernel logs immediately after a freeze (journalctl, dmesg)
4. Test Wayland vs Xorg (`$XDG_SESSION_TYPE`)
5. Try kernel & driver mitigations (kernel cmdline: radeon/amdgpu options; disable ASPM / DPM)
6. Use Live USB to separate hardware vs installed-system faults
7. If hardware is suspected, replace failing parts (SSD, RAM) and prefer Xorg + LTS kernel for long-term stability

---

# Preparation & prerequisites

* Back up all important data (you already did — good).
* Have a second machine available (for downloads and writing Live USB images).

---

# 0 — Quick checklist (run this first)

```text
1. Can you switch to a TTY when it freezes? → Ctrl+Alt+F3
   - Yes → GUI problem / WindowServer / X / Wayland
   - No  → kernel or GPU hard-lock / hardware
2. Check the session type:
   echo $XDG_SESSION_TYPE
   - wayland or x11 (xorg)
3. Check memory:
   free -h
4. Check storage type:
   lsblk -d -o NAME,ROTA
   - ROTA=1 → spinning disk (HDD)
   - ROTA=0 → SSD
```

---

# 1 — Confirm disk & RAM health (non-destructive)

## Disk: SMART

Install and run smartctl (smartmontools):

```bash
sudo apt update
sudo apt install smartmontools -y
lsblk                 # identify drive device (e.g. /dev/sda)
sudo smartctl -a /dev/sda
```

Look for:

* `Reallocated_Sector_Ct`, `Current_Pending_Sector`, `Offline_Uncorrectable`, and “SMART overall-health self-assessment”.
  If any of those are non-zero/FAILED → replace disk (SSD recommended).

## RAM: memtest86 (UEFI friendly)

Two options: memtest86+ in GRUB (may be legacy) or boot official MemTest86 UEFI image. On UEFI systems you will usually see `Memory test (memtest86+x64.efi)` in the GRUB menu — choose that (not the serial console).

To install memtest86+ (if missing):

```bash
sudo apt install memtest86+
sudo update-grub
# Reboot and pick Memory test in GRUB menu
```

Recommended: run **2 full passes**. Any error → replace RAM.

**Clarification from the session:** if GRUB drops to a `grub>` prompt (CLI), you can exit with `exit` and boot again, or show the GRUB menu permanently by editing `/etc/default/grub` and setting `GRUB_TIMEOUT_STYLE=menu` then `sudo update-grub`.

---

# 2 — Rule out I/O stalls & check overall resource pressure

Check memory and swap:

```bash
free -h
swapon --show
```

If swap is tiny, create or increase it (example: 8 GiB swapfile):

```bash
sudo swapoff -a
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Reduce swapping aggressiveness:

```bash
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Check disk type and I/O:

```bash
lsblk -d -o NAME,ROTA
sudo apt install iotop -y
sudo iotop -ao
# Use iotop while reproducing the freeze to see heavy disk I/O
```

If you observe processes in `D` state (uninterruptible sleep) in `htop` or `ps aux`, that typically indicates I/O blocking and possible disk/hardware issues.

---

# 3 — Capture kernel logs (do this immediately after a freeze and reboot)

Collect previous-boot kernel logs and search for key terms:

```bash
# Kernel messages from previous boot (last boot before current)
journalctl -b -1 -k --no-pager | tail -n 200

# Grep for obvious causes
journalctl -b -1 --no-pager | egrep -i 'amdgpu|radeon|gpu|lockup|hang|watchdog|panic|error|fail|oom' -n
```

Watch kernel logs live while reproducing the freeze (open terminal before triggering):

```bash
sudo journalctl -f -k
```

If the system freezes and logs show `radeon ... ring X stalled` or `GPU lockup` lines, you have a GPU driver / firmware issue. If logs show `I/O` or `blk` timeouts and the system stalls, suspect disk.

**Useful commands to collect extra state:**

```bash
uname -r                # kernel version
cat /proc/cmdline       # current kernel cmdline (useful to confirm active mitigations)
ls -1 /boot/vmlinuz-*   # installed kernels
grep "^menuentry" -n /boot/grub/grub.cfg | sed -n '1,200p'  # GRUB menu entries
```

**Clarification from the session:** in macOS we used `log show --last boot | grep -i panic` and ` /Library/Logs/DiagnosticReports` to locate panic reports; on Ubuntu use `journalctl` and `dmesg` for kernel-level diagnostics.

---

# 4 — Split test: Wayland vs Xorg

Old AMD GPUs + Wayland have caused freezes in multiple cases. Check current session type:

```bash
echo $XDG_SESSION_TYPE
```

If it prints `wayland`, switch to Xorg:

* Log out
* At the login screen, click the gear icon and choose **Ubuntu on Xorg** (or “GNOME on Xorg” / “Ubuntu on Xorg”)
* Log in and test for several hours

If switching to Xorg eliminates freezes → stick with Xorg.

**Clarification from the session:** user experienced entire-system freezes with Wayland; switching to Xorg is a simple, high-value test.

---

# 5 — GPU: common driver fixes and the exact cmdline used in the session

If your logs show `radeon` messages (GPU lockup examples):

* `radeon 0000:01:00.0: ring 5 stalled ... GPU lockup`
* `ci_start_dpm failed` / `radeon: dpm resume failed`
* or `displayport link status failed`

Then the fix that worked in the session was to **disable Radeon DPM and runtime PM** and disable PCIe ASPM. Edit GRUB:

```bash
sudo nano /etc/default/grub
```

Find `GRUB_CMDLINE_LINUX_DEFAULT=...` and set it like:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash radeon.dpm=0 radeon.runpm=0 pcie_aspm=off"
```

Then:

```bash
sudo update-grub
sudo reboot
```

Check it took effect:

```bash
cat /proc/cmdline
# Should include radeon.dpm=0 radeon.runpm=0 pcie_aspm=off
```

If your system uses `amdgpu` (some chips) instead of `radeon`, use:

```text
amdgpu.runpm=0 pcie_aspm=off
```

(You can test both if you’re not sure which driver is active; logs will reveal `radeon` vs `amdgpu`.)

**Why these options?**

* `radeon.dpm=0` disables dynamic power management (DPM) which often deadlocks old cards under modern kernels.
* `radeon.runpm=0` stops runtime power management (sleeps) that may hang the GPU.
* `pcie_aspm=off` disables PCIe ASPM which Apple firmwares sometimes implement poorly and can cause lockups.

If disabling DPM resolves freezes, keep it. You lose some power efficiency but gain stability.

---

# 6 — Kernel selection: try an older LTS kernel

Very new kernels can regress older GPU drivers. List available kernels:

```bash
ls -1 /boot/vmlinuz-* | sed 's/\/boot\/vmlinuz-//'
```

If you have older kernels installed (e.g., 6.1 or 6.5), boot them via GRUB → **Advanced options** and test for stability for a day. If you don’t, install the distro LTS meta-package to ensure supported kernels:

```bash
sudo apt update
sudo apt install --install-recommends linux-generic linux-headers-generic
sudo update-grub
```

Then boot the recommended Ubuntu kernel (generally more stable for LTS use).

---

# 7 — Stress tests & reproduction (safe diagnostics)

Use GL and CPU/VM stressors to try to reproduce the freeze in a controlled way (run in Xorg for best reproduction of driver issues):

```bash
sudo apt install mesa-utils stress-ng -y
glxinfo | grep -i "OpenGL"
glxgears               # light GPU test
# CPU + VM stress for 5 minutes:
stress-ng --cpu 4 --vm 1 --vm-bytes 70% --timeout 300s
```

While running, also tail the kernel log:

```bash
sudo journalctl -f -k
```

If a freeze happens, note the last kernel messages — they are diagnostic gold.

---

# 8 — Live USB test (definitive hardware vs software)

Create a fresh Ubuntu 24.04 Live USB (or another small distro live image). Boot the Mac from USB (hold Option ⌥ at power) and run the live session for several hours doing the same actions that previously caused freezes.

* Freeze on Live USB = likely hardware or very low-level firmware bug (GPU or motherboard)
* No freeze on Live USB = likely a configuration / kernel / installed-driver problem in your installed system (you can reinstall or change kernel/driver settings)

---

# 9 — Kernel logs & panic traces (for deep inspection)

If the system previously had kernel panics (macOS) or watchdogs, check `journalctl` and `/var/log`:

```bash
# previous boot errors
journalctl -b -1 -p 3 --no-pager
# general previous boot logs filtered for GPU
journalctl -b -1 | egrep -i 'radeon|amdgpu|gpu|ring|lockup|dpm|atombios' -n
```

On macOS (before switching to Ubuntu) we used `log show --last boot | grep -i panic` and checked panic files in `/Library/Logs/DiagnosticReports/`. For Ubuntu, rely on `journalctl` and `dmesg`.

---

# 10 — Tuning for an AI agent host (Openclaw / 24/7 use)

Since you plan to host Openclaw and use cloud LLM APIs:

* Use **Xorg** for stability if GPUs are older.
* Disable system sleep & auto-suspend:

```bash
# disable suspend/hibernate targets
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

* Ensure SSH enabled for remote management:

```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```

* Run your agent in Docker and enable auto-restart with systemd or Docker restart policies:

```bash
# install Docker
sudo apt install docker.io -y
sudo usermod -aG docker $USER
# test
docker run --rm hello-world
```

* Use ufw for basic firewall:

```bash
sudo ufw allow ssh
sudo ufw enable
```

---

# 11 — Useful commands cheat sheet (copy-paste friendly)

```bash
# hardware info
lsblk -d -o NAME,ROTA
free -h
uname -r
cat /proc/cmdline

# disk health
sudo apt install smartmontools -y
sudo smartctl -a /dev/sda

# memory test (install if needed)
sudo apt install memtest86+ -y
sudo update-grub

# logs (previous boot)
journalctl -b -1 -k --no-pager | tail -n 200
journalctl -b -1 --no-pager | egrep -i 'amdgpu|radeon|gpu|lockup|hang|watchdog|panic|error|fail' -n

# live kernel follow
sudo journalctl -f -k

# install monitoring / stress
sudo apt install htop iotop stress-ng mesa-utils -y
htop
sudo iotop -ao
glxinfo | grep -i "OpenGL"
glxgears
stress-ng --cpu 4 --vm 1 --vm-bytes 70% --timeout 300s

# swap increase
sudo swapoff -a
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# grub editing and update
sudo nano /etc/default/grub
# edit GRUB_CMDLINE_LINUX_DEFAULT line
sudo update-grub
sudo reboot
```

---

# 12 — Where clarifications were needed (and what to document)

Document these common confusions so others don’t get stuck:

* GRUB CLI (`grub>`): you can type `exit` or `reboot`. To permanently show the GRUB menu on boot, edit `/etc/default/grub` and set `GRUB_TIMEOUT_STYLE=menu`, then `sudo update-grub`. (We ran into this while trying to run memtest.)
* Which memtest to pick on a Mac: pick `Memory test (memtest86+x64.efi)` — **not** the serial console variant — because Macs use UEFI.
* Wayland vs Xorg: Wayland can hang older AMD GPUs on Linux; switching at login (gear icon) to Xorg often resolves freezing symptoms immediately.
* Kernel cmdline flags differ by driver:

  * For older Radeon “radeon” driver: `radeon.dpm=0 radeon.runpm=0 pcie_aspm=off`
  * For newer AMD drivers (`amdgpu`): `amdgpu.runpm=0 pcie_aspm=off`

---

# 13 — Final recommendations & long-term fixes

* If you want rock-solid stability for a 24/7 agent node:

  * Replace mechanical HDD with a SATA or NVMe SSD (big stability & performance win).
  * Upgrade RAM to 16GB if you run many browser tabs or local services.
  * Stay on Xorg + LTS kernel (Ubuntu 24.04 with distro kernel).
  * Keep a tested Live USB for quick verification when problems reappear.

* If GPU compute becomes desired later (local LLM inference), the 2015 iMac’s AMD GPU is not suitable for modern ROCm-based local inference — you’ll need external GPU or newer hardware.

---

# Example real-world summary (from the session — paste to illustrate)

* Problem: intermittent full-system freezes after installing Ubuntu on iMac17,1.
* Diagnostics performed: SMART check (disk healthy), memtest (2 passes, 0 errors), lsblk (HDD present), journalctl showed `radeon ... GPU lockup` lines.
* Fix that worked: switch to Xorg + add `radeon.dpm=0 radeon.runpm=0 pcie_aspm=off` to GRUB cmdline and reboot. Result: no more freezes.

---

* Produce a small script to toggle between Wayland/Xorg for fast testing,
* Or create a short troubleshooting flowchart image (markdown + ASCII).
