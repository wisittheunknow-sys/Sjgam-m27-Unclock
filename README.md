# SJGAM M27 — Unclock

Unlock and make the SJGAM M27 more usable.

## Findings

- Remarketing fraud: the chip is marked `RK3568B2`, but it is actually an `RK3188`.
- No Wi-Fi chip, although an antenna is present on the board, probably for wireless PS2-style controllers.
- Console studied as PCB R10 rev 2.3 with an RK3188, 512 MB RAM and Android 4.4 AOSP written entirely to NAND. (Dumps of rev 2.2 are also archived.)
- The stock launcher (`gametemplate.apk`) ships PPSSPP and uses Libretro cores for low-level emulation up to PS1, with native APKs for N64, Dreamcast and PPSSPP.
- DraStic (NDS) is absent, even though the SJGAM frontend refers to it; the NDS category is deliberately disabled.
- The DOSBox core is present but also disabled.
- `com.emu` (RetroArch) is present, but with a minimal quick menu.
- ROM inventory relies on `gamelist.xml` files, compiled by the frontend into `cachegames.db` (SQLite); deleting that database makes the frontend create a new one.
- The right joy-con is mapped together with ABXY.
- The joy-cons are 4-direction buttons, not analog axes: 24 inputs in total (`rk29-keyboard`).
- It is impossible to leave the frontend for Android, and the frontend locks up if the SD card does not contain the expected files.
- ADB is inactive, recovery is blocked by the manufacturer, and loader/maskrom access requires holding VOL- while a USB data cable is connected to a PC.

## Goals

- Enable ADB on the live system.
- Restore Android Settings.
- Restore access to Android and the ability to install APKs.
- Restore a complete, configurable RetroArch quick menu.
- Have a launcher that automatically picks up ROMs added to or removed from the SD card.
- Implement a virtual keyboard navigable with the D-pad.
- Restore NDS and DOSBox emulation.
- Keep the firmware mouse and its OSD, without adding an extra virtual mouse.

## Repository rules

- **No original game data may be versioned.** The player supplies their own copies on the SD card.
- The `ai*.cpp` sources of the M.A.X. port are not modified: the original AI is preserved.
- Do not include a Rockchip loader (unauthorised distribution).
- No binaries, dumps, APKs or secrets (in particular the keystore password) are versioned: only sources, scripts, notes and checksums are tracked.

## Planned layout

- `docs/`: status journals (`STATE.md`), decisions, procedures.
- `tools/`: reproducible scripts (flashing, Mouse OSD rebuild, etc.).
- `kernel/`: configuration and patches for the M38 kernel.
- `frontend/`: frontend sources.
- `max/`: sources of the M.A.X. port (without game data).
- `packaging/`: manifests and procedures (no images or ZIPs).

Currently tracked in the repository: `.gitignore`, `HISTORY.md` and `README.md`.

## Status

- Kernel M38 (build #38) rebuilt and configured from <https://github.com/bbelos/rk3188-kernel/tree/master>: USB gadget added, ADB enabled, right joy-con separated from ABXY, and the reboot-on-first-suspend crash fixed.
- Android 4.4 system preserved (the best compromise for this SoC: fast and efficient).
- `Settings.apk` from a Kingston RK3188 tablet ROM installed and patched because the console's network-connectivity query failed.
- Virtual keyboard implemented and navigable with the D-pad.
- Firmware mouse preserved: `rk29-mouse` + `mouse_daemon` + `com.sjgam.mouseosd`. R1+R2 (~3 s) toggles it and the right stick moves the pointer without being tied to ABXY. No virtual mouse was added to the port, no touch→mouse bridge, and no SELECT-2 s CLIC/TOUCH mode (touch was disabled on 06/11).
- New hybrid frontend (Android/emulation) with ROM refresh at launch and customisation through the `UI` folder on the SD card. The installed build has 7 columns; the firmware archive still has the 6-column build and will be aligned during the final rebuild.
- Complete RetroArch quick menu (SELECT+START).
- Original SJGAM Libretro cores patched (their header made them unusable by RetroArch).
- Original SJGAM APKs kept (Flycast, Mupen64Plus AE) because high-level emulation through cores is too slow; native APKs are required.
- RGB joy-con GPIOs and backlight control integrated into the frontend.
- DraStic installed for NDS and its key mappings configured.
- PPSSPP installed, as it was in the original frontend.
- MX Player added for video and MP3 playback (VLC had problems).
- Test firmware archive: `sjgam-m27-custom-20261124.zip` (not versioned). It will be rebuilt once with the final M.A.X. APK after the remaining user tests.

### Annex — M.A.X. port

Port of <https://github.com/klei1984/max> for Android 4.4 on the SJGAM M27, mainly for fun. The original AI is untouched and no game data is bundled: the player supplies their own copy on SD. The final clean APK is `c6f5a77831898618fd69ccac9dcb2e21`, installed at `/data/app/com.max.port-2.apk` and validated on a new game with 30 clean autosave checks. Details are in `HISTORY.md`.
