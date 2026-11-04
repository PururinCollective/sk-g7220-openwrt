# SK-G7220 — Stage 2 (preferred): keep stock kernel + drivers, swap userland to OpenWrt 25

This **supersedes the "flash full upstream" idea** in [stage2-plan.md](stage2-plan.md)
(which loses GPON). Method modelled on **NijikaWrt** (`dpn-fx3060v-a1-openwrt`): keep the
vendor kernel and its proprietary **PON/OMCI/WiFi/VoIP** drivers, run a modern **OpenWrt
25.12** userland on top, and keep/port the OEM GUI. **Fiber keeps working.**

## Which reference applies

| Project | Approach | Applies to SK-G7220? |
|---|---|---|
| **dpn-fx3060v (NijikaWrt)** | keep vendor kernel+drivers, OpenWrt 25 userland, vendor stack in chroot, config-bridge | **Yes — the model to follow** |
| **ma5671a (Peter-Pon)** | *native* OpenWrt GPON (`omcid` + Lantiq FALC-ON open PON stack, ME/MIB) | **No for PON** — that open stack is **Lantiq-only**; EN7581 has no open PON driver. Useful only as a reference for OMCI/MIB/LuCI-page design |

The decisive fact: EN7581 GPON/OMCI is closed and lives in the **vendor kernel modules +
userland**. Keep them (NijikaWrt way). You cannot replace them with OpenWrt's native omcid
the way the MA5671A (Lantiq) does.

## Why SK-G7220 is an *easier* target than the fx3060v

NijikaWrt's hardest problems come from D-Link's stock being a **Realtek glibc SDK** rootfs.
The SK-G7220's stock is **already OpenWrt 21.02 / musl / UCI** (see
[stage1-auth-findings.md](stage1-auth-findings.md)). So:

| NijikaWrt complexity | SK-G7220 |
|---|---|
| glibc(stock) vs musl(OpenWrt) → full `/rtk` **chroot**, zig-compiled glibc helpers | **musl-on-musl** — likely no chroot, or a thin one; mainly keep the vendor's own lib versions to avoid 21.02→25.12 ABI drift |
| config in a Realtek **MIB** → heavy `omcicli`/`flash`/`diag` + awk bridge | config already in **UCI + ubus** (`skapi`, `rtweb.system`) → bridge is light / partly free |
| stock GUI is **boa** (foreign web server) on 8080 | stock GUI is **LuCI** (customised `skTmCombo` + `MY-TM-skg*`) → reuse or port within LuCI |
| kernel from a proprietary SDK | kernel is **OpenWrt-built 5.4.55** → clean kmods |

## Target architecture

```
stock U-Boot 2014.04 (locked) → FIT "tclinux" (stock kernel 5.4.55) → NEW squashfs
NEW squashfs = OpenWrt 25.12 userland (procd/LuCI/apk)  +  /rtk = vendor 21.02 userland
```
- **Kernel + kmods: stock, untouched** → `xpon`/EN7571 GPON, OMCI, `mt76`/MT7992 Wi-Fi 7,
  switch, VoIP DSP all keep loading.
- **Vendor stack kept** (the current 21.02 rootfs as `/rtk` or a merged lib tree) and
  brought up by its own init (`skinit`, `S13pon`, `S98skapi`, OMCI, wifi, voip) — these
  own PON/WAN/WiFi/VoIP, exactly as today.
- **OpenWrt 25.12 procd owns management:** new LuCI, dropbear/SSH, rpcd, apk packages.
- **Userland target:** build `armsr/armv8` (generic aarch64, kernel-agnostic — NijikaWrt's
  choice) *or* `airoha/an7581` rootfs; in both cases **drop OpenWrt's kernel/kmods** and
  keep the stock kernel. armsr/armv8 is the cleaner "bring-your-own-kernel" base.

### GUI / "move OEM GUI to OpenWrt"
The stock GUI is LuCI already, so two options:
1. **Keep the vendor LuCI** (skTmCombo/skg*) running on the vendor side for PON/OMCI/WAN
   pages (it already talks to `rtweb.system`/`skapi` ubus), add fresh OpenWrt 25 LuCI for
   system management — dual-GUI like NijikaWrt (vendor on a second port, firewalled).
2. **Port the skg* apps** to 25.12 LuCI — harder: LuCI's client moved from Lua CBI (21.02)
   to ucode/JS (25.x); the skg* views (`/usr/lib/lua/luci/.../skTmCombo/*`) would need
   reworking. Start with (1), port incrementally.

