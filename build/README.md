# Aero-PON image build pipeline (host-side only)

Reproducible pipeline that packs a rootfs into the vendor **"tclinux"** FIT
container the stock EcoNet U-Boot 2014.04 boots. Reuses the **stock 5.4.55
kernel LZMA + stock DTB unchanged** and swaps only the squashfs rootfs — the
NijikaWrt-style rootfs swap that keeps the vendor GPON stack working.

> ⚠ **HOST-SIDE ONLY. Nothing here flashes, writes any MTD, or touches the
> device.** Output is a `.bin` to be flashed later, in a separately-authorised
> step, to the **inactive** A/B bank (stock bank stays intact for rollback).

Spec: [../docs/tclinux-container.md](../docs/tclinux-container.md). Graft design:
[../docs/aero-pon-graft-design.md](../docs/aero-pon-graft-design.md).

## Dependencies

`python3`, `mksquashfs`/`unsquashfs` (squashfs-tools ≥ 4.4), `fakeroot`, `xz`,
`wget`, `sha256sum`. **No `mkimage`/`dtc` needed** — the FIT is built by a
pure-Python v17 FDT serialiser ([lib_fdt.py](lib_fdt.py)) that reproduces the
vendor FIT **byte-for-byte** (verified), so what we emit is exactly the shape
the stock U-Boot already boots.

## Quick start

```bash
AERO_PON_REVISION=r16 build/build.sh aero   # RemixWrt -factory.bin + -sysupgrade.bin
build/build.sh stock                         # stock firmware as -sysupgrade.bin
build/build.sh verify build/out/aero-pon-g7220-r16-factory.bin
```

Subcommands: `extract`, `roundtrip`, `stockbase`, `graft`, `graft-full`, `aero`, `stock`, `verify <img>`
(`all` = extract + roundtrip + stockbase + graft, the stage-2 proofs).
Curated-graft knobs: `VENDOR_GROUPS=pon,wifi,...` (override the manifest defaults).

## What each stage does

