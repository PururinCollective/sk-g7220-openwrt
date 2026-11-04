# WireGuard on the stock 5.4.55 kernel: feasibility and result (2026-11-04)

Question: can WireGuard be added to RemixWrt by building a module for the
stock EcoNet 5.4.55 kernel and `insmod`-ing it at boot?

**Answer: possible, but not with a vanilla kernel tree.** The stock kernel's
core networking structs are not the kernel.org layout. A module built from
linux-5.4.55 + the stock config would read and write the wrong offsets of
every `sk_buff`, `net_device` and socket it touches and corrupt kernel
memory. A working module needs those structs rebuilt to the vendor layout
first (below), each verified against the vendor's own code. Not shipped in
r15; the evidence and the tooling are here for that follow-up.

## What is in place

| Prerequisite | Stock kernel |
|---|---|
| Modules | `CONFIG_MODULES=y`, no `MODVERSIONS`, no `MODULE_SIG`: vermagic only (`5.4.55 SMP mod_unload aarch64`) |
| WireGuard's kernel deps | `NET_UDP_TUNNEL=y`, `DST_CACHE=y`, `GRO_CELLS=y`, IPv6 built in |
| Config | `/proc/config.gz` (IKCONFIG), GCC 10.2.0 |
| Source | wireguard-linux-compat 1.0.20220627 (WireGuard for 3.10-5.5, own crypto) |
| Not there | WireGuard, TUN (`/dev/net/tun`), so no userspace WireGuard either |

## Build environment (reproducible)

- linux-5.4.55.tar.xz (kernel.org, sha256 `cc0b3fd3…bf940`) in `build/dl/`,
  wireguard-linux-compat-1.0.20220627.tar.xz (`19b181e5…61461`).
- gcc 10.2: Bootlin aarch64 glibc 2020.08-1 (the NijikaWrt toolchain).
- flex, bison, m4 unpacked from Ubuntu .debs (`apt-get download`, no root).
- `olddefconfig` of the stock config drops only vendor drivers, plus
  `CONFIG_ZONE_DMA` (a vendor backport: arm64 got it in 5.5); a one-line
  Kconfig symbol in the tree restores it so `enum zone_type` /
  `kmalloc_caches` match.
- Tree in `build/work/kmod/linux-5.4.55`, `modules_prepare` done.

## The ABI check

`tools/kabi/` (run from `build/work/kmod`): the stock kernel image
(`build/stock-parts/kernel.lzma` → `Image.stock`, text_offset 0x88000, so
file offset = vaddr − 0xffffffc010088000) and the live `/proc/kallsyms`
(no KASLR) give each exported function's vendor code; `offmap.py <obj>`
compiles nothing, it aligns our object's functions with the vendor's
instruction by instruction and lists every memory offset that differs.
`skboff.c` / `sizes.c` print the vanilla offsets and sizes.

### struct sk_buff: two inserted blocks

| Field(s) | vanilla | vendor |
|---|---|---|
| next … `mark`, flag bytes (128-131), `skb_iif`, `hash` | 0 … 160 | same |
| *(vendor, 8 bytes)* | — | 164 |
| `inner_protocol` | 164 | 172 |
| `inner_transport_header` … `mac_header` | 166 … 178 | 174 … 186 |
| *(vendor, 420 bytes: EcoNet HW NAT/QDMA block)* | — | 188 |
| `tail`, `end` | 180, 184 | 608, 612 |
| `head`, `data`, `truesize`, `extensions` | 192, 200, 208, 216 | 616, 624, 632, 640 |

Evidence: `skb_put`/`skb_pull`/`skb_trim` (tail/end/head/data),
`skb_scrub_packet` (tstamp 32, refdst 88, nfct 104, flags 128, skb_iif 144,
mark 160 unchanged), `skb_udp_tunnel_segment` (`ldrb [x19,#164]` →
`#172`: inner_ipproto), `netif_skb_features`, `udp_gro_complete`.

### Other structs WireGuard depends on

