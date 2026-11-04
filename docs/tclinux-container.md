# SK-G7220 "tclinux" image container format + rebuild recipe

Reverse-engineered from `SK-G7220_V1.0.0.7_20260106.bin` (43,079,680 B). This is the
image format the stock EcoNet U-Boot 2014.04 boots and the OEM updater writes. Needed to
build Aero-PON images (stock kernel + new OpenWrt squashfs). All offsets are in the file.

Tools (no `dtc`/`u-boot-tools` needed): [tools/fitparse.py](../tools/fitparse.py) — a
pure-Python FDT/FIT enumerator. Extracted DTB: [reference/stock-fdt.dtb](../reference/stock-fdt.dtb),
decoded: [reference/stock-fdt.txt](../reference/stock-fdt.txt).

## Overall layout

| offset | size | region |
|---|---|---|
| `0x00000` | `0x2000` (8 KiB) | **Signed header + X.509 cert chain** (TBBR). Outer magic `0x12345678` @ +4; DER certs at `0x400`/`0x1000`/`0x1800` = *Trusted Key* / *Non-Trusted FW Key* / *Non-Trusted FW Content* (valid 2026-01-06 → 2046). **Not enforced** — efuse key absent ("Secure key does not exist"). |
| `0x02000` | `0x100` (256 B) | **`HDRs` tclinux header** (below) |
| `0x02100` | → EOF | **FIT image** (U-Boot, `totalsize 0x29133fb`, embedded data) |

Inside the FIT, the three payload blobs sit at:

| offset | size | blob |
|---|---|---|
| `0x021FC` | `0x39E2` (14,818) | `fdt@1` data = **DTB** (`d00dfeed`) |
| `0x05C5C` | `0x38F256` (3,732,054) | `kernel@1` data = **LZMA** → 14,553,096 B ARM64 `Image` (`ARMd` magic), load/entry `0x80088000` |
| `0x394FB0` | `0x2580000` (39,321,600) | `filesystem@1` data = **squashfs** (xz, `hsqs`) |

## `HDRs` header @ 0x2000 (256 B)

| off | bytes | value | meaning |
|---|---|---|---|
| +0x00 | `48 44 52 73` | `"HDRs"` | magic |
| +0x04 | `00 01 00 00` | | version / flags |
| +0x08 | `fb 34 91 02` | `0x029134fb` | **payload size** = 256 (HDRs) + FIT `totalsize` |
| +0x0C | `5a 96 63 57` | `0x5763965a` | **checksum** — algorithm TBD; not CRC32/sum32 of the obvious ranges. **Not verified at boot** (U-Boot checks only the FIT kernel sha1); likely a flash-tool/`blapi_cmd` field |
| +0x10 | `"V1.0.0.7\n"` | | sw version (16 B) |
| +0x30 | `"V1.0.0.7\n"` | | sw version #2 (16 B) |
| +0x50 | `56 f2 38 00` | `0x0038f256` | **kernel size** (= FIT `kernel@1` data len) |
| +0x54 | `00 00 58 02` | `0x02580000` | **rootfs size** (= FIT `filesystem@1` data len) |
| +0x5C | `"THN21\n"` | | **model** (SK-G7220 internal code) |

The kernel/rootfs size fields feed `tclinux_info` on the kernel cmdline, which is how the
`kernel`/`rootfs` sub-MTDs (`mtd2`/`mtd3`, `root=/dev/mtdblock3`) are **carved at runtime
inside the `tclinux` partition** — they are *not* in the DTB partition table. **So these two
size fields are boot-critical and must be rewritten when the squashfs size changes** (verify
exactly how U-Boot derives `tclinux_info` — see open items).

## FIT structure (`conf@1`)

```
images {
  fdt@1        type=flat_dt     arch=arm64  compression=none   14,818 B      (no hash)
  kernel@1     type=kernel      arch=arm64  compression=lzma   load/entry 0x80088000
               hash@1 { algo=sha1  value=af4440af62db2896ad93a77bc3f7a3c267bebb9c }
  filesystem@1 type=filesystem  arch=arm64  compression=none   39,321,600 B  (no hash)
}
configurations { default=conf@1; conf@1 = kernel@1 + fdt@1 + filesystem@1 }
```

**Only `kernel@1` is hashed.** `fdt@1` and `filesystem@1` have no hash node, so swapping the
squashfs needs **no FIT hash update**, and the stock kernel + its sha1 stay untouched. U-Boot
verified exactly this at boot ("Verifying Hash Integrity … sha1+ OK").

## DTB facts (the port starting point)

