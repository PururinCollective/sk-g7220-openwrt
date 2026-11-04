# RemixWrt IIDX r18 (beta) — release notes

Images (`bin/`, built in `build/out/`): `aero-pon-g7220-r18-sysupgrade.bin`
(from r17: LuCI System > Backup / Flash Firmware or `sysupgrade`, settings
kept) and `aero-pon-g7220-r18-factory.bin` (from the stock web GUI). r17:
[remixwrt-r17.md](remixwrt-r17.md).

## Changes

- **Quick Setup** (top menu, after Status): the stock Quickset page
  (`skTmCombo/quickset.js`) as `aero/pages/quickset.js`, for setting the box
  up again after a factory reset: WAN mode (Combo / RG / Bridge /
  Satellite), the PPPoE account and LAN ports of one WAN connection, the
  primary SSIDs (band steering on: one SSID; off: 2.4G and 5G), VoIP line 1.
  The stock payloads in the stock order (do_set_wanmode), sent for what
  changed only. Combo shows WAN, WLAN and VoIP; RG WAN and WLAN; Bridge WAN
  and VoIP; Satellite the mode only. The connection fields belong to the
  current mode (the stock page hides them for another one); a new mode is
  sent last (`gwweb.wancfg set_wan_mode`), the stock stack then sets itself
  up for it and reboots the box. Differences from the stock page: PPPoE
  fields are sent only for a PPPoE connection and ports only for a bridge
  or an INTERNET connection (stock also wrote the hidden fields), the
  current `maxAssoc` is kept instead of 64.
- **Network > WAN, Binding Port**: no longer required for every
  connection. TR069, VOIP and IPTV routes run inside the ONT (VEIP) and
  need none; a bridge or an INTERNET service needs at least one LAN port or
  SSID; a bridge of another service than INTERNET may not take every port
  (the stock `wancfg_otherBridgeError` rule). The stock page itself checks
  only the last rule.
- **Factory reset resets the stock stack too**: LuCI's reset
  (`firstboot -r -y`), the reset button (OpenWrt's `/etc/rc.button/reset`,
  held 5 s) and `firstboot` all end in fstools' `factoryreset`, which
  empties rootfs_data. The stock stack then restored its old settings from
  `/apps/backup/config` at the next boot. Now `79_aero_overlay_split` sees
  the reset (rootfs_data pending, no build booted, stock bank did not run;
  `/tmp/.aero-factory-reset`, since mount_root empties rootfs_data again
  until a boot completes) and the chroot's `/etc/aero/preinit.sh` does what
  the stock `/etc/preinit` does after `prolinecmd restore default`: drops
  the configuration backups, loads the stock defaults (`sk_parseconfig
  -l`), keeps the PON registration (`pon_registe_info`, else the production
  values) and sets the uplink back to PON. The stock recovery flag
  (`ecnt_sys others recovery_flag`, set by a stock-side factory reset) is
  honoured the same way.
- **Stock web GUI port setting** (`aero.vendor.stockweb`): `lan` never took
  effect, the chroot runner starts from an empty environment; the setting
  (and the factory-reset flag) now go in with `env`.

## Tested

In the LuCI mock (real LuCI 25.12 JS, `tools/luci-mock`, now with
`get_wan_mode` and `get_onuType` answers): WAN page, VOIP edited and saved
without a port, the IPTV bridge refused without one; Quick Setup renders,
mode and band steering switch the right fields, a save sends
setWanCfg_encrypt, wlanGlobalSet, wlanRadioSet / wlanBasicSet_encrypt per
band, reload, setLineInfo, infoReload, hotEvent in the stock order, a mode
change sends only set_wan_mode. On the host: the reset detection of
`79_aero_overlay_split` for pending, running, second boot after a reset,
factory install and no state (marks only the first and the last). On the
device (r17, read only): the stock tools the reset uses answer in the
chroot (`ecnt_sys get others recovery_flag`, `pon_registe_info`,
`sk_test get_boot_switch_flag`).

Not tested on the device: a factory reset (it drops the stock configuration
backups and the ISP settings in them), Quick Setup and the WAN page with a
logged-in browser, a WAN mode change.
