# Aero-PON userland graft — design decision

How the **latest OpenWrt (25.12) userland** gets onto the SK-G7220 while the
proprietary EcoNet GPON/OMCI/Wi-Fi/VoIP stack keeps working. This is the design
the task asked to settle **before** the heavy integration build. The container
packaging is solved and proven (see [tclinux-container.md](tclinux-container.md)
and [../build/README.md](../build/README.md)); this doc is about *what rootfs
tree* we pack.

Decided 2026-10-26, from the full NAND backup (`backup/nand-20261026/`).

## Key correction to the original plan

[stage2-plan-rootfs-swap.md](stage2-plan-rootfs-swap.md) assumed a NijikaWrt
(D-Link/Realtek) shape: a glibc vendor blob confined to a `/rtk` subtree,
chrooted. **That is not the SK-G7220.** The stock rootfs (`mtd3`, carved) is a
**normal OpenWrt 21.02-SNAPSHOT musl FHS** — the vendor stack is integrated
directly into the standard tree, not isolated:

- **No `/rtk`.** Top-level dirs are stock OpenWrt: `bin etc lib sbin usr www`
  plus vendor extras `apps config scripts share userfs`.
- `/lib/modules/5.4.55/` — **165 vendor kmods** (PON `xpon`/`pon_phy`, OMCI,
  MT7992 `mt76`/`connac`, switch, VoIP DSP, NPU, netfilter, …).
- `/lib/firmware/` — vendor blobs: `MT7992*_EEPROM*.bin`, `MT7991*`,
  `EthMD32.dm.bin`/`EthMD32.DSP.bin`, `mediatek/`.
- Vendor daemons + their `.so`s live in the normal `/bin /sbin /usr/{bin,sbin,lib}`.
- Vendor init in `/etc/init.d`: `pon xponconfig omcproxy skapi skinit skwifi
  skupgrade mt76config wifi_fw_path wifi_e2p wifi_testmode arht_wlan_cal
  sgmiiconfig voip gpio_switch led prplmesh beerocks* obuspa tr069_*` (+ the
  stock OpenWrt set).
- Vendor config: `/userfs` (per-SoC LED maps, `npu_data.bin`, `CountrySetting`),
  `/config` (prplmesh/beerocks), `/share/prplmesh_platform_db`.
- libc is **musl** (`/lib/libc.so`, 522 KB). All 5062 inodes are `root:root`;
  the only device node is `/dev/console`.

**Consequence:** this is *musl-on-musl* (the easy case the plan predicted), but
the vendor pieces are **scattered through the FHS**, so a clean "keep `/rtk`,
replace the rest" split does not exist. The graft must either (a) carry a
curated subset of vendor files back onto an OpenWrt-25 base, or (b) park the
whole vendor tree in a sidecar and wire the kernel-facing bits into the live FHS.

## Directions evaluated

| | **D1: OpenWrt-25 base + vendor sidecar** (chosen) | D2: stock 21.02 base + package bumps |
|---|---|---|
| "latest OpenWrt userland" | **Yes** (25.12 procd/busybox/LuCI/apk) | No — a patched 21.02 |
| ABI risk | musl 1.1.x→1.2.5 for vendor daemons (musl is ABI-stable across 1.x; the NijikaWrt bet) | low |
| Integration risk | init ordering + glue between OpenWrt procd and vendor stack | high churn, no modern base |
| Rollback | A/B bank (stock bank untouched) | A/B bank |
| Matches siblings | **Yes** (NijikaWrt model) | No |

**Decision: D1.** Keep the **stock 5.4.55 kernel + DTB + all vendor kmods**
untouched (the container already guarantees this); run an OpenWrt 25.12 userland;
bring the vendor hardware stack up as a sidecar. The kernel module ABI is fixed
to 5.4.55 — **we never rebuild the kernel**, so OpenWrt's own 6.12 kmods are
dropped and the vendor 5.4.55 kmods are authoritative.

## Minimal viable graft (the keep/replace matrix)

**Replace with OpenWrt 25.12** (management plane): `init`/procd, busybox, musl
base libs, `ubus`/`ubox`/`uci`/`libubox`, `rpcd`, `uhttpd` + modern LuCI,
`dropbear`, `apk`, `fstools`/`block-mount`, logging, cron.

**Keep from vendor (the hardware plane) — the minimal set:**
1. `/lib/modules/5.4.55/*` (all 165 kmods) + the vendor `/etc/modules.d` and
   `/etc/modules-boot.d` autoload lists (stock ships no `modules.dep`; OpenWrt's
   `kmodloader` resolves dependencies itself).
