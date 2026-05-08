# Display Debugging — Intel HD 620 on QHD+ panel

This documents the root cause of the display issue and everything that was tried,
so future debugging doesn't repeat the same dead ends.

---

## The symptom

macOS boots fully (SSH accessible, Metal GPU confirmed working), but the physical
screen stays frozen at the Apple logo progress bar at ~20%. WindowServer is running
but rendering to a software/offline display instead of the hardware framebuffer.

---

## Root cause

**`framebuffer-stolenmem` was set to 19MB (`00003001`). This is too small.**

The sequence of failure:

1. EFI/OpenCore drives the eDP panel at native 3200×1800 in linear tiling mode (tiling=0)
2. macOS KBL driver loads, detects the boot pipe is already active ("Legacy Probe Case"),
   and inherits the EFI framebuffer without full DDI re-initialisation
3. IOAccelDisplayPipe tries to transition the framebuffer from linear (EFI) to Y-tiled
   (Metal) — this is the normal path even after Legacy Probe
4. The Metal surface allocation requires ~60–80MB of stolen GPU memory for a 3200×1800
   frame with multiple buffers. With only 19MB reserved, the allocation fails silently
5. Framebuffer stays in linear tiling mode (tiling=0)
6. CoreDisplay checks the framebuffer: sees tiling=0, no real VBLs (Fake VBL reported),
   classifies the display as non-hardware (`kIOWSAA = 0x80000013`)
7. WindowServer falls back to a software/offline display
8. The physical screen never receives any output from macOS — it shows the frozen EFI
   boot animation indefinitely

**The fix:** `framebuffer-stolenmem = 00000004` (64MB). This gives the Metal allocator
enough stolen memory to complete the tiling transition. CoreDisplay sees a tiled
framebuffer, classifies it as hardware, WindowServer renders to it.

---

## Kernel log evidence

From verbose boot, the Legacy Probe and tiling state are visible:

```
[Boot_Transition] Boot plane size : 3199 x 1799      ← EFI at native 3200×1800
[Boot_Transition] Boot plane config : tiling 0        ← LINEAR (not Metal)
[Connection_probe] Legacy Probe Case fb = 0           ← no full DDI reinit
FB0 Not waiting for VBL as path state is not active   ← pipe not fully active
Reporting Fake VBL for pipe 0 on Fb 0                 ← no real hardware VBLs
[WSAA] FB0: Transitioning wsaa from 0 to 80000003     ← offline flag set
```

After the fix, the transition to tiled completes, real VBLs are generated, and
CoreDisplay classifies FB0 as a hardware device.

---

## Things that do NOT fix it

### Platform ID choices
- **0x591B0000** — wrong for this laptop. Has DP connector flags (0x000003C7) on con0
  instead of LVDS/eDP flags (0x00000230). The internal panel is eDP, not DP.
- **0x59160000** — correct. Native LVDS/eDP on con0 with correct flags.

### Connector patches
- `framebuffer-con0-pipe: 12000000` (Pipe 18) — no effect. AAPL,DisplayPipe remained
  0x00000000. Pipe 18 is not a valid KBL hardware pipe (valid: 0/1/2).
- `framebuffer-con0-type`, `framebuffer-con0-busid` without EDID — driver aborts silently
- Connector patches without `framebuffer-con0-enable` — patches not applied

### SMBIOS changes
- **iMac18,1** — AGDC hang on boot: "Didn't find DPMicro". Desktop SMBIOS expects
  desktop display hardware (AGDC DPMicro chip) that doesn't exist on a laptop.
- **MacBookPro15,2** — Breaks APFS keybag. Machine identity mismatch prevents volume mount.

### Boot-args
- `-igfxvesa` — disables Intel framebuffer entirely, VESA only. Removes the problem
  but also removes GPU acceleration.
- `-cdfon` (CoreDisplay Force Online) — forces offline displays online, but doesn't
  change the tiling state or HW classification. Doesn't help.

### OpenCore settings
- `DirectGopRendering: true` — no effect on the Legacy Probe or tiling issue
- `ForceResolution: true` with `Resolution: 1920x1080` — the Intel GOP always drives
  the eDP hardware at native resolution regardless; the KBL driver reads the hardware
  pipe registers (not GOP metadata) and always sees 3199×1799

### WEG boot-args tried
- `igfxfw=3` (force GuC firmware) — no effect on display. Still in boot-args, harmless.
- `igfxonln=1` — needed and kept. Forces connector online state.

### Sleep/wake attempt
Forcing a system sleep and wake via `pmset sleepnow` does cause the KBL driver to do
a fresh probe of the eDP panel (not Legacy Probe). The panel powers off and back on.
However, the display still comes up blank on wake — a separate issue related to
CoreDisplay not reassigning the hardware display to WindowServer after wake.
(This is still unresolved — see sleep/wake section in GUIDE.md.)

### Hallucinated WEG properties (do not use)
These properties were suggested by various AI tools and do not exist in WEG:
- `framebuffer-unifiedmem`
- `framebuffer-con0-has-panel-limits`
- `framebuffer-fbmem-enable`
- `framebuffer-stolenmem-enable`
- `enable-metal`
- `igfxframe=1`
- `igfxsave=1`

---

## Diagnostic commands

```bash
# Check GPU and display state
system_profiler SPDisplaysDataType

# Check IOFramebuffer state (boot-display, pipe, connector type, power state)
ioreg -l -c IOFramebuffer | grep -E "AAPL,boot-display|AAPL,DisplayPipe|connector-type|DriverPowerState|kIOWSAA"

# Check DPCD / link rate / EDID state
ioreg -l | grep -E "dpcd|link.*rate|override-no-connect"

# Check stolen/fbmem allocations
ioreg -l | grep -E "stolen|fbmem|unifiedmem"

# Check kernel log for IGFB messages (run with -v in boot-args for early boot)
sudo dmesg | grep -iE "IGFB|FB0|legacy|boot plane|tiling|VBL|fake|WSAA"
```