| stage | script | output |
|---|---|---|
| **extract** | `extract_stock.py` | splits the live bank A (`backup/nand-20261026/mtd1.bin`) into `stock-parts/{cert-block,hdrs,fit}.bin` + `kernel.lzma` + `fdt.dtb` + `rootfs.squashfs`; verifies DTB==reference, kernel sha1==FIT hash, HDRs math, `tclinux_info`, and a **byte-exact FDT round-trip** |
| **roundtrip** | `assemble.py --verify-stock` | re-feeds the stock squashfs and asserts the rebuilt `.bin` equals the live bank **byte-for-byte** (end-to-end pipeline proof) |
| **stockbase** | `mksquashfs-rootfs.sh` + `assemble.py` | `out/aero-pon-g7220-stockbase.bin` — stock userland repacked; boot-safe baseline, flash this first |
| **graft** | `build-graft-rootfs.sh` (`vendor_closure.py` → `wire_vendor.py` → `check_rootfs.py`) + `assemble.py` | `out/aero-pon-g7220-openwrt2512-graft.bin` — OpenWrt 25.12.6 userland + **curated** vendor closure under `/vendor`, wired into the FHS (**NOT boot-verified**); reports in `out/reports/` |
| **graft-full** | same, `VENDOR_MODE=full` | `out/aero-pon-g7220-openwrt2512-graft-fullvendor.bin` — whole stock tree under `/vendor` (fallback); reports in `out/reports-fullvendor/` |
| **aero** | `build-kmod.sh` + `build-graft-rootfs.sh` (`VENDOR_MODE=full` + `package/`) + `assemble.py` + the target `fwtool` (qemu-user) | `out/aero-pon-g7220-<rev>-factory.bin` (stock web GUI) and `-sysupgrade.bin` (RemixWrt LuCI / `sysupgrade`); copied to `bin/`. OpenWrt packages: `AERO_APK_ADD` added, `AERO_APK_DEL` removed (default: attended sysupgrade and owut, r17: OpenWrt's online upgrade would offer armsr/armv8 builds, never an SK-G7220 image) |
| **stock** | `assemble.py` + the target `fwtool` | `out/sk-g7220-stock-V1.0.0.6-sysupgrade.bin` (this unit's factory firmware, compared byte for byte with the bank backup) and `-V1.0.0.7-sysupgrade.bin` (the official `SK-G7220_V1.0.0.7_20260106.bin`, sha256 checked); metadata strips back to the exact container; copied to `bin/` |
| **verify** | `validate.py` | 28 structural checks + independent `tools/fitparse.py` cross-parse |

## Components

- **`lib_fdt.py`** — FDT v17 reader/writer. Parses the stock FIT into an event
  stream (strings + property-name offsets kept verbatim) so an unmodified tree
  re-serialises byte-exact; `set_prop()` swaps `filesystem@1` data in place.
- **`extract_stock.py`** — disassemble + verify the authoritative stock bank.
- **`assemble.py`** — swap squashfs → re-serialise FIT → rewrite the 3 HDRs size
  fields (`+0x08` payload, `+0x50` kernel, `+0x54` rootfs) → prepend cert block.
- **`mksquashfs-rootfs.sh`** — pack a tree as squashfs v4 / xz / block 262144
  (what the 5.4.55 kernel mounts), under fakeroot, reproducible via
  `SOURCE_DATE_EPOCH`.
- **`build-graft-rootfs.sh`** — assemble the OpenWrt-25 + `/vendor` graft tree
  (`VENDOR_MODE=curated|full`), ship `/dev/console`, install the gated
  `aero-vendor` bring-up scaffold.
- **`vendor_closure.py`** + **`vendor-manifest.ini`** — static dependency
  closure of the vendor stack from grouped seeds (ELF `DT_NEEDED`/interp,
  script/Lua/string refs, kmod path refs); host-owned packages resolved via the
  stock opkg DB; `--explain PATH` prints why a file was pulled.
- **`wire_vendor.py`** — expose `/vendor` at the original absolute paths
  (link / copy `/etc/config` / replace kmod autoload lists / shadow guard /
  hold behaviour-active dirs / host wins on collisions).
- **`check_rootfs.py`** — musl-loader simulation over the final tree (every
  ELF's interpreter + `DT_NEEDED` resolvable, no dangling `/vendor` links);
  fatal gate before packing.
- **`validate.py`** — structural validation against the documented layout.

## Outputs

`bin/` (repository root) holds the current firmware only: `aero` and `stock`
replace their earlier images there and rewrite `bin/SHA256SUMS`; `bin/README.md`
says which file is flashed from where. `out/` keeps the build results:

| file | what |
|---|---|
| `aero-pon-g7220-<rev>-factory.bin`, `-sysupgrade.bin`, `aero-pon-<rev>.squashfs` | RemixWrt (`aero`) |
| `sk-g7220-stock-V1.0.0.6-sysupgrade.bin`, `-V1.0.0.7-sysupgrade.bin` | stock firmware for sysupgrade (`stock`) |
| `aero-pon-g7220-stockbase.bin`, `stockbase.squashfs` | stock 21.02 userland repacked (`stockbase`; HDRs label `AeroPON-sb0.1`) |
| `kmod/5.4.55/` | WireGuard for the stock kernel (`build-kmod.sh`) |
| `reports-aero/` | the last `aero` build's wiring, loader, apk and ucode reports |

`graft` / `graft-full` write `aero-pon-g7220-openwrt2512-graft*.bin` and
`out/reports*/` (stage 2; superseded by `aero`). Earlier revisions are not kept:
2026-11-04 the r1-r15 images, the stage-2 graft and base images and their
squashfs went to the desktop Trash.

## Reproducibility notes

- apk (host, its own loader) writes past fakeroot. A file fakeroot knows that
  apk replaces or deletes leaves a stale entry, and a symlink that reuses the
  inode is then seen with that file's mode (r17: busybox symlinks of the stock
  tree read as regular files, tar "Too many levels of symbolic links").
  `build-graft-rootfs.sh` therefore swaps apk's database for untracked copies
  before apk runs and deletes removed packages' files itself, under fakeroot.

- The FIT is bit-reproducible (deterministic serialiser; stock kernel/DTB fixed).
- Squashfs + graft use `SOURCE_DATE_EPOCH` (default `1700000000`) for stable
  timestamps; `mksquashfs -processors 1` for stable ordering.
- Large intermediate trees go in `build/work/` (overridable via `AERO_WORK`),
  the OpenWrt tarball is cached + sha256-checked in `build/dl/`.

## Not done here (needs a real boot — see graft design doc)

Boot acceptance by U-Boot, the musl 1.1.x→1.2.5 vendor-daemon ABI, the vendor
init bring-up, PON reaching O5, and the HDRs `+0x0C` checksum algorithm.
