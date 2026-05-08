# Dell XPS 13 9360 — macOS Ventura Hackintosh

OpenCore EFI for running macOS Ventura 13.7.8 on the Dell XPS 13 9360 with full Intel HD 620 GPU acceleration, HiDPI display, and Metal 3 support.

## Hardware

| Component | Spec |
|-----------|------|
| CPU | Intel Core i7-7500U (Kaby Lake-U) |
| GPU | Intel HD Graphics 620 (device-id 0x5916) |
| RAM | 16GB LPDDR3 1866MHz |
| Storage | 512GB Toshiba XG5 NVMe |
| Display | Sharp LQ133Z1 QHD+ 3200×1800 touchscreen (SHP1449) |
| WiFi/BT | Broadcom DW1560 (swapped from OEM Killer 1535) |
| Audio | Realtek ALC256 (ALC3246) |
| Service tag | B2KMLH2 |

## Status

| Feature | Status |
|---------|--------|
| GPU acceleration (Metal 3) | ✅ Working |
| Display (3200×1800 HiDPI) | ✅ Working |
| HiDPI scaling (1600×900) | ✅ Working |
| Audio | ✅ Working |
| WiFi | ✅ Working |
| Bluetooth | ✅ Working |
| Battery / power management | ✅ Working |
| Brightness keys | ✅ Working |
| Trackpad (I2C) | ✅ Working |
| Keyboard | ✅ Working |
| USB | ✅ Working |
| NVMe | ✅ Working |
| Sleep / wake | ⚠️ Partial — system sleeps, display blank on wake |
| Touchscreen | ❌ Disabled |
| SD card reader | ❌ Not working |
| Fingerprint reader | ❌ Not supported |

## Kexts

| Kext | Purpose |
|------|---------|
| Lilu | Kernel extension patching framework |
| WhateverGreen | Intel GPU fixes and framebuffer patches |
| AppleALC | Audio — layout-id 56 for ALC256 |
| VirtualSMC | SMC emulation |
| SMCBatteryManager | Battery status |
| SMCProcessor | CPU temperature |
| SMCLightSensor | Ambient light sensor |
| SMCSuperIO | Fan sensors |
| SMCDellSensors | Dell-specific sensor data |
| AirportBrcmFixup | Broadcom WiFi fix |
| BlueToolFixup | Bluetooth fix for macOS 12+ |
| BrcmPatchRAM (via AirportBrcmFixup) | Broadcom Bluetooth firmware |
| VoodooPS2Controller | Keyboard and trackpad |
| VoodooI2C + VoodooI2CHID | I2C trackpad |
| CPUFriend + CPUFriendDataProvider | CPU power management for i7-7500U |
| NVMeFix | NVMe power management |
| USBInjectAll + USBPorts | USB port mapping |
| BrightnessKeys | Fn+F6/F7 brightness control |
| VerbStub | Audio verb patching |

## ACPI Patches

| File | Purpose |
|------|---------|
| SSDT-PLUG | CPU power management |
| SSDT-GPRW | Sleep/wake fix (work in progress) |
| SSDT-PTSWAK | Sleep/wake fix |
| SSDT-USBX | USB power properties |
| SSDT-TPDX | Trackpad ACPI fix |
| SSDT-TYPC | Thunderbolt/USB-C fix |
| SSDT-PCI0 | PCI device fix |
| SSDT-ALS0 | Ambient light sensor |
| SSDT-BCKM | Backlight key mapping |

## BIOS Settings (required before installing)

These UEFI variables must be set using the included `DVMT.efi` tool from the OpenCore Tools menu:

| Variable | Offset | Value | Notes |
|----------|--------|-------|-------|
| CFG Lock | 0x4DE | 0x00 | Disable CFG lock |
| DVMT Pre-allocation | 0x785 | 0x03 | 96MB — minimum for this config |
| DVMT Total Gfx Memory | 0x786 | 0x03 | MAX |

To run: select `OpenShell.efi` from the OpenCore boot menu, then:
```
DVMT.efi
setup_var 0x4DE 0x00
setup_var 0x785 0x03
setup_var 0x786 0x03
```

> **Optional:** Setting DVMT to 0x06 (192MB) gives more headroom and allows raising
> `framebuffer-stolenmem` to `00000005` (80MB) for additional stability.

## Key GPU Configuration

The critical fix that makes the display work is `framebuffer-stolenmem`. This must be
large enough for the Metal Y-tiled framebuffer allocator to handle 3200×1800.

```xml
<!-- DeviceProperties > Add > PciRoot(0x0)/Pci(0x2,0x0) -->
<key>AAPL,ig-platform-id</key>       <data>00001659</data>  <!-- 0x59160000, KBL-U native eDP -->
<key>device-id</key>                  <data>16590000</data>
<key>framebuffer-patch-enable</key>   <data>01000000</data>
<key>framebuffer-stolenmem</key>      <data>00000004</data>  <!-- 64MB — DO NOT lower this -->
<key>framebuffer-fbmem</key>          <data>00000003</data>
<key>AAPL00,override-no-connect</key> <data>[128-byte EDID — see docs/EDID.md]</data>
<key>enable-max-pixel-clock-override</key> <data>01000000</data>
<key>enable-hdmi20</key>              <data>01000000</data>
<key>enable-dpcd-max-link-rate-fix</key> <data>01000000</data>
<key>dpcd-max-link-rate</key>         <data>14000000</data>  <!-- HBR2 -->
```

## SMBIOS

**MacBookPro14,1** — do not change. The APFS volume keybag is tied to this identity.
Changing it will break the encrypted volume and prevent booting.

## Display

The Sharp LQ133Z1 QHD+ panel requires EDID injection (`AAPL00,override-no-connect`).
The correct EDID was extracted from a Ubuntu Live USB:
```
sudo cat /sys/class/drm/card1-eDP-1/edid | xxd -p | tr -d '\n'
```
See [docs/EDID.md](docs/EDID.md) for the full EDID and verification steps.

Result: 3200×1800 native, UI scaled to 1600×900 HiDPI (looks like a Retina MacBook).

## HiDPI Scaling Options

The default is 1600×900 (2× HiDPI). For additional scale options, use
[one-key-hidpi](https://github.com/xzhih/one-key-hidpi).

## Detailed Setup Guide

See [docs/GUIDE.md](docs/GUIDE.md) for the full step-by-step guide including what
doesn't work and why, with the root cause of the display fix explained.

## Credits

- [OpenCore](https://github.com/acidanthera/OpenCorePkg) — BSD 3-Clause
- [WhateverGreen](https://github.com/acidanthera/WhateverGreen) — BSD 3-Clause
- [Lilu](https://github.com/acidanthera/Lilu) — BSD 3-Clause
- [VirtualSMC](https://github.com/acidanthera/VirtualSMC) — BSD 3-Clause
- [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) — GPL 2.0
- [syncmaster851/darkvoid-XPS9360-macOS](https://github.com/syncmaster851/darkvoid-XPS9360-macOS) — reference for DVMT/stolenmem values
- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
