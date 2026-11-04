# RemixWrt IIDX r20 (beta) — release notes

Images (`bin/`, built in `build/out/`): `aero-pon-g7220-r20-sysupgrade.bin`
(from r19: LuCI System > Backup / Flash Firmware or `sysupgrade`, settings
kept) and `aero-pon-g7220-r20-factory.bin` (from the stock web GUI). r19:
[remixwrt-r19.md](remixwrt-r19.md).

## Changes

- **Splash screen after a LuCI login**: the serial and SSH login banner in the
  browser, with the same animation and the live PON line, then the credit
  "A Pururin Collective Project, part of Open PON Foundation". It covers the
  page for about 3.5 s while the page loads underneath; a click or any key
  skips it. On by default, off with **RemixWrt > Vendor Stack > Web interface
  > Splash screen after login** (`aero.banner.splash`, applies at the next
  login). `aero.banner.animate=never` and the browser's reduced motion setting
  show it without animation. `aero.banner.color` is for terminals only.

A configuration kept from r19 has no `aero.banner.splash`, which counts as on,
so no uci-defaults script is needed. Saving the option at its default removes
it from the configuration (LuCI's Flag), which also counts as on.

## How it works

- `/usr/libexec/aero/banner --layout` prints the tagged lines of
  `banner-layout.sh` (after one line of settings), and rpcd `aero.splash` hands
  them over. The logo and the status line have one source, the banner's.
- `aero/splash.js` draws them as `banner.awk` does in truecolor: same colours,
  frame plan and row functions. **A change to the animation in `banner.awk`
  belongs in `splash.js` too.** All 46 frames were compared cell by cell
  (character, colour, bold) for the four combinations of RX and TX light.
- `aero/splash.ut`, included at the end of LuCI's `header.ut` by the build,
  emits a small script only for a session with the option on. A session id the
  browser has not seen is a login: a reload or a second tab stays quiet. The
  last 8 characters of the id are kept in `localStorage` (`aero-splash`). No
  `localStorage`: no splash.
- It must never stand in the way of the page: an error or no banner removes the
  cover at once, no answer from rpcd removes it after 2.5 s.
- The build now compiles LuCI's `header.ut` and `aero/splash.ut` with the
  image's ucode: a template error there would be a 500 on every page, the login
  page included.

## Other

- **Vendor Stack**: "Changes here apply at the next boot" now sits on the
  Settings section instead of the page, because the splash option applies at
  the next login.
- `tools/luci-mock` runs the splash hook and `aero.splash` from the real script.
  `LAYOUT=1 tools/banner-preview.sh` prints what the splash gets.
