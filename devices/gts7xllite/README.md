# Samsung Galaxy Tab S7 FE 5G (gts7xllite)

SM-T736B. Qualcomm SM7225 (Snapdragon 750G, downstream `lagoon`), Adreno 619,
12.4" BOE TS124QDM panel (1600x2560, used as landscape), 5G modem. Perpetua
runs Debian 13 (trixie) with Denial Mobile; the rootfs is pushed over USB
into RAM at every boot and nothing is written to storage yet.

The port's record, plan and private backups live in the port workspace
(`SM-T736B-denial-port`: `ROADMAP.md`, `record/`).

## Access

The kernel's init brings up a USB gadget with a CDC ACM serial port and an
NCM network function (tablet `usb0` 192.168.7.2/24). Without a rootfs push,
init stays on its screen console and serial shell.

## Flashing

- Samsung has no fastboot. Download mode: power off, hold Vol-Up + Vol-Down,
  plug in USB, then Vol-Up on the warning screen (or `adb reboot download`
  from stock). `odin4` writes a ustar tar in the AP slot.
- Only `boot` is ever written. Never `vbmeta*`, `dtbo`, bootloader or
  firmware partitions. Never flash firmware below bootloader revision 9.
- The tablet charges on stock Android until perpetua drives the charger;
  without a driver it charges at the chip's defaults (500 mA input).

## Recovery

Download mode, then the stock `boot` image from the port's backups.

## Boot chain

Android boot header v2, no `vendor_boot` or `init_boot`: kernel, ramdisk and
DTB all in the 96 MiB `boot`. AVB: the stale signed vbmeta stays, with the
footer rewritten. Board-id 05 (dtbo index 3) is the overlay the bootloader
applies.

## Device files

- `gts7-touchd.service`: the FT8203 touchscreen has no kernel driver.
  `gts7-touchd` binds spidev, downloads the firmware (`tsp_focaltech/`,
  proprietary, not here) into the chip's RAM and feeds uinput, reading one
  frame per gpio22 edge. The tty1 login that starts Denial waits for it.
- `logind.conf.d/gts7xllite.conf`: Denial owns the power key
  (`pm8941_pwrkey`).
- `zram-swap` is enabled, as on every device. The kernel builds zram in,
  because the rootfs carries no modules.

The output transform (`DSI-1,270`), the DRM device paths and Denial's
settings (gamut correction off; with it off, landscape measured 60 fps)
are written by the port's `tools/update-rootfs.sh`.