| Struct | vanilla | vendor | Evidence |
|---|---|---|---|
| `net_device` size | 2112 | 2176 (`alloc_netdev_mqs`: `add w0, w19, #0x89f`) | +8 before 752, +64 from 896 (cacheline-aligned members) |
| `napi_struct` size | 384 | 400 (`gro_cells_init`: `__alloc_percpu(424)`, vanilla gro_cell 408) | +8 before `poll` (40→48), +8 later |
| `struct sock` | 728 | larger | 560→568 … 688→696 (+8), 704→792 (+88) |
| `udp_sock` | 1088 | larger | 936→1024, 1024→1088 (+88) |
| `struct net` | | changed | 488→552 (`ip6_output`) |

WireGuard embeds a `napi_struct` in each peer, uses `netdev_priv()` (=
`sizeof(net_device)`), reads `sk_user_data`, and handles every field of
`sk_buff`: all of them must be right.

## The way to a module (follow-up)

1. Patch `include/linux/skbuff.h` with the two padding blocks above; patch
   `netdevice.h` (napi_struct, net_device), `sock.h`, `udp.h` and
   `net_namespace.h` the same way, bracketing each insert between an
   unchanged and a shifted field with more `offmap.py` runs.
2. Gate: rebuild `net/core/{skbuff,dev,sock,dst_cache,gro_cells}.o`,
   `net/ipv4/{udp,udp_tunnel,ip_output,route}.o`, `net/ipv6/*` and require
   `offmap.py` to report no offset differences in every function that
   aligns.
3. Build wireguard-linux-compat against that tree (vermagic must read
   `5.4.55 SMP mod_unload aarch64`), check its undefined symbols against
   `/proc/kallsyms`, load it by hand with the crash guard of NijikaWrt
   (`/etc/…/wireguard.loading` written and synced before `insmod`, refuse
   on the next boot if it is still there), then wire `wireguard-tools`, the
   firewall rules in the stock iptables, and the LuCI page (NijikaWrt's
   `nijika-wireguard` and `wireguard.js` port over).

## Result (2026-11-04, follow-up session): ABI-correct module built, gated, tunnel works

### Header patch (`tools/kabi/vendor-abi.patch`, against kernel.org 5.4.55)

| Struct | Insert | Bracket (vendor evidence) |
|---|---|---|
| `sk_buff` | 8 B after `mark` (164..171, inside headers_start..end) | vendor `__copy_skb_header` memcpy 128..188 |
| `sk_buff` | 420 B HW NAT block after `headers_end` (188..607); size 648 | `skb_init`: kmem_cache size 0x288, fclones 1304 |
| `sk_buff` HW NAT tag | magic u16 @204, FOE entry @206, CPU reason @208 | `hw_nat.ko` `hwnat_skb_to_foe` |
| `napi_struct` | +8 after `gro_bitmask`, +8 after `dev`; size 400 | vendor `netif_napi_add` stores |
| `net_device` | +8 between `ieee80211_ptr` (736, unchanged) and `dev_addr` (752→760); +8 before `index_hlist` (872→888); `_tx` 896→960; size 2176 | net-sysfs, dev.o |
| `sock` | +8 between `sk_user_data` (640, unchanged) and `sk_memcg` (648→656); +80 between `sk_destruct` (696→704) and `sk_reuseport_cb` (704→792); size 816 | `setup_udp_tunnel_sock`, `__sk_mem_*`, `sock_init_data` |
| `udp_sock` / `inet_sock` | none needed (follows sock; encap_rcv 1048, size 1152) | `setup_udp_tunnel_sock` |
| `ipv6_devconf` | 64 B vendor sysctl `slaac_addr` after `use_oif_addrs_only` (addr_gen_mode 180→244) | vendor `addrconf_sysctl[]` table in the image |
| `neigh_parms` | 64 B vendor sysctl `default_route` before `data_state` (136→200) | vendor neigh sysctl template, neighbour.o |
| `ipv4_devconf`, `struct net`, `ipv6_stub`, `udp_tunnel_sock_cfg`, flowi | unchanged | devinet sysctl table (32 entries, same indices), `ipv6_stub_impl` table in the image |

