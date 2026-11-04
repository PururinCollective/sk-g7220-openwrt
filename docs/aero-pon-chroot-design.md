# Aero-PON stage-3 design: stock stack in a chroot, OpenWrt/LuCI on top

Decided 2026-10-27. Model copied from the `dpn-fx3060v-a1-openwrt` reference
(docs/ARCHITECTURE.md there), adapted for SK-G7220 (EcoNet/Airoha EN7581).

## The split (why it isn't "all or nothing")

The user's concern is correct for the **WAN forwarding path**: EcoNet HWNAT
(`hw_nat.ko`) and the PON WAN are driven by the vendor drivers + userland, not
upstream netfilter/netifd. But two things narrow what actually has to leave
netifd:

- Stock `/etc/init.d/network` (S20) is ordinary netifd and its `setup_switch()`
  is a **no-op** — the internal switch is programmed by the drivers
  (`eth.ko` + `xponconfig`'s modules) and by the bridge/VLAN/macvlan definitions
  in UCI, all of which 25.12 netifd's standard device handlers can create.
- So the **management LAN** (br-lan over the per-port VLANs) is plain L2 and
  safe under host netifd. HWNAT only matters for routed/NAT'd LAN↔WAN flows.

Therefore:

| plane | owner | why |
|---|---|---|
| management LAN (br-lan, LAN IP, DHCP, LuCI reachability) | **host netifd** | plain Linux bridge/VLAN; no HWNAT/PON; lets LuCI come up now |
| fiber/optics bring-up (xpon/bob/phy modules) | preinit hook (host) | kernel-global modules; already proven (`80_aero_hwinit` + an xpon block) |
| PON/OMCI (OLT registration, WAN iface creation) | **chroot** `omci` daemon | wholly vendor; no OSS equivalent |
| WAN forwarding + HWNAT binding | **chroot** stock stack | `portbind`/offload are vendor netifd/driver features |
| WiFi (mt_wifi), VoIP | **chroot** vendor daemons | proprietary |

## Layout

The fullvendor graft already ships the entire stock tree at **`/vendor`** — that
is our `/rtk`. The stock userland is itself OpenWrt 21.02 (procd/netifd/ubus),
so the chroot runs the vendor **daemons** (omci, wifi, voip), not a second procd.

```
/                     OpenWrt 25.12 (musl) — procd, ubus, netifd (LAN only), LuCI
└── vendor/           stock EcoNet rootfs (21.02, glibc?/musl vendor libs) == "/rtk"
    ├── proc sys dev  bind-mounted by /etc/init.d/aero-vendor
    ├── var           own tmpfs; var/config = the MIB (mtd: ubifs/others — TBD)
    └── etc/init.d/*  stock scripts, run selectively in the chroot
```

Path mirrors as needed (kernel worker threads resolve against PID 1's root):
`/userfs` is already in the rootfs; `/lib/firmware`, `/etc/wireless`, `/etc/iNIC`
were wired by `wire_vendor.py`.

## aero-vendor init.d (new; ~S11, before S20 network)

1. bind `/proc /sys /dev /dev/pts` into `/vendor`; `/vendor/var` tmpfs; mount the
   MIB config partition at `/vendor/var/config`.
2. run the stock PON/optics init (`xponconfig`) — or keep its insmods in the
   preinit hook; whichever proves cleaner on-device.
3. start the chrooted vendor daemons through a `run` wrapper (closes the procd
   lock fd, sets the stock PATH/HOME), as procd instances so they respawn:
   `omci` (pon), `skwifi`/mt76, `voip`. Each gated behind an `AERO_*` switch,
   default off until bring-up is validated.
4. host netifd keeps only loopback + the management LAN.

## Stock lines to neutralise in the chroot (parallels the reference's patch table)
- don't let stock re-mount `/` or `/var` (host owns them);
- don't let stock take `/proc/sys/kernel/hotplug` / run `mdev` (procd owns it);
- don't let stock arm `/dev/watchdog` (procd/U-Boot own it);
- move any stock web server off :80 (LuCI owns it); dropbear owns :22.

## Open questions to settle on-device (read-only / reversible)
1. Does `hw_nat.ko` auto-offload once interfaces are up, or does it need the
   vendor `portbind`? (decides whether WAN can ever be plain-netifd or must be
   fully stock.)
2. Does the `omci` daemon register (ONT → O5) and create the `pon`/`nas_vid*`
   WAN interfaces when run on the graft?
3. Which mtd holds the MIB config (`ubifs`/`others`), and does the vendor config
   tooling (`flash`, `omcicli`, `diag`) run on the graft for the LuCI bridge?

## First increment
Bring the **management LAN** up under host netifd (vendor br-lan over
`eth0.5/eth0.2/eth0.3/eth0.4`, `bridge_empty 1`, 192.168.0.1) so **LuCI is
reachable** — then build the chroot for PON/WAN/HWNAT/WiFi/VoIP.
