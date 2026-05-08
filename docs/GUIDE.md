# Setup Guide — Dell XPS 13 9360 macOS Ventura

Full walkthrough from a fresh Windows machine to a working macOS Ventura Hackintosh
with Intel HD 620 GPU acceleration on the QHD+ display.

---

## Prerequisites

- A Mac or another machine to create the USB installer
- A USB drive (16GB+) for the macOS installer
- A second USB drive (1GB+) for the OpenCore EFI fallback
- Access to a Ubuntu Live USB (for EDID extraction — optional if using this EFI as-is)

---

## Step 1 — BIOS settings

Enter BIOS (F2 at Dell logo) and configure:

- SATA mode: AHCI
- Secure Boot: Disabled
- TPM: can be left on
- CFG Lock: must be disabled via UEFI variable (see Step 2)

---

## Step 2 — Set required UEFI variables

These cannot be set in the normal BIOS UI. Boot OpenCore, select `OpenShell.efi`,
then run `DVMT.efi` (included in `EFI/OC/Tools/`):

```
fs0:
cd EFI\OC\Tools
DVMT.efi
setup_var 0x4DE 0x00    # Disable CFG Lock
setup_var 0x785 0x03    # DVMT Pre-alloc = 96MB
setup_var 0x786 0x03    # DVMT Total = MAX
```

Reboot after setting these.

> **Note:** Offset 0x785 = 0x03 gives 96MB DVMT. This is the minimum for this config.
> Setting it to 0x06 (192MB) gives more headroom and is recommended if possible.

---

## Step 3 — Create macOS installer USB

On a Mac, download macOS Ventura from the App Store and run:
```bash
sudo /Applications/Install\ macOS\ Ventura.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/USB
```

Then copy the EFI folder from this repo to the EFI partition of the installer USB.

---

## Step 4 — Install macOS

Boot from the USB. Install macOS to the internal NVMe. This is straightforward —
the tricky part is what comes after.

---

## Step 5 — Move OpenCore to internal EFI

After installation, OpenCore is still on the USB. To boot without the USB:

1. Boot macOS (with USB plugged in)
2. Mount the internal EFI:
   ```bash
   sudo diskutil mount disk0s1
   ```
3. Copy the EFI folder from the USB to the internal EFI partition:
   ```bash
   sudo cp -r /Volumes/EFI/EFI /Volumes/EFI\ 1/
   ```

---

## Step 6 — SMBIOS serial numbers

The `config.plist` in this repo has placeholder serials. You must generate your own
using [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS):

```
SMBIOS: MacBookPro14,1
```

**Critical:** Once you boot with a specific SMBIOS identity and macOS creates an APFS
volume, do not change the `SystemProductName`, `SystemUUID`, `MLB`, `ROM`, or
`SystemSerialNumber`. Changing them will break the APFS keybag and the volume will
not mount. The keybag is tied to the machine identity at first boot.

---

## Step 7 — GPU and display

This is the most complex part. The EFI in this repo is pre-configured with the correct
values. The key settings are in `DeviceProperties > Add > PciRoot(0x0)/Pci(0x2,0x0)`.

**The critical value:**
```
framebuffer-stolenmem = 00000004  (64MB)
```

With less than ~60MB of stolen memory, the Metal Y-tiled framebuffer allocator cannot
allocate a surface for 3200×1800. The allocation fails silently. The framebuffer stays
in the EFI-inherited linear mode. CoreDisplay classifies the display as non-hardware.
WindowServer falls back to a software display. The screen stays frozen at the Apple logo.

See [DISPLAY-DEBUGGING.md](DISPLAY-DEBUGGING.md) for the full root cause analysis and
everything that was tried.

**Expected boot behaviour once configured correctly:**
1. OpenCore picker appears
2. Apple logo with progress bar
3. Brief brightness dip (KBL driver takes over the backlight from EFI)
4. Login screen at 1600×900 HiDPI

---

## Step 8 — Audio

Audio uses `AppleALC` with layout-id `56`. This is set in DeviceProperties for the
audio device. Should work out of the box with this EFI.

ComboJack is included in `EFI/OC/Tools/ComboJack.zip` for headphone/microphone
combo jack support. Install it separately if needed.

---

## Step 9 — Keep the USB as a fallback

Always keep a USB with the working EFI as a fallback. If you break the internal EFI:

1. Boot from the USB
2. SSH in or use the terminal
3. Mount the internal EFI and fix it:
   ```bash
   sudo diskutil mount disk0s1
   ```

The USB EFI should have `-igfxvesa` in boot-args as a safe fallback that always boots
(VESA mode, no GPU acceleration, but the system is accessible).

---

## Updating the EFI

When you change the internal EFI config, sync it to the USB:
```bash
sudo diskutil mount disk0s1   # internal = /Volumes/EFI 1
sudo diskutil mount disk2s1   # USB = /Volumes/EFI
sudo cp "/Volumes/EFI 1/EFI/OC/config.plist" /Volumes/EFI/EFI/OC/config.plist
```

---

## Known issues / to do

- **Sleep/wake:** System sleeps correctly. On wake, SSH returns and the KBL driver
  powers the panel on, but the display stays blank (login screen does not reappear).
  First thing to try: add `darkwake=4` to boot-args.

- **Touchscreen:** Works. VoodooI2CHID is enabled in the config.

- **SD card reader:** Not working, no driver available for macOS.

- **Fingerprint reader:** Works.
