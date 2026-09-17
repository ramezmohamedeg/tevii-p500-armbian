# Armbian 5.9 on the TEVII P500 (Allwinner H6 / Tanix TX6-compatible)

This documents how I got **Armbian with the 5.9 kernel** booting on a **TEVII P500** Android TV box. I couldn't find any existing guide for the 5.9 build specifically on this device, so I'm writing this up in case it helps someone else.

## The hardware

- **Device:** TEVII P500 (Android TV box)
- **SoC:** Allwinner H6
- Hardware/device-tree-compatible with the **Tanix TX6**, so it uses the `sun50i-h6-tanix-tx6` device tree from the standard multi-SoC Armbian builds for these boxes.

## What already worked: Armbian, 5.7 kernel

I had a working setup on the 5.7 kernel build, using a two-step flash:

1. **Rufus** — writes the full Armbian image (boot + rootfs partitions) to the SD card/eMMC.
2. **Balena Etcher** — writes a device-specific `u-boot-allwinner-h6-tanix-tx6.img` on top.

That second step isn't a full-disk image — it's exactly the size of the region *before* the first partition (the file ends precisely at the boot partition's starting offset). Etcher writes it byte-for-byte, so it only overwrites the raw bootloader area and leaves the partitions Rufus already wrote untouched.

On 5.7, boot configuration was handled by `boot.cmd`/`boot.scr` + `uEnv.txt`, with `uEnv.txt` pointing `FDT` at `/dtb/allwinner/sun50i-h6-tanix-tx6.dtb`. This booted fine.

## The problem with 5.9

Repeating the exact same two-step flash with the 5.9 build produced no boot at all — and I couldn't find any guide covering 5.9 on this device to compare against.

## Root cause

Comparing a full backup of the working 5.7 boot partition against the non-booting 5.9 one turned up the real issue: **the boot mechanism changed entirely between the two kernel versions.**

- 5.7 boots via `boot.scr` + `uEnv.txt`.
- 5.9 drops `boot.scr` and uses U-Boot's built-in **distro boot** via `/extlinux/extlinux.conf` instead. (Confirmed from the U-Boot binary's own embedded boot script: it checks `extlinux/extlinux.conf` *before* it ever looks for a `boot.scr`.)

The 5.9 image ships one `extlinux.conf` template covering several SoC families (Rockchip RK3399, RK3328, Amlogic, and Allwinner H6), and by default **the active, uncommented block was set to Rockchip RK3399** — wrong device tree, wrong serial console, wrong everything for this hardware. The correct Allwinner H6 block was present in the same file, just fully commented out.

So even with a correctly-flashed, H6-compatible U-Boot, the kernel was being told to boot as a Rockchip RK3399 board. That's a guaranteed failure — wrong device tree means no matching hardware, no working console, nothing.

## The fix

Edit (or replace) `/extlinux/extlinux.conf` on the boot partition so the **Allwinner H6** block is the active one instead of RK3399:

```
LABEL Armbian
LINUX /zImage
INITRD /uInitrd
FDT /dtb/allwinner/sun50i-h6-tanix-tx6.dtb
APPEND root=LABEL=ROOTFS rootflags=data=writeback rw console=ttyS0,115200 console=tty0 no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 video=HDMI-A-1:e
```

(Comment out or remove the RK3399/RK3328/Amlogic blocks — the full corrected file is in this repo as [`extlinux.conf`](./extlinux.conf).)

## Download

Download Armbian_20.10_Arm-64_bullseye_current_5.9.0_desktop.img.xz from :  https://drive.google.com/drive/folders/1CJEsZ6jdRGFC7XpOFVp8eG-DO0tbrOOb
Download my patched extlinux.conf file from this repo.


### Full flashing procedure for 5.9

1. Flash the 5.9 Armbian image to the SD card/eMMC with **Rufus**, same as before.
2. Replace `/extlinux/extlinux.conf` on the boot partition with the corrected version above.
3. Flash `u-boot-allwinner-h6-tanix-tx6.img` on top with **Balena Etcher**, same as the 5.7 process — this only touches the bootloader region, not the files from steps 1–2.
4. Boot.

## Troubleshooting

- **No display:** the corrected config uses `video=HDMI-A-1:e` (auto-detect via EDID). If your display doesn't show anything, try forcing a resolution instead: `video=HDMI-A-1:1920x1080@60e`.
- **Still nothing:** the `APPEND` line sets `console=ttyS0,115200` as the primary UART. A USB-TTL adapter on the board's debug UART pins will show you exactly where U-Boot or the kernel is stalling.

## Disclaimer

This is personal testing on one specific device. It should generalize to other Allwinner H6 boxes using the same `sun50i-h6-tanix-tx6` DTB, but your results may vary on different hardware. Corrections and improvements welcome.