- `compatible = "econet,en7581", "airoha,en7581"`; model `THN21` (from HDRs).
- `bootargs = "root=/dev/mtdblock3 ro console=ttyS0,115200n8 earlycon init=/sbin/init"`
  (U-Boot appends the dynamic args: `tclinux_info`, `ethaddr`, `sky_vid`, serdes…).
- CPU: Cortex-A53 ×4, GIC-v3; `econet,ecnt-npu`; NAND `econet,ecnt-nand@1fa10000`.
- PON: `xpon@1fb64000` (`econet,ecnt-xpon`), `pon_phy@1faf0000`, `xpon_usxgmii@1fa80000`.
- Net: `mdio-bus` (`airoha,arht-mdio`, `phy@13/14/15`); `pcie0/1/2` (MT7992 Wi-Fi on PCIe);
  serdes/pcie/usb PHYs; `i2c` (SFP/optics).
- GPIO LEDs: power, xponStatus, xponLos, internet, voip_hook, wps, usb_0, phy_tx_power_disable,
  phy_tx_ben. Buttons: reset, wps.
- Partition table (`fixed-partitions`):

  | node | offset | size |
  |---|---|---|
  | bootloader@0 | `0x0000000` | `0x080000` (512 K) |
  | tclinux@80000 | `0x0080000` | `0x3c00000` (60 M) — **bank A** |
  | tclinux_slave@3c80000 | `0x3c80000` | `0x3c00000` (60 M) — **bank B** |
  | rootfs_data@7880000 | `0x7880000` | `0x800000` (8 M) — jffs2 overlay |
  | ubifs@8080000 | `0x8080000` | `0x2000000` (32 M) — /apps |
  | others@a080000 | `0xa080000` | `0x200000` (2 M) — dualbob/bootflag |
  | art | (auto) | `0x380000` (3.5 M) — radio calibration |

  `kernel`/`rootfs` are **not** here — carved at runtime from `tclinux_info` (see HDRs).

## Rebuild recipe (Aero-PON image)

Goal: `new tclinux = stock kernel + stock DTB + NEW squashfs` (OpenWrt 25 userland + the
vendor `/rtk` tree), accepted by stock U-Boot and the OEM updater.

1. **Build the new squashfs** (block `262144`; match the stock xz/compression). Budget: the
   `tclinux` bank is 60 M; stock uses ~3.6 M header+kernel + 37.5 M rootfs, so there's headroom.
2. **Reuse the stock kernel + DTB** unchanged:
   - kernel LZMA: `dd skip=0x5C5C count=0x38F256` (→ valid ARM64 Image, load `0x80088000`).
   - DTB: [reference/stock-fdt.dtb](../reference/stock-fdt.dtb) (modify only if adding nodes).
3. **Assemble the FIT** (`mkimage -f aero-pon.its`, FIT version 17) with `kernel@1` (lzma,
   load/entry `0x80088000`, sha1 — mkimage recomputes), `fdt@1` (DTB, none), `filesystem@1`
   (new squashfs, `type=filesystem`, none), `conf@1` default. Mirror the vendor node names
   and `description` strings.
4. **Prepend the `HDRs` header** (256 B): magic `HDRs`, version `00 01 00 00`, payload size =
   `256 + FIT_total`, **kernel size = new `kernel@1` len**, **rootfs size = new squashfs len**,
   model `THN21`, version strings. Checksum: reuse the algorithm once identified, or (if
   unverified) any value — confirm by booting a size-changed image.
5. **Prepend the 8 KiB header+cert block** (`0x0–0x2000`) from the stock image verbatim: the
   certs aren't enforced, and the chip-id string `7581` that `platform_check_image` reads at
   `0x2251` lives in the DTB and is preserved. (Confirm the outer header holds no payload-size
   field needing a fixup.)
6. **Flash** the whole image to the **inactive** bank (`tclinux_slave`) and **flip the active
   bank**, booting with the OEM bank intact for rollback. Two on-device paths (both confirmed
   2026-10-26, see [stage3-ondevice-findings.md](stage3-ondevice-findings.md) §7):
   - **Vendor `sysupgrade`** (what the WebGUI upgrade page drives): the OEM LuCI page
     `skgupgrade.js` uploads via `cgi-upload`, calls ubus `system validate_firmware_image
     {path}`, then ubus `file exec {command:"/sbin/sysupgrade", …}`. The vendor-patched
     `/sbin/sysupgrade` writes the inactive bank **and flips the boot flag itself**. ⚠
     `validate_firmware_image` may reject a non-OEM image — if so, flash from CLI instead.
   - **Manual CLI**: write the inactive bank with `ecnt_mtd`, flip the active bank with
     **`ecnt_sys swap bootflag`** (confirmed command; the flag lives in the `others` mtd =
     mtd9, read by the U-Boot *dualbob* loader), then `reboot`. Roll back with another
     `ecnt_sys swap bootflag`. `ecnt_sys` flags: `reserve, bob/dualbob, wifi_*, bootflag`.

