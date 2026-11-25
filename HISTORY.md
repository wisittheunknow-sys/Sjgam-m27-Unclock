# History of the SJgam M27 console / Sjgam-m27-Unclock

This document recounts the work carried out on the SJgam M27 console (Android 4.4.2 / API 19, RK3188, 1024×600 screen): ADB/root unlocking, kernel and drivers, firmware mouse, restoration of `Settings.apk` and the virtual keyboard, frontend, emulation, then packaging of a reproducible firmware archive. The M.A.X. port is a side project to the console: it is reported in Appendix A, and its 25/11 diagnosis in Appendix B.

It is deliberately **stripped down**: no original game data, no binaries, no dumps, no ROM images and no secrets are copied into it. Only the paths, dates, decisions, key codes and checksums needed for understanding are kept.

Sources: the work logs `STATE.md` / `STATE.MD` kept on the development machine. The raw logs are not versioned: they contain a local keystore secret. When information was missing or ambiguous in a log, it is flagged as such rather than reconstructed.

## Preservation rules

- No original M.A.X. data may be embedded in the APK or in the pack; the player provides their own copy on SD card (`/mnt/external_sd/MAX`).
- The four `ai*.cpp` sources of the M.A.X. port are not modified: the original enemy AI is preserved.
- Fixed target: Android 4.4 / API 19 + NDK r25c + SDL2 (`v0.7.2`); no ROM ≥ 5.0 for this console.
- Firmware mouse preserved: `rk29-mouse`, `mouse_daemon`, `com.sjgam.mouseosd`. No virtual mouse in the port, no R1+R2 chord added, no touch→mouse bridge, right stick left free.
- `/system/usr/keylayout/rk29-keypad.kl` is touched only as a last resort; reference md5 `29fb5f55eba641d01de7a11fd6120396`.
- No Rockchip loader is distributed; no flashing or installation on the console without explicit confirmation.
- Chosen format: a Rockchip flashable archive containing everything validated, excluding game data. A single final rebuild, after the user tests.
- The M.A.X. port is not the central subject: see Appendix A and Appendix B at the end of the document.

## Overview — milestones

| Date | Milestone |
| --- | --- |
| 05–08/07/2026 | Factory dumps of the partitions and SD card, read-only. |
| 07/10 | First kernel flash (build9), USB ADB root working. |
| 08/10 | Discovery of the real panel timings (600×1024 portrait); clean build13. |
| 10/10 | Root cause of the controls found: `rk29-keypad`, not the USB pad; build17 validates 24 buttons. |
| 11/10 | Joycon LEDs; `Settings.apk` restored by re-enabling the network in `services.jar`. |
| 12/10 | Leanback keyboard navigable with the D-pad: `framework.jar` fixed, larger keys. |
| 12–13/10 | HDMI/RGA, builds M19→M33, software rotation, landscape logo. |
| 13–14/10 | R1+R2 + right stick mouse mode, `mouse_daemon`, Mouse OSD, auto-start. |
| 19/10 | GPU/CPU DVFS measurements, no persistent changes. |
| 23/10–03/11 | SD dumps archived. |
| 30/10 | M38 kernel: fix for the reboot on first sleep; correction of partition numbering (kernel = p2). |
| 01/11 | Hardware remap of A/B/X/Y in the system keylayout. |
| 02–04/11 | ARM32 libretro cores installed and validated (mGBA, Snes9x, MAME 2003 Plus). |
| 06/11 | `sjgam_touchd`, then final disabling of the click→touch bridge. |
| 09–24/11 | SJGAM frontend iterated up to the selected 7-column APKS menu. |
| 20–24/11 | Google/Vending stubs; billing kept for Riptide GP2; 5 stubs disabled. |
| 23/11 | `search` service restored in `services.jar` (MX Player); M.A.X. port in progress (appendix). |
| 22–24/11 | M.A.X. port: JNI/SD paths, stretched 640×480 rendering, mapping, original AI, audio watchdog, French language. |
| 24/11 | Firmware archive `sjgam-m27-custom-20261124.zip`, console cleanup, DraStic mappings, GitHub repository created. |
| 25/11 | Diagnosis, fix and validation of the connector/anti-air overlap in the M.A.X. autosave (details in Appendix B). |

## Part I — Kernel, drivers and mouse (07/10 → 24/11)


Source: `/var/mnt/stockage/sjgamM27/kernelbuild/STATE.md` (3393 lines, read in full). Concise technical notes; no game data or binary content reproduced (only paths and checksums are kept). Points not reported in STATE.md are flagged as such.

### Phase 0 — Background, factory dumps and hardware (partial dates / undated)

