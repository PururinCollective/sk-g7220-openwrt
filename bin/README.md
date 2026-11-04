# Current firmware (SK-G7220)

Copies of the current images from `build/out/` (`build/build.sh aero` and
`build/build.sh stock` replace their earlier images here). Checksums:
`SHA256SUMS`. Details: `docs/aero-pon-upgrade.md`.

| file | flash it from | what it does |
|---|---|---|
| `aero-pon-g7220-<rev>-sysupgrade.bin` | RemixWrt: LuCI System > Backup / Flash Firmware, or `sysupgrade` | updates RemixWrt in the running bank, settings kept; the other bank is left alone |
| `aero-pon-g7220-<rev>-factory.bin` | the stock web GUI (Management > Upgrade) | installs RemixWrt in the other bank and boots it; the stock bank stays as the fallback |
| `sk-g7220-stock-V1.0.0.6-sysupgrade.bin` | RemixWrt: LuCI / `sysupgrade` | back to stock: this unit's factory firmware V1.0.0.6, byte for byte (from the bank backup), written over the running bank |
| `sk-g7220-stock-V1.0.0.7-sysupgrade.bin` | RemixWrt: LuCI / `sysupgrade` | back to stock: the official update `SK-G7220_V1.0.0.7_20260106.bin`, unchanged, written over the running bank |

The stock images replace RemixWrt in the bank it runs from; the other bank
keeps what it has. To return to RemixWrt afterwards, flash
`aero-pon-g7220-<rev>-factory.bin` in the stock web GUI (the same revision
brings its OpenWrt settings back), or, if the other bank still holds
RemixWrt, switch banks on the serial console: `ecnt_sys swap bootflag; reboot`.

The stock web GUI's own file is the official `SK-G7220_V1.0.0.7_20260106.bin`
in the repository root.
