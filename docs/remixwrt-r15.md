# RemixWrt IIDX r15 (beta) — release notes

RemixWrt IIDX is the new name of Aero-PON: the second remix (**II**) of
OpenWrt and the vendor's Wrt, deluxe (**DX**). Internal names (the `aero`
CLI, `/etc/config/aero`, rpcd `aero`, `AERO_PON_*` keys) are unchanged, so
an r14 configuration carries over.

Images (`build/out/`): `aero-pon-g7220-r15-sysupgrade.bin` (from r14:
LuCI System > Flash firmware or `sysupgrade`, settings kept) and
`aero-pon-g7220-r15-factory.bin` (from the stock web GUI).

## Changes

- **Name and banner**: RemixWrt IIDX everywhere (LuCI firmware version,
  `/usr/lib/os-release`, LuCI branch string, login banner, image
  sw-version `RemixWrt-r15`). New animated login banner: the figlet logo,
  "RemixWrt" in the sky-to-violet sweep, "IIDX" in a rose-to-gold gradient
  with a glint pass, the TX 1310 nm / RX 1490 nm light rules and the live
  PON line. The animation now actually runs on the device (busybox `sleep`
  takes whole seconds only; frames are paced with `ucode -e 'sleep(8)'`).
- **Status > Overview**: OpenWrt's own overview again, with two RemixWrt
  cards in Peter-Pon's style: *Fibre (GPON)* (animated SK-G7220 drawing,
  state, light in on the class B+ bar, laser out, optics/chip temperature,
  internet WAN, live traffic, release, links, GPON SN, details line) and
  *WAN* (every stock WAN connection). Storage no longer lists the /vendor
  chroot mounts; the stock DDNS/minidlna boxes are gone. Data: rpcd
  `aero.overview` (one vendor round trip, cached 3 s, no credentials).
- **Menus**: the stock Status pages (WAN, LAN, WLAN, VoIP, TR-069) moved to
  a new top menu **OEM**; the Aero-PON menu is **RemixWrt** (ONT Auth, ONT
  Identity, OMCI Topology, OLT Status, Vendor Stack).
- **DDNS** (Advanced > DDNS): the stock client is replaced by NijikaWrt's
  (`/usr/libexec/aero/ddns`, `aero-ddns`): DynDNS v2 family, GnuDIP,
  freedns.afraid.org, Cloudflare; HTTPS via curl; CGNAT detection.
- **Encrypted DNS** (Network > LAN > Encrypted DNS): DoH (https-dns-proxy)
  or DoT (stubby) for everything the stock DHCPv4/DHCPv6 servers serve.
  The stock dnsmasq gets a conf-dir drop-in (`no-resolv`, `server=` the
  local proxy); `ntp.org` and the ISP's domain stay on plain DNS so the
  clock can be set before TLS. Off by default; presets Cloudflare,
  Cloudflare Families, Google, Quad9, AdGuard, Mullvad.
- **Clock fix**: OpenWrt's sysntpd never ran (ujail cannot drop
  capabilities 38+ on a 5.4 kernel); the clock stayed at the build date.
  ntpd now runs unjailed and sets it within a minute of the WAN coming up.
- **Packages**: https-dns-proxy, stubby (+ getdns, libopenssl3), curl,
  installed with OpenWrt's apk at build time (`build/dl/apk-cache`).
- **WireGuard**: not included. The stock kernel's struct layout is not
  vanilla 5.4.55, so a module needs that ABI rebuilt first; see
  [wireguard-feasibility.md](wireguard-feasibility.md).

## Tested on the device (2026-11-04)

r15 written to bank B with `aero upgrade --other-bank`, booted; then the
final build installed over it with `sysupgrade` (settings kept), booted.
PON O5 (−9.4 dBm in, 2.4 dBm out), PPPoE WAN up, NTP set at ~70 s, overview
cards, OEM/RemixWrt menus, OLT Status, DDNS page and backend, DoH and DoT
(dnsmasq statistics: every query to the local proxy, none to the ISP), the
off path, DoH across a reboot, LuCI Save & Apply of Encrypted DNS. Bank A
still holds r14 (`ecnt_sys swap bootflag && reboot` to go back).

Known: load average reads ~8 with an idle CPU (vendor kernel threads such
as `LDDLA_task_wait` and `dis_if` sit in uninterruptible sleep).