### Config bridge
Vendor config is UCI + ubus already. New LuCI can read PON/WAN/WiFi status via the vendor
`skapi`/`rtweb.system` ubus (and OMCI via the on-device OMCI tool — confirm which exists).
No MIB/awk parser layer like NijikaWrt needed.

## Install, A/B, rollback (reuses Stage-1 findings)

- Package: **stock kernel FIT + new squashfs**, wrapped in the vendor **"tclinux" FIT**
  container (magic `sRDH`), flashed to the **inactive bank** (`tclinux_slave`, mtd4-6) with
  `ecnt_mtd`; switch the **dualbob** slot with `ecnt_sys set bootflag`. Active OEM bank and
  **bootloader (mtd0) stay untouched** → guaranteed rollback.
- Two image flavours (NijikaWrt pattern): a **factory** image for the OEM WebGUI's
  Load-Configuration/firmware path (if the container format matches), and a **direct
  `ecnt_mtd`/sysupgrade** flash from the root shell.
- **Recovery:** stock U-Boot console is **locked** (no TFTP), so in-Linux root + the A/B
  slot is the *only* recovery channel. Keep the OEM bank pristine; back up `art` (mtd10,
  per-unit radio calibration) and never erase it; treat any `mtd0` write as brick-class.

## Bring-up checklist (adapted from NijikaWrt docs/PORTING.md)

1. **Full UART log** of stock boot (have it) + of every custom boot during bring-up.
2. **Probe on-device (root shell):** `lsmod` (the exact kmod set + load order), the vendor
   boot tools (`ecnt_sys`, `ecnt_mtd`, `blapi_cmd`, OMCI CLI, `skapi`), `ubinfo -a`, the
   vendor init order (`/etc/init.d` START values), and which libs the PON/OMCI/wifi daemons
   link (`ldd`).
3. **Full mtd backup** to host over SSH (enable LAN SSH first) — all of mtd0–10, verify
   checksums; `art` (mtd10) especially.
4. **Reverse the "tclinux" FIT container**: how the stock kernel + squashfs are packed, load
   addresses, the `sRDH` header, and the dualbob→mtd mapping — so we can rebuild an image
   with our squashfs and the *stock* kernel.
5. **ABI probe:** run a vendor PON/OMCI daemon under an OpenWrt 25.12 musl chroot/sysroot;
   decide "keep vendor libs beside the binaries" vs "thin chroot".
6. **Build** OpenWrt 25.12 (armsr/armv8), drop kmods, add LuCI/rpcd/uhttpd/dropbear;
   assemble `new-squashfs = OpenWrt + /rtk`; wrap with stock kernel into the tclinux FIT.
7. **Flash to the slave bank**, switch bootflag, boot **with rollback ready** (OEM bank
   intact). Confirm on serial: drivers load, PON reaches O5, WAN/IP, Wi-Fi (MT7992), LAN
   switch, VoIP. Verify `bootcount`/dualbob auto-fallback by forcing a bad boot once.
8. **With fibre:** capture an OMCI/PON baseline; confirm the ISP's provisioning (GPON SN,
   PLOAM, VLANs) still matches and internet works.

## Open questions to resolve first
- Exact vendor **tclinux FIT** layout + dualbob slot mapping (step 4).
- musl ABI drift 21.02→25.12 for the vendor daemons (step 5) — chroot vs merged.
- Does OpenWrt 25.12 procd/fstools run on the **5.4.55** kernel? (very likely; confirm the
  min-kernel assumptions — we keep vendor netfilter, not nftables).
- Which **OMCI CLI / ubus** the vendor exposes for the config bridge.
- Reuse vs port of the **skg\*** LuCI apps.

## Suggested project shape
Mirror `dpn-fx3060v-a1-openwrt`: `build.conf`, `scripts/{extract-stock,build-rootfs,
assemble,patch-vendor,mkfw}.sh`, `devices/sk-g7220/device.conf`, a vendor-boot init that
starts the stock stack, a `platform.sh` driving `ecnt_mtd`/`ecnt_sys`, and `-factory` /
`-sysupgrade` outputs. Keep every UART log. Reuse that repo's scripts as templates where the
boot/flash differs (FIT+dualbob here vs UBI+sw_tryactive there).

## Bottom line
- **Viable and the right path** for keeping fiber: stock kernel/drivers + OpenWrt 25 userland.
- **ma5671a's native-OMCI route is not usable** on EN7581 (Lantiq-only open PON).
- **Lower functional risk** than full-upstream (fiber survives); main effort is the tclinux
  image packaging, vendor-stack integration, and the GUI. A real multi-iteration project —
  start with the backups, the container reverse, and an ABI probe before any flash.
