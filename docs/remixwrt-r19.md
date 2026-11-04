# RemixWrt IIDX r19 (beta) — release notes

Images (`bin/`, built in `build/out/`): `aero-pon-g7220-r19-sysupgrade.bin`
(from r18: LuCI System > Backup / Flash Firmware or `sysupgrade`, settings
kept) and `aero-pon-g7220-r19-factory.bin` (from the stock web GUI). r18:
[remixwrt-r18.md](remixwrt-r18.md).

## Changes

- **Encrypted DNS on by default**: `aero.dns` now ships with DNS over HTTPS
  through `https://b.hitoha.moe/dns-query` (bootstrap `1.1.1.1,1.0.0.1`,
  local proxy port 5053). Plain DNS domains are `pool.ntp.org` and
  `my.pool.ntp.org`, which cover the default NTP servers
  (`0-3.openwrt.pool.ntp.org`). `tm.com.my` is no longer in the default list.

The new defaults reach a box whose OpenWrt settings start empty: a factory
reset, or a `-factory.bin` of a new revision. A sysupgrade with settings kept
leaves the existing `aero.dns` as it is, on purpose: `off` there may be a
choice. DoT defaults are unchanged (Cloudflare).