2. `/lib/firmware/*` vendor blobs (MT7992/MT7991 EEPROMs, EthMD32 DSP, mediatek).
3. Vendor **PON/OMCI/Wi-Fi/VoIP daemons + their `.so` dependencies** (resolve via
   `ldd`/`readelf -d` on the stock binaries; carry only the closure).
4. Vendor **init for the hardware stack**: `skinit pon xponconfig omcproxy skapi
   skwifi mt76config wifi_fw_path wifi_e2p arht_wlan_cal sgmiiconfig voip
   gpio_switch led` (NOT the stock `network`/`firewall`/`dropbear`/`uhttpd` —
   those are the OpenWrt-25 side's job).
5. Vendor **config/data**: `/userfs` (LED maps, `npu_data.bin`, `CountrySetting`),
   `/config`, `/share/prplmesh_platform_db`, the `skapi`/`skyworth_user_config`
   seed, and the `/etc/config/*` the vendor daemons read (`skapi`, `xpon`, `wifi`,
   `voip`, `system` incl. `sshpasswd`).

## Curated graft — IMPLEMENTED (2026-10-27)

`build/build.sh graft` now builds `/vendor` as the **dependency closure** of the
keep-set above instead of the whole stock tree (`graft-full` keeps the old
whole-tree layout as a fallback). Two steps, both static (nothing aarch64 runs):

**1. Closure** — [`build/vendor_closure.py`](../build/vendor_closure.py) +
[`build/vendor-manifest.ini`](../build/vendor-manifest.ini). Seeds are grouped
(`pon`, `wifi`, `voip`, `hw-ui`, `platform`, `isp-mgmt`, `kernel` on by
default; `mesh`, `vendor-gui`, `vpn`, `extras`, `debug-tools`, `factory`,
`chroot-base` off). From the seeds it follows ELF `PT_INTERP`/`DT_NEEDED`
(+RPATH), script/Lua words naming executables, `require`s, absolute-path strings
(including inside **kmods** — see below), and pulls *specific* referenced dirs.
- **Host-owned packages** (procd, netifd, ubox/ubus/uci, base-files, firewall,
  dnsmasq, LuCI, rpcd, uhttpd, dropbear, fstools, …) are resolved through the
  stock **opkg database**: a vendor reference to them does *not* drag the 21.02
  copy in — the OpenWrt-25 host provides it. `DT_NEEDED` libs are always carried.
- **Vendor-extended `base-files`:** Skyworth packed its own glue into
  `base-files` (`xponconfig`, `br_conf.sh`, `clean_up_xpon_intf.sh`,
  `hsgmii_init.sh`, `intf_refresh.sh`, `prodef.sh`, `skBoard`, LED maps,
  `skyworth_*_config`, `/sbin/sysupgrade`); those files are denied only where the
  host ships the same path.
- Report per group/package, unresolved `DT_NEEDED` (**0**), and an `--explain`
  provenance chain for any file.

**2. Wiring** — [`build/wire_vendor.py`](../build/wire_vendor.py). Parking files
under `/vendor` is **not enough**: the stack opens hard-coded absolute paths, some
from the **kernel** (`mt_wifi.ko` reads `/etc/wireless/mediatek/*.dat` and
`/etc/iNIC/tclinux.bin`; `xponconfig` insmods `/lib/modules/5.4.55/*.ko`; daemons
read `/userfs/*`). The first-pass full-`/vendor` graft only wired modules +
firmware, so the MT7992 driver could not have found its profiles. Policy:
link `/<path> -> /vendor/<path>` where the host lacks it (whole dirs when the
host lacks the dir); **copy** vendor `/etc/config/*` (writable on the overlay);
**replace** `/etc/modules.d` + `/etc/modules-boot.d` (the host's 21+3 entries name
the deleted 6.12 kmods → the vendor's 86+12 autoload lists for 5.4.55, so the
host `kmodloader` loads the vendor driver set); `vendor_wins` for `/sbin/wifi`;
**shadow guard** (never link a lib/executable whose basename exists in another
host lib/PATH dir); **hold** behaviour-active dirs (`/lib/upgrade` — host
sysupgrade sources every `*.sh` there —, preinit, hotplug.d, uci-defaults,
board.d, netifd handlers, ubus/rpcd ACLs, account DBs, dropbear keys: stock ships
zero-byte host-key placeholders that would break key generation); host wins on
any other collision. `/etc/rc.d` is never wired — `aero-vendor` stays the single
gated entry point, now running the enabled groups' init scripts from
`/vendor/etc/init.d` in stock START order.

**3. Gate** — [`build/check_rootfs.py`](../build/check_rootfs.py) simulates the
musl loader over the finished tree: every ELF's interpreter + `DT_NEEDED`
resolvable, no dangling `/vendor` links (fatal; negative-tested).

**Result** (curated vs whole-tree `/vendor`, same OpenWrt 25.12.6 base):

| | curated (`graft`) | full (`graft-full`) |
|---|---|---|
| `/vendor` uncompressed | 70.5 MiB / 769 entries (57% of stock 123.4 MiB) | 123.4 MiB |
| squashfs | 26.9 MB | 42.8 MB |
| image / 60 MiB bank | **30.6 MB / 49%** | 46.6 MB / 74% |
| dynamic ELFs, all resolved | 294 | 565 |
| wiring: linked / held / collisions | 244 / 5 / 14 | 805 / 226 / 323 |

The 70 MiB is dominated by what must stay: kmods (33 MiB, `mt_wifi.ko` alone
14 MiB), firmware (7 MiB), `/etc/iNIC` (4 MiB), VoIP, OpenSSL 1.1, the xPON/OMCI
apps. Dropped by default: prplMesh/beerocks (11.5 MiB), the stock LuCI GUI,
strongSwan, gdb/tcpdump/pciids, DLNA/UPnP/FTP/DDNS, the opkg DB.
Reports land in `build/out/reports/` (`vendor-closure-report.md`,
`vendor-wiring-report.md`, `rootfs-loader-check.md`).

### Static findings from the curation (de-risking the boot)

- **Sonames don't collide.** OpenWrt 25.12 ships versioned sonames
  (`libuci.so.20250120`, `libubox.so.20260213`, `libubus.so.20251202`,
  `libnl-tiny.so.1`); vendor daemons link `libuci.so`, `libubox.so.20210516`,
  `libubus.so.20210630`, `libnl-tiny.so`, `libssl.so.1.1`. Both sets coexist in
  `/lib`/`/usr/lib`, so vendor daemons load their **own ABI-matched** libs with no
  `LD_LIBRARY_PATH`. Shared with the host: musl itself, plus stable-soname
  `libgcc_s.so.1`, `libjson-c.so.5`, `libz.so.1`, `libblkid.so.1`,
  `libiwinfo.so.20230701`.
- **nf_tables is built into the stock kernel** (`nft_compat`/`nft_ct`/
  `nf_tables_api` in the image) — fw4 is not blocked; x_tables/iptables also
  built-in, so the vendor's legacy `iptables`/`ebtables` keep working.
- **No devtmpfs in the stock kernel, and none needed:** both 21.02 and 25.12
  procd mount tmpfs on `/dev` (`mode=0755,size=512K`) and coldplug with
  `udevtrigger`. The graft now ships a static `/dev/console` (c 5,1) like stock,
  so init has stdio before that.
- **`/sbin/sysupgrade` collision:** the host's generic armsr sysupgrade wins in
  the FHS; the vendor binary that writes the inactive tclinux bank + flips the
  bootflag is at `/vendor/sbin/sysupgrade` (`ecnt_sys`/`ecnt_mtd` are wired).
  An Aero-PON `platform.sh` is a bring-up item.

## ABI strategy

- **Kernel/kmods:** fixed at 5.4.55 (stock). Non-negotiable — the container keeps
  the stock kernel LZMA + sha1 byte-exact. Vendor kmods load against it as today.
- **musl:** OpenWrt 25.12 ships musl 1.2.5; vendor daemons were built against
  21.02-era musl. musl keeps a stable ABI across 1.x, so vendor binaries are
  expected to run under 1.2.5. Everything *above* libc is already separated by
  soname (see "Static findings"), so musl itself is the remaining shared-ABI
  exposure. If a daemon faults, the fallback is its own vendor musl: the
  vendor `libc.so` is carried in `/vendor/lib` (invoke via
  `/vendor/lib/libc.so --library-path /vendor/lib:/vendor/usr/lib <daemon>`, a
  procd wrapper) — a thin chroot (`chroot-base` group) only if that still fails.
  **Open ABI item — must be proven on a real boot.**
- **ubus wire protocol:** vendor `libubus.so.20210630` clients talk to the host's
  2025 `ubusd` — protocol, not ABI; confirm at boot (skapid registers
  `rtweb.*`/`skapi` objects).
- **Firmware:** kernel `firmware_class` path pointed at `/vendor/lib/firmware`
  (or merged), so request_firmware() finds the vendor EEPROM/DSP blobs.
- **netfilter vs nftables:** keep the **vendor netfilter** kmods + iptables
  userland for the data path the vendor daemons drive; do not force nftables.
  The OpenWrt-25 side manages its own services only.

## Bring-up plan (the heavy build — next, separately-authorised, needs a boot)

1. Flash **`aero-pon-g7220-stockbase.bin`** to the inactive bank first and boot
   it — proves the container is accepted, the kernel+DTB+sha1 path, the HDRs
   size fields, `tclinux_info` carving, and A/B rollback, with the *stock*
   userland (lowest risk). Prove a stock↔stock `ecnt_sys swap bootflag` round
   trip before trusting a userland change.
2. Boot **`aero-pon-g7220-openwrt2512-graft.bin`** (curated): confirm OpenWrt-25
   procd/init comes up on 5.4.55, serial console + LAN, `/dev` on tmpfs +
   coldplug (no devtmpfs — see findings). `aero-pon-g7220-openwrt2512-graft-fullvendor.bin`
   is the fallback if the curated closure turns out to miss something.
3. Confirm `kmodloader` loaded the vendor driver set from the vendor
   `/etc/modules.d` (compare with the stock unit's live `lsmod`) and PON/OMCI/
   MT7992 probe; set `AERO_ENABLE=1` in `/etc/init.d/aero-vendor` to run the
   vendor init chain (xponconfig → … → skapi/voip, stock START order).
4. Bring PON to O5 on fibre, confirm OMCI provisioning / WAN / Wi-Fi 7 / LAN
   switch / VoIP. Bridge status into modern LuCI via the vendor `skapi`/
   `rtweb.system` ubus (no MIB parser needed — config is already UCI+ubus).
5. Port the **held** items that matter (vendor hotplug/netifd handlers, if any
   are needed for WAN), decide the `/etc/config` merge (host vs vendor
   `network`/`wireless`/`firewall` — the config bridge), add an Aero-PON
   `platform.sh`/sysupgrade, and the WebGUI bank-switch toggle.

## On-device flash test (2026-10-27) — see [aero-pon-bringup-findings.md](aero-pon-bringup-findings.md)

First real flash **proved the big bets**: U-Boot accepts the rebuilt FIT, A/B +
HDRs + `tclinux_info` carving work (stockbase ran a full stock boot from bank B),
and **OpenWrt 25.12 procd/init runs on the stock 5.4.55 kernel** (the graft
reached `procd: - ubus -`). Two issues, both procedural, not image defects:
1. **Graft hang** = the shared `rootfs_data` overlay (mtd7) still holds the stock
   21.02 config/passwd/dropbear and gets mounted over the OpenWrt-25 rootfs →
   wipe it (`mtd erase rootfs_data`) before the first graft boot. stockbase
   booted with that same overlay only because its userland matches it.
2. **Curated "verification failed"** = the OEM WebGUI's `validate_firmware_image`
   gate rejecting a non-OEM image → flash via CLI (as stockbase/fullvendor were).

## Open items (carried)

- **musl 1.1.x→1.2.5 ABI** for vendor daemons — narrowed to musl itself (libs
  above it are soname-separated); only a boot confirms it.
- **Vendor init order:** the static START order is now encoded in
  `aero-vendor`; what `skinit`/`skapid` orchestrate at runtime still needs the
  live unit (`lsmod`, `ps`, `logread` of a stock boot).
- **procd service-name clash:** vendor `led` (and any future vendor service named
  like a host one) shares procd's namespace with the host's service.
- **Does OpenWrt-25 procd/fstools run on 5.4.55?** /dev handling verified
  compatible statically; confirm at boot.
- **Closure blind spots:** paths built at runtime (`$var/file`, `sprintf`) can't
  be seen statically; the loader gate proves libraries, not data files. If
  bring-up hits a missing file, add a seed or use `graft-full`.
- **The 60 M bank budget:** stockbase 43.2 MB (69%), curated graft 30.6 MB (49%),
  full graft 46.6 MB (74%).
- **HDRs `+0x0C` checksum** still unidentified (see container doc); not
  boot-verified, left at the stock value.
