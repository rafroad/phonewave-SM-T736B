# Samsung Galaxy Tab S7 FE 5G (gts7xllite)

SM-T736B. Qualcomm SM7225 (Snapdragon 750G, downstream `lagoon`), Adreno 619,
12.4" BOE TS124QDM panel (1600x2560, used as landscape), 5G modem. Perpetua
boots from `boot`, and the image's Arch Linux ARM with Denial runs from the
microSD card. The internal UFS keeps Android: the kernel does not drive it.

Works: display (60 fps), GPU, touch, Wi-Fi, battery and charging status,
suspend (s2idle), RTC, the Book Cover Keyboard (EF-DT730) and the S Pen in
apps. Not yet: Bluetooth, auto-rotation and the other ADSP sensors, audio,
the modem, cameras.

The port's record, plan and private backups live in the port workspace
(`SM-T736B-denial-port`: `ROADMAP.md`, `record/`).

## Access

The kernel's init brings up a USB gadget with a CDC ACM serial port and an
NCM network function. On the card's system the tablet serves DHCP on `usb0`
(10.77.71.2; the computer gets 10.77.71.1) and runs SSH.

## Flashing

- Samsung has no fastboot. Download mode: power off, hold Vol-Up + Vol-Down,
  plug in USB, then Vol-Up on the warning screen (or `adb reboot download`
  from stock). `odin4` writes a ustar tar in the AP slot.
- Only `boot` is ever written. Never `vbmeta*`, `dtbo`, bootloader or
  firmware partitions. Never flash firmware below bootloader revision 9.
- Perpetua reads the SM5714 gauge and charger status. It charges at the
  chip's power-on 500 mA, or 1500 mA in / 1000 mA into the battery on a wall
  charger between 10 and 45 C; everything else stays at Samsung's defaults.

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
- `zram-swap` is enabled, as on every device. The kernel builds zram in.
- `rmtfs.service.d/10-gts7xllite.conf`: Wi-Fi's firmware service runs on the
  modem DSP, which needs rmtfs. It serves copies of `modemst1`, `modemst2`
  and `fsg` from `/var/lib/gts7xllite-efs`, read-only (`-r`: the modem's
  writes stay in memory), and starts the modem (`-s`). The kernel does not
  drive the UFS disk yet; the copies are the port's backups (they hold the
  IMEI, so they are not here).
- `NetworkManager/conf.d/gts7xllite.conf`: NetworkManager (what Denial's Wi-Fi
  settings use) leaves `usb0`, the USB link to the workstation, alone.
- `sleep.conf.d/80-gts7xllite-sleep.conf`: `AllowSuspend=yes` over the image's
  `70-phone-no-sleep.conf`, so logind lets Denial suspend the tablet
  (`CanSuspend` = yes). Hibernation stays off.

## Suspend

s2idle is the only sleep state (`/sys/power/mem_sleep` = `[s2idle]`). It
resumes on battery: the power key and an RTC alarm wake it, Wi-Fi, USB and
SSH come back (2026-11-18). With the USB cable in, the gadget keeps the
tablet awake (dwc3 'HS-PHY not in L2'), so test it unplugged.

Denial suspends on idle, but the mobile shell has no setting for it (the
image leaves Denial's desktop Settings out). Set it in
`~/.config/denial/settings.json`:

```json
"power": {
  "idleDpmsEnabled": true,
  "idleDpmsTimeoutSeconds": 30,
  "idleSuspendEnabled": true,
  "idleSuspendTimeoutSeconds": 300,
  "suspendMode": "s2idle"
}
```

- `suspendMode` must be `s2idle`: Denial's default is `deep`, which this
  tablet does not have.
- deniald applies the file live, but only with a `"version"` (30) and a
  `"revision"` higher than before; otherwise it logs "rejected externally
  edited Denial settings".
- When applied, deniald logs `configured automatic inactivity policy
  dpms_timeout_seconds=30 suspend_timeout_seconds=300 suspend_mode="s2idle"`.
- A short press of the power key only blanks the screen; it does not suspend.

The battery shows in Denial and UPower once the fuel gauge reports
`charge_full` (10090 mAh, stock `battery,battery_full_capacity`).

The output transform (`DSI-1,270`), the DRM device paths and Denial's
settings (gamut correction off; with it off, landscape measured 60 fps)
are written by the port's `tools/alarm-card.sh`. The suspend settings above
are not written by it yet.
