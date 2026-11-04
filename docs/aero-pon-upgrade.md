# Aero-PON images and upgrades (SK-G7220)

## The two banks

The SK-G7220 has two NAND banks of 60 MiB, `tclinux` (A, mtd1) and
`tclinux_slave` (B, mtd4). Each holds one stock-format "tclinux" container:
`[8 KiB cert][HDRs @0x2000][FIT @0x2100][zero pad]`. U-Boot (dualbob) boots
the bank selected by the bootflag in mtd `others`: `bootflag=0` (A) or `1` (B),
visible in `/proc/cmdline`. `ecnt_sys swap bootflag` toggles it.

Rule: one bank keeps the stock firmware. It is the only recovery: the U-Boot
console is locked, there is no TFTP recovery.

`rootfs_data` (mtd7, jffs2) is **shared** by both banks. The stock firmware
uses its `upper/` as its overlay, Aero-PON its `upper/` too, plus
`aero-vendor/` (the configuration of the stock stack in the /vendor chroot),
`aero-host.tgz` (a copy of the OpenWrt settings) and `stock-upper/`.

## The two images

`build/build.sh aero` (revision from `AERO_PON_REVISION`) writes:

| image | for | contents |
|---|---|---|
| `aero-pon-g7220-<rev>-factory.bin` | the **stock** web GUI (Management > Upgrade) | the tclinux container, padded to the size the cert declares (the stock secure-upgrade size check) |
| `aero-pon-g7220-<rev>-sysupgrade.bin` | **Aero-PON**: LuCI System > Backup / Flash Firmware, or `sysupgrade` | the same container + OpenWrt metadata (`supported_devices: skyworth,sk-g7220`), written by the image's own `fwtool` under qemu-user |

Every image carries `AERO_PON_BUILD` (revision + hash of `package/`) in
`/etc/aero-pon-release`.

## Factory install (from the stock firmware)

1. Boot the stock bank (from Aero-PON: `ecnt_sys swap bootflag; reboot`).
2. Flash `-factory.bin` in the stock web GUI. The stock updater writes the
   other bank (the old Aero-PON one) and switches the bootflag.
3. First boot (`/lib/preinit/79_aero_overlay_split`): `upper/etc/config` has
   `skapi` and `ucitrack` (the stock userland ran: the stock image ships no
   `/etc/config`, every stock boot writes both; OpenWrt 25.12 has no
   `ucitrack`) and the build has not booted here before ->
   **factory install**: the stock configuration in `upper/etc/config` is copied
   to RAM, rootfs_data is emptied (fstools' `.fs_state` stays), the copy becomes
   `aero-vendor/upper/etc/config` (ISP provisioning, PPPoE, WiFi, VoIP in the
   chroot). OpenWrt starts from its defaults (empty root password).

## Back from the stock bank (same build)

`skapi` and `ucitrack` in `upper/` but the build booted before: `upper/` is moved to
`stock-upper/`, the OpenWrt settings come back from `aero-host.tgz`
(refreshed every minute by `aero-housekeep` and at shutdown), and the vendor
boot takes over `stock-upper/etc/config` once (`.aero-split` round-trip
counter), so changes made in the stock GUI follow.

A factory install resets the OpenWrt settings (root password, LuCI) on
purpose. To keep them, upgrade from Aero-PON instead: LuCI System > Backup /
Flash Firmware (or `sysupgrade`) with `-sysupgrade.bin`. It rewrites the
running bank, keeps rootfs_data and the stock bank. The stock updater can be
told apart in a UART log: it pipes the image ("Writing from <stdin> to
tclinux") and writes the bootflag to "others" afterwards.

## Factory reset

LuCI's reset (System > Backup / Flash Firmware, `firstboot -r -y`), the
reset button (held 5 s, OpenWrt's `/etc/rc.button/reset`) and `firstboot`
all run fstools' `factoryreset`: rootfs_data is emptied (both the OpenWrt
settings and the chroot's `aero-vendor/`). From r18 the stock stack resets
with it: `79_aero_overlay_split` sees the empty, pending rootfs_data and the
chroot preinit drops `/apps/backup/config`, loads the stock defaults and
keeps the PON registration, as the stock firmware does after
`prolinecmd restore default`. Before r18 the stock stack restored its newest
backup instead. Quick Setup (top menu) sets the box up again.

## sysupgrade (from Aero-PON)

`/lib/upgrade/platform.sh`:

- `platform_check_image`: HDRs magic at 0x2000, chip id `7581` at 0x2251, FIT
  at 0x2100, size <= the bank. `REQUIRE_IMAGE_METADATA=1`: a `-factory.bin`
  needs `-F`.
- `platform_do_upgrade`: strips the metadata, writes the **running** bank with
  `mtd write`, then `mtd verify`. The other bank (stock) is not touched. A
  failed write or verify switches the bootflag to the other bank, so the next
  boot is the fallback.
- Settings: rootfs_data is not touched, so everything is kept. `sysupgrade -n`
  leaves `/overlay/.aero-wipe-host`: the next boot resets the OpenWrt settings;
  the stock configuration in the chroot stays.
- Risk: a power cut while the running bank is written leaves it broken; recover
  by switching to the stock bank (serial console: `ecnt_sys swap bootflag`;
  whether U-Boot falls back by itself is not verified).

## The stock method from Aero-PON

`aero upgrade --other-bank [--replace-stock] [-r] <image>` writes the other
bank and switches the bootflag, as the stock updater does; the running bank
becomes the fallback. If the other bank holds the stock firmware (HDRs
sw-version not `AeroPON*`) it refuses without `--replace-stock`. `aero bank`
shows both banks.

## Back to stock from RemixWrt

`build/build.sh stock` writes the stock firmware as sysupgrade images (in
`build/out/`, copied to `bin/`):

| image | contents |
|---|---|
| `sk-g7220-stock-V1.0.0.6-sysupgrade.bin` | this unit's factory firmware V1.0.0.6: reassembled from `stock-parts`, identical to the first 42,948,608 bytes of the bank-A backup (`backup/nand-20261026/mtd1.bin`) |
| `sk-g7220-stock-V1.0.0.7-sysupgrade.bin` | the official update `SK-G7220_V1.0.0.7_20260106.bin`, unchanged (sha256 `aed26cc8…`) |

Both carry the same metadata as the RemixWrt images (`supported_devices
skyworth,sk-g7220`, compat 1.0, dist "Skyworth"), so LuCI System > Backup /
Flash Firmware and `sysupgrade` take them without `-F`. Like any sysupgrade
they write the **running** bank: RemixWrt is replaced there, the other bank
keeps what it has. `platform_do_upgrade` strips the metadata first, so the
bank gets the stock container byte for byte (the build checks this with the
target fwtool). The stock firmware then boots on the shared rootfs_data and
restores its own configuration (`/apps/backup/config`), as after any bank
switch to stock. "Keep settings" makes no difference to stock; unticked, it
also leaves `.aero-wipe-host`, so a later boot of the same RemixWrt build
starts from its defaults.

To return: the stock web GUI with `aero-pon-g7220-<rev>-factory.bin`, or the
serial console (`ecnt_sys swap bootflag; reboot`) when the other bank still
holds RemixWrt. Checked on the host (2026-11-04): OpenWrt 25.12.6
`fwtool_check_image` and this `platform_check_image` accept both images. Not
yet flashed this way on the device. V1.0.0.7 has not been booted on this unit
in these sessions; V1.0.0.6 is what it shipped with.
