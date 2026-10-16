# Quadro P3000 (MXM) in the iMac 27" Mid 2010: enabling the internal display in Windows

Tested on an iMac11,3 (27" Mid 2010) with a Quadro P3000 Mobile (GP104, 10de:1bb8). Windows is installed in UEFI mode, OpenCore 1.0.8. No macOS and no OCLP. The same behavior occurred with the R470 and R580 drivers.

## Requirements

The card must be flashed with an iMac vBIOS that includes a GOP driver and has the internal panel enabled. The original laptop vBIOS does not light this panel.

OpenCore must be installed on the iMac, or you need a working bootable USB stick with it.

With that vBIOS and the two OpenCore settings below, the internal display works in the OpenCore menu and in Windows until the NVIDIA driver loads.

### vBIOS

The vBIOS is flashed with a hardware programmer. Firmware images and the flashing procedure are described in these MacRumors posts:

**Santa's Little Helper** — https://forums.macrumors.com/threads/imac-2011-maxwell-and-pascal-gpu-upgrade.2300989/page-7?post=30349621#post-30349621

**m0bil** — https://forums.macrumors.com/threads/imac-2011-maxwell-and-pascal-gpu-upgrade.2300989/page-9?post=30638999#post-30638999

### OpenCore

Windows has to be booted through OpenCore. Booting Windows directly does not apply the ACPI patch, and the driver will turn the panel off again.

In `UEFI → Quirks`:

- `ForgeUefiSupport` = `True`
- `ReloadOptionRoms` = `True`

These two settings make the 2010 firmware run the GOP from the card, so the panel is on before the operating system starts.

## The ACPI patch

After the NVIDIA driver loads, the internal panel turns off and only the external monitor keeps working. The vBIOS and the wiring are fine. The driver disables the panel because of this iMac's ACPI.

In the stock DSDT the graphics card is described as the old Radeon. The `GFX0._DOD` method returns a single display identifier, `0x0110`. The NVIDIA driver checks this list to decide which output is the internal panel, does not find its own eDP output there, and disables the panel.

Rename `_DOD` to `XDOD`. The driver no longer sees the method and detects the panel itself. The `LCD` device and its brightness methods (`_BCL`, `_BCM`, `_BQC`) stay in place. On this iMac, brightness is controlled by the SMC, not by the graphics card.

Add this entry under `ACPI → Patch`:

```xml
<dict>
    <key>Base</key>
    <string></string>
    <key>BaseSkip</key>
    <integer>0</integer>
    <key>Comment</key>
    <string>GFX0 _DOD to XDOD</string>
    <key>Count</key>
    <integer>1</integer>
    <key>Enabled</key>
    <true/>
    <key>Find</key>
    <data>FA1fRE9EAA==</data>
    <key>Limit</key>
    <integer>0</integer>
    <key>Mask</key>
    <data></data>
    <key>OemTableId</key>
    <data></data>
    <key>Replace</key>
    <data>FA1YRE9EAA==</data>
    <key>ReplaceMask</key>
    <data></data>
    <key>Skip</key>
    <integer>0</integer>
    <key>TableLength</key>
    <integer>0</integer>
    <key>TableSignature</key>
    <data>RFNEVA==</data>
</dict>
```

`Find` is the bytes `14 0D 5F 44 4F 44 00`, the AML header of `Method (_DOD, 0)`. `Replace` changes the name to `XDOD`. `TableSignature` limits the patch to the DSDT. In the stock iMac11,3 firmware this sequence occurs once.

## After the patch

- The internal 2560×1440 panel stays on with the NVIDIA driver.
- The brightness slider works.
- The image comes back after sleep.
- The external monitor works at the same time as the internal panel.

No separate SSDT or extra kexts are required.
