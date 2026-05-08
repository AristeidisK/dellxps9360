# History — How We Got Here

This is the full account of what was tried before the config in this repo worked.
It starts with a Monterey install that never achieved GPU acceleration, and ends
with Ventura working fully. The display fix is covered in detail in
[DISPLAY-DEBUGGING.md](DISPLAY-DEBUGGING.md) — this document is the broader story.

---

## Hardware

Dell XPS 13 9360 (service tag B2KMLH2)
- Intel Core i7-7500U (Kaby Lake-U)
- Intel HD Graphics 620 (0x5916)
- 16GB LPDDR3 1866MHz
- 512GB Toshiba XG5 NVMe
- 13.3" QHD+ 3200×1800 Sharp LQ133Z1 touchscreen
- Broadcom DW1560 WiFi/BT (swapped from OEM Killer 1535)

---

## Chapter 1 — Monterey (abandoned)

### Starting point

macOS Monterey 12.6.1 installed on the internal NVMe. OpenCore on the internal EFI
partition. SMBIOS: MacBookPro14,1. The machine booted, but only in VESA mode
(`-igfxvesa` in boot-args) — no GPU acceleration, single resolution at native
3200×1800 with UI elements tiny and unscaled.

### Attempt 1 — Default WhateverGreen platform-id (Monterey)

Platform-id: `0x59260002` (Iris Plus 640 — wrong for HD 620, but the WEG default
at the time).

Result: kernel panic on every boot at `AppleIntelController.cpp:29869`. The
symbol `IsTypeCOnlySystem` couldn't be resolved by Lilu. This looked like the
root cause, but turned out to be a red herring — the panic was a symptom of the
framebuffer driver failing to initialise, not a Lilu symbol issue.

Workaround: `-igfxvesa` suppressed the panic by disabling the Intel framebuffer
entirely, leaving the system on VESA fallback. Stable, but no GPU acceleration.

### Attempt 2 — SMBIOS swap to iMac18,1

Community guidance suggested that changing the SMBIOS to `iMac18,1` would bypass
the `IsTypeCOnlySystem` assertion that Kaby Lake laptop iGPUs sometimes hit.

**First try:** Regenerated all SMBIOS values (serial, UUID, MLB, ROM). The APFS
volume keybag is tied to the machine identity at first boot — changing all of
these mid-install broke the keybag. Boot hung silently at userspace.

**Second try:** Kept the original UUID, MLB, serial, and ROM. Only changed
`SystemProductName` to `iMac18,1`.

Result: Boot reached userspace, then a silent hang at AGDC with the kernel log
showing `"Didn't find DPMicro"`. The iMac18,1 SMBIOS tells AGDC to look for a
desktop DPMicro display chip that doesn't exist on a laptop. There is no way to
make this work on the 9360.

Added `agdpmod=vit9696` then `agdpmod=pikera` — the hang shifted but did not
resolve. Reverted to MacBookPro14,1.

### Attempt 3 — mrking95 community Monterey EFI

A community-known-working EFI for the 9360 on Monterey 12.6.3. Swapped our EFI
tree with it verbatim, keeping only our SMBIOS identity values. Added
`disable-agdc=true` and `-igfxnotelemetryload`.

Result: `AppleIntelKBLGraphicsFramebuffer` loaded but did not bind to the iGPU.
The config used a device-id spoof to `0x9D71` — this is an audio device-id, and
it prevented the kext personality from matching the GPU.

Fix attempt: removed the device-id spoof so the real HD 620 (0x5916) was exposed.
The kext bound, but the system still hung at userspace bringup.

### Monterey conclusion

Every combination tried on Monterey either kernel panicked or hung on this exact
hardware. No combination of platform-id, SMBIOS, boot-args, or kext config
produced a working Intel framebuffer on Monterey 12.x with this GPU.

Monterey GPU acceleration was abandoned. The machine sat in VESA-only mode on
Monterey for a period before the decision was made to switch OS.

---

## Chapter 2 — Switch to Ventura

A fresh install of macOS Ventura 13.7.8 was performed using a USB installer
created with [macadmin-scripts](https://github.com/munki/macadmin-scripts)
(`installinstallmacos.py`), which downloads a vanilla macOS image outside of the
App Store — useful when you don't have a Mac handy.

The switch to Ventura resolved the Monterey-specific kernel panic entirely. The
Kaby Lake framebuffer driver on Ventura initialises cleanly with the correct
platform-id and SMBIOS.

From this point, GPU acceleration was confirmed working via SSH (`system_profiler
SPDisplaysDataType` showing Metal and 1536MB VRAM) — but the physical display
remained frozen at the Apple logo. That is a separate issue documented fully in
[DISPLAY-DEBUGGING.md](DISPLAY-DEBUGGING.md).

The short version: `framebuffer-stolenmem` was set to 19MB, which is not enough
for the Metal Y-tiled framebuffer allocator to handle 3200×1800. Setting it to
64MB fixed it. The clue came from
[syncmaster851/darkvoid-XPS9360-macOS](https://github.com/syncmaster851/darkvoid-XPS9360-macOS)
which uses 80MB stolenmem for the same panel.

---

## Current state

As of May 2026, the config in this repo is fully working on Ventura 13.7.8:

- GPU acceleration (Metal 3, 1536MB VRAM) ✅
- QHD+ display at 1600×900 HiDPI ✅
- Audio, WiFi, Bluetooth, battery, trackpad, keyboard, brightness keys ✅
- Sleep/wake: system sleeps correctly, display stays blank on wake ⚠️

The sleep/wake display issue is the one known remaining problem.