- [dumps 05–08/07/2026] `Original memory dumps SjgameM27/system.bin` = reference factory system; `android_image/system.bin` = later dump (root tools added), not usable as a reference; `build.prop` and `framework-res.apk` identical in both.
- [undated in STATE.md] **First console access / ADB root**: not reported. State observed afterwards: root `adbd`, `ro.secure=0`, `ro.debuggable=1`, SELinux Disabled, no `su` binary for apps.
- [undated] dumps/recovery: `android_image/kernel.bin` (factory kernel #1857), `android_image/recovery.bin`, `work/recovery_kernel.img`, `payload1857.bin` / `payload562.bin` (disassembled extracts), `boot_current_20261007_190957.img` (p4).
- [undated] board confirmed = `arch/arm/mach-rk3188/board-rk3188-ds1006h.c` (paths present in `payload1857.bin` and `payload562.bin`; `dmesg_562.txt` traces `--->board-rk3188-ds1006h.c`). The defconfig seed `rk3188_ds1006h_android-4.4` was therefore correct.
- [undated] panel CDO7018010-30A / 2025082701982 (7"); user feedback 07/10: physically damaged by bending during disassembly (haze, streaks), FPC reseated with no effect.
- [undated] Wi-Fi: no onboard chip; user USB dongle identified as **RTL8188ETV `0bda:0179`**, natively supported by `drivers/net/wireless/rkusbwifi/rtl8188eu`; `CONFIG_RTL8188EU=y` option considered but not adopted at this stage.
- [undated] USB controller `0810:0001` "Twin USB JoysticK" (`drivers/hid/hid-pl.c`): 2 interfaces / 2 reports, 2 FFC cables; later identified as a wireless gamepad receiver (neutral traffic is normal when no gamepad is connected).
- [undated] custom ROM platform key (SHA-256 `71:5E:B4:D7:4F:69:A4:26:A5:CC:8E:57:C6:D8:B5:28:CC:49:67:D0:B3:50:20:05:AA:AF:23:AA:EB:50:75:90`) → do not re-sign APKs with `sharedUserId=android.uid.system`.

### 07/10 — First flash, build9 boot, clocks and black-screen cause

- [2026-10-07] recovery flash: p3 = `kernel_MID.img` (build9, re-read md5 `bf5cec5439c5c29f08b25ba11a529f7e`), p4 = factory backup (re-read md5 `5da0f206ca49f13bb5b7165ea73f82c2`), p10 (kpanic) zeroed; backup `boot_current_20261007_190957.img`.
- [2026-10-07] build9 boot validated: USB ADB working (root `adbd`, `/dev/android_adb`, `f_adb` gadget), error-free display chain (primary lcdc1, `load_screen`, win0, backlight, sleep/wake), ATV launcher active on `fb0`, `sys.boot_completed=1`; panel physically dead.
- [2026-10-07] clock/pixclock session: stock #1857 contains neither 528 MHz nor 71 MHz; 1 GHz literal at `0xc0459650`; `rk29fb_screen` offsets confirmed (pixclock@20, hs_tx_clk@88); active PHY dump (prediv=4, fbdiv=187).
- [2026-10-07] cause of the black screen in earlier builds: no MIPI panel in `.config` → `only mipi dsi lcd is supported`; creation of `drivers/video/rockchip/screen/lcd_M27_mipi.c` (MIPI-DSI panel via RK616, 4 lanes, 203 DCS commands from the #1857 disassembly).
- [2026-10-07 22:07:37 CEST] build11: `gpio_set_value` → `gpio_direction_output` in `rk_lcd_init` / `rk_lcd_standby` (GPIO 166 reset, 168 power, already claimed by the battery and `rk_fb_io_init`); Image md5 `8f331cf742719a2383f00a31138f53c7`, container `kernel_M11.img` md5 `aaf7e276cde7ccddfe2d2c564d10de2f`.
- [2026-10-07] factory reference kept: `android_image/kernel.bin` md5 `ed8d06450140e40f768f5e4f089eda5d`.

### 08/10 — Timing revelation, builds 12/13, USB diagnosis

- [2026-10-08] **revelation**: the panel is 600×1024 (portrait), not 1280×800; disassembled stock #1857 `set_lcd_info`: H 60/80/1, V 25/35/1, DCLK 80 MHz, LCDC_ACLK 300 MHz, MIPI HS 560 MHz, 4 lanes, DCLK_POL=1. Builds 11 and earlier used the `ds1006h` timings (1280×800/71 MHz/528 MHz): a real mismatch explaining the black screen.
- [2026-10-08 19:07:48 CEST] build12: exact stock timings in `lcd_M27_mipi.c`; Image md5 `b5bacc5d844da453514f108ef16d9a47`, container `kernel_M12.img` md5 `6f40aba58fe0b6971282729c9ea27bef`; binary check of the `set_lcd_info` immediates identical to stock.
- [2026-10-08 21:10] build13 flashed: `rk_fb.c rk_fb_set_par` patch aligning `xres_virtual` on `ALIGN(600,16)=608` (stride 2432, consistent with HWC/IPP); patched file md5 `9c8b8bd72907434f880da0e1088d7eb6`; Image md5 `3d9ad62f43380426343ea57ece6538e9`, `kernel_M13.img` md5 `90159bafc94c797a2b7346d011b395dd`; p4 backup `boot_current_20261008_211011.img`.
- [2026-10-08] build13 validation: clean, upright logo/loading/launcher, `virtual_size=608,3072`, `stride=2432`, pitch correlation peak at 2432; build13 = reference kernel at the time.
- [2026-10-08 evening] system diagnosis: `connectivity` service absent from ServiceManager (Play licence blocked); noted as a cross-cutting system issue, not a kernel fault.
- [2026-10-08 evening] USB diagnosis: `usb20_host` → `0810:0001` (`hid-pl.c`); `Vendor_0810_Product_0001.kl` keylayout loaded; `getevent` captures on event2/event3 = zero events; frozen HID reports `01/02 80 80 80 80 0f 00 00`; the same OUTPUT `00 00 00 00` send exists in stock #1857 → not a regression.
- [2026-10-08] tool `work/poke_usb_min.c` → `work/poke_usb` (freestanding ARM, `swi` syscalls); first SET_REPORT attempt = EBUSY (`usbfs: process did not claim interface 0`) → fix to be made: `USBDEVFS_CLAIMINTERFACE` before SET_REPORT.

### 09/10 — Stock vs builds A/B, USB gadget hypothesis

- [2026-10-09] major discovery: under the **factory #1857** kernel, all controls work (vol±, navigation, A/B, sticks); builds M10→M13: chip enumerated but no buttons → kernel regression, not hardware.
- [2026-10-09 22:43] reflash p3 = stock `ed8d0645...`, p4 = `5da0f206...`, p10 zero, md5 re-reads OK; backup `boot_current_20261009_224358.img`.
- [2026-10-09] candidate delta = USB/OTG device gadget: build13 with `USB_GADGET`, `USB_G_ANDROID`, `DWC_OTG_DEVICE_ONLY`, `DWC_CONN_EN`; stock without any `android_usb`/gadget string.
- [2026-10-09 23:2x] build14 (kernel #12): `HOST_ONLY` OTG, gadget disabled; Image md5 `59bb1996be6489f53648adbe10547f45`, `kernel_M14.img` md5 `ba4e8d8265907abdfe7c5615c6e59e5e`; gadget strings absent. Erratum: first build14 corrupted by a python script (duplicate config, conflicting choice, `USB20_OTG_EN` prompt) → restoration of `.config.build13.bak` md5 `47aa2cc9...`.
- [2026-10-09] build14 result negative: controls still dead → **gadget/OTG hypothesis refuted**. The gadget may return for ADB.
- [2026-10-09] new lead: `recovery.bin` = same #1857 kernel as p3 but with gadget (recovery ADB) → different factory configs; `extract-ikconfig` fails on stock and recovery (no `IKCFG_ST`).

### 10/10 — Factory GPIO, `rk29-keypad` root cause, joycon LEDs

- [2026-10-10] scan/diff of `gpio_request` in board init: stock `gpio_request(178,"usb_power")` + HIGH output (178 = `RK30_PIN0_PC2`), `gpio169 "led"` + LOW output + `gpio_export`, `gpio264 "sd_det"`, `gpio190 "HDMI"`; build12 had no occurrence of 169/178.
- [2026-10-10] SDMMC1/PB1 lead refuted: `CONFIG_ARCH_RK29` absent and `USE_SDMMC1_FOR_WIFI_DEVELOP_BOARD` off → `set_iomux` never called for SDIO1.
- [2026-10-10] build16: idempotent `M27-FACTORY-GPIO` patch in `machine_rk30_board_init` (169 led LOW, 178 usb_power HIGH + exports); board md5 `f5af0ac7b680e307b454dedfc1dabab3`, Image md5 `7b3d51abd561e7cb3c52035499753c36`, `kernel_M16.img` md5 `6e1ea73e0e3e7e6de949e3cf044ba7c1`.
- [2026-10-10] **root cause found**: `rk29_keys_platform_data` table @`0xc0a77f5c` in the factory kernel = **24 GPIO buttons** (`up/down/left/right`, 4 directions per stick, A/B/X/Y, L1/L2/R1/R2, SELECT/START, vol±). The built-in controls go through `rk29-keypad` (`event0`), not the USB pad. `l1 = GPIO160` = `power_hold`, driven as an output by our board instead of being an input.
- [2026-10-10] live evidence under M16: GPIOs held 160, 164, 170, 202, 205, 207; `rk29_keys` fails wholesale (`goto fail2`) as soon as one `gpio_request` fails; `CONFIG_CT36X_TS=y` held GPIO207 (no touchscreen on the M27, i2c-2 spam).
- [2026-10-10] build17 (kernel #14): exact 24-button table, `POWER_ON_PIN`→`INVALID_GPIO`, `dc_det_pin`/`LCD_CS_PIN`/`LCD_PWR_PIN`→`INVALID_GPIO`, `CT36X_TS` off; board md5 `e41935379251221e78c6100c1597cc97`, Image md5 `b4564eb99fc74dea7f0cf6d140ef9634`, `kernel_M17.img` md5 `87d3561344f1a96b1a28b163ac5f0011`; log `kernelbuild/build17.log`, script `build17.sh`.
- [2026-10-10] M17 flash: p3 = `87d356...`, p4 = `5da0f206...`, p10 zero, re-reads OK; **controls 100% functional** validated by the user (navigation, A/B/X/Y, L/R, sticks, start/select, vol); no `gpio_request` failure, no more CT36X.
- [2026-10-10] LED: `gpio169` = power master of the LED/RGB block (simple on/off; rotating pattern internal to the module). Board patch (md5 `9f3d65cb04908f35a5613d833ceaf7df`): `leds-gpio` "joycon-led", `default_state=ON`, removal of `gpio_request(169)` from the `M27-FACTORY-GPIO` block.
- [2026-10-10 evening] build18 (kernel #15): build13 base + `CT36X_TS` off; `LEDS_TRIGGERS=y` attempt abandoned (NEW symbol `NETFILTER_XT_TARGET_LED` blocks `silentoldconfig`); Image md5 `c49ac04ffca696967e0dc90442ac87a2`, `kernel_M18.img` md5 `3f8a38de847e044fc8b72063e51cf64f`.
- [2026-10-10] `/system/etc/install-recovery.sh` hook: LED section `chmod 666` on `/sys/class/leds/joycon-led/brightness` (root at boot via the init service `flash_recovery`); pushed hook md5 `bb9a76f9164f87da1874844df4f0509b`, console `.bak` backup.
- [2026-10-10 night] M18 flash: p3 = `3f8a38de...`, p4 = `5da0f206...`, p10 zero; `/sys/class/leds/joycon-led` present, `brightness=255`, permissions 666, log `led chmod rc=0 val=255`; ON/OFF toggle validated, RGB rings included.

### 11/10 — LED app, boot hook, system services and virtual keyboard

- [2026-10-11] "SJGAM Control" app (`com.sjgam.control`): LED ON/OFF + blinking + "apply at startup" (original: "appliquer au démarrage"), screen brightness via `WRITE_SETTINGS`; packaging of `classes.dex` at the root of the zip (`INSTALL_FAILED_DEXOPT` trap fixed); installed then promoted to a system app `/system/app/SJGAMControl.apk` (md5 `218818287c0500ec985bdaba42bb5b7f`), `/data/data` data preserved.
- [2026-10-11] streamlined LED hook: no more blinking or "apply" checkbox, state still remembered (`led_on`) and applied from boot (~6 s) by reading prefs in the hook; hook md5 `6f6d554d04facc9ea7f9b620289e07d5`, then `991d65b124452debaa23c130b8b6c428` (prefs logs).
- [2026-10-11] fix for "the app turned the LEDs back on" (original: "l'app rallumait les LED"): `LedService` with no action, always reads the pref; `onResume` resynchronises the Switch and writes only if the hardware state differs; `saveEnabled=false`; `BootReceiver` with no action; ON/OFF cycles validated with `input tap` + reboot.
- [2026-10-11] network restored on the system side: `services.jar` had `config.disable_network=true` and `const/16 v86,0x1` in `ServerThread` → smali patch to `0x0`/`false`, jar pushed back, dalvik cache purged; `connectivity`, `wifi`, `network_management`, `netpolicy`, `netstats` present again.
- [2026-10-11] IME: `LeanbackIme.apk` (`com.google.android.leanback.ime`, source md5 `234d5eeebfbc33763b3690a0d1d592bc`) installed; the cause of the non-navigable keyboard was finally identified on 12/10 (see below).

### 12/10 — D-pad keyboard, HDMI, builds M19→M28

- [2026-10-12] **keyboard root cause**: `ViewRootImpl$ImeInputStage.onProcess` neutralised (stub `return 0`) in `framework.jar` → keys never offered to the IME; restoration of the AOSP KitKat code via `access$700` (`isInLocalFocusMode`); trap fixed: inverted condition `if-eqz` → `if-nez`.
- [2026-10-12] keyboard validation: `KBDTEST`/`KBDPATCH` logs, LEFT/RIGHT move the selection, DPAD_CENTER confirms the key; `framework_clean.jar` md5 `41a4adb67874252b71bf33827b894e56` pushed, logs removed; `LatinIME` removed (backup md5 `68e8367b9290e30413d126dcde1c0c89`), Leanback is the only remaining IME.
- [2026-10-12] keys enlarged by binary patch of `resources.arsc` (`key_height` 28→44 dp, `key_width` 28→60 dp); tool `/tmp/opencode/leanback4/patch_arsc.py`.
- [2026-10-12 evening] striped HDMI: HWC writes 600×1024 stride 608, `lcdc0` mirror reads 1024 px/line (transpose branch) → shearing; temporary fix `ro.sf.hwrotation` 90→0 in `/system/build.prop` (backup `build_prop_live_20261013_portrait.bak`).
- [2026-10-12 20:43] build19 (kernel #16): panel timing reversed to native 1024×600 landscape; Image 7 311 396 bytes, `kernel_M19.img` md5 `e3060a2b875da03ded9576759b1ecee3`; clean TV but rotated image; rotation1 confirmed to come from `libsurfaceflinger.so` via `ro.sf.hwrotation=90`.
- [2026-10-12 ~22:00] `ro.sf.hwrotation=0` applied: TV upright; HDMI sync losses first attributed to I2S audio (volumes set to 0), then to the panel's DSI stream (test `echo 1 > /sys/class/graphics/fb0/blank`: 9 min without loss).
- [2026-10-12 ~23:45] M21 build (kernel #19): exclusive HDMI priority (panel blanked on hotplug, anti-wake guard); `kernel_M21.img` md5 `267bf109498d0c53e4444fa6dbf251c1`. M20 (permanently sleeping panel) never flashed.
- [2026-10-12 ~22:50] M21 test: panel lit with horizontal lines and blanking has no effect (`fbops->fb_blank` short-circuits the backlight notifier).
- [2026-10-12 22:51] M22 build (kernel #20): back to the 600×1024 portrait timing; `rk_fb_blank_panel()` calls `fb_blank()` (fbmem) → `FB_EVENT_BLANK` → backlight off; Image md5 `c822cbd163ba5642edd1da9b58fc4237`, `kernel_M22.img` md5 `606bd7fb7f2fd39ca996ebf7ef47c249`; test: panel proportions OK, off when HDMI plugged in OK, but TV striped and unstable.
- [2026-10-12 23:10] M23 build (kernel #21/#22 after rebuild): consistent mirror (fb2 = exact geometry of fb0, no more transpose, letterbox); Image md5 `36d0428841466a37b8de7505e89fb30f`, `kernel_M23.img` md5 `56ff1cb8d5a99c69b5a30a82e63de1f1`; first flash did not take (boot #20), second attempt OK (boot #22 23:04:30).
- [2026-10-12 23:15] M23 test: TV clear but content rotated 90°, `mOverrideDisplayInfo rotation=1` forced by the ATV launcher; rk3188_lcdc has no hardware rotation.
- [2026-10-12] M26/M27/M28 builds: attempts to "lock" native landscape (MADCTL `0x36=0x20`, then CASET/PASET window `0x2A/0x2B`); md5 `kernel_M26.img 3d53d9e98eb64a9d408faf4b1eacad79`, `kernel_M27.img 467671b5d690915c2009de3d6ac7b963`, `kernel_M28.img c4884d78e9bff21b8a1a9efb9bc188f1`; all produce bands/streaks.

### 13/10 — M25/M29/M31/M32, firmware mouse, auto-start, recovery

- [2026-10-13 00:05] "all native landscape" decision abandoned (M24/M27/M28 in bands); back to the M23 portrait + RGA rotation plan on the HDMI side.
- [2026-10-13] M25 build (kernel #28): `rk_fb.c` + `dma_alloc_coherent` of a 1024×600 rotbuf, `m25_rga_rotate_to_landscape()` (RGA 90° CCW, `sina=-65536`), HDMI hotplug → full-screen landscape mirror; `kernel_M25.img` md5 `2728bee87ba7a30ba73d74b6ed5750e7`, Image md5 `bf28d4a8ce19a1f55251e9c71bea6956`; flashed, **clean panel** validated; logo/loading still rotated.
- [2026-10-13] map of the 24 `rk29-keypad` inputs completed: right joycon = `abs_Y/A/X/B` (codes 288/289/290/291) and ABXY buttons = `key_*` (same codes) → indistinguishable; no stick axis reported; `/dev/mem` = Bad address, no `CONFIG_DEBUG_GPIO`.
- [2026-10-13 14:12] M29 build (kernel #29): CW rotation of the `logo_linux_clut224.ppm` logo (formula `fb_rotate_logo_cw`, 480×480, 222 colours), `.c` regenerated by `pnmtologo`, zero kernel code; `kernel_M29.img` md5 `2316d6256e91d6ec6d1be4f38d62eb6b`; binary check: only 22 lines outside the logo area differ from M25.
- [2026-10-13 14:2x] M30 build (CCW logo fallback): `kernel_M30.img` md5 `3d74635bd248e07087d3e0aa39a6bddf`, not needed.
- [2026-10-13 17:52] M29 flash: p4 backup `boot_current_20261013_175239.img` md5 `5da0f206...`, p3 re-read `2316d625...`, p4 re-read `5da0f206...`, p10 zero; **boot logo upright in landscape validated**; current base = `build29.sh`.
- [2026-10-13 18:41] `ro.sf.hwrotation=90` in `/system/build.prop` (single line, md5 `65a16bdb3ccedd938aedbeadcf57e744`): flat loading and ATV directly in landscape; ramdisk route abandoned after corruption of the first 1024 bytes of p4 (`boot_hwrot90.bin` md5 `24e87b9a2c3381f0e2922468b9040310`, forensic `boot_p4_corrupt_20261013.img` md5 `c16300c303b7cd58ba8a3d37a5c997af`, restoration `boot_current_20261013_182907.img`).
- [2026-10-13] M31 build (kernel #31): recompute `vir_stride` in the M25 branch of `pan_display`; `kernel_M31.img` md5 `ce2fcb478a907cf198314ba600144e76`; user-validated ("panel, logo and HDMI clean and full-screen" (original: "dalle, logo et HDMI propres et plein écran")).
- [2026-10-13] M32 build (kernel #32, diagnostic): lcdc `BUS_ERR`/`WIN0_EMPTY` printk, sysfs attribute `hdmi_rga`, backlight printk; `kernel_M32.img` md5 `d4f8e518e6a0c2167e8f82deddbd3d0c`; Image md5 `99db0faab22d0c376554c1f75f4f1eb9`.
- [2026-10-13 20:08] M32 flash: p4 backup `boot_current_20261013_200856.img` md5 `5da0f206...`, p3 re-read `d4f8e518...`, p4 re-read `5da0f206...`, p10 zero.
- [2026-10-13] M32 tests: backlight actually off (`eff=0`), no `BUS_ERR`, `hdmi_rga=0` changes nothing → **RGA, audio, DMA and backlight ruled out**; the "jump" = external HDMI link (HDMI→VGA adapter / old monitor), replugged then stable in direct HDMI; source tree restored to M31 (`*.M32diag.bak` backups).
- [2026-10-13 21:48] M33 build (kernel #33): `key_button[]` remap to make the sticks distinguishable (left 183–186, right 187–190); board md5 `5231ca28a67aeaf12b82a4f78a1c4d64`, Image md5 `10da2ffb454188952e3f5029f74db836`, `kernel_M33.img` md5 `7e2a00cd753a605c5b09f54a1233e1be`; p3 flash OK, backup `boot_current_20261013_214755.img` (md5 `5da0f206...`).
- [2026-10-13] physical R1/R2 confirmed: R1 = 295 (`BTN_BASE2`), R2 = 293 (`BTN_PINKIE`); factory keylayout inverted → corrected without flashing.
- [2026-10-13] **mouse mode validated**: toggle by holding R1+R2 ≥ 3 s, right stick = movement, R1 = left click, R2 = right click; `mouse_daemon` userspace reads `event0` and creates a `uinput` mouse (`CONFIG_INPUT_UINPUT=y`); source `mouse_daemon/mouse_daemon.c` (v10), `.arm` binary md5 `51f03bb517873e46a3b10722d274bb03`; `POINTER_KEEPALIVE=1` (alternating +1/−1 px / 2 s).
- [2026-10-13] keylayout `/system/usr/keylayout/rk29-keypad.kl` (md5 `57edc0c05745fc5c7d5b352dbde29bb8`): 183–186 → `DPAD_*`, A(290) → `DPAD_CENTER`, B(289) → `BACK`, X/Y unmapped, R1/R2 corrected.
- [2026-10-13] Mouse OSD `com.sjgam.mouseosd`: polls `/dev/sjgam_mouse` (0666) and shows a "Mouse Mode ON/OFF" Toast, autostart OK.
- [2026-10-13] cursor: SurfaceFlinger "Sprite" layer, survives `UI_DEV_DESTROY`; patch of `/system/lib/libinputservice.so` md5 `a6b2466483cd3b9b4c8d28bc27cb8546` (original `f537f6f472379ba407110ae26d94e568`): `fadePointer` neutralised at `0x26758`, idle timer 15 s → 3 s at `0x31d78`; device and PC backups.

### 14/10 — Mouse auto-start, umask, bootloader/recovery, system inventory

- [2026-10-14] auto-start: `install-recovery.sh` hook v3 (md5 `caa2e6a1de23428f81603a0b9833e7f4`) ends with `exec /system/bin/sjgam_mouse.sh` (md5 `fff9645ffd367c5d2b5876d0e2c0477c`); supervisor waits for `/dev/uinput` and `event0`, restarts the daemon 3 s after a crash, log `/data/local/tmp/mouse_daemon_autostart.log`; verified `ppid=1`, `init.svc.flash_recovery=running`.
- [2026-10-14] umask fix: `/dev/sjgam_mouse` created 0600 by init (`umask 077`) → OSD unable to read it; added `umask 0` in `sjgam_mouse.sh` (new md5 `02d6a27432503e707a4b8d7cdcfa1e49`); validated "perfect, it works great" (original: "impec ca fonctionne nikel").
- [2026-10-14] "no Toast after reboot" incident (original: "pas de Toast après reboot") = false alarm: toggle performed 3 s before the OSD started (`BOOT_COMPLETED`); deliberate `last=-1` logic; final validation after the UI appeared.
- [2026-10-14 evening] discovery: Vol- + USB + ON = **Rockchip loader mode** (`2207:310b`), neither recovery nor Mask ROM; `rkdeveloptool db` fails ("device does not support"), direct `rl/wl` as root work (LBA = 512-byte sectors); SAMSUNG eMMC 3776 MB / 7 733 248 sectors.
- [2026-10-14 evening] factory recovery flash via `flash_recovery_rockchip.sh` (sudo): LBA `0x12000`, length `0xE000`; p4 before `1dbf15db7d5811e8b74581b3bf03502b` (our recovery v3), after `7aac2177533953c012b61de748d55f9d`; backup `recovery_p4_avant_rockchip_20261014_205715.img`, parameter `parameter_device_lu_20261014.bin`.
- [2026-10-14 evening] **recovery entry found**: hook v3 writes `boot-recovery` at offset 0 of `misc` (`print -n 'boot-recovery' > $BCB`), read by UBoot on the next boot; `RECOVER_KEY` commented out in `parameter`; factory recovery waits for a key then times out → automatic return to the system.
- [2026-10-14] recovery v4 prepared (not flashed): full factory image + `socket adbd stream 660 system system` in the adbd service; `work/recovery_sjgam_v4.img` md5 `99f5716960baa0e8948de67792ac7a02`, script `work/mk_recovery_v4.py`.
- [2026-10-14] by-name numbering in normal boot fixed: `misc=p1 kernel=p2 boot=p3 recovery=p4 backup=p5 cache=p6 userdata=p7 metadata=p8 kpanic=p9 system=p10 user=p11` (under `/dev/block/platform/emmc/by-name/`).
- [2026-10-14 night] comparison with the factory system: removal of the 6 GApps, then AOSP leftovers (`CalendarProvider`, `WallpaperCropper`, `DownloadProviderUi`); no kernel impact.

### 19/10 — GPU/CPU clocks (measurements, no kernel modification)

- [2026-10-19] GPU: `/proc/clocks` `aclk_gpu` = 132 MHz idle, 445.5 MHz under GL load (table max), back to 297 then 132 → **GPU DVFS functional**; the earlier note "DVFS unregulated" (original: "DVFS non régulé") was inaccurate (it is the voltage that cannot be controlled, only a "dummy" regulator is exposed).
- [2026-10-19] CPU: 1/2/3/4-thread saturation → 1200/1200/1200/1008 MHz; `temp_limit_freq` follows the number of busy cores; 1416 MHz unreachable (ACT8846 rails fixed at 1.0 V for `vdd_cpu`, DVFS table requiring 1.225–1.3 V); no cpufreq transition (`total_trans=0`).
- [2026-10-19] no persistent modification; `retroarch-core-options.cfg` restored (md5 `e35d00a6603c02a126adb483b6f67d86`).

### 30/10 — M38 build: reboot-on-screen-off fix

- [2026-10-30] symptom: automatic reboot on first sleep; `/proc/last_kmsg`: NULL pointer dereference @`0x4`, `PC=sram_gpio_set_value`, `LR=act8846_device_suspend+0x20` (Oops 5).
- [2026-10-30] cause: on PCB r10v2.3 the i2c-1 bus is silent (0x5a/0x51 without ack) → `act8846_probe()` exits before `act8846_set_init()`, `pmic_vsel.base=NULL` → panic on suspend; the PMIC is only used for voltage setting and the power button (fixed rails).
- [2026-10-30] patch `arch/arm/mach-rk30/board-pmu-act8846.c`: `if (!pmic_vsel.base) return;` at the start of `act8846_device_suspend()` **and** `act8846_device_resume()`.
- [2026-10-30] **tools/compiler**: incremental kbuild impossible (`cmd_bounds` rewrites `include/generated/bounds.h` on every `make` → everything is recompiled; GCC 16 breaks on old sources). Workaround: recompilation of `board-rk3188-ds1006h.o` only via `arm-eabi-4.6` wrapper + GCC 16.2.1 (`-std=gnu89 -Os`), then manual link chain `/tmp/opencode/link38.sh` (built-in.o → `.tmp_vmlinux1` → `scripts/kallsyms` → `.tmp_kallsyms2` → `vmlinux` → `System.map` checked with `cmp` → `Image`); `NM=` exported for `scripts/mksysmap`; `init/version.o` recompiled after forcing `include/generated/compile.h` to `#38`. No NDK documented in STATE.md for this build.
- [2026-10-30] Image 7 311 396 bytes md5 `8adeb20a...`; `kernel_M38.img` 7 311 408 bytes md5 `7f3b10641e5149d7c9dd6764247fe63f`; banner `Linux version 3.0.36+ #38 SMP PREEMPT Fri Oct 30 21:34:50 CET 2026`.
- [2026-10-30] **major flash-script correction**: `TOOLS/flash_p3.sh` and `TOOLS/restore_good.sh` are wrong for this board (they write the kernel to p3, the boot to p4 and zero the start of p10 = system). The real kernel is `mmcblk0p2` (12 MB); p3 = boot ramdisk (KRNL + gzip → cpio), p4 = recovery boot.img (`ANDROID!`). Proof: p2 payload byte-identical to the M33 Image; p3 payload decompressed into cpio (`070701`).
- [2026-10-30] real flash from Android root, without recovery: `busybox dd if=kernel_M38.img of=/dev/block/mmcblk0p2 bs=65536`, md5 re-read OK; rollback `kernel_p2_M33_avant_M38_20261030.img` (12 MB, md5 `3b7634b2...`).
- [2026-10-30] validation: boot #38 OK (~25 s), ADB OK; test trap = USB cable plugged in → `usb_pcd` wakelock prevents deep sleep (empty `suspend_time`); valid test with **cable unplugged**: sleep without panic and wake by any game button (the power button goes through the absent PMIC); `hym8563` RTC inert (no scheduled wake).
- [2026-10-30] final settings: `screen_off_timeout=180000`, `stay_on_while_plugged_in=7`; `pm_test=none`; patch present in the tree but **not committed**.

### 01/11 — Gamepad A/B remap (keylayout)

- [2026-11-01] discovery: the physical A/B/X/Y buttons emit kernel codes **188/190/189/187** (`abs_A/B/X/Y`), not 288–291; the earlier validation via `sendevent(290)` was a false positive; L1/L2/R1/R2 = 294/292/293/295, SELECT/START = 296/297.
- [2026-11-01] final `/system/usr/keylayout/rk29-keypad.kl` (1077 bytes, md5 `e52538eccb02774919b1ff29de460474`): 188 → `DPAD_CENTER` (A=OK), 190 → `BACK` (B=Back), 189 → `BUTTON_X`, 187 → `BUTTON_Y`, 288–291 → F5–F8 (unused), 292/293/294/295 → `BUTTON_L2/R1/L1/R2`, 296/297 → `BUTTON_SELECT/START`; 183–186 remain `DPAD_*`. Reboot required (Android 4.4, no hot reload).
- [2026-11-01] emulator configs adjusted to the new mapping (RetroArch autoconfig/global, DraStic prefs, PPSSPP inis); physical tests valid; backups in `/tmp/opencode/kl/backup_20261101/`.

### 06/11 — Virtual mouse restoration, then `sjgam_touchd`, then touchscreen disabling

- [2026-11-06] `start_touch` re-enabled in `/system/bin/sjgam_mouse.sh` (md5 `944eda69d702f87f6ba4b53d554da7c5`) after isolating the right joycon; hot restart via `setprop ctl.stop/start flash_recovery` (do not launch from adb: the process does not survive the shell closing).
- [2026-11-06] validation: `init → sjgam_mouse.sh → sjgam_touch_daemon.sh (event4 "rk29-mouse") → mouse_daemon (event0)`; `/dev/sjgam_mouse=1`, log `tap (520,285)` = R1 click translated into a tap at the cursor; `/system` back to ro.
- [2026-11-06 (continued)] native translator `sjgam_touchd` (C, direct uinput, ~ms) replaces the shell (`getevent`+`input tap`, ~0,69 s): `/system/bin/sjgam_touchd` md5 `2d64a01671502cab95d22572c373f462`, `sjgam_mouse.sh` md5 `9d561446d24e2cf9cb2e38daccead8f1`, `sjgam_touch_daemon.sh` fallback kept; project `/var/mnt/stockage/sjgamM27/sjgam_touchd/`.
- [2026-11-06] uinput device "sjgam-touch": `BUS_VIRTUAL`, ABS_X 0..1023, ABS_Y 0..599, ABS_PRESSURE, `BTN_TOUCH`, `INPUT_PROP_DIRECT`; Android Sources `0x00001002`, DeviceType touchScreen.
- [2026-11-06] mandatory calibration (portrait panel 600×1024 rotated to 1024×600): `x_brut=(599-y_ecran)*1024/600`, `y_brut=x_ecran*600/1024`; verified (100,100)→(99.0,100.8), (900,500)→(899.4,501.0).
- [2026-11-06] behaviour: R1 click in ON mode = DOWN/UP, R1 held + right stick = drag & drop, R2 alone = right click without touch, R1+R2 chord < 400 ms = cancellation, OFF mode = no injection; logs `/data/local/tmp/sjgam_touchd.log` (auto-rotation at 512 KB).
- [2026-11-06 evening] **user decision: mouse click only, touchscreen disabled** (the tap interfered with validation in some menus, needed elsewhere): `TOUCH=0` in `sjgam_mouse.sh` (md5 `41f705f19843127c18ec58fe3c8197fe`); `sjgam_touchd` remains installed but inactive (md5 `ead02066e3b537693e5f3cdf408306e2`); final state: `mouse_daemon` alone, no "sjgam-touch" device, `/dev/sjgam_mouse=0`, `/system` ro. Reactivation: `TOUCH=1` + `ctl.stop/start flash_recovery`.
- [2026-11-06] note: `APK2SD` block of `install-recovery.sh` located after `exec`/`exit 0` → never reached (to be handled separately).

### 20/11 — R1+R2 release fix (spurious click)

- [2026-11-20] after an ON/OFF toggle, the first finger lifted produced a spurious click (left, or right = Back): `sync_buttons()` only inhibited while `r1 && r2` were held down.
- [2026-11-20] fix: `toggle_armed=0` as an exit lock, clicks ignored until both touches are fully released + `RELEASE_GUARD_MS=120` guard; log "chord lock: clic parasite ignore" (verbatim log string); source reconstructed `kernelbuild/mouse_daemon/mouse_daemon.c` (v10 + fix), old binary saved to `/mnt/internal_sd/mouse_daemon.pre-release-lock`.
- [2026-11-20] rebuild with the GCC 16 wrapper (GCC 4.6 unavailable): 5444-byte binary instead of 37278; **user validation "everything works great" (original: "tout marche nikel")**, no rollback.

### 24/11 — Firmware packaging inventory, cleanup and final checks

- [2026-11-24] active kernel confirmed: `/proc/version` = `3.0.36+ (wisit@bazzite) #38 SMP PREEMPT Fri Oct 30 21:34:50 CET 2026`.
- [2026-11-24] reference image `kernelbuild/kernel_M38.img` 7 311 408 bytes md5 `7f3b10641e5149d7c9dd6764247fe63f` (copied into `PACKAGING/stage/`); config `rk3188-kernel/.config` md5 `2dcf5b0d272d9d30426584fc739eb27b`.
- [2026-11-24] boot partition `mmcblk0p3` md5 `5da0f206ca49f13bb5b7165ea73f82c2` (= factory dump); ramdisk `/init.rc` contains `service flash_recovery /system/etc/install-recovery.sh` (line 576): customisation is in `/system`, not in the boot.
- [2026-11-24] parameter `SPARE/parameter_device_lu_20261014.bin` md5 `db875b3140e427d9b9caf453d5564b76`; `build.prop` md5 `65a16bdb3ccedd938aedbeadcf57e744`.
- [2026-11-24] mouse: `/system/bin/mouse_daemon` deployed md5 `5fbeaecc3805a2a6de8c863b6916c631` (source + `build.sh` included in the stage); keylayout `rk29-keypad.kl` md5 `29fb5f55eba641d01de7a11fd6120396`; `rk29-mouse.kl` / `Vendor_1234_Product_5678.kl` md5 `e539a9a9eccba38a195d0616e50e4ba8`; `Vendor_0810_Product_0001.kl` md5 `034fdb78933fcff9be2765fc63770c4c`.
- [2026-11-24] flashable archive `PACKAGING/sjgam-m27-custom-20261124.zip`: `kernel.img` = `kernel_M38.img` padded with zeros to 12 582 912 bytes, archive md5 `4f85e77197d10c722355fa7cfec632c1`; factory `boot.img` and current `recovery.img` (`7aac2177...`) included as-is; `misc.img` captured but not written; LBA in `flash.sh`: kernel `0x4000/0x6000`, boot `0xa000/0x8000`, recovery `0x12000/0xe000`, system `0x282000/0x180000`; no loader distributed, no flash performed.
- [2026-11-24] console cleanup: backup `DUMPS & SAVES/console-cleanup-20261124/` (135 files, 25 MB), 125 temporary files deleted (`/data/local/tmp` 24,7 MB → 1,5 MB); active configs/logs kept (`sjgam_mouse_speed`, `sjgam_touchd.pid/.log`, `mouse_daemon.log`, `mouse_daemon_autostart.log`, `usbmark_hook`, …); kernel #38, keylayout and system binaries unchanged.
- [2026-11-24] keylayout/mouse preserved during the latest UI and emulation tweaks: no system file, keylayout or binary modified; keylayout remained at `29fb5f55...`; "firmware mouse (rk29-mouse / mouse_daemon / Mouse OSD) left intact" (original: "souris firmware (rk29-mouse / mouse_daemon / Mouse OSD) laissée intacte") (wording in the log); Mouse OSD is sufficient for on-screen presses, no virtual mouse added.

### Undocumented or ambiguous points in STATE.md

- First console access (ADB/root), initial acquisition of root and the entire partition-recovery phase before 07/10 are not reported: only the later state is documented.
- NDK/compiler tools for the M38 build: only the `arm-eabi-4.6` wrapper + GCC 16.2.1 is documented; no NDK is mentioned for the kernel (the NDK references in the file concern app/core builds, not the kernel).
- "right stick = firmware mouse" (original: "stick droit = souris firmware"): STATE.md describes **no dedicated firmware** for the right stick. The documented implementation is (a) M33 kernel remap of GPIO codes, (b) userspace `mouse_daemon` + `uinput` daemon, (c) OSD. The USB receiver `0810:0001` is driven by `hid-pl.c`, identically to stock. The exact meaning of the term "firmware" in the request remains to be clarified.
- Partition numbering: STATE.md reports a discrepancy between recovery and normal boot for the historical `flash_p3.sh` / `flash_p3` scripts; the definitive correction (kernel = p2) is only recorded in the M38 section (30/10). Earlier flashes (M17→M33) are described using the recovery convention, and the exact target cannot be confirmed retrospectively from this file alone.
- Build12 dates (19:07:48 CEST) and a few M14/M21 builds are deduced from their position in the log; the year is not explicitly repeated on every line (2026 from context).
- No game data or binary content was deliberately included; only paths, key codes, device types and checksums needed for the technical history were kept.

## Part II — Android system: `Settings.apk` and virtual keyboard (11/10 → 12/10, continued 23/11)

These two workstreams are at the heart of the console: they make the system genuinely usable without a touchscreen. They concern neither the M.A.X. port nor games.

### `Settings.apk`: reimplementation / restoration (11/10, then 23/11)

- **Symptom**: the Kingston `Settings.apk`, placed in `/system/priv-app`, installed fine (`codePath=/system/priv-app/Settings.apk`, `PRIVILEGED` flags) but crashed as soon as it was opened.
- **logcat diagnosis**: `NullPointerException` in `com.android.settings.Settings.updateHeaderList(Settings.java:587)`. `ServiceManager.getService("network_management")` returned `null`, then the call to `INetworkManagementService.isBandwidthControlEnabled()` raised the exception (the `try/catch` covered only `RemoteException`, not the NPE).
- **Root cause**: in `/system/framework/services.jar`, the class `com.android.server.ServerThread` (method `initAndLoop`) had networking hard-disabled by the "root" ROM: `SystemProperties.set("config.disable_network", "true")` and `const/16 v86, 0x1` (`disableNetwork=true`). The corresponding branch skipped the entire network block: `NetworkManagement`, `NetworkStats`, `NetworkPolicy`, Wi-Fi P2P, Wi-Fi, `Connectivity`, Ethernet and `Nsd` were never started. The original Kingston ROM read `config.disable_network` with `false` as the default, hence the difference in behaviour.
- **Fix applied**: decompilation of `services.jar` with baksmali 3.0.10, patch of `ServerThread.smali` (`0x1` → `0x0` and string `"true"` → `"false"`), reassembly of `classes.dex`, replacement in a copy of the jar, push to `/system/framework/services.jar` (permissions 644, `root:root`), purge of the dalvik dex cache, then reboot. The bootclasspath jar is not signature-verified, which makes the patch possible.
- **Result**: after reboot, `network_management`, `connectivity`, `netpolicy`, `netstats`, `wifi`, ethernet and `servicediscovery` are registered again. `Settings.apk` is back in its **original, unmodified Kingston version**; `am start com.android.settings/.Settings` displays the activity in ~636 ms, with no `FATAL`.
- **Signature constraint**: the ROM platform key is custom (SHA-256 `71:5E:B4:D7:4F:69:A4:26:A5:CC:8E:57:C6:D8:B5:28:CC:49:67:D0:B3:50:20:05:AA:AF:23:AA:EB:50:75:90`), different from AOSP `test-keys`. Any application with `sharedUserId=android.uid.system` (Settings, SystemUI, Keyguard…) must remain **unre-signed**: otherwise `PackageManager` rejects the package with `Signature mismatch for shared user android.uid.system`. A re-signed `Settings_v1_aligned.apk` attempt was rejected; the original APK was kept.
- **Follow-up on 23/11**: MX Player crashed due to the missing `search` service. The `SearchManager` / `ISearchManager` (`framework.jar`) and `SearchManagerService` (`services.jar`) classes already existed; only the registration in `ServerThread` was missing. smali patch (`ServiceManager.addService("search", new SearchManagerService(mContext))` after the Clipboard block, with `try/catch reportWtf`), reassembly with `--api 19`, jar push and cache purge. Verification: `service list` shows `search`, `dumpsys search` sees MX Player, no `FATAL EXCEPTION` during the search test. Backups: `BACKUPS/2026-11-23-search-service/` on PC and `/mnt/internal_sd/services.jar.pre-search` on the console.
- **Vocabulary note**: here, "reimplementation" (original: "réimplémentation") means restoring the original APK to service and repairing the system framework that prevented it from working — not rewriting the Settings application. No application code was rewritten.

### Virtual keyboard: from display to D-pad navigation (11/10 → 12/10)

- **Initial state**: the ROM had no working keyboard. `default_input_method` pointed to `com.ndcsolution.androidtv.leankeykeyboard/.LeanbackImeService`, which was not installed (APK removed during debloating): no text input possible.
- **First workaround**: installation of `LatinIME.apk` in `/system/app`, `ime enable` + `ime set`, `enabled_input_methods` corrected. The keyboard appeared and responded (`mCurMethodId=LatinIME`, `mHaveConnect=true`), verified by framebuffer capture. But this IME is not navigable with the D-pad, and the console has no touchscreen.
- **Chosen option**: `com.google.android.leanback.ime` v1.0.2.2423292 (minSdk 16), service `com.google.leanback.ime.LeanbackImeService`, installed in `/system/app/LeanbackIme.apk` (source md5 `234d5eeebfbc33763b3690a0d1d592bc`). `com.android.inputmethod.leanback` (min API 35) is unusable on API 19 (`Requires newer sdk version #35`). `LatinIME` is kept as a temporary fallback.
- **Documented dead end**: the IME window carries `FLAG_NOT_FOCUSABLE` (`fl=#1800108`). A first hypothesis attributed the lack of navigation to this flag. A smali patch of `LeanbackImeService` adding `makeWindowFocusable()` (`clearFlags(0x8)` + `requestFocus`) did remove the flag (`fl=#1800100`) but broke the IME state (`mHasSurface=false`, `isReadyForDisplay=false`, `mInputShown=false`): the keyboard no longer appeared. The original APK was restored and the hypothesis discarded.
- **Actual root cause (12/10)**: in the ROM's `framework.jar`, the method `android.view.ViewRootImpl$ImeInputStage.onProcess(QueuedInputEvent)I` was **neutralised**: it only did `return 0` (`FORWARD`). Keys were therefore never offered to the IME, regardless of the keyboard installed. All the surrounding machinery was sound (`dispatchInputEvent`, `ImeInputEventReceiver`, `IInputMethodSessionWrapper`, `onFinishedInputEvent`).
- **Proof**: temporary instrumentation of `Leanback.onKeyDown` with a `KBDTEST` log — never triggered before the patch, triggered on every key after.
- **Fix**: restoration of the AOSP KitKat logic of `onProcess` in `framework.jar` via smali: if `mLastWasImTarget && !isInLocalFocusMode()`, call `InputMethodManager.peekInstance().dispatchInputEvent(...)`; `FINISH_HANDLED` (1), `FINISH_NOT_HANDLED` (2), `DEFER` (3), otherwise `FORWARD` (0). Since `isInLocalFocusMode()` is private, the synthetic accessor `ViewRootImpl.access$700()` is used. Trap encountered and fixed: the first version tested `if-eqz` (inverted condition), resulting in a permanent branch to `FORWARD`; corrected to `if-nez`.
- **Functional validation**: with the instrumented APK, `KBDPATCH` logs show `res=IN_PROGRESS` (DEFER), then live reading of the `EditText` confirms `LEFT` then `CENTER` → `r`, `RIGHT` then `CENTER` → `t`: the arrow moves the selection and the centre confirms the key. The physical gamepad is consistent: `rk29-keypad` maps A to `BUTTON_A` (96), and Leanback's `getSimplifiedKey()` already maps `BUTTON_A` → `DPAD_CENTER` and `BUTTON_B` → `BACK` — no key remapping was needed.
- **Capture trap**: the keyboard does not appear in `/dev/graphics/fb0`: the IME window is composited as a hardware overlay on RK3188 (`mShownFrame=[0,366][1024,600]`, `layer=21010`) while the framebuffer only contains the application layer. Visual validation of the keyboard therefore cannot be done by `fb0` capture on this ROM.
- **Finishing**: enlarged keys without `apktool`, by binary patch of `resources.arsc` (type `dimen`, encoding `dp*256 | 0x01`): `key_height` 28→44 dp and `key_width` 28→60 dp. The IME window goes from 234 to 330 px in height and the keyboard from about 428 to 780 px in width. A right margin is kept for the "Next" button (do not exceed ~70 dp).
- **Cleanup**: removal of temporary `KBDPATCH` logs from the framework and `KBDTEST` from the IME. Clean `framework_clean.jar` md5 `41a4adb67874252b71bf33827b894e56` is pushed to the console. `LatinIME.apk` is deleted (backup md5 `68e8367b9290e30413d126dcde1c0c89`), its dex cache and data purged: Leanback becomes the only active and default IME.
- **Final state verified after cold reboot**: `sys.boot_completed=1`, clean `framework.jar`, package `com.android.inputmethod.latin` absent, `default_input_method = enabled_input_methods = Leanback`. Persistent text entry validated (text entered then extended via `DPAD_CENTER` after reboot). The pre-existing `NullPointerException` in `com.google.process.location` (location hard-disabled) remains unchanged and unrelated to the keyboard.
- **Vocabulary note**: the IME was not rewritten from scratch. The work consisted of repairing the framework (re-enabling key dispatch), then patching the existing IME at the level of its resources and smali code, with systematic restores of the original versions.

## Part III — Frontend, emulation, stubs and packaging (10/10 → 24/11)

### Initial context, goals and preservation rules (undocumented date; document current to 2026-10-27)
- Hardware: RK3188, 512 MB RAM, non-touch 7" screen (1024×600 landscape), 24 physical inputs seen by Android as a keyboard, 3000 mAh battery, SD slot, ON/OFF switch cutting the battery, no original Wi-Fi/Bluetooth/ADB/recovery.
- Two consoles: console 1 with NAND, screen damaged during disassembly, kernel without USB gadget, spare-parts donor; console 2 requiring SD boot, kernel with USB gadget but empty `adbd` folder, chosen as the work base.
- Original state: locked-down SJGAM Android 4.4 AOSP, proprietary frontend in the foreground encompassing the interface, RetroArch and PPSSPP, a few native emulators; Android inaccessible, poor quick menu, non-dynamic roms (deleting a `.DB` at the SD root required).
- Goals: open system with APK installation from SD, custom MediaBox-style launcher (platform icons, rom list on the left, PNG preview on the right with default image), emulation as broad as the RK3188 allows, acceptable performance, R1+R2 virtual mouse, independent right joycon, APK2SD, persistent libretro configs, lightweight system and firmware flashable via Rockchip tools.
- Doubt: the R1+R2 chord duration is given as 3 s in the project document and 5 s in the very first notes; 3 s is the value adopted afterwards.
- Strict rule: original dumps of both consoles' partitions and SD images = read-only, never altered; extract into a folder if needed.

### First ADB/root access, dumps and transfers (2026-10-07 → 2026-11-03)
- Undocumented date for the very first ADB/root access; earliest dated evidence: 2026-10-07, boot of the build9 kernel, root `adbd`, `/dev/android_adb` and `f_adb` gadget operational.
- 2026-10-07: recovery flash p3 (build9 kernel), p4 backed up then rewritten identically (md5 `5da0f206ca49f13bb5b7165ea73f82c2`), p10 zeroed, reboot left to the user.
- 2026-10-07 → 08: several successive builds for display/panel; build13 validated on 08/10 (clean logo and launcher), then kept as a rollback reference.
- 2026-10-10: discovery that the 24 controls go through `rk29-keypad` (GPIO), not the USB gamepad; complete input mapping established and keylayouts corrected without another flash.
- 2026-10-10 → 11: joycon LED/RGB controllable via gpio169 and `SJGAM Control` app created, anglicised then promoted to a system app `/system/app/SJGAMControl.apk` (md5 `218818287c0500ec985bdaba42bb5b7f`), data preserved.
- 2026-10-11: batch of 7 original system APKs (Kingston) transferred via `adb push` to `/system/app` and `/system/priv-app` (MediaProvider, CalendarProvider, DownloadProvider, WallpaperCropper, DownloadProviderUi, DocumentsUI, LatinIME), local md5 = console md5, reboot and `pm path` verification.
- 2026-10-19: mouse-click → touchscreen-press translator installed (`/system/bin/sjgam_touch_daemon.sh`) because Unity games ignore `BTN_LEFT`.
- 2026-10-23: console 2 dumps saved under `DUMPS & SAVES/dumps originaux m27 pcb r10 v2.2/` (including `system_adb_20261023.img`, `boot_adb_patch_20261023.img`, flash logs).
- 2026-10-26: complete raw image of the factory SD card (52,1 GB) saved with SHA-256 and a zstd copy in `DUMPS & SAVES/factory sdcard m27 r10v2.2/`, read-only.
- 2026-10-27: chosen architecture = development on console 2 (SD boot, ADB, iterations without brick risk), NAND migration only at the end of the project; `/data` nearly full (~11,7 MB free), the SD remains the data medium.
- 2026-11-03: image `DUMPS & SAVES/factory sdcard M27 r10v2.3.img` archived.
- Note: kernel build/flash details are covered in the separate kernel history.

### Virtual mouse and associated firmware (2026-10-13 → 2026-11-21)
- 2026-10-13: "mouse via right stick" project: holding R1+R2 toggles the mode, right stick = movement, R1 = left click, R2 = right click, userspace daemon via `/dev/uinput`.
- 2026-10-13: M33 gives dedicated codes to the sticks (183..190) to distinguish right stick from ABXY; physical R1 = 295, R2 = 293, factory keylayout corrected without flashing.
- 2026-10-13/14: first `/data/local/tmp/mouse_daemon` daemon validated; `com.sjgam.mouseosd` OSD shows ON/OFF toasts; `umask 0` fix in `/system/bin/sjgam_mouse.sh` so `/dev/sjgam_mouse` is readable by the OSD.
- 2026-10-14: user validation "perfect, it works great" (original: "impec ça fonctionne nikel"); the missing toast after reboot was a timing issue (toggle before the BOOT_COMPLETED service started), not a bug.
- 2026-11-06: click→tap translator restored then replaced by `sjgam_touchd` in C (direct uinput, drag & drop, short chord cancelled); final decision: mouse click only, touch disabled (the tap disrupted MediaBox and the DraStic menu).
- 2026-11-20: the daemon's right click converted to `KEYCODE_BACK` by the Rockchip framework; attempt to patch `android.policy.jar` abandoned, framework restored to original (`android.policy.jar.orig`, md5 `ff2c543f...`) and reboot.
- 2026-11-20: OSD fixed (inverted labels, toast on first poll) then archived as `WORKING/SJGAMMouseOSD-mode-fixed.apk`.
- 2026-11-21: smooth movement (10 ms loop, sub-pixel accumulator); RK3188 uinput fix (a single `input_event` per `write()`, 3 writes REL_X/REL_Y/SYN_REPORT), user validation "oh yes! now it's beautiful, it's clean!" (original: "ha oui ! la c'est beau, c'est propre !").
- 2026-11-21: `/system/bin/mouse_daemon` binary md5 `5fbeaecc3805a2a6de8c863b6916c631`, live setting `/data/local/tmp/sjgam_mouse_speed` (factor 1/128, bounds 32..320), backup `/mnt/internal_sd/mouse_daemon.pre-smooth`.
- 2026-11-21: OSD toasts switched to English by an in-place DEX patch (equal sizes, DEX signature/checksum recalculated, same debug key), `WORKING/SJGAMMouseOsd-i18n.apk` md5 `927623418e6ec2dc413e42e220726579`.
- 2026-11-23: START+SELECT 3 s combo grafted onto `sjgam_touchd` to force-stop the foreground app, with OSD toast `App stopped: <pkg>`; binary md5 `0c1784a2e446abf59f1c5effb9343380`, APK md5 `7dc0df3a847c975746acd66a7bd0cc1e`.
- 2026-11-24: the firmware mouse (rk29-mouse + `mouse_daemon` + Mouse OSD) is confirmed as the only mechanism: no chord in the MAX port and no touch bridge added.

### libretro cores and GBA/mGBA milestone (2026-11-02 → 2026-11-04)
- 2026-11-02: mGBA sources available, Android NDK r15c build (API 19, armeabi-v7a) prepared as a fallback.
- 2026-11-03: official buildbot cores (NDK r29, API 21) fail on Android 4.4 (`__strncpy_chk2` etc.); SJGAM cores from the SD are not encrypted, only the first 32 bytes of the ELF header are replaced and can be reconstructed.
- 2026-11-03: recovered SJGAM mGBA core installed by default in `/data/data/com.retroarch.ra32/cores/mgba_libretro_android.so` (md5 `790f7ef47f732fd96a2012c8cdc01a51`); r15c fallback md5 `db27aec1f45d4d67362b3a665d0cc352`.
- 2026-11-03: user validation on 4 GBA roms (no modification of dumps/roms), 240×160 geometry, OpenSL audio active, ~83 MB RSS; SJGAM Snes9x (md5 `e1d8df49fa53fdc15166c172ed69bc6a`) also validated in-game.
- 2026-11-03: `com.emu` (the original target of the SJGAM frontend) remains stuck before display; do not spend time on it, go through `com.retroarch.ra32`.
- 2026-11-04: 23 useful ARM32 cores installed in the private RetroArch folder (UID 10042, modes 755), ~126 MiB, MD5 identical to the original SJGAM cores; in-game validation limited to mGBA, Snes9x and MAME 2003 Plus, the others remaining statically verified.
- Doubt: the two recovered Desmume cores are AArch64 and incompatible with the M27.

### SJGAM frontend and APKS menu (2026-11-02 → 2026-11-24)
- 2026-11-02: hybrid frontend (MediaBox base + rom browser) chosen, preserving the platform scanner and launchers.
- 2026-11-09: v0.2: home screen redone as a 1280×720 mosaic adapted to 1024×600, RetroArch/My Apps/Settings/Total Commander/Clean RAM tiles, platform banner from `/mnt/external_sd/iroms`; v0.3: switch to `com.retroarch.ra32` + internal cores; v0.4: carousel-style strip.
- 2026-11-12 → 13: v0.5/0.6 (recent games, fixed scan, thumbnails), then v0.7 → 0.36 (missing platforms, UI theme, case-insensitive icons, per-platform backgrounds, strip and LAST GAMES settings).
- 2026-11-15: v0.37 → 0.95: visual styling (DVD case, transparencies, shadows, tilted cover art, `NoPreview.png`), GBA renaming, direct DraStic intent (v0.83, `ACTION_VIEW` + `Uri.fromFile`), Back key ignored (v0.95).
- 2026-11-16: v1.0 → 1.24: integration of SJGAM Control, CPU control added then removed, English interface, adjusted tiles and panels, 8 columns in "My Apps", forced landscape orientation.
- 2026-11-17: v1.05: `RomLauncher.standaloneIntent()` extended to all standalone emulator APKs (DraStic, Mupen64Plus AE, Flycast, PPSSPP); NDS/N64/PSP validation, Flycast v2.5 patched (`Uri.getPath()`), Dreamcast BIOS added and performance settings applied.
- Doubt: numbering is not monotonic in the sources (v1.24 on 16/11, v1.05 on 17/11); both versions coexist in the logs, with no explicit explanation.
- 2026-11-19: MediaBox Launcher uninstalled after backup `WORKING/mediaboxlauncher-installed-before-removal.apk`; the SJGAM frontend becomes the only HOME launcher.
- 2026-11-20: frontend and Mouse OSD moved to `/system/app/SJGAMFrontend.apk` and `/system/app/SJGAMMouseOsd.apk` to hide them from the grid; preferences restored (`recent_roms.xml`, `rom_inventory.xml`), "Apps" grid reduced to 10 entries; future update = replace the system APK + reboot, not `adb install -r`.
- 2026-11-23: system `search` service restored in `services.jar` (patch to `ServerThread`) to fix MX Player force-close; backups `BACKUPS/2026-11-23-search-service/`.
- 2026-11-24: 6/7-column confusion in the APKS menu (`MyAppsActivity`): the deployed build `d02dbe9e...` (versionCode 122) had 6 columns, the working build `a1641ac1...` (versionCode 124) had 7.
- 2026-11-24: after the user corrected the interpretation ("it's the 7-column one that's the fix" (original: "c'est la 7 colonnes qui est le correctif")), the 7-column build is reinstalled in `/data/app` and visually validated in the APKS menu; the firmware archive remains at 6 columns by decision to postpone the rebuild.

### Google/Vending stubs and Riptide billing (2026-11-20 → 2026-11-24)
- 2026-11-20: Riptide GP2 crashed at `IabHelper.startSetup` because `com.android.vending` was absent; AIDL stub `com.android.vending.billing.InAppBillingService.BIND` (13 KB) answering `BILLING_UNAVAILABLE` (code 3), without patching the game.
- 2026-11-20: stub installed in `/system/app/VendingStub.apk` (root:root 644) to be invisible in the frontend grid; reboot verified, game launched, `/system` returned to read-only, archive `WORKING/VendingBillingStub.apk`.
- 2026-11-23: 20 presence stubs made (`STUBS/`, minimal DEX mandatory otherwise dexopt fails) then, at the user's request, 14 social network/browser/Amazon stubs deleted.
- 2026-11-23: 6 stubs kept: VendingStub, GmsStub, GsfStub, PlayGamesStub, YouTubeStub, GoogleSearchStub; documented limits: presence only, no sign-in/purchases/SafetyNet (microG needed for real APIs).
- 2026-11-24: 5 stubs disabled via `pm disable` (GMS, GSF, Play Games, YouTube, Google Search), VendingStub kept because it makes Riptide GP2 work; backups `DUMPS & SAVES/stubs-cleanup-20261124/`, no physical deletion applied.
- 2026-11-24: Drift Zone 2 displays "Google Play Store signature invalid" (stub signed with the debug key): no stub can fix this without patching the game or installing a real Play Store.

### DraStic: A/B, L2, left joycon, firmware mouse mappings (2026-11-24)
- 2026-11-24: A and B swapped in DraStic (`_KeyMapConfigs_0_2/0_3` in `/data/data/com.dsemu.drastic/shared_prefs/_Dra$t1c_Pref$_.xml`); swap 4↔23 applied after stopping the app, validated "it's better" (original: "c'est mieux"), backup `DUMPS & SAVES/drastic-ab-fix-20261124/prefs-avant.xml`.
- 2026-11-24: `.ui.Settings` opens without crashing; the crash occurs only when navigating within the panel, not reproduced in the logged sequence.
- 2026-11-24: passing through `.ui.KeyMapper` rewrites the XML and restores the old A/B mapping; fix reapplied after `force-stop`; warning: any pass through the KeyMapper UI overwrites the fix.
- 2026-11-24: L2 = Open Menu (index 20 = keycode 104 `BUTTON_L2`) written directly into the preferences, validated "perfect, it works" (original: "impec ca marche"), consistent with L1=102/R1=103 and the M.A.X. port mapping.
- 2026-11-24: left joycon assigned to Screen-swap (index 16 = 198), Fast forward (index 17 = 196) and 1/2 Screen Swap (index 25 = 199), identification by `getevent -l -t` / logcat correlation, validated "it's great" (original: "c'est nikel").
- 2026-11-24: no system keylayout modified for these settings; the firmware mouse handles on-screen presses, no additional virtual mouse added.

### Firmware archive packaging (2026-11-24)
- 2026-11-24: creation of the `PACKAGING/` folder (`MANIFEST.md`, `stage/`, 30 files, `FILES.txt`, `MD5SUMS.txt`); rule adopted: the source of truth is the state actually deployed and validated on the console, not the working copies.
- 2026-11-24: divergences corrected on the Linux side: frontend code resynchronised with the deployed version (6 columns at that point), `TOOLS/sjgam_mouse.sh` (md5 `e9b6bd761db9ee845f0e64eb153d4081`) and `TOOLS/sjgam_touchd.arm` (md5 `0c1784a2e446abf59f1c5effb9343380`).
- 2026-11-24: user decision: Rockchip flashable archive with "everything validated, except game data" (original: "tout le valide, hors données de jeu"); creation of `PACKAGING/rockchip-m27-custom/` and `PACKAGING/sjgam-m27-custom-20261124.zip`.
- 2026-11-24: archive of 231 399 853 bytes, md5 `4ac17e4c7c14d56ae206dceb86361140`, `unzip -t` OK; Mouse OSD sources included (`rebuild-mouseosd.sh` producing an identical DEX md5 `98838ca8...`).
- 2026-11-24: packed images: `parameter` (1024 bytes), `kernel.img` (M38 padded to 12 MB), factory `boot.img` (md5 `5da0f206ca49f13bb5b7165ea73f82c2`), `recovery.img` (md5 `7aac2177...`), `misc.img` (md5 `cf4825bb...`), raw ext4 `system.img` (805 306 368 bytes, md5 `af895a12...`).
- 2026-11-24: `system.img` = factory dump + 19 validated files written via `debugfs`; `e2fsck -fn` clean, 1188 files identical to the console, one single game APK added in `/system/app`; delta = 6 modified files + 13 added, 75 identical symbolic links.
- 2026-11-24: `flash.sh` (bash -n OK): detection + backup by default, `--write` (kernel/boot/recovery/system then re-read and `cmp`), `--restore`, `KRNL` boundary checks at 0x4000 and 0xa000; no formatting of `userdata`/`cache`, no SD writes, no loader distributed.
- 2026-11-24: nothing flashed or modified on the console at this stage; open points before fabrication: distribution format, exact scope, Mouse OSD rebuild script.

### Console cleanup and end of session (2026-11-24)
- 2026-11-24: full backup of `/data/local/tmp` before cleanup (`DUMPS & SAVES/console-cleanup-20261124/data-local-tmp/`, 135 files, 25 MB, `MD5SUMS.txt` md5 `86793897f1aa7438b6a868287a17211a`).
- 2026-11-24: 125 temporary files deleted (framebuffer captures ~21 MB, stub copies, old OSDs, diagnostics, test videos, old binaries/scripts); `/data/local/tmp` goes from 24,7 MB to 1,5 MB.
- 2026-11-24: kept: `sjgam_mouse_speed`, `sjgam_touchd.pid/.log`, `mouse_daemon.log`, `mouse_daemon_autostart.log`, `settings.ini.bak-italian`, `services_patched.jar`, `sd_boot.bin`, `usbmark_hook`, `ctsScreenShot/`.
- 2026-11-24: post-cleanup checks: kernel #38, keylayout `29fb5f55...`, apps present, `mouse_daemon` (pid 757) and `sjgam_touchd` (pid 758) active; no game/SD/rom data touched.
- 2026-11-24: attempt to replace the 7-column frontend with the 6-column one, then correction of the interpretation; back to build `a1641ac1...` via `adb install -r` (upgrade), 6-column version backed up in `DUMPS & SAVES/frontend-7col-20261124/`.
- 2026-11-24: end of session "let's stop here" (original: "on stop la"): no flash, no forced reboot, no other modification; console in 7-column frontend, DraStic fixed, 5 stubs disabled, firmware mouse intact.
- 2026-11-24: archive unchanged (6 columns, 6 stubs); to resume: Riptide GP2 (billing), Drift Zone 2, MX Player (search) and APKS menu tests, then a single rebuild of the archive after validation.
- 2026-11-24: new writable GitHub repository `git@github.com:wisittheunknow-sys/Sjgam-m27-Unclock.git`, local clone `/var/mnt/stockage/sjgamM27/Sjgam-m27-Unclock`, key `~/.ssh/id_ed25519_m27`, commit `ab370a6` (README + .gitignore, no game data or binaries).
- 2026-11-24: HISTORY.md task requested but postponed to the next day; three extraction agents produced `/tmp/opencode/history-{kernel,max,global}.md` (perishable outputs, to be regenerated if missing); do not copy the raw logs into the repository (local keystore password).
- Scope: the M.A.X. port is detailed in Appendix A and the kernel/drivers in Part I; this part is limited to the frontend, emulation, stubs and packaging. The two system workstreams are covered in Part II.

## Appendix A — M.A.X. port on SJgam M27 (22/11 → 24/11)

Appendix: work ancillary to the console, kept here for reference and reproducibility. The console remains the main subject (Parts I to III).


Sole source: `/var/mnt/stockage/sjgamM27/projet MAX/STATE.MD` (1280 lines, read in full). No game data or binary content is reproduced here: only paths, quantities, decisions and results are kept. Doubts are flagged explicitly at the end of the document.

### Phase 0 — Scoping and inventory (date not specified, before 22/11)

- Objective: an APK installable on SJgam M27 (Android 4.4 / API 19, 32-bit RK3188 ARMv7-A, Mali-400), the game M.A.X. (1996, Interplay), playable single-player against the enemy AI, no multiplayer (no Wi-Fi on the console).
- Input constraints: 24 physical keys seen as an Android keyboard + virtual mouse; no touchscreen; letterbox scaling to be provided.
- Initial inventory `C:\MAX`: `MAX.iso` (638 740 480 bytes), installed `MAX/` (532 files, 388 200 894 bytes), `max-master.zip` (M.A.X. Port, SDL3, klei1984), `maxr-develop.zip` (M.A.X. Reloaded, SDL2/C++20, client-server).
- Both archives were extracted and flattened into `C:\MAX\max-master` and `C:\MAX\maxr-develop`; the `.zip` archives and ISO kept intact.
- No build tools at the start: no cmake, ninja, git, Java, gradle, adb or C/C++ compiler; python = WindowsApps stub; `ANDROID_HOME` / `ANDROID_SDK_ROOT` / `NDK_HOME` undefined.
- `maxr-develop` audit: no enemy AI (UI stub, `isHuman() → true`, `data/ABOUT` "features still missing", only automatic movement of prospectors exists), local single-player only, C++20 + SDL2_net + OggVorbis + `resinstaller` mandatory, no Android/JNI code; favourable points: 2D rendering to an ARGB8888 texture, 640×480 minimum, key remapping via `keys.json`.
- AI audit: `MAXRUN.EXE` contains assertion strings from the original sources (`ai_*.cpp`) that correspond one-to-one with the tasks in `max-master`; the AI in `max-master` is therefore the reconstructed original AI, under the MIT licence.
- Measured AI scope in `max-master`: facade `src/ai.hpp`, strategy `src/aiplayer.hpp`, 51 task types in `src/task_manager.hpp`, 67 `.cpp` (~22 100 lines) + ~7 000 lines of handlers, activation in `src/game_manager.cpp`; AI cost = 0 if this engine is kept.
- Architecture comparison: grafting the AI onto MAXR judged unrealistic (~30 kLOC to translate plus an intent adapter to `cAction`); reverse-engineering `MAXRUN.EXE` redundant and costly.
- Verified version conflict: SDL3 (used by the master) requires API 21+ / NDK r28c / SDK 36+, SDL2 accepts API 19; NDK r25c = last KitKat-compatible version, r26+ requires API 21+.
- `max-master` master: 53 SDL3 files to back-port and 59 `std::format` occurrences incompatible with LLVM 14; two coherent stacks identified: A = API 19 + SDL2 + NDK r25c, B = ROM ≥ Android 5.0 + SDL3 as-is.
- User decision: stay on Android 4.4 / API 19 (option A); option B is discarded but the code remains compatible if a ROM appears.
- User decision: no original game data will be embedded in the APK (M.A.X. is not abandonware); the end user supplies their original copy on the SD card.
- Data audit: installed tree = 13 `.MSC` (190,5 MB), 3 `.MVE` (100,1 MB), 24 `.WRL` (29,4 MB), 272 `.WAV` (25,9 MB), `MAX.RES` (17,8 MB), 71 `.FLC` (10,0 MB), scenarios and saves; `MAX.RES` = 1 828 records with 721 references to external media, including tolerable absences.
- `MAX.INI` contains no resolution key: the 640×480 comes from the port, not from the original configuration.
- Port assets in `C:\MAX\max-master\assets` = 1 859 676 bytes (`PATCHES.RES`, `language.json`, `help.json`, `maxint.json`, `settings.json`); `PATCHES.RES` loaded before `MAX.RES` and takes priority on tag collisions.
- Derived constraints: `READ_EXTERNAL_STORAGE` permission sufficient on API 19, custom home screen for entering the SD path instead of the SDL3 picker, audio paths kept as-is (miniaudio compiled without Vorbis/Opus/FLAC/MP3).

### Phase 0bis — Major revision of the codebase (22/11)

- Discovery: the official repository migrated from SDL2 to SDL3 on 26 November 2025; tag `v0.7.2` is the last SDL2 version (SDL2 2.32.6, official API 19, NDK r15c+).
- Audited comparison: `master` = SDL3 + 59 `std::format` + 12 dependencies; `v0.7.2` = native SDL2, 0 `std::format`, 9 dependencies, C++20 limited to `concept`/`requires` (LLVM 14 OK).
- The enemy AI is complete in `v0.7.2` (110 `ai*`/`task*` files, `Ai_BeginTurn` and `Ai_CheckReactions` in `game_manager.cpp`); only the gameplay fixes are less recent (~defect 250 vs ~306).
- Decision: start from `v0.7.2` extracted into `C:\MAX\max-android`, not from the SDL3 master; avoids 53 SDL rewrites, 59 `std::format` uses and 3 dependencies without losing the AI.
- `max-master` and the original archives are not modified; the master remains available to retrieve occasional fixes.
- v0.7.2-specific blockers identified: `GetBasePath()`/`GetPrefPath()`/`GetGameDataPath()` in `src/resource_manager.cpp` (Windows/Linux branches only), `TestMouse()` exiting without a mouse, `SDL_Init` without `SDL_INIT_AUDIO`, fixed mapping table in `src/input.cpp`.

### Phase 1 — Toolchain (22/11)

- The user provides `C:\MAX\android-19_r04.zip` (63 871 092 bytes): Android 4.4.2 / API 19 SDK Platform, extracted to `C:\MAX\sdk\platforms\android-19`; archive kept.
- Clarification: this package provides `android.jar` and skins, but no C/C++ compiler, `aapt`, `dx` or `zipalign`.
- Build-tools 19.1 installed in `C:\MAX\sdk\build-tools\19.1` (`aapt`, `dx`, `zipalign` working); platform-tools in `C:\MAX\sdk\platform-tools` (adb 1.0.41 / 37.0.3).
- NDK r25c (25.2.9519653) installed in `C:\MAX\sdk\ndk\25.2.9519653` (446 MB archive); last NDK compatible with API 19.
- JDK: the Adoptium asset returned behind the proxy was a 1867-byte XML instead of a ZIP; replaced by Amazon Corretto 8 (1.8.0_514) in `C:\MAX\sdk\jdk8`.
- CMake 3.28.6 and Ninja 1.12.1 installed in `C:\MAX\sdk\...`; archives kept in `C:\MAX`.
- Full toolchain validated on 22/11: `armeabi-v7a` cross-compilation possible; `adb` immediately usable.

### Phase 2 — Native core and APK skeleton (22/11)

- CMake/dependencies: `SHARED` target under `ANDROID`, no `-static`/debuglink, `SDL_STATIC ON` + `SDL2main`; `cmake/enet.cmake` and `cmake/sha2.cmake` use FetchContent `_SOURCE_DIR` variables; `cmake/sdl2.cmake` enables joystick and haptic under Android to avoid unresolved JNI symbols.
- Statically linked dependencies (9 local sources in `C:\MAX\deps-src`); fixes: nlohmann/json 3.12 `path::u8string()` converted byte by byte for libc++ 25, `lua_getlocaledecpoint()` forced to `'.'` under Android (`localeconv()` absent before API 21), `<filesystem>/<sstream>/<system_error>` includes added in `resource_manager.cpp`.
- Android port: `GetBasePath()`/`GetPrefPath()` go through `SDL_GetPrefPath()` (private `getFilesDir()` directory) where the Java launcher copies the assets; `GetGameDataPath()` reads `game_data.txt` written by the launcher (no native folder picker on API 19); `TestMouse()` bypassed under Android.
- Java (`C:\MAX\android`): `MAXLauncher` copies `PATCHES.RES`, `settings.ini` and the 6 `lang_*.ini` files to the private directory, browses the SD, checks for `MAX.RES` (or `MAX/MAX.RES`) and persists the path in `game_data.txt`.
- `MAXActivity` extends `SDLActivity`, `getLibraries() = {"max"}`, optionally remaps keycodes via `keymap.txt`; `SDLSurface` bridges touch→mouse; `SDLAudioManager` replaces Java 8 streams/lambdas with loops (dx 19.1); `HIDDeviceManager` uses `BLUETOOTH_CONNECT` as a literal string (API 31 constant).
- Java compiled against API 28 `android.jar` (`SDK_INT` guards kept), package and resources with API 19 `android.jar`.
- Reproducible scripts: `C:\MAX\build_android.ps1` (configure/build/strip) and `C:\MAX\build_apk.ps1` (R.java, javac, dx, aapt, signing, zipalign).
- Result: `C:\MAX\build-android\libmax.so`, ELF32 ARM, `SDL_main` exported, 4,6 MB after strip; `C:\MAX\android\bin\MAX.apk`, 12,5 MB, signed, `minSdk=19`, `armeabi-v7a`; `aapt list` check: no original data embedded.
- "Offline" additions (console not yet connected): manual SD path entry in `MAXLauncher` with resolution of the parent containing `MAX/MAX.RES`; keymap read priorities `/sdcard/max_keymap.txt`, `/sdcard1/max_keymap.txt`, `/storage/sdcard1/max_keymap.txt`, `<game_data>/keymap.txt`, private file.
- `C:\MAX\keymap_example.txt` (commented template), `C:\MAX\console_capture.ps1` (automated ADB capture), `C:\MAX\install_console.ps1` (install, push keymap, launch via `monkey`, `-Log` option); APK rebuilt after these changes.
- User clarification: the firmware virtual mouse is activated by the R1+R2 chord and can produce a real `SOURCE_MOUSE` or a simulated touch; implementation initially added (`vm_r1=`/`vm_r2=`, `sVirtualMouseActive`, touch→mouse bridge) then removed on 23/11 in favour of the firmware mouse.

### Phase 3 — First start on console: signature, JNI and paths (23/11)

- The console is connected; goal of the session: start the engine on M27 (1024×600) without embedding any game data, original AI intact.
- APK signature: `-sigalg SHA1withRSA -digestalg SHA1`; SHA-384 was rejected by Android 4.4 (`INSTALL_PARSE_FAILED_NO_CERTIFICATES`).
- JNI/paths: guard `Android_JNI_GetEnv() == NULL` in `SDL_android.c` (returns `/data/data/com.max.port/files`); `HIDDeviceManager.acquire()` forced to `null` because `libmax.so` lacks the HID JNI symbols.
- Linkage: relink under Linux with `ld.lld` 22 (objects and libraries from the Windows NDK r25c reused) via `WORKING/max/rebuild-sdl-android.sh`.
- Dead end: an EGL crash diagnosed as `SDL_EGL_LoadLibrary` was in fact a bad `load bias`; the real stack was `memcpy` ← `SDL_memcpy` ← `Svga_Blit` ← `Svga_SetPaletteColor` ← `Color_Init`.
- Real cause: framebuffer advertised as 24-bit (pitch 2458 bytes for 819 pixels) while `Svga_Blit` copied `bounds.w * 4` bytes per line; read outside the mapping.
- Fixes in `max-android/src/svga.cpp`: force `SDL_PIXELFORMAT_ARGB8888` when the advertised depth is not 32-bit; replace `SDL_LockTexture` + manual `memcpy` with `SDL_UpdateTexture()` using the real pitch.
- Temporary diagnostic removed before delivery (`usleep(10s)` for gdbserver).
- Result: APK rebuilt and installed, intro cinematic visible (user validation), no SIGSEGV, stable process, files `/mnt/external_sd/MAX/MAX.RES` and `MAXINT.MVE` opened, 44.1 kHz audio active.
- Tools kept: `rebuild-sdl-android.sh` (recompiles `SDL_android.c` and `svga.cpp`, relinks and pushes into `/data/app-lib/com.max.port-*/`), `rebuild-apk-linux.sh` (dex + SHA1-signed APK), `diag-gdb-stack.sh`; captures in `WORKING/max/capture-2026-11-23/`.
- Immediate follow-up: user confirms the main menu ("we have the menu!!!!" (original: "on a le menu !!!!")); 8-bit → 32-bit texture rendering also valid for the interface, ~25% CPU, no crash.

### Phase 4 — Full-screen 640×480 → 1024×600 display (23/11)

- Problem: `WindowManager_GetScale() = screen_h/480 = 1,25` centres an 800×600 interface in the buffer, hence a rectangle surrounded by black with `window_width=1024`, `window_height=600`.
- Attempt 1: `settings.ini` at 800×600 + `disable_ar_correction=1` and new `Svga_ApplyStretch()` (`SDL_RenderSetScale` + `SDL_RenderSetViewport`) called in `Svga_Init()` and `Svga_Flush()`/`Svga_Flip()`; framebuffer check 98,4% non-black pixels.
- Attempt 2: black bands around the menu background (640×480 centred in 800×600); addition of the `rescale=true` parameter to the 3 `MAINPIC` calls in `menu.cpp`; check 95,6% non-black pixels, validated "magnificent!!!" (original: "magnifiquuueeee !").
- Rollback: `rescale=true` shifted the menu out of the black frame because `ScaleUlx/Uly` applies only an offset while the generic scaler multiplies by 1,25; fix cancelled, `menu.cpp` restored identical to upstream.
- Locked final state: `settings.ini` with `window_width=640`, `window_height=480`, `disable_ar_correction=1`; `WindowManager_GetScale()=1,0`; `Svga_ApplyStretch()` stretches the 640×480 texture to 1024×600 with `scale=(1,6 ; 1,25)` and a full-screen viewport.
- Local SDL2 converts mouse/touch events according to the renderer scale, so clicks remain consistent with the stretched image.
- Final check: stable process, `/dev/graphics/fb0` capture 98,6% non-black pixels, no black bands; user validation "Yes, perfect!" (original: "Oui, parfait !"); instruction not to touch `svga.cpp`, `settings.ini` or `menu.cpp` again for this session.

### Phase 5 — Firmware mouse, removal of the virtual mouse and pointer (23/11)

- User request: remove the virtual mouse embedded in the game, which duplicates the one in the custom firmware.
- Firmware state observed: `/system/bin/mouse_daemon` (PID 776, root), `com.sjgam.mouseosd` (PID 730), `rk29-mouse` device = event4.
- Java: removal of the R1+R2 chord and Toast in `MAXActivity`, of the `forwardTouchAsMouse()` bridge and its calls in `SDLSurface`, of the `sVirtualMouseActive` field in `SDLActivity`; real `SOURCE_MOUSE` events to `onNativeMouse()` are kept; parsing now ignores `vm_*` keys.
- Trap fixed: `rebuild-apk-linux.sh` only recompiled `HIDDeviceManager.java`; it now compiles all 4 Java files and copies all classes into `obj/` before dex.
- Verification: `strings classes.dex` with no trace of the virtual mouse; APK installed, stable PID, no crash; resumption point = confirm a single click (no more double click).
- Pointer drift reported (zero at top/left, increasing towards bottom/right): first cause ruled out (relative mouse mode of `SDL_WINDOW_FULLSCREEN`, disabled in `GNW_mouse_init()`), real cause = `SDL_GetMouseState()` returns physical 1024×600 coordinates not converted by `SDL_RendererEventWatch`.
- Fix: `Svga_MouseToLogical()` in `svga.h/cpp` via `SDL_RenderWindowToLogical()` (accounting for viewport and scale), `Svga_WarpMouse()` via `SDL_RenderLogicalToWindow()`, called in `mouse.c` `mouse_info()`; `rebuild-sdl-android.sh` also compiles `mouse.c` with `-D_FORTIFY_SOURCE=0`.
- User validation: "perfect, it follows the mouse" (original: "impec ça suit la souris"); mouse locked down.

### Phase 6 — Stabilisation: training, FLIC, cursor and log (23/11)

- `CLANATRB` assertion when launching training: French tokens from the original data triggered the assert while `settings.ini` was in English; `AttribTokenMatches()` now accepts both English and French, without modifying game data.
- FLIC/horizontal bands: signed `char psize` on ARM interpreted literal runs as huge sizes, causing heap corruption; fixed to `int8_t psize` in `flicsmgr.cpp`; validated "oh! perfect, we have the image and it's stable on the map!!!" (original: "ho ! impec on a l'image et c'est stable sur la map !!!").
- Persistent log: `ResourceManager_LogOutputHandler()` ends with `std::endl` + `flush()`; `/data/data/com.max.port/files/stdout.txt` recoverable after a crash.
- "right joycon up" crash: SIGSEGV in `Cursor_DrawAttackPowerCursorHelper()` (`cursor.cpp`) due to a cached resource descriptor invalidated after `ResourceManager_Realloc()`; addition of `Cursor_RefreshDescriptor()` before draws in `game_manager.cpp` (attack bar and stealth chance), with the overlay skipped if the resource is absent/invalid; `cursor.cpp` added to the rebuild script.
- Validation: "Yes, no more crash" (original: "Oui, plus de crash"); APK rebuilt and installed (`Success`).

### Phase 7 — Calibration of the 24 buttons and mapping (23/11)

- Raw `getevent -l /dev/input/event0` capture → `/tmp/opencode/calib-keypad.txt`: D-pad 103/105/106/108, left stick 183..186 (Android 196..199), A/B/X/Y declared in `/system/usr/keylayout/rk29-keypad.kl` = 290/289/291/288 → Android 23/4/99/100, shoulders 294/292/293/295, Select/Start 296/297.
- Spurious right-stick actions: `/sdcard/max_keymap.txt` still contained `183=8`, `184=9`, `185=10`, `186=11` (PROG_RED..BLUE in addition to the `rk29-mouse` REL_X/REL_Y); lines and `vm_*` keys removed, backups `-20261123`, pushed copy with `chmod 644` / `chown 1000:1015`, app restarted; validated "Yes, fixed" (original: "Oui, corrigé").
- First ABXY deduction via `getevent` corrected by a temporary `MAXKEY` log in `MAXActivity.dispatchKeyEvent()`: A=23 (scan 290), B=4 (scan 289), X=100 (scan 288), Y=99 (scan 291); log removed, APK rebuilt, `MAXKEY` absent from `classes.dex`.
- The Android `.kl` file remains unchanged ("not the Android kl, in max" (original: "pas le kl android, dans max")): only the table read by the game is modified (`WORKING/max/max_keymap.txt`, `keymap_example.txt`, SD backups `-20261123{, -2, -3}`).
- Definitive D-pad calibration with the game stopped (in-game actions skewed the logs): up = Y/end turn, right = B/back, down = A/confirm, left = X/centre; user confirmation of the physical layout A=down, B=right, X=left, Y=up.
- Root cause of the chaotic mapping: comments in the SD table had shifted Linux numbers by +1 (BTN_THUMB2 = 290 and not 291); the entry `104=34` sent F=centre from L2; table corrected to `23=66`, `4=111`, `99=34`, `100=33`, `102=69`, `104=70`, backup `-6`.
- System file check: unchanged mtime (2026-11-18 17:36:22), md5 `29fb5f55eba641d01de7a11fd6120396`; no APK to rebuild (table read at launch).
- A/Y swap requested for testing (`23=33`, `100=66`, backup `-7`) then reverted to the previous state (`23=66`, `4=111`, `99=34`, `100=33`, `102=69`, `104=70`, backup `-9`); validation "Yes" (original: "Oui").
- Start/Select shortcuts: addition of `injectKey()` in `MAXActivity.java` sending a synthetic KeyEvent to SDL (Start 108 → Alt+F, Select 109 → Alt+P, release in reverse order), based on the native shortcuts in `game_manager.cpp`; the old Tab/Space mappings trigger nothing.
- APK rebuilt (`apk-shortcuts.log`), lines `108=61`/`109=62` removed from the SD table (backup `-8`); validation: Start opens FILES/SAVE, Select pauses (user choice: keep Select = Pause, preferences accessible with the mouse).
- All diagnostic logs removed from `MAXActivity.java`; APK rebuilt (`apk-clean-final.log`), installed, `MAXKEY` absent from `classes.dex`.
- Final mapping locked: A=Enter/end turn (23=66), B=back (4=111), X=centre (99=34), Y=change cursor (100=33), L1 zoom out (102=69), L2 zoom in (104=70), left stick = arrows, right stick = unmapped firmware mouse; `.kl` intact.

### Phase 8 — Functional tests and port closure (23/11)

- User validation of the final mapping: "Yes" (original: "Oui") (A=Enter, B=Escape, X=F/search, Y=E/end turn in the table then in effect).
- Save/load: `WINDOW_FILES_BUTTON` (logical rect 17,4–83,21, i.e. ~27,5–133,26 on 1024×600), FILES → SAVE then FILES → LOAD: OK (user validation).
- Final AI test: game started with 4 "computer" opponents in parallel; feedback "stable, it attacks, it launches construction projects" (original: "stable, ça attaque, ça lance des constructions"), `pidof com.max.port` active, `stdout.txt` with no FATAL log or tombstone.
- Compliance: the 4 `src/ai*.cpp` sources were never modified; the original enemy AI absolutely preserved.
- Gameplay observation during the 4-AI test: "the reds get crushed by the greens" (original: "les rouges se font défoncer par les verts"); noted as an observation, not a bug.
- Archives: `WORKING/max/logs/stdout-final-20261123.txt` and `WORKING/max/logs/tombstones-list-20261123.txt`.
- 23/11 summary: port completed and validated on M27 (Android 4.4 / API 19, RK3188, Mali-400, 1024×600): native 640×480 rendering stretched, firmware mouse preserved, SD mapping calibrated, training stable, save/load OK, original AI functional; no original data in the APK (player copy on `/mnt/external_sd/MAX`).
- Sound: occasional loss just after a restart, logs `AudioFlinger: write blocked` (366/398 msecs) and API 19 warnings from `SDLAudioManager`; normal volumes, no initialisation error; sound returned on its own ("now it's good, I have sound in max" (original: "là c'est bon j'ai du son dans max")), with no code change.

### Phase 9 — Text language (23/11 → 24/11)

- `src/localization.cpp`: `GetLanguage()` reads `SETUP/language` in `<prefpath>/settings.ini` then loads `lang_<langue>.ini` from the APK assets; the 6 language files are embedded; Voice/Sfx/Music point to the player's MAX copy (no audio-language setting in the engine).
- 23/11: `language=french` written in `/data/data/com.max.port/files/settings.ini` and in `android/assets/settings.ini` (template); CRLF preserved, `chown 10041:10041`, `chmod 600` (busybox `sed -i` had put the file back to root:root); restart via `am force-stop` + `monkey`; validated "Yes, everything is in French" (original: "Oui, tout est français").
- 23/11: first-run language selector added in `MAXLauncher.java` (`LANGUAGES[]`, spinner, `readLanguage()`/`writeLanguage()` via regex `(?m)^language=`, ISO-8859-1 write to preserve inode, permissions and CRLF), called before `writeDataPath()` + `startGame()`.
- Trap fixed: `rebuild-apk-linux.sh` compiled only 4 Java files and reused an old `MAXLauncher.class`; `MAXLauncher.java` added to the compilation and `FileInputStream` import added; new APK verified in `classes.dex`.
- Test: Italian language chosen in the launcher with the folder `/mnt/external_sd/MAX` → Italian texts, `language=italian`, CRLF and permissions intact; cinematic remained French, as expected since it is read from the player's copy.
- Read-only verification: unique `MAXINT.MVE` (71 202 530 bytes), `MAXMVE1.MVE` and `MAXMVE2.MVE` = other cinematics, `INTRO1.CAM` = campaign; a single occurrence of `INTROFLC` in `MAX.RES`, `Movie_PlayIntro()` → `movie_run(INTROFLC)`; no language suffix or folder: single-language intro, no audio/video choice to implement.
- 24/11: back to `language=french` in the private `settings.ini` (backup `/data/local/tmp/settings.ini.bak-italian`), CRLF and permissions preserved; `lang_french.ini` already present; visual validation of the texts by the user.
- 24/11: French texts confirmed, port considered complete (sound, rendering, icon, controls, language). A possible later enhancement: file `/sdcard/max_language.txt` applied by `MAXActivity` before `super.onCreate()` (without changing the native code) or a real selector in Preferences (C++).

### Phase 10 — Visual export and icon (24/11)

- "backgrounds" export at the user's request: read-only extraction of 29 640×480 images from `MAX.RES`, 6-bit palette of `COLOR.PAL` converted to 8-bit (×4), RLE decoding conforming to `WindowManager_DecodeBigImage` (stream from byte 776 = transparent colour).
- Results in `EXPORT/tank-sprites/backgrounds/` (thumbnails `thumbs/`, sheet `ALL_BACKGROUNDS_sheet.png`) and `EXPORT/presentation/` (4 composites, raw backgrounds, portraits, sheet of the 29 backgrounds, `LISEZ-MOI.txt`); reconstructed production screens `FACBUILD`/`CONBUILD` + composite tank sprite at (11,13), position read in `buildmenu.cpp`, pixel-by-pixel verification OK.
- Reminder: the user wanted the production screen, neither the `TANK03.FLC`/`ALNTANK.FLC` animation nor the RLE sprites; no MAX data, `ai*.cpp` source or APK modified.
- Icon: source `icon_final.png` provided by the user (144×144 RGBA, transparent background), no original image used; `install-icon.sh` generates 48/72/96/144, updates the manifest, rebuilds and installs the APK; old APK saved in `BACKUPS/2026-11-24-MAX-pre-icon.apk`.
- Checks: `aapt dump badging`, `unzip -l` (4 densities), `pm path` → `/data/app/com.max.port-1.apk`, `settings.ini` still at `language=italian` at that point; icon APK md5 = `571e1b3ae3fbe415fc820a1f27c4b762`.

### Phase 11 — Intermittent sound: diagnosis, watchdog and batched rendering (24/11)

- Symptom: sound absent everywhere on first launch, present after "quit + relaunch"; not reproduced over 6 force-stop + `monkey` cycles (all `ma_engine_init()` succeeded); volumes 100/100/50 and `disable_music/fx/voice=0`, no system mute.
- Reproducible sequence: Burger in the foreground, quit, launch M.A.X. → silence; captures in `/tmp/opencode/audio-bad/`.
- Diagnosis: Burger keeps 43 threads, 2 `AudioTrack`, 26,5 MB PSS; at M.A.X. startup, `AudioFlinger` reports `write blocked for 395 msecs, 3 delayed writes`; the M.A.X. track in state `S`, flags `0x601`, `UndFrmCnt=2230`, absent from `active tracks`; after stopping Burger: state `A`, flags `0x000`, present in `active tracks`.
- RAM ruled out: 512 MB, 274 MB free/cacheable, no swap, kswapd0 inactive; retained cause = OpenSL ES underrun/scheduling linked to an app's background `AudioTrack`.
- Interim user decision: START + SELECT 3 s combo on the foreground app before launching M.A.X. (force-stop already in place); no native watchdog retained at this stage, no recompilation.
- Optimisation review (analysis only): 1 `SDL_RenderPresent` per `Svga_Blit` (dirty rects) → batch; full-screen `SDL_LowerBlit` 8→32 on every palette change (~30 Hz, colour cycling); move `SDL_HINT_RENDER_SCALE_QUALITY` before `SDL_CreateRenderer`, remove `SVGA_NO_TEXTURE_UPDATE` (dead code), replay `Svga_ApplyStretch()` only on size change; settings `scale_quality=1`, `enhanced_graphics=1`, films disabled; RAM largely sufficient.
- Linux NDK r25c installed, script `WORKING/max/build-native-linux.sh` created, baseline compiled, full backup before modification in `BACKUPS/2026-11-24-MAX-pre-audio-render/`.
- Audio watchdog v2 in `sound_manager.cpp`: check every 500 ms, `ma_device_start()` if `ma_device_get_state() == ma_device_state_stopped`, log limited to 5 restarts.
- Batched rendering in `svga.cpp`: `Svga_Flush()` = a single `SDL_RenderCopy`/`SDL_RenderPresent` per frame, `Svga_Blit()` now only uploads + sets a dirty flag, 25 ms fallback; quality hint moved before the renderer, dead constant removed; flush at the start of `GNW_do_bk_process()` in `input.c`.
- Verification: `Svga_Flush` present in the `.so` (`llvm-nm`), stripped `.so` 4,8 MB, APK of 2 856 567 bytes installed, AI intact; automatic test: clean start, active track, no crash.
- User validation: sound OK after Burger, rendering "really smooth" (original: "vachement fluide"), FLV/intro can be enabled, watchdog not triggered during the test but present and logged; no visual regression or tombstone; installed version `/data/app/com.max.port-2.apk`, 2 856 567 bytes.
- Final feedback: map navigation also smooth, overall FPS gain confirmed, rendering fix definitive.
- Watchdog v3: silence was reproduced with the device in the "started" state (v2 blind) and the AudioFlinger track blocked; detection via audio clock in `sound_manager.cpp`: if `ma_engine_get_time_in_pcm_frames()` stops advancing for 1500 ms (500 ms check), force stop+start of the OpenSL device and log (max 5 times); fields `audio_stall_time`, `audio_last_engine_time`, constant `SOUND_MANAGER_STALL_MS=1500`.
- APK v3: 2 856 905 bytes, md5 `064460b9...` (value truncated in the source), `.so` `fb2a1f86...` (truncated), installed; test with Burger in the background: sound OK, active track, no restart logged; diagnostics to collect in `DIAG/2026-11-24-audio-watchdog-v3/` if silence returns.

### Phase 12 — Console actions outside the MAX scope (24/11)

- Console audit: CPU healthy (~64% idle), RAM not saturated (245 MB reclaimable cache) but aggressive LMK (14–36 MB); Drift Zone 2 at ~91 MB PSS kills MAX and Burger on launch; Drift error = invalid Vending stub signature (debug key); nothing changed on the console.
- Cleanup of `/data/local/tmp` after full backup in `DUMPS & SAVES/console-cleanup-20261124/`: 125 files deleted, 24,7 MB → 1,5 MB; no MAX data on SD touched, no `ai*.cpp` modified; `settings.ini.bak-italian` and `services_patched.jar` kept.
- After cleanup: `com.max.port` (`/data/app/com.max.port-1.apk`, md5 `064460b9...`) and the mouse remain operational.
- Frontend (outside MAX): 7 → 6 → 7-column oscillation in the APKS menu; correction validated "now it's good" (original: "la c'est bon") = 7-column build `a1641ac1...` (versionCode 124) reinstalled via `adb install -r`, final path `/data/app/com.sjgam.frontend-2.apk`, 6-column backup in `DUMPS & SAVES/frontend-7col-20261124/`; the firmware archive still contains the 6-column build in `/system/app` (resynchronisation decision to be made).
- Drastic (outside MAX): A/B inversion fixed in `_KeyMapConfigs_0_2 = 23` / `_0_3 = 4` (backup `prefs-avant.xml`), special L2 = Open Menu mapping (`_0_20 = 104`), screen-swap = left joycon (198), fast forward = up (196), 1/2 screen swap = right (199); user validation "it's great" (original: "c'est nikel"); `.ui.Settings` still crashes after opening the panel, logcat ready; no link with the M.A.X. port or the original firmware mouse.

### Resolved anomalies — summary

- SHA-384 signature refused by Android 4.4 → SHA1withRSA/SHA1 signature.
- Native crash at startup (misleading EGL stack) → 24-bit framebuffer and wrong pitch in `Svga_Blit` → forced ARGB8888 format + `SDL_UpdateTexture`.
- Unresolved HID JNI symbols → `HIDDeviceManager.acquire()` to null and joystick/haptic backends enabled in SDL2.
- SDL3 folder picker absent on API 19 → Java launcher with SD navigation, manual entry and persisted path.
- No mouse detected at startup → `TestMouse()` bypassed under Android.
- Double click / duplicate mouse → complete removal of the virtual mouse from the port in favour of `rk29-mouse`.
- Cursor drift (physical vs logical coordinates) → `Svga_MouseToLogical()` / `Svga_WarpMouse()`.
- Black bands and menu offset → native 640×480 buffer + `Svga_ApplyStretch()`; `menu.cpp` restored to upstream.
- `CLANATRB` assertion on French tokens → bilingual `AttribTokenMatches()`, data intact.
- FLIC bands/corruption on ARM → `psize` as `int8_t`.
- Cursor crash after resource reallocation → `Cursor_RefreshDescriptor()`.
- Spurious right-stick actions and mapping shifted by SD table comments → keymap corrected, `.kl` intact.
- Unexpected recentring by L2 → removal of `104=34`, table corrected.
- Tab/Space having no effect for FILES/Pause → synthetic Alt+F / Alt+P shortcuts.
- Stale Java build (`rebuild-apk-linux.sh` recompiled only one file) → script extended to the 5 Java sources.
- `path::u8string()` / `localeconv()` / libc++ includes → NDK r25c compatibility fixes.
- Intermittent sound loss → watchdog v2 then v3 by audio clock, batched rendering (smoothness validated).
- Permissions and CRLF of `settings.ini` altered by `sed -i` → `chown`/`chmod` restoration.
- Italian then French language → `language=` driven by the launcher and the private file.

### APK checksums and identifiers reported in the source

- System keylayout table (`/system/usr/keylayout/rk29-keypad.kl`, never modified): md5 `29fb5f55eba641d01de7a11fd6120396`, mtime 2026-11-18 17:36:22.
- Icon APK (before the audio/rendering changes): md5 `571e1b3ae3fbe415fc820a1f27c4b762`.
- Watchdog v3 APK / last MAX version reported: 2 856 905 bytes, md5 `064460b9...` (truncated in `STATE.MD`); associated `.so` `fb2a1f86...` (truncated).
- Intermediate batch/watchdog v2 version: 2 856 567 bytes, installed under `/data/app/com.max.port-2.apk`.
- Non-MAX frontend: 7-column build `a1641ac1...` (versionCode 124, retained); 6-column build `d02dbe9e...` (versionCode 122).

### Doubts and points not to invent

- `STATE.MD` has "last update: 2026-11-22" (original: "dernière mise à jour : 2026-11-22") at the top, but its content covers up to 24/11: the header date is unreliable for the end of the log.
- Several "final mapping" versions succeed one another on 23/11 (D-pad calibration, the +1 shift discovered afterwards, A/Y swap, rollback); the retained table is that of phase 7 (A=Enter, B=back, X=centre, Y=change cursor), but the intermediate functional-test report still mentions X=F and Y=E: a nomenclature inconsistency not to be smoothed over artificially.
- APK installation paths diverge across passages: `/data/app/com.max.port-1.apk` (icon, after cleanup) and `/data/app/com.max.port-2.apk` (2 856 567-byte version); the link between these paths and the v3 APK is not spelled out.
- The md5 values of the v3 APK and the `.so` are truncated in the source ("..."): no complete checksum of the very last APK is recorded there; the `571e...` value concerns the icon APK, not necessarily the final version.
- Watchdog v2 was never triggered in tests and the original silence was not reproduced after v3: definitive effectiveness unproven, monitoring recommended.
- The resolution of the sound problem was first decided in favour of the START+SELECT combo without a watchdog, then a native watchdog was implemented the same day: successive decisions, not to be presented as a single simultaneous choice.
- The "definitive" D-pad calibration of 23/11 used Linux numbers shifted by the SD table comments; it was corrected afterwards. Intermediate codes (X=104, etc.) must not be presented as final.
- The old ISO inventory ("111 files / 110,7 Mo" (original: "111 fichiers / 110,7 Mo")) is explicitly wrong and replaced by 1 075 files / 637 053 313 bytes; do not keep the old figure.
- Several 24/11 actions are outside the MAX scope (frontend, Drastic, LMK, cleanup) and some were undone/redone: they are reported for context, with no link to the port APK or the mappings.

## Appendix B — 25/11/2026: `corruption_detected` assertion in M.A.X.

- User report: `assertion failed at saveloadmenu_save (.../max-android/src/saveloadmenu.cpp:7) triggered 1 time : 'corruption_detected'`. The path shown is a string compiled into the APK (`__FILE__` of the `SDL_assert` macro): the console never attempted to read that path on the PC.
- Verification: the code is identical to the upstream tag `klei1984/max` `v0.7.2` (`saveloadmenu.cpp:699-716`); this assertion was not added by the port. The local build enables assertions with `SDL_ASSERT_LEVEL=2`, hence the display; "triggered 1 time" is SDL's one-shot behaviour.
- Trigger identified: `SaveLoadMenu_Save(..., play_voice=false, backup=true)` called from `game_manager.cpp:2737` = end-of-turn autosave. Manual saves from the menu pass `play_voice=true` and skip the test.
- Original mechanism: `SaveLoadMenu_RunPlausibilityTests()` runs `Defect11` (depot/warehouse), `Defect183` (map hash) and `Defect151` (connector overlapping a unit). If a fix applies, the state is repaired in memory, `corruption_detected` becomes true and the function `return`s before writing the file: **that specific autosave is not written**, without a crash.
- The console's `stdout.txt` confirms the SDL report and `always ignore: yes`: the user chose "Always ignore" in the dialog box. Future identical messages will no longer appear, but the `return` remains active.
- Observed files: `SAVE10.BAK` 00:04 and `SAVE10.DTA` 00:08 (an autosave succeeded at 00:08); no `ai_log.txt` present; `settings.ini` measured at 0 bytes then rewritten to 1 196 bytes at launch. The user confirms the texts remain in French: nothing to change.
- User decision: "diagnose more finely" (original: "diagnostiquer plus fin"). Temporary `MAXDBG` logs added **only** in `saveloadchecks.cpp` (7 points: Defect11, Defect151, four Defect183 branches) and a summary in `SaveLoadMenu_RunPlausibilityTests()` of `saveloadmenu.cpp`. No modification of the `ai*.cpp` files (md5 verified before/after).
- Prior backup: `BACKUPS/2026-11-25-MAX-corruption-diag/` (sources before, installed APK `064460b913819ea3c0c214081dfa7e43`, `stdout.txt.avant`, md5 inventory of the 4 `ai*.cpp`).
- Native rebuild via `WORKING/max/build-native-linux.sh`, `llvm-strip` strip: `.so` `556ae81cc97191f2a0b3e567ead2d2ee` (4 806 040 bytes) containing the 7 `MAXDBG` strings. APK rebuilt via `WORKING/max/rebuild-apk-linux.sh`: `b15c8f34a5cd07ab0e6a28b94d8eec5f` (2 857 125 bytes, versionName 0.7.2, API 19, SHA1 signature).
- Installation authorised via `adb install -r`: final path `/data/app/com.max.port-2.apk`, installed md5 verified identical, embedded library verified. Data preserved.
- Results during the 4-AI game: the first 10 autosaves are clean (`Defect11=0 Defect183=0 Defect151=0 result=0`), then the 11th triggers `MAXDBG Defect151 connector at [78,81] overlaps unit type=35 at [78,81]` and `MAXDBG plausibility summary: Defect11=0 Defect183=0 Defect151=1 result=1`. So it is indeed the original `Defect151` check that is responsible: a connector overlapped a stationary unit of type 35 at [78,81]. The upstream code destroys that connector in memory (`UnitsManager_DestroyUnit()` in `saveloadchecks.cpp:223`) then skips writing this autosave.
- No crash, the game continues; the assertion did not appear because the user had chosen "Always ignore" in the SDL dialog. No modification of the `ai*.cpp`, no game data touched, no flash/reboot. The diagnostic APK is **temporary**: return to the validated APK `064460b9...` once the game is finished, and do not rebuild the firmware archive before the end of the user tests.
- Root cause identified by read-only analysis: the generic engine allows a friendly stationary unit on its own connector (`access.cpp:208-218`: only another connector or an enemy connector is blocked) and the construction pipeline finalises the `ANTIAIR` on the utility tile without removing the connector (`unitinfo.cpp:4022-4027`), while this cleanup already exists in `UnitsManager_BuildNext()` (`units_manager.cpp:5687-5691`) for units built with a path. `UnitInfo::SetPosition()` adds the unit to the hash without an occupancy test (`unitinfo.cpp:1962`). The AI is only a scenario trigger via generic calls (`taskmanagebuildings.cpp`, `builder.cpp`): no `ai*.cpp` is at fault.
- Fix applied at the user's request (who had explicitly authorised stopping the game in progress): backup of the state in `BACKUPS/2026-11-25-antiair-fix/`, `am force-stop com.max.port`, then two modifications without touching the `ai*.cpp` files or game data:
  - `unitinfo.cpp` (`SpawnNewUnit()`): just before `UnitsManager_DeployUnit()`, if the built unit is stationary and is not `CNCT_4W`/`ROAD`/`BRIDGE`/`WTRPLTFM`, remove the connector present on the utility tile via `UnitsManager_RemoveConnections()` then `UnitsManager_DestroyUnit()` (the `RemoveConnections()` will be removed after the crash described below). Cones/ribbons remain handled as before, and the cleanup no longer depends on the existence of a path.
  - `saveloadchecks.cpp` (`Defect151`): initially call `UnitsManager_RemoveConnections()` before destroying the connector, so that old repaired saves do not retain stale network links (call later removed, see below).
  - No guard added in `Access_IsAccessible()`: it could invalidate construction sites already planned by the AI. The AI was not modified (md5 of the four `ai*.cpp` verified before/after).
- Intermediate verification build via `WORKING/max/build-native-linux.sh`: stripped `.so` `c8352e25a911585c3e4f5e604641a079`, APK `df211558a87fc904c9e51e9cb161b412` (`/data/app/com.max.port-1.apk`). Restart without `FATAL`: five consecutive `MAXDBG plausibility summary ... Defect151=0 result=0` reports. The user confirms the mobile AA gun has the correct sprite; no other symptom reported.
- Clean final APK after confirmation: backup of the sources with logs (`sources-correctif-avec-logs/`) and of the intermediate APK (`APK-correctif-avec-logs.apk`) in `BACKUPS/2026-11-25-antiair-fix/`, then removal of the eight temporary `MAXDBG` / `TEMP DIAG` blocks without touching the fix. `.so` `5b6fb42467b4d324f0854f1494592bb7`, APK `06fe08f34689e129927f34e9dbf4ca57` installed via `adb install -r` -> `/data/app/com.max.port-2.apk` (md5 verified), data preserved (`settings.ini` 1 196 bytes, `SAVE10.DTA` 459 843 bytes). Restart without `FATAL`: `stdout.txt` now contains only the pixel-format line. This APK was later replaced after recurrence (see below).
- No game data touched, no flash or reboot. The archive `PACKAGING/sjgam-m27-custom-20261124.zip` remains unchanged for now: it still contains the old APK `064460b9...` and must be rebuilt once with the final APK `c6f5a778...` after the user tests.

- After this APK, the user again reports the `!corruption_detected` assertion at end of turn. Backup of the final APK (`APK-final-avant-diag.apk`, md5 `06fe08f3...`), of `stdout.txt` and of `SAVE10.DTA`, then reinstallation of the diagnostic APK `df211558...` (fix + logs) on confirmation. Reproduction: five clean reports then `Defect151 connector at [73,12] overlaps unit type=35 at [73,12]` and `Defect151=1 result=1`. The first patch covered "tower deployed on connector" (original: "tour déployée sur connecteur"), but not "connector deployed on an already present tower" (original: "connecteur déployé sur une tour déjà présente").
- Read-only analysis: `MakeConnectors()` (`taskmanagebuildings.cpp:1329-1357`) plans connectors without rechecking occupancy at deployment; `Access_IsAccessible()` is only tested when issuing the order; `GameManager_DeployUnit()` (`game_manager.cpp:1000-1005`) deploys directly for quick-build/network. No `ai*.cpp` is at fault, no game data is modified.
- Complementary fix: symmetric guard in `unitinfo.cpp` before `UnitsManager_DeployUnit(CNCT_4W)` (refuse if the tile already contains a stationary unit other than `GROUND_COVER`/`CONNECTOR_UNIT` and other than the transient state `ORDER_IDLE` + `ORDER_STATE_BUILDING_READY`, with `Access_DestroyUtilities()` then `ClearBuildListAndPath()`), and equivalent guard in `game_manager.cpp` before direct deployment.
- A first verification build `0f84fa4d255c9d33f6ade27925d8e330` caused a crash: tombstone in `UnitsManager_RemoveConnections(UnitInfo*)`, a missing network neighbour being dereferenced. The extra `RemoveConnections()` calls added earlier in `unitinfo.cpp` and `saveloadchecks.cpp` were removed; connector destruction remains that of the original game (`UnitsManager_DestroyUnit()` alone). The stdout, save and tombstone of the crash are kept in `BACKUPS/2026-11-25-antiair-fix/`.
- Diagnostic build `904dc0019996821de0dc9f9157eaa2bb` installed then validated by the user on a **new game** (rather than reloading the quicksave, as they suggested): 30 consecutive `Defect151=0 result=0` reports, no crash, fixed AA gun displayed correctly. Backup in `BACKUPS/2026-11-25-antiair-fix/`: `APK-diagnostic-valide.apk`, `stdout-diagnostic-valide.txt`, `SAVE10.DTA-diagnostic-valide`.
- Clean final APK after removal of the eight temporary `MAXDBG` / `TEMP DIAG` blocks (no residue): stripped `.so` `c84dd8078f0f0a4f45e088994a1ef74b`, APK `c6f5a77831898618fd69ccac9dcb2e21` installed via `adb install -r` -> `/data/app/com.max.port-2.apk` (md5 verified), data preserved (`settings.ini` 1 196 bytes, `SAVE10.DTA` 354 554 bytes). Restart without `FATAL`, `stdout.txt` now contains only the pixel-format line. Final backup: `APK-final-sans-logs.apk` and `sources-final-sans-logs/`. It is this APK, and no longer `06fe08f3...`, that must be included in the archive. No `ai*.cpp` modified (md5 verified), no game data modified, no flash or reboot.

## Notes on source divergences

- Two md5 values are associated with the 24/11 packaging: `4f85e77197d10c722355fa7cfec632c1` in the kernel log (presumably the `kernel.img` padded to 12 582 912 bytes) and `4ac17e4c7c14d56ae206dceb86361140` in the global log (final ZIP archive, 231 399 853 bytes, `unzip -t` OK). The latter identifies the chosen archive; do not confuse them.
- The MAX logs mention several successive "final mapping" versions on 23/11 and APK paths (`com.max.port-1.apk` / `com.max.port-2.apk`) that reflect reinstallations. The retained table is that of phase 7 (A=Enter, B=Back, X=Centre, Y=Change cursor, L1/L2=Zoom) and the current path after the last APK of 25/11 (`c6f5a778...`) is `-2.apk` (intermediate builds alternated between `-1.apk` and `-2.apk`).
- The header dates of some logs are out of date relative to their content; the dates given here are those of the entries themselves, not those of the headers.
- The first ADB/root access, before 07/10, is not reported in the logs: only the later state is documented.
