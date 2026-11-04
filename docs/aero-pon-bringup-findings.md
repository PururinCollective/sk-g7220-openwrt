# Aero-PON first on-device flash test — findings (2026-10-27)

First real-hardware flash of the Aero-PON images. UART logs: `putty-stockbase.log`
(stockbase boot), `puttygraft-fullvendor.log` (graft boot attempt),
`putty-bridge_mode.log` / `putty-default_mode.log` (stock reference boots).

## Results

| image | flash | boot |
|---|---|---|
| `aero-pon-g7220-stockbase.bin` | OK (bank B / `tclinux_slave`) | **Full boot** — kernel sha1 OK, 5.4.55 up, Wi-Fi/VoIP/SLIC all came up |
| `aero-pon-g7220-openwrt2512-graft.bin` (curated) | **"verification failed"** | not reached |
| `aero-pon-g7220-openwrt2512-graft-fullvendor.bin` | accepted | **Hang** at `procd: - ubus -` |

## What is now PROVEN on real hardware

- **The container + pipeline are correct.** U-Boot accepted our rebuilt FIT:
  `Verifying Hash Integrity ... sha1+ OK`, DTB OK, `Starting kernel`, 5.4.55 up.
- **A/B + HDRs + `tclinux_info` work.** stockbase ran from bank B (`bootflag==1
  --> booting from second image`), cmdline carried
  `tclinux_info=...,0x259f000` (our rewritten rootfs size), and the runtime split
  carved `kernel_slave`/`rootfs_slave` with rootfs at bank+`0x395068`. Exactly as
  designed.
- **OpenWrt 25.12 userland runs on the stock 5.4.55 kernel.** The graft reached
  `Run /sbin/init`, `init: Console is alive`, kmodloader (modules-boot.d), preinit,
  `mount_root`, overlay switch, `procd: - early - / - ubus -`. procd-on-5.4.55
  is viable (the core NijikaWrt bet).

## Root cause 1 — graft hang: the stale, shared config overlay

`rootfs_data` (mtd7, `0x7880000`, 8 MiB) is a **single overlay shared by BOTH
banks** (it is not per-bank in the DTB partition table). It still contained the
**stock 21.02** overlay: `jffs2dump` of the backup shows 332 entries incl.
`/etc/config/{network,wireless,system,gpon,upnpd,skqos,…}`, `/etc/passwd`,
`/etc/shadow`, `/etc/dropbear/*` host keys.

The graft boot log shows `mount_root: switching to jffs2 overlay` — so OpenWrt-25
mounted that stock overlay on top of its squashfs. Stock 21.02 `/etc/config/*`,
`/etc/passwd`/`shadow` and dropbear keys then **shadow** the OpenWrt-25 versions,
and procd stalls applying incompatible config → hang after `- ubus -`.

**The tell:** stockbase booted fine *with the same overlay* because its userland
is itself stock 21.02 — the overlay is compatible for it. Swap in OpenWrt-25 and
the same overlay is poison. This is the textbook "flashed incompatible firmware
without wiping config" case (OpenWrt `sysupgrade -n`).

Console is not the problem: OpenWrt-25's `/etc/inittab` has `ttyS0::askfirst`,
matching the kernel's `console=ttyS0,115200n8`.

### Fix: wipe the overlay before first graft boot
From the running stock system (or graft failsafe), erase the shared overlay so
the graft boots on pure squashfs + a fresh overlay:

```bash
mtd erase rootfs_data        # clears /overlay (mtd7); OpenWrt reformats it on boot
```

⚠ Because the overlay is **shared**, this also clears the **stock** bank's saved
config (e.g. the `lansshEnable=1` from stage 3). Stock regenerates defaults on
its next boot; the overlay backup is `backup/nand-20261026/mtd7.bin` if you need
to restore it. A permanent Aero-PON needs its **own** overlay (see Follow-ups).

## Root cause 2 — curated "verification failed"