Correction to the table above: "struct net 488→552" was `inet6_dev.stats.ipv6` (from `ipv6_devconf`), and "sock 560→568" was `sk_destruct` addressed from `sk+0x88`.

### Gate (`build/work/kmod/ana/offmap2.py`, `ana/gate_wg.sh`)

- 35 core objects (net/core skbuff dev sock dst_cache gro_cells net-sysfs rtnetlink neighbour net_namespace flow_dissector dst; net/ipv4 udp udp_tunnel ip_output route devinet arp icmp ip_tunnel_core af_inet fib_frontend; net/ipv6 ip6_udp_tunnel ip6_output route udp addrconf ndisc ip6_icmp af_inet6 ip6_fib mcast; net/netlink af_netlink genetlink; net/ethernet eth): **0 struct-offset differences in every fully aligned function** (~800 functions). One remaining hit is section-relative (`neigh_table_clear`, `.data..read_mostly`+0x548), not a field.
- Every kernel function wireguard.ko imports (144 code symbols, 57 objects): 53 align fully with 0 offset differences, 91 align partially (register allocation / scheduling only) with 0 differences in the aligned runs; 3 benign hits (stack-canary load `current+0x408` written two ways, `__crypto_memneq` loop stride).
- Not patched (WireGuard does not use them): IPv6 multicast source-filter structs (`ifmcaddr6`/`ip6_sf_list`, mcast.o partial), `nf_conn` (+8, `skb_flow_dissect_ct`).
- No vendor code reads `net_device_ops` past 528 B or `rtnl_link_ops` past 200 B (whole-text scan, `ana/opsscan.py`).

### Module

wireguard-linux-compat 1.0.20220627 + `tools/kabi/wireguard-ecnt-hwnat.patch` (built with `KCFLAGS=-DWG_ECNT_HWNAT`): `wg_reset_packet` zeroes the PPE flow-tag magic, so hw_nat cannot bind a tunnelled flow onto the other packet's headers. vermagic `5.4.55 SMP mod_unload aarch64`, all 161 imports present in the stock export table (`__ksymtab_strings`). sha256 `d94166dc…59c2cd`.

### On the device (RemixWrt r15, bank B; insmod from /tmp, `panic_on_oops=1` only during the test)

