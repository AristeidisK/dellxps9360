# Sharp LQ133Z1 EDID

Panel: Sharp LQ133Z1 QHD+ 3200×1800 touchscreen
Vendor ID: 0x4D10 (SHP)
Product ID: 0x144A
EDID checksum: 0xAE

## Raw EDID (128 bytes)

```
00ffffffffffff004d104a14000000001e190104a51d11780ede50a3544c99260f
505400000001010101010101010101010101010101cd9180a0c00834703020350026
a510000018a47480a0c00834703020350026a510000018000000fe0052584e343981
4c513133335a31000000000002410328001200000b010a202000ae
```

Single line (for use in config.plist):
```
00ffffffffffff004d104a14000000001e190104a51d11780ede50a3544c99260f505400000001010101010101010101010101010101cd9180a0c00834703020350026a510000018a47480a0c00834703020350026a510000018000000fe0052584e3439814c513133335a31000000000002410328001200000b010a202000ae
```

## How to extract from the hardware

Boot a Ubuntu Live USB and run:
```bash
sudo cat /sys/class/drm/card1-eDP-1/edid | xxd -p | tr -d '\n'
```

Verify the last byte is `ae` (checksum). If transcribing manually from a photo,
expect errors — always extract from the live file.

## Verification

Sum all 128 bytes mod 256 should equal 0. The last byte (0xAE) is the checksum byte
chosen to make the total sum to 0.

The descriptor block at offset 0x48 contains:
```
000000fe00  (monitor name descriptor tag)
52584e3439  "RXN49"
814c513133335a31  "LQ133Z1"
```
This confirms it is the correct Sharp LQ133Z1 panel EDID.

## Why EDID injection is needed

The Sharp eDP panel does not assert HPD (Hot Plug Detect) in the way the
AppleIntelKBLGraphicsFramebuffer driver expects for internal displays. Without the
EDID override (`AAPL00,override-no-connect`), the connection probe returns no EDID
and the driver aborts silently, leaving the display uninitialised.