Almost certainly the **OEM WebGUI upgrade gate**, not an image defect. The page
`skgupgrade.js` calls ubus `system validate_firmware_image` and requires
`valid==true && tests.fwtool_signature==true`. That path runs
`fwtool_check_signature` + `fwtool_check_image` + `platform_check_image`
(`lib/upgrade/platform.sh`), which expects the OEM wrapper (fwtool metadata/ucert
trailer, and magic/chip-id at OEM offsets — `platform_check_image` even strips a
512 KiB bootloader prepend and reads chip-id `7581` at `+0x251` of the result).
Our **bank image** layout is `[8 KiB cert][HDRs][FIT]`, which doesn't satisfy that
gate, so the WebGUI rejects it. stage3 §7 flagged exactly this ("does
`validate_firmware_image` reject non-OEM images? → CLI flash").

stockbase and fullvendor booted/flashed, so a **working CLI flash path already
exists on this unit** — it just bypasses the WebGUI gate. The curated image is
almost certainly a valid image that was offered to the WebGUI instead of CLI.

### Fix: flash the graft via CLI (the proven path), not the WebGUI
Use the same CLI method that flashed stockbase + fullvendor. The curated image is
byte-structurally identical in header/cert/HDRs to those (only the squashfs
differs), so if they flashed via CLI, curated will too.

## Retry procedure (curated graft)

1. On the running **stock** system, get the image to the device (SSH/`nc`/tftp) →
   e.g. `/tmp/aero.bin`.
2. Wipe the shared overlay: `mtd erase rootfs_data`.
3. CLI-flash to the **inactive** bank + flip the bootflag with the **same method
   used for stockbase/fullvendor** (vendor `sysupgrade`/`ecnt_mtd` →
   `ecnt_sys swap bootflag`). Do **not** use the WebGUI upgrade page.
4. Capture the full serial log of the boot. Expected past the previous hang:
   `procd: - init -`, service startup, then a `ttyS0` login (press Enter).
5. Roll back any time with `ecnt_sys swap bootflag` (stock bank untouched).

## Update — clean-overlay retry (`putty-rootfs_data.log`, 2026-10-27)

Overlay wiped (`mtd erase rootfs_data` from failsafe), so this boot came up with
`mount_root: no usable overlay filesystem found, using tmpfs overlay` — a clean
overlay. **It still hangs at exactly `procd: - ubus -`.** So the stale overlay
was *not* the hang cause (wiping it is still correct, but it's a separate issue).

The hang is now isolated to a **constant**: the OpenWrt-25.12 base on the stock
5.4.55 kernel reaches `procd: - ubus -` and never prints `- init -`, i.e. procd
is stuck in the UBUS stage → **ubusd is not coming up** (procd blocks waiting for
the ubus socket). Config/overlay differences don't change it; the vendor module
autoload happens later (at `- init -`, never reached), so it's not that either.

A second, *separate* defect showed up and is **fixed**: `uci: Parse error
(section of different type overwrites prior section...)` during board-file
generation. Cause: `wire_vendor.py` had copied the vendor **seed** configs
(`skyworth_user_config`, `skyworth_factory_config`, `tm_*_network_config`, and a
shell script `sk_parseconfig.sh`) into the live `/etc/config`; they carry
`package <other>` headers that redeclare other packages' sections, which
OpenWrt-25's generic `uci` parse rejects. Fix: the whole vendor `/etc/config` is
now **held** under `/vendor` (not placed live); the vendor chain stages what it
needs at bring-up. This is not the hang (the pure base has no vendor config), but
it was polluting every graft boot.

### Probes to localise the `- ubus -` hang

Built two **pure OpenWrt base** images (no vendor graft at all — just the stock
base rootfs, 6.x kmods dropped, a static `/dev/console`), in `build/out/`:

| image | base | size |
|---|---|---|
| `aero-pon-g7220-base-25126.bin` | OpenWrt 25.12.6 | 7.1 MB |
| `aero-pon-g7220-base-23056.bin` | OpenWrt 23.05.6 (kernel-5.15 era, musl 1.2.4) | 6.8 MB |

Diagnostic order (fastest first):

1. **Failsafe shell, no reflash** — the device already drops to a failsafe shell
   (that's where `mtd erase` was run). On the stuck image: reboot → press **f** +
   Enter at the failsafe prompt → root shell (pure squashfs, before procd's ubus
   stage). Then run ubusd by hand and watch for a crash:
   ```sh
   /sbin/ubusd & sleep 1; ls -l /var/run/ubus* 2>&1; ubus list; dmesg | tail -20
   ```
   A segfault / "illegal instruction" / "undefined instruction" in `dmesg` says
   it's a binary/CPU/syscall incompatibility (OpenWrt-25 userland vs 5.4.55),
   which no config change fixes. Also useful: `logread 2>/dev/null | tail`,
   `/sbin/ubusd -h`, `ldd /sbin/ubusd`.
2. **`base-25126`** — pure OpenWrt 25.12. If it boots to a `ttyS0` login, the
   graft content is the problem (I bisect). If it hangs at `- ubus -`, the 25.12
   base is incompatible with 5.4.55.
3. **`base-23056`** — pure OpenWrt 23.05 (much closer to the 5.4-era userland). If
   25.12 hangs but this boots, we rebase the graft on 23.05 and have a working
   OpenWrt-on-stock-kernel system; then push the version up. If this hangs too,
   it's a deeper 5.4.55 incompatibility (next: kernel-config / syscall probe).

## Resolved — `- ubus -` root cause + the ubusd bridge (2026-10-27)

Walked the hang all the way down in the graft's failsafe shell (no reflash).
Chain of evidence:

1. The 25.12 `ubusd` **runs** on 5.4.55 — `ldd` resolves every lib, the musl
   loader executes, it binds `/var/run/ubus/ubus.sock` (it even self-creates the
   `/var/run/ubus` dir) and parks in `epoll_wait` (`wchan=do_epoll_wait`). But a
   client connect never completes (`ubus list` → "Failed to connect"), and
   `dmesg` shows **no crash** (no SIGILL/segfault/trap).
2. `/proc/net/unix` confirms the socket is a healthy listener (`SO_ACCEPTCON`,
   st 01), and `/proc/<pid>/fdinfo` confirms the listen fd **is** in the epoll
   set, registered **edge-triggered** (`events: 80002019` = `EPOLLET|EPOLLIN`).
3. Watching ubusd across a 20 s client connect: its fd set never grows and it
   never leaves `epoll_wait` — **it never `accept()`s**. The connection sits in
   the listen queue; procd (which forks `/sbin/ubusd`, then `ubus_connect(NULL)`
   in a retry loop) waits on it forever → the `- ubus -` stall. This failsafe
   reproduction is faithful to the boot hang.
4. **Not** edge-triggering per se: the stock **21.02 `ubusd` from `/vendor`**,
   run on this same kernel (via the vendor musl loader + `/vendor/lib`),
   registers its listener edge-triggered too (`80002019`) and **does** accept
   (fd set grows, client gets a `status:0` reply).
5. Cross-test pins the broken half:
   - 25.12 ubusd + **vendor** client → **fail** ("Failed to connect")
   - **vendor** ubusd + 25.12 client → **ok** (empty list, rc 0)

   So the **25.12 `ubusd` (server) is the defect**; the 25.12 libubus/client is
   fine, and — crucially — the **ubus wire protocol is compatible** between the
   21.02 ubusd and the 25.12 clients.

Root cause: a **25.12 `ubusd`/libubox-uloop regression** that stops the listen
socket's accept-ready event from waking its epoll loop on this vendor 5.4.55
kernel. Independent of config, overlay and the vendor graft (a pure base hangs
identically — which is why `base-25126`/`base-23056` were never needed to
localise it).

### Fix: run the vendor 21.02 ubusd as the system message bus (no recompile)

Since the vendor ubusd works here and the protocol is compatible, the graft now
ships a `/sbin/ubusd` shim that execs the vendor ubusd on the 25.12 default
socket path:

```sh
#!/bin/sh
mkdir -p /var/run/ubus 2>/dev/null
exec /vendor/lib/ld-musl-aarch64.so.1 --library-path /vendor/lib:/vendor/usr/lib \
     /vendor/sbin/ubusd -s /var/run/ubus/ubus.sock "$@"
```

procd forks this shim → the vendor ubusd serves → every 25.12 daemon connects.
Validated at the socket layer on-device: vendor ubusd on
`/var/run/ubus/ubus.sock` + the default 25.12 `ubus list` → rc 0. Build wiring:
- `build/build-graft-rootfs.sh` installs the shim when `/vendor/sbin/ubusd` and
  the vendor loader are present (`AERO_UBUSD_SWAP=1`, default), keeps the 25.12
  ubusd as `/sbin/ubusd.openwrt`, and records `AERO_PON_UBUSD=` in
  `/etc/aero-pon-release`.
- `build/vendor-manifest.ini` has a new `[aero-ubusd]` group seeding `/sbin/ubusd`
  so the **curated** closure also carries the vendor ubusd + its lib closure
  (fullvendor already includes it).

**Status: built into both grafts, NOT yet boot-verified.** The next boot should
clear `- ubus -` and reach `- init -`. Watch for a *next* stall: netifd/rpcd use
25.12 uloop too, so if the regression is broader than listen-socket accept, a
later uloop-driven step could stall — still real progress, and A/B-recoverable.
Long-term fix: rebuild 25.12 `libubox`/`ubus` (toolchain) once the exact uloop
regression is pinned, then drop the bridge.

## Follow-ups (de-risking the next boot)

- **Dedicated Aero-PON overlay.** The shared single `rootfs_data` makes stock⇄
  Aero-PON dual-boot fight over one config store. Options: give Aero-PON its own
  overlay region, or an auto `jffs2reset` guarded by a firmware stamp in
  `/lib/preinit` (keyed off `/etc/aero-pon-release`; left out for now because an
  untested preinit auto-reset risks a boot loop — but recoverable via A/B).
- **If the graft still hangs at `- ubus -` with a clean overlay**, it's a
  base-userland issue on 5.4.55, not config. Next probes: boot with the console
  shell forced early, try `procd` debug, or test an OpenWrt release whose kernel
  is nearer 5.4 (23.05 = 5.15) to isolate a procd/ubusd-vs-5.4.55 regression.
- **WebGUI-flashable image**: decoded — see the section below. Our images are
  *already* in OEM upgrade format; the only open question is the secure-header
  check. Not needed while CLI flashing works, but it may "just work".

## WebGUI upgrade gate decoded — our image is already OEM-format (2026-10-27)

Pulled apart the stock upgrade path (`/usr/libexec/validate_firmware_image`,
`/lib/upgrade/{fwtool,platform}.sh`, `www/.../skgupgrade.js`) and compared our
images to the OEM `SK-G7220_V1.0.0.7_20260106.bin`.

**The WebGUI flow** (`skgupgrade.js`): upload to `/tmp/firmware.bin` → ubus
`system validate_firmware_image` must return `valid==true` **and**
`tests.fwtool_signature==true` → `sysupgrade --test` → `sysupgrade -u`.

**`validate_firmware_image`** sets `valid = (fwtool_check_signature ok) &&
(fwtool_check_image ok) && (platform_check_image ret 0)`, and
`tests.fwtool_signature = err_to_bool(fwtool_check_signature)`. So:

- **Signature is a free pass** (corrects the earlier "signature required" note in
  root cause 2): there is **no `/usr/bin/ucert`** in the stock rootfs and
  **`REQUIRE_IMAGE_SIGNATURE` is never set to 1**, so `fwtool_check_signature`
  returns 0 for *any* image → `fwtool_signature==true` even unsigned. The OEM
  image itself carries no ucert/fwtool trailer (its tail is zero padding).
- **Device match is a free pass**: no fwtool metadata → `REQUIRE_IMAGE_METADATA`
  unset → returns 0.
- So **`valid` hinges entirely on `platform_check_image`**.

**`platform_check_image`** (this is a **secure-boot** unit — the chip-id `"7581"`
sits at offset **8785 (0x2251)**, the secure path's `GET_CHIPID` offset, not the
non-secure 593): it reads magic `"HDRs"` + chip-id `"7581"` and calls
`/userfs/bin/blapi_cmd system check_secure_tclinux 0 <img>` for the FIP
secure-header offset (`-1` ⇒ "Secure header check fail" ⇒ fail). The
512 KiB-bootloader-strip + `ecnt_mtd writeflash … bootloader` branch fires **only**
when `"HDRs"` is found at offset 512 KiB; our image has `HDRs` at 0x2000, so that
branch never triggers → `validate_firmware_image` / `sysupgrade --test` is
**read-only** on our image (no bootloader write).

**The OEM image is byte-for-byte our bank format** — `[8 KiB cert@0 (FIP/TBBR,
magic `0x12345678`@+4)][HDRs@0x2000][FIT]`, **no** 512 KiB prefix, no fwtool
trailer. Verified our rebuilt bridge images match the OEM at every offset the
checks read: `HDRs@8192`, chip-id `"7581"@8785/8799/22437`, identical cert head
`01 00 64 aa 78 56 34 12 …`. **So there is nothing to "OEM-wrap" — the format
already matches.**

### RESOLVED — the gate is a size/short-read check, not a signature (2026-10-27)

On-device `validate_firmware_image` on three of our images settled it:

| image | file size | cert@0xc0 declared | result |
|---|---:|---:|---|
| `aero-pon-g7220-stockbase.bin`          | 43,206,067 | 42,948,608 | `valid:true` |
| `…-openwrt2512-graft-fullvendor.bin`    | 46,585,267 | 42,948,608 | `valid:true` |
| `…-openwrt2512-graft.bin` (curated, old)| 30,619,059 | 42,948,608 | **`valid:false` "Secure header check fail"** |

All three carry the **identical** cert (copied verbatim from the V1.0.0.6
`mtd1.bin`), so the discriminator is **not** the cert bytes and **not** a signature
over the payload (fullvendor's payload is modified yet passes). The one property
that tracks pass/fail is **total file size**: pass ⇔ `size ≥ ~42.9 MB`.

**Why, confirmed by decompiling the stock binaries** (Ghidra headless, aarch64):

1. `do_blapi_system_check_secure_tclinux` (in `lib/libblapi_system.so`) runs
   `blapi_system_secure_upgrade(0x100, offset=0, path, 4)` — ioctl **`0x40886300`**
   on **`/dev/secure_upgrade`** — then prints
   `"Secure tclinux check %s. FIP header length is %d"`; `platform.sh`'s
   `awk '{print $9}'` reads that length, and `-1` ⇒ fail.
2. The driver is **`lib/modules/5.4.55/secureupgrade.ko`** (not stripped). Its ioctl
   handler, for `0x40886300`:
   - reads the first **0x2000** bytes, checks FIP magic **`0xAA640001`** (= cert
     bytes `01 00 64 aa`);
   - `parse_fip()` walks 40-byte FIP entries and matches the tclinux image UUID
     **`d6d0eea7-fcea-d54b-9782-9934f234b6e4`**, reading its **offset** and **size**;
   - seeks to `offset` and reads **`size`** bytes in 1 MiB chunks —
     **`if (chunk_requested != chunk_read) return -1`** (the short-read bail);
   - SHA-256s the payload and calls `ecnt_arm_smc(img_smc, hash, cert, 0x2000)`
     (TrustZone verify), returning the SMC result.
3. The matched FIP entry (from the verbatim cert) is **offset `0x2000` (8192),
   size `0x28f35b3` (42,939,827)** → it reads bytes `[8192 : 42,948,019]`. Any
   image **shorter than 42,948,019** triggers the short-read → `-1` → fail. That is
   exactly the curated image (30.6 MB). `cert@0xc0` = `0x28f5800` (42,948,608) is the
   FIP end-marker entry (the 2 KiB-aligned image end), a safe pad target.
4. **The SMC verify is not enforced on this unit.** `init_module` probes the key
   slot via `ecnt_arm_smc(0x82000001,"CKEY",1,…)`; `/proc/tc3162/secure_boot_check_key`
   returns *"This key is empty in this device"* when empty. Proof: fullvendor's
   **modified** payload returned `valid:true` — a real signature check would reject
   it. So the only effective gate is the short-read size check. (Caveat: a unit with
   a provisioned key could enforce the SMC and reject a modified payload. This device
   does not.)

**Fix (build invariant, `build/assemble.py`):** pad the final image up to the size
the prepended cert declares at **+0xC0** (`42,948,608` for the V1.0.0.6 cert). The
original bank and the OEM image already zero-pad to exactly this, so
`--verify-stock` stays **byte-exact**, and U-Boot (boots the FIT by totalsize) and
the runtime (mounts squashfs by the HDRs rootfs size) ignore the trailing zeros.
Rebuilt curated image: 42,948,608 B, `sha256=b9d7b67b5f27196c`; fullvendor is
already larger so it is unchanged (pad 0).

**Bottom line: the WebGUI route is OPEN.** Any of our images ≥ 42,948,019 B with the
verbatim cert passes `validate_firmware_image` and can be flashed from the WebGUI
"Update firmware" page (which writes to the inactive bank and toggles the bootflag,
so A/B rollback still protects a bad boot). `fullvendor` is confirmed `valid:true`
on-device; the padded `curated` should behave identically (one more read-only
`validate_firmware_image /tmp/aero.bin` run confirms it).
