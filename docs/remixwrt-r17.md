# RemixWrt IIDX r17 (beta) — release notes

Images (`bin/`, built in `build/out/`): `aero-pon-g7220-r17-sysupgrade.bin`
(from r16: LuCI System > Backup / Flash Firmware or `sysupgrade`, settings
kept) and `aero-pon-g7220-r17-factory.bin` (from the stock web GUI).
r16 (WireGuard) is described in [wireguard-feasibility.md](wireguard-feasibility.md).

## Changes

- **No online upgrade check**: OpenWrt's attended sysupgrade is removed
  (`luci-app-attendedsysupgrade`, `owut`, `attendedsysupgrade-common`,
  `rpcd-mod-rpcsys`; `AERO_APK_DEL` in `build/build-graft-rootfs.sh`), and
  with it the "Check online for firmware upgrades" dialog on the first
  Status > Overview visit and System > Attended Sysupgrade. It looks for
  OpenWrt's builds of the running target (armsr/armv8), never an SK-G7220
  image. Flashing from LuCI or `sysupgrade` is unchanged. An upgraded
  device keeps an unused `/etc/config/attendedsysupgrade`.
- **Login banner animated on the serial console too**:
  `aero.banner.animate` defaults to `always` (was `ssh`);
  `uci-defaults/93-remixwrt-banner` moves a kept `ssh` to `always`. On the
  console (`TERM=vt102`, 16 colours) the 46 frames are ~45 KB, ~4 s at
  115200 baud; `uci set aero.banner.animate=ssh` brings back the static
  banner there.
- **Build**: `build/build.sh stock` writes the stock firmware as
  `-sysupgrade.bin` (V1.0.0.6 and V1.0.0.7, see
  [aero-pon-upgrade.md](aero-pon-upgrade.md) "Back to stock from RemixWrt");
  `bin/` holds the current firmware. Removing packages exposed a fakeroot
  trap (host apk writes past fakeroot: a symlink on a freed inode read as a
  regular file), fixed in `build-graft-rootfs.sh` (`build/README.md`).

## Tested

On the host: r16 and r17 `unsquashfs -lln` listings differ only in the
removed packages' files, apk's database and `/etc/opkg` (now a link to
`/vendor/etc/opkg`; OpenWrt 25.12 uses apk). On the device (r16, bank B):
`aero.banner.animate=always` set with the 93 script; the banner then draws
all frames with `TERM=vt102` (45,174 bytes instead of 1,199); the setting is
in `aero-host.tgz`. r17 itself is not flashed yet.