Peer: `tools/kabi/wgpeer.py`, an independent userspace WireGuard (Python) on the LAN host.
- Handshake initiated by the kernel; kernel public key equals Python's X25519 of the same private key.
- 20/20 pings each way; 20/20 fragmented 4000-byte pings; 5000 MTU-sized (1420) packets flooded at the device, all decrypted, 0 authentication failures (echo replies capped by the vendor's `icmp_ratemask=6169`, which rate-limits echo replies: stock policy).
- Forwarding (temporary FORWARD + MASQUERADE rules): 4000/4000 inner UDP datagrams wg0→br-lan→LAN host and back, exact payloads, nothing but test payloads at the LAN echo (no HW NAT mis-bind).
- `ip link del` + `rmmod` + re-insmod clean, twice; dmesg clean throughout.
- Rekey soak (7.5 min, keepalive 25 s, pings both ways every 0.5-1 s): 4 handshakes (start + rekeys at 125, 245, 365 s, REKEY_AFTER_TIME 120 s), 400/400 device pings and 736/736 peer pings answered, 0 bad packets, one in-flight packet accepted on the previous session; dmesg clean.
- Not tested: IPv6 inside or outside the tunnel, throughput, a real phone client (the QR/config path is checked, not scanned).

## In RemixWrt r16 (Advanced > WireGuard)

Ported from NijikaWrt (DPN-FX3060V, same author), RemixWrt names:

| Piece | File |
|---|---|
| module build | `build/build-kmod.sh` (toolchain sha256, kernel.org tree + `vendor-abi.patch`, stock config read from `stock-parts/kernel.lzma` IKCONFIG, WireGuard + hwnat patch, vermagic and export-table checks) → `build/out/kmod/5.4.55/`; run by `build.sh aero` (`AERO_KMOD=0` skips) |
| image | `build-graft-rootfs.sh`: `AERO_APK_ADD` + `wireguard-tools qrencode` (their 6.12 kmod autoload entries removed), module to `/usr/lib/aero/kmod/5.4.55/`, `S97/K10aero-wireguard` |
| backend | `/usr/libexec/aero/wireguard` (up, retry, down, firewall, status, client, debug, genkey...) |
| service | `/etc/init.d/aero-wireguard` (procd, reload on `aero` changes) |
| crash guard | `/etc/aero/wireguard.loading` written and synced before `insmod`, removed after; left over = no load until `wireguard retry` |
| link | created with the vendor chroot's iproute2 (`/usr/libexec/aero/run /sbin/ip link add ... type wireguard`): busybox ip cannot, and ip-full would pull an elfutils libelf over the vendor-wired one |
| firewall | stock iptables (`aero/run`): INPUT udp port and `-i wg0`, FORWARD in/out, MASQUERADE of the tunnel subnets out of the WAN; re-applied every minute by `aero-housekeep` (`wireguard firewall`), with wg0's policy rules (pref 90, before the stock tables) and routes, which the stock network bring-up flushes after S97 |
| rpcd | `aero` ucode plugin: `wg_status` (read), `wg_keys`, `wg_client` (write; config + QR SVG) |
| LuCI | `view/aero/wireguard.js`, menu Advanced > WireGuard (order 55) |
| config | `aero.wireguard` (off, 51820, 10.7.0.1/24) + `wireguard_peer` sections; `uci-defaults/92-remixwrt-wireguard` adds the section to a kept configuration |

Note for later uci-defaults: rootfs_data (the overlay) survives sysupgrade, so a uci-defaults script that ran once leaves a whiteout and never runs again under the same name, even from a new image. New defaults need a new file (91-remixwrt-sections did not add `aero.wireguard` on the first r16 flash; 92 does).

Checked on the device (2026-11-04, `aero-pon-g7220-r16-sysupgrade.bin` sha256 `37422905…`, sysupgrade over r15 in bank B, settings kept; bank A still r14):
- `aero.wireguard` added by 92-remixwrt-wireguard on upgrade; `wg_status`, `wg_keys` over ubus.
- Enabled from UCI with a peer: private key generated and stored, module loaded through the guard, wg0 created, peer synced, address, policy rule, firewall rules; tunnel to the host test peer: 20/20 and 40/40 pings, 0 bad.
- `wg_client`: client configuration (address, LAN DNS, WAN endpoint, keepalive) and a QR SVG; an invalid section name is refused.
- Crash guard: a left-over `.loading` blocks the load with the message in `status`; `retry` loads and clears it.
- Reboot with WireGuard on: wg0 and its peers up from S97; firewall and policy rules back within ~2.5 min (housekeep); dmesg clean; tunnel works.
- LuCI page and `aero/common.js` as served by the device parse (the page opened logged in: see below).
- Left on the device afterwards: r16, WireGuard off, no peers, no key (the defaults).

Checked by the user (2026-11-04, same r16, uptime 6 h):
- Advanced > WireGuard opened logged in: interface settings and the peer table render, with live status (handshake age, transfer, endpoint) and the QR/Edit/Delete actions.
- A real peer: a MikroTik RouterOS router over the internet, site-to-site (peer AllowedIPs `192.168.89.0/24`). Handshake done; ping from the device (serial console) to `192.168.89.1` answered, 10-26 ms.
- Forwarding both ways, routed (no NAT, TTL 63 = one hop): a LAN host pings `192.168.89.1`, 6/6; the MikroTik pings the LAN host `192.168.0.2`, 9/9, 10-18 ms.
- After ~40 min with the tunnel up: dmesg clean (no oops, no WireGuard warnings), crash guard file gone, handshakes current.
- Still not tested: a sustained TCP/UDP transfer through the tunnel (hw_nat binds TCP/UDP flows, not ICMP; the host test peer covered 4000 forwarded UDP datagrams), throughput, a phone client scanning the QR code, IPv6.
