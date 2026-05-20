# iMac 2015 (Bonaire) Linux GPU Black-Screen Diagnosis Guide

**Hardware:** iMac17,1 (27" Retina 2015) — AMD Radeon R9 M380 Mac Edition
(actually a rebadged FirePro M6100 / Bonaire / GCN 1.1 "Sea Islands" / **CIK**),
PCI ID `1002:6640`.

**OS:** Ubuntu 24.04 LTS, kernel 6.17.0-29-generic.

**Symptom investigated:** browsing GPU-accelerated web pages (heavy WebGL,
animated 3D backgrounds) → instant black screen → forced hard reboot.

This document captures the full diagnostic path, dead-ends, and the
final layered fix. Useful for anyone with the same iMac generation, and
for future-me if symptoms recur after a kernel/Mesa/Chrome upgrade.

---

## TL;DR — final working configuration

Three layered changes, in order of importance:

1. **Kernel cmdline** (`/etc/default/grub`):
   `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`
   — *no* `radeon.dpm=0`, *no* `radeon.runpm=0`, *no* `pcie_aspm=off`.
2. **Kernel driver:** legacy `radeon` (amdgpu does not work on this Apple-firmware Bonaire — see Dead Ends below).
3. **Chrome launch flags:** `--disable-webgl --disable-webgl2`
   — kills only WebGL; compositing, video decode, raster stay on GPU.

Firefox is the fallback for the rare site that genuinely needs WebGL —
it uses a different rendering path that does not trigger the bug.

---

## Layered root cause

The original failure mode was actually **two unrelated GPU bugs stacked**:

### Bug 1 — Thermal/clock stress from disabled DPM

The pre-existing `/etc/default/grub` had:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash radeon.dpm=0 radeon.runpm=0 pcie_aspm=off"
```

`radeon.dpm=0` disables dynamic power management, pinning the card at a fixed
(usually high) clock state. Under sustained GPU load this caused thermal
stress and ring-3 lockups:

```
radeon 0000:01:00.0: ring 3 stalled for more than 10081msec
radeon 0000:01:00.0: GPU lockup (current fence id ... on ring 3)
```

**Fix:** remove all radeon-prefixed flags. With DPM working, GPU now idles
at low clocks (~42°C at rest, well under thermal limits even under WebGL stress).

### Bug 2 — Chrome ANGLE+radeonsi shader bug on heavy WebGL

After Bug 1 was fixed, the aquarium WebGL demo (1000 fish) and 4K YouTube
ran cleanly for 15+ minutes. But https://openai.com/codex/ still triggered
instant black screen.

Same page in **Firefox**: renders fine, no crash. Difference is rendering path:

- Chrome on Linux: WebGL → **ANGLE** → desktop GL → Mesa `radeonsi` (ACO) → `radeon` kernel
- Firefox on Linux: WebGL → native GL → Mesa `radeonsi` → `radeon` kernel

ANGLE re-translates WebGL into a desktop GL shader stream that the Mesa+Bonaire
combination cannot compile/run safely for certain shader patterns
(post-processing, displacement maps, etc).

**Fix:** disable WebGL only in Chrome via launch flag.
Compositing, video, raster stay GPU-accelerated.

---

## Diagnostic methodology that worked

1. **Identify hardware + driver in use**

   ```bash
   lspci -nnk | grep -A3 VGA
   # look for "Kernel driver in use:" — radeon, amdgpu, or empty (=simpledrm/EFI fb)
   ```

2. **Inspect kernel logs from previous boot for GPU events**

   ```bash
   journalctl -k -b -1 --no-pager | grep -iE 'radeon|amdgpu|drm|gpu lockup|ring.*stall|failed to schedule|fence'
   ```

   Key signatures to look for:
   - `ring N stalled for more than ... msec` → GPU lockup on engine N
   - `GPU lockup (current fence id ...)` → confirmed lockup
   - `Failed to schedule IB` → command buffer submission failed
   - `[drm:radeon_gem_va_update_vm] BO_VA (-4)` → -EINTR, usually benign noise
   - `[drm:ci_dpm_set_power_state] ci_upload_dpm_level_enable_mask failed`
     → DPM state transition failed (logged but non-fatal on CIK Bonaire)
   - `[drm:uvd_v1_0_start] UVD not responding` → video decode engine init failed

3. **Check current power state under load**

   ```bash
   cat /sys/class/drm/card*/device/power_dpm_state              # balanced / battery / performance
   cat /sys/class/drm/card*/device/power_dpm_force_performance_level   # auto / low / high / manual
   cat /sys/class/drm/card*/device/hwmon/hwmon*/temp1_input     # millidegrees C
   ```

4. **Inspect Chrome's GPU pipeline**

   - Open `chrome://gpu`
   - Look at:
     - `GL_RENDERER` → reveals whether ANGLE is in the path, and which Mesa driver
     - `Vulkan` status → Disabled = no RADV (radeon kernel driver lacks Vulkan)
     - `WebGL`, `WebGL2`, `WebGPU` status → which are exposed to pages
     - "Problems Detected" section → driver-specific workarounds Chrome auto-applies
   - Verify WebGPU actually exposed to pages: visit https://webgpureport.org/

5. **A/B test rendering path with Firefox**

   - Open the offending URL in Firefox. Different GPU process model, no ANGLE.
   - If Firefox handles it cleanly → Chrome/ANGLE is the culprit; small targeted fix possible.
   - If Firefox also crashes → driver-level bug under heavy WebGL; need bigger hammer.

6. **Set up crash-survivable logging before deliberate repro**

   - Enable persistent journald (already default on Ubuntu): `/var/log/journal/` exists.
   - Confirm pstore mounted (for kernel panics in EFI NVRAM):
     `mount | grep pstore`
   - Best: SSH from a phone or another machine, run
     `journalctl -kf | grep --line-buffered -iE 'radeon|drm|gpu|stall|lockup|fence'`
     sshd usually outlives Xorg by tens of seconds after a GPU lockup.
   - Alternative TTY (Mac keyboard: **Ctrl + Option + Fn + F3** or `sudo chvt 3`).
   - **Wait 60+ seconds on black screen before hard-rebooting** — the radeon
     lockup detector fires after 10s and then writes diagnostics. Hard-rebooting
     too early loses the log.

---

## Dead ends (don't waste time re-trying)

### Switching to amdgpu kernel driver

Bonaire is CIK, so amdgpu needs `amdgpu.cik_support=1 radeon.cik_support=0`.
With those flags + radeon blacklisted, amdgpu **does** load, but on this
specific Apple-firmware variant it hangs the display engine during early
modesetting → black screen at boot, requiring `nomodeset` rescue.

The documented community workaround is to add `amdgpu.dc=0` to disable
Display Core (Apple's BIOS doesn't expose the eDP panel the way amdgpu DC
expects). This was *not* attempted in this session and remains the only
known path to full GPU acceleration + Vulkan on this hardware. Try only
with a USB rescue stick ready.

### Chrome `--use-gl=desktop`

Deprecated in Chrome 148+. The flag is accepted but translated to
`gl=none, angle=none`, which fails the allow-list check. Result:
GPU process exits at startup → Chrome falls back to pure software
rendering for *everything* (compositing, raster, video — all CPU).
Works but heavy on CPU. Use `--disable-webgl --disable-webgl2` instead
to disable *only* WebGL while keeping compositing GPU-accelerated.

### chrome://flags WebGPU toggles

The "Unsafe WebGPU Support" and "WebGPU Developer Features" flags only
control unsafe/dev WebGPU features — toggling them off does NOT disable
WebGPU. On Linux radeon, WebGPU is effectively unavailable anyway
(requires Vulkan, which requires amdgpu), so it's never the trigger here.

---

## Reproducible repair recipe

Starting from a system in the broken state (`radeon.dpm=0` set, GPU
lockups under WebGL):

```bash
# --- 1. Clean kernel cmdline ---
sudo cp -a /etc/default/grub /etc/default/grub.bak-$(date +%Y%m%d-%H%M%S)
sudo sed -i 's|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"|' /etc/default/grub

# --- 2. Make sure radeon is NOT blacklisted (only if a previous amdgpu attempt left one) ---
sudo rm -f /etc/modprobe.d/blacklist-radeon.conf

# --- 3. Rebuild boot artifacts ---
sudo update-initramfs -u
sudo update-grub

# --- 4. Persist Chrome flags so WebGL is disabled by default ---
mkdir -p ~/.local/share/applications
cp /usr/share/applications/google-chrome.desktop ~/.local/share/applications/
sed -i 's|Exec=/usr/bin/google-chrome-stable|Exec=/usr/bin/google-chrome-stable --disable-webgl --disable-webgl2|g' \
  ~/.local/share/applications/google-chrome.desktop
update-desktop-database ~/.local/share/applications/ 2>/dev/null

# --- 5. Reboot to apply kernel cmdline ---
sudo reboot
```

Post-reboot verification:

```bash
lspci -nnk | grep -A3 VGA                 # expect: Kernel driver in use: radeon
cat /proc/cmdline                          # expect ONLY: quiet splash (+ BOOT_IMAGE etc)
cat /sys/class/drm/card1/device/power_dpm_state                  # expect: balanced
cat /sys/class/drm/card1/device/power_dpm_force_performance_level  # expect: auto
journalctl -k -b 0 | grep -iE 'gpu lockup|ring.*stall' | head    # expect: nothing
```

Stress test:

- WebGL: https://webglsamples.org/aquarium/aquarium.html with 1000 fish — should run for >15 min without lockup
- Video: any 4K YouTube fullscreen for >5 min — should run without UVD errors
- Chrome `--disable-webgl --disable-webgl2` + https://openai.com/codex/ — page renders without 3D background, no crash
- Firefox + https://openai.com/codex/ — full 3D background renders, no crash

---

## Useful one-liners reference

```bash
# What's bound to the GPU PCI slot?
readlink /sys/bus/pci/devices/0000:01:00.0/driver

# DRM device source (real driver vs EFI simpledrm fallback)?
ls -la /sys/class/drm/ | grep card

# Connector status + preferred mode per output
for f in /sys/class/drm/card*-*/status; do
  conn="${f%/status}"
  printf '%s: %s (preferred=%s)\n' \
    "${conn##*/}" "$(cat "$f")" \
    "$(head -1 "${conn}/modes" 2>/dev/null)"
done

# Live kernel log filter for GPU/DRM events
journalctl -kf | grep --line-buffered -iE 'radeon|drm|gpu|stall|lockup|fence'

# Force a fixed DPM level (escalation if `auto` flaps)
echo low  | sudo tee /sys/class/drm/card1/device/power_dpm_force_performance_level
echo high | sudo tee /sys/class/drm/card1/device/power_dpm_force_performance_level
echo auto | sudo tee /sys/class/drm/card1/device/power_dpm_force_performance_level

# GPU temperature (millidegrees C → divide by 1000)
cat /sys/class/drm/card1/device/hwmon/hwmon*/temp1_input
```

---

## Future escalation if symptoms return

If a future Mesa/kernel/Chrome update breaks even WebGL-disabled Chrome:

- **First**: re-check `chrome://gpu` to see what changed in Chrome's pipeline.
  Some Chrome updates add new GPU-accelerated features that may bypass the
  WebGL disable.
- **Second**: try forcing a fixed DPM state to avoid the `ci_dpm` transition bug:
  `echo low > /sys/class/drm/card1/device/power_dpm_force_performance_level`
  (make persistent via udev rule or systemd unit if it helps).
- **Last resort (full GPU acceleration restored)**: try amdgpu with the
  Apple-Bonaire workaround:
  ```
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash radeon.modeset=0 radeon.cik_support=0 amdgpu.cik_support=1 amdgpu.dc=0"
  ```
  plus `blacklist radeon` in `/etc/modprobe.d/`.
  Have a USB rescue stick / TTY recovery plan ready before rebooting. If it
  boots, you regain Vulkan, RADV, hardware WebGPU, and stable heavy-WebGL
  rendering.

---

## References

- Kernel doc on AMD CIK split between radeon and amdgpu:
  https://docs.kernel.org/gpu/amdgpu/index.html
- Chrome ANGLE backend selection:
  https://chromium.googlesource.com/angle/angle/+/main/doc/DevSetup.md
- Mesa radeonsi driver: https://docs.mesa3d.org/drivers/radeonsi.html
- Arch Wiki — AMDGPU & Radeon driver pages have the most up-to-date
  iMac15/iMac17-series quirk notes:
  https://wiki.archlinux.org/title/AMDGPU
  https://wiki.archlinux.org/title/ATI