**WebGUI stock⇄Aero-PON switch (NijikaWrt-style):** the stock WebGUI has no bare "flip bank"
button — switching is bundled into a firmware upgrade. A dedicated toggle (like DPN-FX3060V
NijikaWrt) is a small LuCI page that runs `ecnt_sys swap bootflag && reboot`, or reuses the
`file exec` ubus method. Easy to add once Aero-PON is on the inactive bank.

## Open items to verify on-device (before trusting a rebuild)

> **RESOLVED on-device (see [stage3-ondevice-findings.md](stage3-ondevice-findings.md)).**
> `tclinux_info` = 5 values/bank: `[FIT totalsize, kernel-off, kernel-size, rootfs-off,
> rootfs-size]`, offsets expressed as `phys − 0x100`. **kernel-size/rootfs-size equal the
> HDRs `+0x50`/`+0x54` fields verbatim (delta 0)**; a vendor kernel MTD-split registers
> `kernel`/`rootfs`(+`_slave`) from it. Carves verified by md5 against the live device:
> `mtd2 = bank[0:0x2900000]`, `mtd3 = bank[0x395068:end]`. NB: the **live-flashed** HDRs
> values differ from this on-disk `.bin` (different V1.0.0.7 build — same structure); build
> tooling off the live layout. So only the two HDRs size fields (and the FIT lengths) need
> rewriting for a new squashfs; the stock-kernel rootfs offset stays `0x395068`.

1. ~~Exactly how U-Boot derives `tclinux_info`~~ — **resolved above.**
2. **The HDRs checksum algorithm** and whether anything verifies it (`blapi_cmd`, the updater,
   U-Boot). Quickest test: flash a size-changed image to the slave bank and boot.
3. Whether the **8 KiB outer header** has any length/offset field tied to payload size.
4. That `mkimage` reproduces a FIT the stock U-Boot accepts (same v17 layout + `filesystem`
   subimage type).
5. Does OpenWrt 25.12's build produce a squashfs the **5.4.55** kernel mounts (squashfs v4 + xz
   — yes, stock already uses it).

## Build pipeline — IMPLEMENTED (2026-10-26)

Host-side build pipeline in [../build/](../build/) (no `mkimage`/`dtc`; pure-Python
FIT serialiser). See [../build/README.md](../build/README.md). Key results:

- **Byte-exact reassembly proven.** Disassembling the live bank A
  (`backup/nand-20261026/mtd1.bin`) into cert + HDRs + FIT + {kernel, DTB,
  squashfs} and reassembling reproduces the bank **byte-for-byte** (`build/build.sh
  roundtrip`). The Python FDT v17 serialiser reproduces the vendor FIT exactly
  (incl. the 960-byte zero tail libfdt/mkimage leave after the strings block), so
  what we emit is the exact shape U-Boot already boots. An Aero-PON image differs
  from stock **only** in the squashfs bytes + HDRs `+0x08`/`+0x54` + FIT
  totalsize/downstream offsets.
- **Live bank A is `V1.0.0.6`**, not the OEM-distributed `V1.0.0.7` `.bin`. Its
  HDRs/`tclinux_info` match [stage3-ondevice-findings.md](stage3-ondevice-findings.md)
  §3 exactly (payload `0x28f35b3`, kernel `0x38f30d`, rootfs `0x2560000`, cksum
  `0x48df266a`). The pipeline builds off this live bank (authoritative).
- **HDRs `+0x0C` checksum still unidentified.** A sweep of crc32 / adler32 /
  sum32 / 16- & 32-bit word sums (LE/BE, signed) over the obvious ranges (payload,
  FIT-only, HDRs-only, with the cksum dword zeroed/skipped) matches **neither**
  known (image,cksum) pair. Left at the stock value; **not boot-verified** (U-Boot
  checks only the FIT kernel sha1), flag until a size-changed image is booted.
- First images produced + structurally validated (28 checks each):
  `build/out/aero-pon-g7220-stockbase.bin` and
  `build/out/aero-pon-g7220-openwrt2512-graft.bin`. **Unflashed.**
