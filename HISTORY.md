# Historique de la console SJgam M27 / Sjgam-m27-Unclock

Ce document retrace le travail réalisé sur la console SJgam M27 (Android 4.4.2 / API 19, RK3188, écran 1024×600) : déverrouillage ADB/root, noyau et pilotes, souris firmware, rétablissement de `Settings.apk` et du clavier virtuel, frontend, émulation, puis packaging d'une archive firmware reproductible. Le port de M.A.X. est un projet annexe à la console : il est rapporté en Annexe A, son diagnostic du 25/11 en Annexe B.

Il est volontairement **dépouillé** : aucune donnée originale de jeu, aucun binaire, aucun dump, aucune image ROM et aucun secret n'y sont copiés. Seuls les chemins, dates, décisions, codes de touches et checksums nécessaires à la compréhension sont conservés.

Sources : les journaux de travail `STATE.md` / `STATE.MD` tenus sur la machine de développement. Les journaux bruts ne sont pas versionnés : ils contiennent un secret local de keystore. Quand une information manquait ou était ambiguë dans un journal, elle est signalée comme telle plutôt que reconstituée.

## Règles de conservation

- Aucune donnée originale de M.A.X. ne doit être embarquée dans l'APK ni dans le pack ; le joueur fournit sa propre copie sur carte SD (`/mnt/external_sd/MAX`).
- Les quatre sources `ai*.cpp` du port M.A.X. ne sont pas modifiées : l'IA adverse d'origine est préservée.
- Cible figée : Android 4.4 / API 19 + NDK r25c + SDL2 (`v0.7.2`) ; pas de ROM ≥ 5.0 pour cette console.
- Souris firmware conservée : `rk29-mouse`, `mouse_daemon`, `com.sjgam.mouseosd`. Pas de souris virtuelle dans le port, pas de chord R1+R2 ajouté, pas de pont touch→souris, stick droit laissé libre.
- `/system/usr/keylayout/rk29-keypad.kl` n'est touché qu'en dernier recours ; md5 de référence `29fb5f55eba641d01de7a11fd6120396`.
- Aucun loader Rockchip n'est distribué ; aucun flash ni installation sur la console sans confirmation explicite.
- Format retenu : archive flashable Rockchip contenant tout ce qui a été validé, hors données de jeu. Une seule reconstruction finale, après les tests utilisateur.
- Le port M.A.X. n'est pas le sujet central : voir Annexe A et Annexe B en fin de document.

## Vue d'ensemble — jalons

| Date | Jalon |
| --- | --- |
| 05–08/07/2026 | Dumps usine des partitions et de la carte SD, lecture seule. |
| 07/10 | Premier flash noyau (build9), ADB root USB opérationnel. |
| 08/10 | Découverte des vrais timings de dalle (600×1024 portrait) ; build13 net. |
| 10/10 | Cause racine des contrôles trouvée : `rk29-keypad`, pas le pad USB ; build17 valide 24 boutons. |
| 11/10 | LED des joycons ; `Settings.apk` rétabli en réactivant le réseau dans `services.jar`. |
| 12/10 | Clavier Leanback pilotable au D-pad : `framework.jar` corrigé, touches agrandies. |
| 12–13/10 | HDMI/RGA, builds M19→M33, rotation logicielle, logo paysage. |
| 13–14/10 | Mode souris R1+R2 + stick droit, `mouse_daemon`, Mouse OSD, démarrage auto. |
| 19/10 | Mesures DVFS GPU/CPU, aucune modification persistante. |
| 23/10–03/11 | Dumps SD archivés. |
| 30/10 | Noyau M38 : correction du reboot à la première veille ; correction de la numérotation des partitions (noyau = p2). |
| 01/11 | Remap matériel A/B/X/Y dans le keylayout système. |
| 02–04/11 | Cores libretro ARM32 installés et validés (mGBA, Snes9x, MAME 2003 Plus). |
| 06/11 | `sjgam_touchd` puis désactivation finale du pont clic→tactile. |
| 09–24/11 | Frontend SJGAM itéré jusqu'au menu APKS 7 colonnes retenu. |
| 20–24/11 | Stubs Google/Vending ; billing conservé pour Riptide GP2 ; 5 stubs désactivés. |
| 23/11 | Service `search` restauré dans `services.jar` (MX Player) ; port de M.A.X. en cours (annexe). |
| 22–24/11 | Port de M.A.X. : JNI/chemins SD, rendu 640×480 étiré, mapping, IA d'origine, watchdog audio, langue française. |
| 24/11 | Archive firmware `sjgam-m27-custom-20261124.zip`, nettoyage console, mappings DraStic, dépôt GitHub créé. |
| 25/11 | Diagnostic puis correction du chevauchement connecteur/anti-air dans l'autosave de M.A.X. (détail en Annexe B). |

## Partie I — Noyau, pilotes et souris (07/10 → 24/11)


Source : `/var/mnt/stockage/sjgamM27/kernelbuild/STATE.md` (3393 lignes, lu intégralement). Notes techniques concises ; aucune donnée de jeu ni contenu binaire repris (seuls les chemins et les checksums sont conservés). Les points non relatés dans STATE.md sont signalés comme tels.

### Phase 0 — Antécédents, dumps usine et matériel (dates partielles / non daté)

- [dumps 05–08/07/2026] `Original memory dumps SjgameM27/system.bin` = système usine de référence ; `android_image/system.bin` = dump postérieur (outils root ajoutés), non utilisable comme référence ; `build.prop` et `framework-res.apk` identiques dans les deux.
- [non daté dans STATE.md] **Premier accès console / ADB root** : non relaté. État observé ensuite : `adbd` root, `ro.secure=0`, `ro.debuggable=1`, SELinux Disabled, aucun binaire `su` pour les apps.
- [non daté] dumps/récupération : `android_image/kernel.bin` (noyau usine #1857), `android_image/recovery.bin`, `work/recovery_kernel.img`, `payload1857.bin` / `payload562.bin` (extraits désassemblés), `boot_current_20261007_190957.img` (p4).
- [non daté] board confirmée = `arch/arm/mach-rk3188/board-rk3188-ds1006h.c` (chemins présents dans `payload1857.bin` et `payload562.bin` ; `dmesg_562.txt` trace `--->board-rk3188-ds1006h.c`). Le seed defconfig `rk3188_ds1006h_android-4.4` était donc correct.
- [non daté] dalle CDO7018010-30A / 2025082701982 (7") ; retour utilisateur 07/10 : physiquement endommagée par flexion pendant le démontage (brume, stries), FPC reseaté sans effet.
- [non daté] WiFi : aucun chip embarqué ; clé USB utilisateur identifiée **RTL8188ETV `0bda:0179`**, supportée nativement par `drivers/net/wireless/rkusbwifi/rtl8188eu` ; option `CONFIG_RTL8188EU=y` envisagée mais non retenue à ce stade.
- [non daté] contrôleur USB `0810:0001` « Twin USB JoysticK » (`drivers/hid/hid-pl.c`) : 2 interfaces / 2 rapports, 2 nappes FFC ; plus tard identifié comme récepteur de manettes sans fil (flux neutre normal en l'absence de manette).
- [non daté] clé plateforme ROM personnalisée (SHA-256 `71:5E:B4:D7:4F:69:A4:26:A5:CC:8E:57:C6:D8:B5:28:CC:49:67:D0:B3:50:20:05:AA:AF:23:AA:EB:50:75:90`) → ne pas resigner les APK `sharedUserId=android.uid.system`.

### 07/10 — Premier flash, boot build9, horloges et cause écran noir

- [2026-10-07] flash recovery : p3 = `kernel_MID.img` (build9, relu md5 `bf5cec5439c5c29f08b25ba11a529f7e`), p4 = backup usine (relu md5 `5da0f206ca49f13bb5b7165ea73f82c2`), p10 (kpanic) remis à zéro ; sauvegarde `boot_current_20261007_190957.img`.
- [2026-10-07] boot build9 validé : ADB USB fonctionnel (`adbd` root, `/dev/android_adb`, gadget `f_adb`), chaîne d'affichage sans erreur (lcdc1 primaire, `load_screen`, win0, backlight, veille/réveil), `fb0` launcher ATV actif, `sys.boot_completed=1` ; dalle physiquement morte.
- [2026-10-07] séance horloges/pixclock : stock #1857 ne contient ni 528 MHz ni 71 MHz ; littéral 1 GHz en `0xc0459650` ; offsets `rk29fb_screen` confirmés (pixclock@20, hs_tx_clk@88) ; dump PHY actif (prediv=4, fbdiv=187).
- [2026-10-07] cause écran noir des builds antérieurs : aucun panneau MIPI dans `.config` → `only mipi dsi lcd is supported` ; création de `drivers/video/rockchip/screen/lcd_M27_mipi.c` (panel MIPI-DSI via RK616, 4 lanes, 203 commandes DCS issues du désassemblage #1857).
- [2026-10-07 22:07:37 CEST] build11 : `gpio_set_value` → `gpio_direction_output` dans `rk_lcd_init` / `rk_lcd_standby` (GPIO 166 reset, 168 power, déjà réclamés par la batterie et `rk_fb_io_init`) ; Image md5 `8f331cf742719a2383f00a31138f53c7`, conteneur `kernel_M11.img` md5 `aaf7e276cde7ccddfe2d2c564d10de2f`.
- [2026-10-07] référence usine conservée : `android_image/kernel.bin` md5 `ed8d06450140e40f768f5e4f089eda5d`.

### 08/10 — Révélation des timings, builds 12/13, diagnostic USB

- [2026-10-08] **révélation** : la dalle est 600×1024 (portrait), pas 1280×800 ; `set_lcd_info` stock #1857 désassemblé : H 60/80/1, V 25/35/1, DCLK 80 MHz, LCDC_ACLK 300 MHz, MIPI HS 560 MHz, 4 lanes, DCLK_POL=1. Les builds 11 et antérieurs utilisaient les timings `ds1006h` (1280×800/71 MHz/528 MHz) : divergence réelle expliquant l'écran noir.
- [2026-10-08 19:07:48 CEST] build12 : timings stock exacts dans `lcd_M27_mipi.c` ; Image md5 `b5bacc5d844da453514f108ef16d9a47`, conteneur `kernel_M12.img` md5 `6f40aba58fe0b6971282729c9ea27bef` ; vérification binaire des immédiats `set_lcd_info` identicales au stock.
- [2026-10-08 21:10] build13 flashé : patch `rk_fb.c rk_fb_set_par` alignant `xres_virtual` sur `ALIGN(600,16)=608` (stride 2432, cohérent avec HWC/IPP) ; fichier patché md5 `9c8b8bd72907434f880da0e1088d7eb6` ; Image md5 `3d9ad62f43380426343ea57ece6538e9`, `kernel_M13.img` md5 `90159bafc94c797a2b7346d011b395dd` ; backup p4 `boot_current_20261008_211011.img`.
- [2026-10-08] validation build13 : logo/loading/launcher nets et droits, `virtual_size=608,3072`, `stride=2432`, pic de corrélation de pitch à 2432 ; build13 = noyau de référence du moment.
- [2026-10-08 soir] diagnostic système : service `connectivity` absent du ServiceManager (licence Play bloquée) ; noté comme problème système transversal, pas un défaut noyau.
- [2026-10-08 soir] diagnostic USB : `usb20_host` → `0810:0001` (`hid-pl.c`) ; keylayout `Vendor_0810_Product_0001.kl` chargé ; captures `getevent` sur event2/event3 = zéro événement ; rapports HID figés `01/02 80 80 80 80 0f 00 00` ; le même envoi OUTPUT `00 00 00 00` existe dans le stock #1857 → pas une régression.
- [2026-10-08] outil `work/poke_usb_min.c` → `work/poke_usb` (ARM freestanding, syscalls `swi`) ; 1er essai SET_REPORT = EBUSY (`usbfs: process did not claim interface 0`) → correction à prévoir : `USBDEVFS_CLAIMINTERFACE` avant SET_REPORT.

### 09/10 — A/B stock vs builds, hypothèse gadget USB

- [2026-10-09] découverte majeure : sous noyau **usine #1857**, tous les contrôles fonctionnent (vol±, navigation, A/B, sticks) ; builds M10→M13 : chip énuméré mais aucun bouton → régression noyau, pas matériel.
- [2026-10-09 22:43] reflash p3 = stock `ed8d0645...`, p4 = `5da0f206...`, p10 zéro, relectures md5 OK ; backup `boot_current_20261009_224358.img`.
- [2026-10-09] delta candidat = gadget USB/OTG device : build13 avec `USB_GADGET`, `USB_G_ANDROID`, `DWC_OTG_DEVICE_ONLY`, `DWC_CONN_EN` ; stock sans aucune chaîne `android_usb`/gadget.
- [2026-10-09 23:2x] build14 (kernel #12) : OTG `HOST_ONLY`, gadget désactivé ; Image md5 `59bb1996be6489f53648adbe10547f45`, `kernel_M14.img` md5 `ba4e8d8265907abdfe7c5615c6e59e5e` ; chaînes gadget absentes. Erratum : 1er build14 corrompu par un script python (config en doublon, choice en conflit, prompt `USB20_OTG_EN`) → restauration `.config.build13.bak` md5 `47aa2cc9...`.
- [2026-10-09] résultat build14 négatif : contrôles toujours morts → **hypothèse gadget/OTG réfutée**. Le gadget pourra revenir pour l'ADB.
- [2026-10-09] nouveau lead : `recovery.bin` = même noyau #1857 que p3 mais avec gadget (ADB recovery) → configs usine différentes ; `extract-ikconfig` échoue sur stock et recovery (pas de `IKCFG_ST`).

### 10/10 — GPIO usine, cause racine `rk29-keypad`, LED joycon

- [2026-10-10] balayage/diff des `gpio_request` du board init : stock `gpio_request(178,"usb_power")` + output HIGH (178 = `RK30_PIN0_PC2`), `gpio169 "led"` + output LOW + `gpio_export`, `gpio264 "sd_det"`, `gpio190 "HDMI"` ; build12 n'avait aucune occurrence de 169/178.
- [2026-10-10] piste SDMMC1/PB1 réfutée : `CONFIG_ARCH_RK29` absent et `USE_SDMMC1_FOR_WIFI_DEVELOP_BOARD` off → `set_iomux` jamais appelé pour SDIO1.
- [2026-10-10] build16 : patch idempotent `M27-FACTORY-GPIO` dans `machine_rk30_board_init` (169 led LOW, 178 usb_power HIGH + exports) ; board md5 `f5af0ac7b680e307b454dedfc1dabab3`, Image md5 `7b3d51abd561e7cb3c52035499753c36`, `kernel_M16.img` md5 `6e1ea73e0e3e7e6de949e3cf044ba7c1`.
- [2026-10-10] **cause racine trouvée** : table `rk29_keys_platform_data` @`0xc0a77f5c` du noyau usine = **24 boutons GPIO** (`up/down/left/right`, 4 directions de chaque stick, A/B/X/Y, L1/L2/R1/R2, SELECT/START, vol±). Les contrôles intégrés passent par `rk29-keypad` (`event0`), pas par le pad USB. `l1 = GPIO160` = `power_hold`, piloté en sortie par notre board au lieu d'être une entrée.
- [2026-10-10] preuve live sous M16 : GPIO tenus 160, 164, 170, 202, 205, 207 ; `rk29_keys` échoue en bloc (`goto fail2`) dès qu'un `gpio_request` échoue ; `CONFIG_CT36X_TS=y` retenait GPIO207 (touch absent du M27, spam i2c-2).
- [2026-10-10] build17 (kernel #14) : table 24 boutons exacte, `POWER_ON_PIN`→`INVALID_GPIO`, `dc_det_pin`/`LCD_CS_PIN`/`LCD_PWR_PIN`→`INVALID_GPIO`, `CT36X_TS` off ; board md5 `e41935379251221e78c6100c1597cc97`, Image md5 `b4564eb99fc74dea7f0cf6d140ef9634`, `kernel_M17.img` md5 `87d3561344f1a96b1a28b163ac5f0011` ; journal `kernelbuild/build17.log`, script `build17.sh`.
- [2026-10-10] flash M17 : p3 = `87d356...`, p4 = `5da0f206...`, p10 zéro, relectures OK ; **contrôles 100 % fonctionnels** validés par l'utilisateur (navigation, A/B/X/Y, L/R, sticks, start/select, vol) ; aucun échec `gpio_request`, plus de CT36X.
- [2026-10-10] LED : `gpio169` = maître d'alimentation du bloc LED/RGB (on/off simple ; motif tournoyant interne au module). Patch board (md5 `9f3d65cb04908f35a5613d833ceaf7df`) : `leds-gpio` « joycon-led », `default_state=ON`, retrait du `gpio_request(169)` du bloc `M27-FACTORY-GPIO`.
- [2026-10-10 soir] build18 (kernel #15) : base build13 + `CT36X_TS` off ; tentative `LEDS_TRIGGERS=y` abandonnée (symbole NEW `NETFILTER_XT_TARGET_LED` bloque `silentoldconfig`) ; Image md5 `c49ac04ffca696967e0dc90442ac87a2`, `kernel_M18.img` md5 `3f8a38de847e044fc8b72063e51cf64f`.
- [2026-10-10 soir] hook `/system/etc/install-recovery.sh` : section LED `chmod 666` sur `/sys/class/leds/joycon-led/brightness` (root au boot via le service init `flash_recovery`) ; md5 hook poussé `bb9a76f9164f87da1874844df4f0509b`, backup console `.bak`.
- [2026-10-10 nuit] flash M18 : p3 = `3f8a38de...`, p4 = `5da0f206...`, p10 zéro ; `/sys/class/leds/joycon-led` présent, `brightness=255`, droits 666, log `led chmod rc=0 val=255` ; toggle ON/OFF validé, anneaux RGB inclus.

### 11/10 — Application LED, hook boot, services système et clavier virtuel

- [2026-10-11] app « SJGAM Control » (`com.sjgam.control`) : LED ON/OFF + clignotement + « appliquer au démarrage », luminosité écran via `WRITE_SETTINGS` ; empaquetage du `classes.dex` à la racine du zip (piège `INSTALL_FAILED_DEXOPT` corrigé) ; installée puis promue en app système `/system/app/SJGAMControl.apk` (md5 `218818287c0500ec985bdaba42bb5b7f`), données `/data/data` conservées.
- [2026-10-11] hook LED épuré : plus de clignotement ni case « appliquer », état toujours mémorisé (`led_on`) et appliqué dès le boot (~6 s) par lecture des prefs dans le hook ; md5 hook `6f6d554d04facc9ea7f9b620289e07d5`, puis `991d65b124452debaa23c130b8b6c428` (logs prefs).
- [2026-10-11] correctif « l'app rallumait les LED » : `LedService` sans action, lit toujours la pref ; `onResume` resynchronise le Switch et n'écrit que si l'état matériel diffère ; `saveEnabled=false` ; `BootReceiver` sans action ; cycles ON/OFF validés par `input tap` + reboot.
- [2026-10-11] réseau restauré côté système : `services.jar` avait `config.disable_network=true` et `const/16 v86,0x1` dans `ServerThread` → patch smali à `0x0`/`false`, jar repoussé, dalvik-cache purgé ; `connectivity`, `wifi`, `network_management`, `netpolicy`, `netstats` de nouveau présents.
- [2026-10-11] IME : `LeanbackIme.apk` (`com.google.android.leanback.ime`, md5 source `234d5eeebfbc33763b3690a0d1d592bc`) installé ; cause du clavier non pilotable finalement identifiée le 12/10 (voir ci-dessous).

### 12/10 — Clavier D-pad, HDMI, builds M19→M28

- [2026-10-12] **cause racine clavier** : `ViewRootImpl$ImeInputStage.onProcess` neutralisée (stub `return 0`) dans `framework.jar` → touches jamais proposées à l'IME ; restauration du code AOSP KitKat via `access$700` (`isInLocalFocusMode`) ; piège corrigé : condition inversée `if-eqz` → `if-nez`.
- [2026-10-12] validation clavier : logs `KBDTEST`/`KBDPATCH`, LEFT/RIGHT déplacent la sélection, DPAD_CENTER valide la touche ; `framework_clean.jar` md5 `41a4adb67874252b71bf33827b894e56` poussé, logs retirés ; `LatinIME` supprimé (backup md5 `68e8367b9290e30413d126dcde1c0c89`), seul Leanback reste IME.
- [2026-10-12] clavier agrandi par patch binaire de `resources.arsc` (`key_height` 28→44 dp, `key_width` 28→60 dp) ; outil `/tmp/opencode/leanback4/patch_arsc.py`.
- [2026-10-12 soir] HDMI strié : HWC écrit 600×1024 stride 608, miroir `lcdc0` lit 1024 px/ligne (branche transpose) → cisaillement ; correctif temporaire `ro.sf.hwrotation` 90→0 dans `/system/build.prop` (backup `build_prop_live_20261013_portrait.bak`).
- [2026-10-12 20:43] build19 (kernel #16) : timing dalle inversé en 1024×600 paysage natif ; Image 7 311 396 o, `kernel_M19.img` md5 `e3060a2b875da03ded9576759b1ecee3` ; TV nette mais image tournée ; rotation1 confirmée venir de `libsurfaceflinger.so` via `ro.sf.hwrotation=90`.
- [2026-10-12 ~22:00] `ro.sf.hwrotation=0` appliqué : TV droite ; pertes de synchro HDMI d'abord attribuées à l'audio I2S (volumes mis à 0), puis au flux DSI de la dalle (test `echo 1 > /sys/class/graphics/fb0/blank` : 9 min sans perte).
- [2026-10-12 ~23:45] build M21 (kernel #19) : priorité HDMI exclusive (blank de la dalle au hotplug, garde anti-réveil) ; `kernel_M21.img` md5 `267bf109498d0c53e4444fa6dbf251c1`. M20 (dalle veille permanente) jamais flashé.
- [2026-10-12 ~22:50] test M21 : dalle allumée en lignes horizontales et blank sans effet (`fbops->fb_blank` court-circuite le notifier de rétroéclairage).
- [2026-10-12 22:51] build M22 (kernel #20) : retour au timing portrait 600×1024 ; `rk_fb_blank_panel()` appelle `fb_blank()` (fbmem) → `FB_EVENT_BLANK` → rétroéclairage coupé ; Image md5 `c822cbd163ba5642edd1da9b58fc4237`, `kernel_M22.img` md5 `606bd7fb7f2fd39ca996ebf7ef47c249` ; test : dalle proportionality OK, extinction au branchement HDMI OK, mais TV striée et instable.
- [2026-10-12 23:10] build M23 (kernel #21/#22 après rebuild) : miroir cohérent (fb2 = géométrie exacte de fb0, plus de transpose, letterbox) ; Image md5 `36d0428841466a37b8de7505e89fb30f`, `kernel_M23.img` md5 `56ff1cb8d5a99c69b5a30a82e63de1f1` ; 1er flash n'a pas pris (boot #20), 2e tentative OK (boot #22 23:04:30).
- [2026-10-12 23:15] test M23 : TV claire mais contenu tourné de 90°, `mOverrideDisplayInfo rotation=1` forcé par le launcher ATV ; rk3188_lcdc sans rotation matérielle.
- [2026-10-12] builds M26/M27/M28 : tentatives de « calage » paysage natif (MADCTL `0x36=0x20`, puis fenêtre CASET/PASET `0x2A/0x2B`) ; md5 `kernel_M26.img 3d53d9e98eb64a9d408faf4b1eacad79`, `kernel_M27.img 467671b5d690915c2009de3d6ac7b963`, `kernel_M28.img c4884d78e9bff21b8a1a9efb9bc188f1` ; toutes donnent bandes/stries.

### 13/10 — M25/M29/M31/M32, souris firmware, boot auto, recovery

- [2026-10-13 00:05] décision « tout paysage natif » abandonnée (M24/M27/M28 en bandes) ; retour au plan M23 portrait + rotation RGA côté HDMI.
- [2026-10-13] build M25 (kernel #28) : `rk_fb.c` + `dma_alloc_coherent` d'un rotbuf 1024×600, `m25_rga_rotate_to_landscape()` (RGA 90° CCW, `sina=-65536`), hotplug HDMI → miroir paysage plein écran ; `kernel_M25.img` md5 `2728bee87ba7a30ba73d74b6ed5750e7`, Image md5 `bf28d4a8ce19a1f55251e9c71bea6956` ; flashé, **dalle nette** validée ; logo/loading encore tournés.
- [2026-10-13] carte des 24 entrées `rk29-keypad` complétée : joycon droit = `abs_Y/A/X/B` (codes 288/289/290/291) et boutons ABXY = `key_*` (mêmes codes) → indistinguables ; aucun axe de stick remonté ; `/dev/mem` = Bad address, pas de `CONFIG_DEBUG_GPIO`.
- [2026-10-13 14:12] build M29 (kernel #29) : pivot CW du logo `logo_linux_clut224.ppm` (formule `fb_rotate_logo_cw`, 480×480, 222 couleurs), `.c` régénéré par `pnmtologo`, zéro code noyau ; `kernel_M29.img` md5 `2316d6256e91d6ec6d1be4f38d62eb6b` ; vérification binaire : seules 22 lignes hors zone logo diffèrent de M25.
- [2026-10-13 14:2x] build M30 (fallback logo CCW) : `kernel_M30.img` md5 `3d74635bd248e07087d3e0aa39a6bddf`, non nécessaire.
- [2026-10-13 17:52] flash M29 : backup p4 `boot_current_20261013_175239.img` md5 `5da0f206...`, p3 relu `2316d625...`, p4 relu `5da0f206...`, p10 zéro ; **boot logo droit en paysage validé** ; base courante = `build29.sh`.
- [2026-10-13 18:41] `ro.sf.hwrotation=90` dans `/system/build.prop` (une seule ligne, md5 `65a16bdb3ccedd938aedbeadcf57e744`) : loading à plat et ATV en paysage direct ; voie ramdisk abandonnée après corruption des 1024 premiers octets de p4 (`boot_hwrot90.bin` md5 `24e87b9a2c3381f0e2922468b9040310`, forensic `boot_p4_corrupt_20261013.img` md5 `c16300c303b7cd58ba8a3d37a5c997af`, restauration `boot_current_20261013_182907.img`).
- [2026-10-13] build M31 (kernel #31) : recalcul `vir_stride` dans la branche M25 de `pan_display` ; `kernel_M31.img` md5 `ce2fcb478a907cf198314ba600144e76` ; validé utilisateur (« dalle, logo et HDMI propres et plein écran »).
- [2026-10-13] build M32 (kernel #32, diagnostic) : printk lcdc `BUS_ERR`/`WIN0_EMPTY`, attribut sysfs `hdmi_rga`, printk backlight ; `kernel_M32.img` md5 `d4f8e518e6a0c2167e8f82deddbd3d0c` ; Image md5 `99db0faab22d0c376554c1f75f4f1eb9`.
- [2026-10-13 20:08] flash M32 : backup p4 `boot_current_20261013_200856.img` md5 `5da0f206...`, p3 relu `d4f8e518...`, p4 relu `5da0f206...`, p10 zéro.
- [2026-10-13] tests M32 : rétroéclairage réellement coupé (`eff=0`), aucun `BUS_ERR`, `hdmi_rga=0` ne change rien → **RGA, audio, DMA et rétroéclairage écartés** ; « saut » = liaison HDMI externe (adaptateur HDMI→VGA / vieil écran), rebranchée puis stable en HDMI direct ; arbre source restauré en M31 (backups `*.M32diag.bak`).
- [2026-10-13 21:48] build M33 (kernel #33) : remap `key_button[]` pour rendre les sticks distinguables (gauche 183–186, droit 187–190) ; board md5 `5231ca28a67aeaf12b82a4f78a1c4d64`, Image md5 `10da2ffb454188952e3f5029f74db836`, `kernel_M33.img` md5 `7e2a00cd753a605c5b09f54a1233e1be` ; flash p3 OK, backup `boot_current_20261013_214755.img` (md5 `5da0f206...`).
- [2026-10-13] R1/R2 physiques confirmés : R1 = 295 (`BTN_BASE2`), R2 = 293 (`BTN_PINKIE`) ; keylayout usine inversé → corrigé sans flash.
- [2026-10-13] **mode souris validé** : bascule R1+R2 maintenus ≥ 3 s, stick droit = déplacement, R1 = clic gauche, R2 = clic droit ; `mouse_daemon` userspace lit `event0` et crée une souris `uinput` (`CONFIG_INPUT_UINPUT=y`) ; source `mouse_daemon/mouse_daemon.c` (v10), binaire `.arm` md5 `51f03bb517873e46a3b10722d274bb03` ; `POINTER_KEEPALIVE=1` (alternance +1/−1 px / 2 s).
- [2026-10-13] keylayout `/system/usr/keylayout/rk29-keypad.kl` (md5 `57edc0c05745fc5c7d5b352dbde29bb8`) : 183–186 → `DPAD_*`, A(290) → `DPAD_CENTER`, B(289) → `BACK`, X/Y non mappés, R1/R2 corrigés.
- [2026-10-13] Mouse OSD `com.sjgam.mouseosd` : poll de `/dev/sjgam_mouse` (0666) et Toast « Mouse Mode ON/OFF », autostart OK.
- [2026-10-13] curseur : layer SurfaceFlinger « Sprite », survit à `UI_DEV_DESTROY` ; patch de `/system/lib/libinputservice.so` md5 `a6b2466483cd3b9b4c8d28bc27cb8546` (origine `f537f6f472379ba407110ae26d94e568`) : `fadePointer` neutralisé en `0x26758`, timer d'inactivité 15 s → 3 s en `0x31d78` ; sauvegardes device et PC.

### 14/10 — Démarrage auto souris, umask, bootloader/recovery, inventaire système

- [2026-10-14] démarrage auto : hook `install-recovery.sh` v3 (md5 `caa2e6a1de23428f81603a0b9833e7f4`) termine par `exec /system/bin/sjgam_mouse.sh` (md5 `fff9645ffd367c5d2b5876d0e2c0477c`) ; superviseur attend `/dev/uinput` et `event0`, relance le démon 3 s après chute, journal `/data/local/tmp/mouse_daemon_autostart.log` ; vérifié `ppid=1`, `init.svc.flash_recovery=running`.
- [2026-10-14] correctif umask : `/dev/sjgam_mouse` créé en 0600 par init (`umask 077`) → OSD incapable de lire ; ajout `umask 0` dans `sjgam_mouse.sh` (nouveau md5 `02d6a27432503e707a4b8d7cdcfa1e49`) ; validation « impec ca fonctionne nikel ».
- [2026-10-14] incident « pas de Toast après reboot » = faux problème : bascule effectuée 3 s avant le démarrage de l'OSD (`BOOT_COMPLETED`) ; logique `last=-1` volontaire ; validation finale après affichage de l'interface.
- [2026-10-14 soir] découverte : Vol- + USB + ON = **mode loader Rockchip** (`2207:310b`), ni recovery ni Mask ROM ; `rkdeveloptool db` échoue (« device does not support »), `rl/wl` directs en root fonctionnent (LBA = secteurs 512 o) ; eMMC SAMSUNG 3776 Mo / 7 733 248 secteurs.
- [2026-10-14 soir] flash recovery usine via `flash_recovery_rockchip.sh` (sudo) : LBA `0x12000`, longueur `0xE000` ; p4 avant `1dbf15db7d5811e8b74581b3bf03502b` (notre recovery v3), après `7aac2177533953c012b61de748d55f9d` ; sauvegarde `recovery_p4_avant_rockchip_20261014_205715.img`, parameter `parameter_device_lu_20261014.bin`.
- [2026-10-14 soir] **entrée en recovery trouvée** : le hook v3 écrit `boot-recovery` à l'offset 0 de `misc` (`print -n 'boot-recovery' > $BCB`), lu par UBoot au boot suivant ; `RECOVER_KEY` commenté dans `parameter` ; la recovery d'usine attend une touche puis timeout → retour automatique au système.
- [2026-10-14] recovery v4 préparée (non flashée) : image usine intégrale + `socket adbd stream 660 system system` dans le service adbd ; `work/recovery_sjgam_v4.img` md5 `99f5716960baa0e8948de67792ac7a02`, script `work/mk_recovery_v4.py`.
- [2026-10-14] numérotation by-name en boot normal figée : `misc=p1 kernel=p2 boot=p3 recovery=p4 backup=p5 cache=p6 userdata=p7 metadata=p8 kpanic=p9 system=p10 user=p11` (sous `/dev/block/platform/emmc/by-name/`).
- [2026-10-14 nuit] comparaison au système usine : retrait des 6 GApps, puis de reliquats AOSP (`CalendarProvider`, `WallpaperCropper`, `DownloadProviderUi`) ; aucun impact noyau.

### 19/10 — Horloges GPU/CPU (mesures, pas de modification noyau)

- [2026-10-19] GPU : `/proc/clocks` `aclk_gpu` = 132 MHz au repos, 445,5 MHz sous charge GL (max table), retour 297 puis 132 → **DVFS GPU fonctionnel** ; la note antérieure « DVFS non régulé » était inexacte (c'est la tension qui n'est pas pilotable, seul un régulateur « dummy » est exposé).
- [2026-10-19] CPU : saturation 1/2/3/4 threads → 1200/1200/1200/1008 MHz ; `temp_limit_freq` suit le nombre de cœurs occupés ; 1416 MHz inatteignable (rails ACT8846 fixés à 1,0 V pour `vdd_cpu`, table DVFS réclamant 1,225–1,3 V) ; aucune transition cpufreq (`total_trans=0`).
- [2026-10-19] aucune modification persistante ; `retroarch-core-options.cfg` restauré (md5 `e35d00a6603c02a126adb483b6f67d86`).

### 30/10 — Build M38 : correctif reboot à l'extinction d'écran

- [2026-10-30] symptôme : redémarrage automatique à la première veille ; `/proc/last_kmsg` : NULL pointer dereference @`0x4`, `PC=sram_gpio_set_value`, `LR=act8846_device_suspend+0x20` (Oops 5).
- [2026-10-30] cause : sur PCB r10v2.3 le bus i2c-1 est muet (0x5a/0x51 sans ack) → `act8846_probe()` sort avant `act8846_set_init()`, `pmic_vsel.base=NULL` → panic au suspend ; le PMIC ne sert qu'au réglage de tension et au bouton power (rails fixes).
- [2026-10-30] patch `arch/arm/mach-rk30/board-pmu-act8846.c` : `if (!pmic_vsel.base) return;` en tête de `act8846_device_suspend()` **et** `act8846_device_resume()`.
- [2026-10-30] **outils/compilateur** : kbuild incrémental impossible (`cmd_bounds` réécrit `include/generated/bounds.h` à chaque `make` → tout est recompilé ; GCC 16 casse sur les vieux sources). Contournement : recompilation de `board-rk3188-ds1006h.o` uniquement via wrapper `arm-eabi-4.6` + GCC 16.2.1 (`-std=gnu89 -Os`), puis chaîne de link manuelle `/tmp/opencode/link38.sh` (built-in.o → `.tmp_vmlinux1` → `scripts/kallsyms` → `.tmp_kallsyms2` → `vmlinux` → `System.map` vérifié par `cmp` → `Image`) ; `NM=` exporté pour `scripts/mksysmap` ; `init/version.o` recompilé après forcage de `include/generated/compile.h` en `#38`. Aucun NDK documenté dans STATE.md pour ce build.
- [2026-10-30] Image 7 311 396 o md5 `8adeb20a...` ; `kernel_M38.img` 7 311 408 o md5 `7f3b10641e5149d7c9dd6764247fe63f` ; bannière `Linux version 3.0.36+ #38 SMP PREEMPT Fri Oct 30 21:34:50 CET 2026`.
- [2026-10-30] **correction majeure des scripts de flash** : `TOOLS/flash_p3.sh` et `TOOLS/restore_good.sh` sont faux pour cette carte (ils écrivent le noyau en p3, le boot en p4 et zéroent le début de p10 = system). Le noyau réel est `mmcblk0p2` (12 Mo) ; p3 = ramdisk boot (KRNL + gzip → cpio), p4 = boot.img recovery (`ANDROID!`). Preuve : payload de p2 byte-identique à l'Image M33 ; payload de p3 décompressé en cpio (`070701`).
- [2026-10-30] flash réel depuis Android root, sans recovery : `busybox dd if=kernel_M38.img of=/dev/block/mmcblk0p2 bs=65536`, relecture md5 OK ; rollback `kernel_p2_M33_avant_M38_20261030.img` (12 Mo, md5 `3b7634b2...`).
- [2026-10-30] validation : boot #38 OK (~25 s), ADB OK ; piège de test = câble USB branché → wakelock `usb_pcd` interdit la veille profonde (`suspend_time` vide) ; test valide **câble débranché** : veille sans panic et réveil par n'importe quel bouton de jeu (le bouton power passe par le PMIC absent) ; RTC `hym8563` inerte (pas de réveil programmé).
- [2026-10-30] réglages finaux : `screen_off_timeout=180000`, `stay_on_while_plugged_in=7` ; `pm_test=none` ; patch présent dans l'arbre mais **non committé**.

### 01/11 — Remapping manette A/B (keylayout)

- [2026-11-01] découverte : les boutons physiques A/B/X/Y émettent les codes noyau **188/190/189/187** (`abs_A/B/X/Y`), pas 288–291 ; la validation antérieure par `sendevent(290)` était un faux positif ; L1/L2/R1/R2 = 294/292/293/295, SELECT/START = 296/297.
- [2026-11-01] `/system/usr/keylayout/rk29-keypad.kl` final (1077 o, md5 `e52538eccb02774919b1ff29de460474`) : 188 → `DPAD_CENTER` (A=OK), 190 → `BACK` (B=Retour), 189 → `BUTTON_X`, 187 → `BUTTON_Y`, 288–291 → F5–F8 (inutilisés), 292/293/294/295 → `BUTTON_L2/R1/L1/R2`, 296/297 → `BUTTON_SELECT/START` ; 183–186 restent `DPAD_*`. Redémarrage nécessaire (Android 4.4, pas de reload à chaud).
- [2026-11-01] configs émulateurs ajustées au nouveau mapping (RetroArch autoconfig/global, DraStic prefs, PPSSPP inis) ; tests physiques valides ; sauvegardes `/tmp/opencode/kl/backup_20261101/`.

### 06/11 — Restauration souris virtuelle, puis `sjgam_touchd`, puis désactivation tactile

- [2026-11-06] `start_touch` réactivé dans `/system/bin/sjgam_mouse.sh` (md5 `944eda69d702f87f6ba4b53d554da7c5`) après isolement joycon droit ; redémarrage à chaud via `setprop ctl.stop/start flash_recovery` (ne pas lancer depuis adb : le processus ne survit pas à la fermeture du shell).
- [2026-11-06] validation : `init → sjgam_mouse.sh → sjgam_touch_daemon.sh (event4 « rk29-mouse ») → mouse_daemon (event0)` ; `/dev/sjgam_mouse=1`, log `tap (520,285)` = clic R1 traduit en tap au curseur ; `/system` remis en ro.
- [2026-11-06 (suite)] traducteur natif `sjgam_touchd` (C, uinput direct, ~ms) remplace le shell (`getevent`+`input tap`, ~0,69 s) : `/system/bin/sjgam_touchd` md5 `2d64a01671502cab95d22572c373f462`, `sjgam_mouse.sh` md5 `9d561446d24e2cf9cb2e38daccead8f1`, fallback `sjgam_touch_daemon.sh` conservé ; projet `/var/mnt/stockage/sjgamM27/sjgam_touchd/`.
- [2026-11-06] device uinput « sjgam-touch » : `BUS_VIRTUAL`, ABS_X 0..1023, ABS_Y 0..599, ABS_PRESSURE, `BTN_TOUCH`, `INPUT_PROP_DIRECT` ; Sources Android `0x00001002`, DeviceType touchScreen.
- [2026-11-06] calibration obligatoire (dalle portrait 600×1024 tournée en 1024×600) : `x_brut=(599-y_ecran)*1024/600`, `y_brut=x_ecran*600/1024` ; vérifié (100,100)→(99.0,100.8), (900,500)→(899.4,501.0).
- [2026-11-06] comportement : clic R1 en mode ON = DOWN/UP, R1 maintenu + stick droit = drag & drop, R2 seul = clic droit sans tactile, chord R1+R2 < 400 ms = annulation, mode OFF = aucune injection ; logs `/data/local/tmp/sjgam_touchd.log` (rotation auto à 512 Ko).
- [2026-11-06 soir] **décision utilisateur : clic souris seul, tactile désactivé** (le tap perturbait la validation dans certains menus, nécessaire ailleurs) : `TOUCH=0` dans `sjgam_mouse.sh` (md5 `41f705f19843127c18ec58fe3c8197fe`) ; `sjgam_touchd` resté installé mais inactif (md5 `ead02066e3b537693e5f3cdf408306e2`) ; état final : `mouse_daemon` seul, device « sjgam-touch » absent, `/dev/sjgam_mouse=0`, `/system` ro. Réactivation : `TOUCH=1` + `ctl.stop/start flash_recovery`.
- [2026-11-06] note : bloc `APK2SD` de `install-recovery.sh` situé après `exec`/`exit 0` → jamais atteint (à traiter séparément).

### 20/11 — Correctif relâchement R1+R2 (clic parasite)

- [2026-11-20] après bascule ON/OFF, le premier doigt relevé produisait un clic parasite (gauche, ou droit = Back) : `sync_buttons()` n'inhibait que pendant `r1 && r2` enfoncés.
- [2026-11-20] correctif : `toggle_armed=0` comme verrou de sortie, clics ignorés jusqu'au relâchement complet des 2 touches + garde `RELEASE_GUARD_MS=120` ; log « chord lock: clic parasite ignore » ; source reconstituée `kernelbuild/mouse_daemon/mouse_daemon.c` (v10 + correctif), ancien binaire sauvegardé `/mnt/internal_sd/mouse_daemon.pre-release-lock`.
- [2026-11-20] reconstruction avec le wrapper GCC 16 (GCC 4.6 indisponible) : binaire 5444 octets au lieu de 37278 ; **validation utilisateur « tout marche nikel »**, aucun retour arrière.

### 24/11 — Inventaire packaging firmware, nettoyage et vérifications finales

- [2026-11-24] noyau actif confirmé : `/proc/version` = `3.0.36+ (wisit@bazzite) #38 SMP PREEMPT Fri Oct 30 21:34:50 CET 2026`.
- [2026-11-24] image de référence `kernelbuild/kernel_M38.img` 7 311 408 o md5 `7f3b10641e5149d7c9dd6764247fe63f` (copiée dans `PACKAGING/stage/`) ; config `rk3188-kernel/.config` md5 `2dcf5b0d272d9d30426584fc739eb27b`.
- [2026-11-24] partition boot `mmcblk0p3` md5 `5da0f206ca49f13bb5b7165ea73f82c2` (= dump usine) ; `/init.rc` du ramdisk contient `service flash_recovery /system/etc/install-recovery.sh` (ligne 576) : personnalisation dans `/system`, pas dans le boot.
- [2026-11-24] parameter `SPARE/parameter_device_lu_20261014.bin` md5 `db875b3140e427d9b9caf453d5564b76` ; `build.prop` md5 `65a16bdb3ccedd938aedbeadcf57e744`.
- [2026-11-24] souris : `/system/bin/mouse_daemon` déployé md5 `5fbeaecc3805a2a6de8c863b6916c631` (source + `build.sh` joints au stage) ; keylayout `rk29-keypad.kl` md5 `29fb5f55eba641d01de7a11fd6120396` ; `rk29-mouse.kl` / `Vendor_1234_Product_5678.kl` md5 `e539a9a9eccba38a195d0616e50e4ba8` ; `Vendor_0810_Product_0001.kl` md5 `034fdb78933fcff9be2765fc63770c4c`.
- [2026-11-24] archive flashable `PACKAGING/sjgam-m27-custom-20261124.zip` : `kernel.img` = `kernel_M38.img` complétée par des zéros jusqu'à 12 582 912 o, md5 archive `4f85e77197d10c722355fa7cfec632c1` ; `boot.img` usine et `recovery.img` actuel (`7aac2177...`) inclus tels quels ; `misc.img` capturé mais non écrit ; LBA `flash.sh` : kernel `0x4000/0x6000`, boot `0xa000/0x8000`, recovery `0x12000/0xe000`, system `0x282000/0x180000` ; aucun loader distribué, aucun flash effectué.
- [2026-11-24] nettoyage console : backup `DUMPS & SAVES/console-cleanup-20261124/` (135 fichiers, 25 Mo), 125 fichiers temporaires supprimés (`/data/local/tmp` 24,7 → 1,5 Mo) ; configs/logs actifs conservés (`sjgam_mouse_speed`, `sjgam_touchd.pid/.log`, `mouse_daemon.log`, `mouse_daemon_autostart.log`, `usbmark_hook`, …) ; noyau #38, keylayout et binaires système inchangés.
- [2026-11-24] keylayout/souris préservés lors des derniers réglages d'interface et d'émulation : aucun fichier système, keylayout ou binaire modifié ; keylayout resté à `29fb5f55...` ; « souris firmware (rk29-mouse / mouse_daemon / Mouse OSD) laissée intacte » (formulation du journal) ; Mouse OSD suffit aux appuis écran, aucune souris virtuelle ajoutée.

### Points non documentés ou ambigus dans STATE.md

- Premier accès à la console (ADB/root), acquisition initiale du root et toute la phase de récupération des partitions avant le 07/10 ne sont pas relatés : seul l'état ultérieur est documenté.
- Outils NDK/compilateur pour le build M38 : seul le wrapper `arm-eabi-4.6` + GCC 16.2.1 est documenté ; aucun NDK n'est mentionné pour le noyau (les références NDK du fichier concernent des builds d'applications/cœurs, hors noyau).
- « stick droit = souris firmware » : STATE.md ne décrit **aucun firmware dédié** au stick droit. L'implémentation documentée est (a) remap noyau M33 des codes GPIO, (b) démon userspace `mouse_daemon` + `uinput`, (c) OSD. Le récepteur USB `0810:0001` est piloté par `hid-pl.c`, identiquement au stock. Le sens exact du terme « firmware » dans la demande reste à clarifier.
- Numérotation des partitions : STATE.md signale une divergence entre recovery et boot normal pour les scripts `flash_p3.sh` / `flash_p3` historiques ; la correction définitive (noyau = p2) n'est consignée qu'à la section M38 (30/10). Les flashs antérieurs (M17→M33) sont décrits avec la convention recovery, sans qu'il soit possible de confirmer rétroactivement la cible exacte depuis ce seul fichier.
- Dates de build12 (19:07:48 CEST) et de quelques builds M14/M21 sont déduites de leur position dans le journal ; l'année n'est pas explicitement répétée à chaque ligne (2026 d'après le contexte).
- Aucune donnée de jeu ni contenu binaire n'a été inclus volontairement ; seuls les chemins, codes de touches, types de périphériques et checksums nécessaires à l'historique technique ont été conservés.

## Partie II — Système Android : `Settings.apk` et clavier virtuel (11/10 → 12/10, suite 23/11)

Ces deux chantiers sont au cœur de la console : ils rendent le système réellement utilisable sans dalle tactile. Ils ne concernent ni le port M.A.X., ni les jeux.

### `Settings.apk` : réimplémentation / remise en service (11/10, puis 23/11)

- **Symptôme** : l'APK `Settings.apk` de Kingston, placé dans `/system/priv-app`, s'installait bien (`codePath=/system/priv-app/Settings.apk`, flags `PRIVILEGED`) mais plantait dès l'ouverture.
- **Diagnostic logcat** : `NullPointerException` dans `com.android.settings.Settings.updateHeaderList(Settings.java:587)`. `ServiceManager.getService("network_management")` renvoyait `null`, puis l'appel à `INetworkManagementService.isBandwidthControlEnabled()` levait l'exception (le `try/catch` ne couvrait que `RemoteException`, pas le NPE).
- **Cause racine** : dans `/system/framework/services.jar`, la classe `com.android.server.ServerThread` (méthode `initAndLoop`) avait le réseau coupé en dur par la ROM « root » : `SystemProperties.set("config.disable_network", "true")` et `const/16 v86, 0x1` (`disableNetwork=true`). Le branchement correspondant sautait tout le bloc réseau : `NetworkManagement`, `NetworkStats`, `NetworkPolicy`, Wi-Fi P2P, Wi-Fi, `Connectivity`, Ethernet et `Nsd` n'étaient jamais démarrés. La ROM Kingston d'origine lisait `config.disable_network` avec `false` par défaut, d'où la différence de comportement.
- **Correctif appliqué** : décompilation de `services.jar` avec baksmali 3.0.10, patch de `ServerThread.smali` (`0x1` → `0x0` et chaîne `"true"` → `"false"`), réassemblage du `classes.dex`, remplacement dans une copie du jar, push vers `/system/framework/services.jar` (droits 644, `root:root`), purge du dex cache dalvik puis reboot. Le jar du bootclasspath n'est pas vérifié par signature, ce qui rend le patch possible.
- **Résultat** : après redémarrage, `network_management`, `connectivity`, `netpolicy`, `netstats`, `wifi`, ethernet et `servicediscovery` sont de nouveau enregistrés. `Settings.apk` est remis en place dans sa version **Kingston d'origine, non modifiée** ; `am start com.android.settings/.Settings` affiche l'activité en ~636 ms, sans `FATAL`.
- **Contrainte de signature** : la clé plateforme de la ROM est personnalisée (SHA-256 `71:5E:B4:D7:4F:69:A4:26:A5:CC:8E:57:C6:D8:B5:28:CC:49:67:D0:B3:50:20:05:AA:AF:23:AA:EB:50:75:90`), différente des `test-keys` AOSP. Toute application en `sharedUserId=android.uid.system` (Settings, SystemUI, Keyguard…) doit rester **non resignée** : sinon `PackageManager` rejette le paquet avec `Signature mismatch for shared user android.uid.system`. Une tentative `Settings_v1_aligned.apk` resignée a été rejetée ; l'APK d'origine a été conservé.
- **Suite le 23/11** : MX Player plantait faute de service `search`. Les classes `SearchManager` / `ISearchManager` (`framework.jar`) et `SearchManagerService` (`services.jar`) existaient déjà ; il manquait seulement l'enregistrement dans `ServerThread`. Patch smali (`ServiceManager.addService("search", new SearchManagerService(mContext))` après le bloc Clipboard, avec `try/catch reportWtf`), réassemblage `--api 19`, push du jar et purge du cache. Vérification : `service list` montre `search`, `dumpsys search` voit MX Player, aucun `FATAL EXCEPTION` au test de recherche. Sauvegardes : `BACKUPS/2026-11-23-search-service/` sur PC et `/mnt/internal_sd/services.jar.pre-search` sur la console.
- **Précision de vocabulaire** : ici, « réimplémentation » signifie remettre en service l'APK d'origine et réparer le framework système qui l'empêchait de fonctionner — pas réécrire l'application Settings. Aucun code d'application n'a été réécrit.

### Clavier virtuel : de l'affichage à la navigation au D-pad (11/10 → 12/10)

- **État initial** : la ROM n'avait aucun clavier fonctionnel. `default_input_method` pointait vers `com.ndcsolution.androidtv.leankeykeyboard/.LeanbackImeService`, qui n'était pas installé (APK retiré au debloat) : aucune saisie possible.
- **Premier palliatif** : installation de `LatinIME.apk` dans `/system/app`, `ime enable` + `ime set`, `enabled_input_methods` corrigé. Le clavier s'affichait et répondait (`mCurMethodId=LatinIME`, `mHaveConnect=true`), vérifié par capture du framebuffer. Mais cet IME n'est pas navigable au D-pad, alors que la console n'a pas d'écran tactile.
- **Choix retenu** : `com.google.android.leanback.ime` v1.0.2.2423292 (minSdk 16), service `com.google.leanback.ime.LeanbackImeService`, installé dans `/system/app/LeanbackIme.apk` (md5 source `234d5eeebfbc33763b3690a0d1d592bc`). `com.android.inputmethod.leanback` (min API 35) est inutilisable sur API 19 (`Requires newer sdk version #35`). `LatinIME` est conservé en secours provisoirement.
- **Fausse piste documentée** : la fenêtre IME porte `FLAG_NOT_FOCUSABLE` (`fl=#1800108`). Une première hypothèse attribuait à ce flag l'absence de navigation. Un patch smali de `LeanbackImeService` ajoutant `makeWindowFocusable()` (`clearFlags(0x8)` + `requestFocus`) retirait bien le flag (`fl=#1800100`) mais cassait l'état de l'IME (`mHasSurface=false`, `isReadyForDisplay=false`, `mInputShown=false`) : le clavier ne s'affichait plus. L'APK d'origine a été restauré et l'hypothèse écartée.
- **Cause racine réelle (12/10)** : dans le `framework.jar` de la ROM, la méthode `android.view.ViewRootImpl$ImeInputStage.onProcess(QueuedInputEvent)I` était **neutralisée** : elle ne faisait que `return 0` (`FORWARD`). Les touches n'étaient donc jamais proposées à l'IME, quel que soit le clavier installé. Toute la machinerie autour était saine (`dispatchInputEvent`, `ImeInputEventReceiver`, `IInputMethodSessionWrapper`, `onFinishedInputEvent`).
- **Preuve** : instrumentation temporaire de `Leanback.onKeyDown` avec un log `KBDTEST` — jamais déclenché avant le patch, déclenché à chaque touche après.
- **Correctif** : restauration de la logique AOSP KitKat de `onProcess` dans `framework.jar` via smali : si `mLastWasImTarget && !isInLocalFocusMode()`, appeler `InputMethodManager.peekInstance().dispatchInputEvent(...)` ; `FINISH_HANDLED` (1), `FINISH_NOT_HANDLED` (2), `DEFER` (3), sinon `FORWARD` (0). La méthode `isInLocalFocusMode()` étant privée, l'accesseur synthétique `ViewRootImpl.access$700()` est utilisé. Piège rencontré et corrigé : la première version testait `if-eqz` (condition inversée), d'où un branchement permanent vers `FORWARD` ; correction en `if-nez`.
- **Validation fonctionnelle** : avec l'APK instrumenté, les logs `KBDPATCH` montrent `res=IN_PROGRESS` (DEFER), puis la lecture live de l'`EditText` confirme `LEFT` puis `CENTER` → `r`, `RIGHT` puis `CENTER` → `t` : la flèche déplace bien la sélection et le centre valide la touche. La manette physique est cohérente : `rk29-keypad` mappe A sur `BUTTON_A` (96), et `getSimplifiedKey()` du Leanback mappe déjà `BUTTON_A` → `DPAD_CENTER` et `BUTTON_B` → `BACK` — aucun remappage de touches n'était nécessaire.
- **Piège de capture** : le clavier n'apparaît pas dans `/dev/graphics/fb0` : la fenêtre IME est composée en overlay matériel sur RK3188 (`mShownFrame=[0,366][1024,600]`, `layer=21010`) alors que le framebuffer ne contient que la couche applicative. Les validations visuelles du clavier ne peuvent donc pas se faire par capture `fb0` sur cette ROM.
- **Finition** : agrandissement des touches sans `apktool`, par patch binaire de `resources.arsc` (type `dimen`, encodage `dp*256 | 0x01`) : `key_height` 28→44 dp et `key_width` 28→60 dp. La fenêtre IME passe de 234 à 330 px de haut et le clavier d'environ 428 à 780 px de large. Une marge à droite est conservée pour le bouton « Next » (ne pas dépasser ~70 dp).
- **Nettoyage** : retrait des logs temporaires `KBDPATCH` du framework et `KBDTEST` de l'IME. `framework_clean.jar` md5 `41a4adb67874252b71bf33827b894e56` est poussé sur la console. `LatinIME.apk` est supprimé (backup md5 `68e8367b9290e30413d126dcde1c0c89`), son dex cache et ses données purgés : Leanback devient le seul IME actif et par défaut.
- **État final vérifié après reboot à froid** : `sys.boot_completed=1`, `framework.jar` propre, paquet `com.android.inputmethod.latin` absent, `default_input_method = enabled_input_methods = Leanback`. Saisie persistante validée (texte saisi puis étendu via `DPAD_CENTER` après redémarrage). Le `NullPointerException` préexistant de `com.google.process.location` (localisation coupée en dur) reste inchangé et sans lien avec le clavier.
- **Précision de vocabulaire** : l'IME n'a pas été réécrit depuis zéro. Le travail a consisté à réparer le framework (réactivation de la distribution des touches), puis à patcher l'IME existant au niveau de ses ressources et de son code smali, avec restores systématiques des versions d'origine.

## Partie III — Frontend, émulation, stubs et packaging (10/10 → 24/11)

### Contexte initial, objectifs et règles de conservation (date non documentée ; document arrêté au 2026-10-27)
- Matériel : RK3188, 512 Mo de RAM, écran 7" non tactile (1024×600 en paysage), 24 entrées physiques vues par Android comme un clavier, batterie 3000 mAh, slot SD, switch ON/OFF coupant la batterie, pas de Wi-Fi/Bluetooth/ADB/recovery d'origine.
- Deux consoles : console 1 à NAND, écran endommagé lors du démontage, kernel sans gadget USB, réserve de pièces ; console 2 à boot SD obligatoire, kernel avec gadget USB mais dossier `adbd` vide, retenue comme base de travail.
- État d'origine : Android 4.4 AOSP verrouillé par SJGAM, frontend propriétaire au premier plan englobant interface, RetroArch et PPSSPP, quelques émulateurs natifs ; Android inaccessible, quickmenu pauvre, roms non dynamiques (suppression d'un `.DB` à la racine SD nécessaire).
- Objectifs : système ouvert avec installation d'APK depuis la SD, launcher type MediaBox personnalisé (icônes de plateformes, liste de roms à gauche, aperçu PNG à droite avec image par défaut), émulation aussi large que le RK3188 le permet, performances acceptables, souris virtuelle R1+R2, joycon droit indépendant, APK2SD, configs libretro persistantes, système léger et firmware flashable via outils Rockchip.
- Doute : la durée du chord R1+R2 est indiquée comme 3 s dans le document de projet et 5 s dans les toutes premières notes ; 3 s est la valeur retenue ensuite.
- Règle stricte : dumps originaux des partitions des deux consoles et images SD = lecture seule, jamais altérés ; extraire dans un dossier si besoin.

### Premiers accès ADB/root, dumps et transferts (2026-10-07 → 2026-11-03)
- Date non documentée pour le tout premier accès ADB/root ; première preuve datée : 2026-10-07, boot du kernel build9, `adbd` root, `/dev/android_adb` et gadget `f_adb` opérationnels.
- 2026-10-07 : flash de récovery p3 (kernel build9), p4 sauvegardé puis réécrit à l'identique (md5 `5da0f206ca49f13bb5b7165ea73f82c2`), p10 zeroisé, reboot laissé à l'utilisateur.
- 2026-10-07 → 08 : plusieurs builds successifs pour affichage/dalle ; le build13 est validé le 08/10 (logo et launcher nets) puis conservé comme référence de rollback.
- 2026-10-10 : découverte que les 24 contrôles passent par `rk29-keypad` (GPIO), pas par la manette USB ; cartographie complète des entrées établie et keylayouts corrigés sans flash supplémentaire.
- 2026-10-10 → 11 : LED/RGB des joycons pilotables via gpio169 et application `SJGAM Control` créée, anglaisée puis promue en app système `/system/app/SJGAMControl.apk` (md5 `218818287c0500ec985bdaba42bb5b7f`), données conservées.
- 2026-10-11 : lot de 7 APK système d'origine (Kingston) transférés par `adb push` vers `/system/app` et `/system/priv-app` (MediaProvider, CalendarProvider, DownloadProvider, WallpaperCropper, DownloadProviderUi, DocumentsUI, LatinIME), md5 local = md5 console, reboot et vérification `pm path`.
- 2026-10-19 : traducteur clic-souris → appui tactile installé (`/system/bin/sjgam_touch_daemon.sh`) car les jeux Unity ignorent `BTN_LEFT`.
- 2026-10-23 : dumps console 2 enregistrés sous `DUMPS & SAVES/dumps originaux m27 pcb r10 v2.2/` (dont `system_adb_20261023.img`, `boot_adb_patch_20261023.img`, logs de flash).
- 2026-10-26 : image brute complète de la carte SD usine (52,1 Go) sauvegardée avec SHA-256 et copie zstd dans `DUMPS & SAVES/factory sdcard m27 r10v2.2/`, lecture seule.
- 2026-10-27 : architecture retenue = développement sur console 2 (boot SD, ADB, itérations sans risque de brick), migration NAND uniquement en fin de projet ; `/data` saturé (~11,7 Mo libres), la SD reste le support de données.
- 2026-11-03 : image `DUMPS & SAVES/factory sdcard M27 r10v2.3.img` archivée.
- Note : les détails de builds/flash kernel sont traités dans l'historique noyau séparé.

### Souris virtuelle et firmware associé (2026-10-13 → 2026-11-21)
- 2026-10-13 : projet « souris via stick droit » : R1+R2 maintenus bascule le mode, stick droit = déplacement, R1 = clic gauche, R2 = clic droit, daemon userspace via `/dev/uinput`.
- 2026-10-13 : M33 donne des codes dédiés aux sticks (183..190) pour distinguer stick droit et ABXY ; R1 physique = 295, R2 = 293, keylayout usine corrigé sans flash.
- 2026-10-13/14 : premier daemon `/data/local/tmp/mouse_daemon` validé ; OSD `com.sjgam.mouseosd` affiche les toasts ON/OFF ; correctif `umask 0` dans `/system/bin/sjgam_mouse.sh` pour que `/dev/sjgam_mouse` soit lisible par l'OSD.
- 2026-10-14 : validation utilisateur « impec ça fonctionne nikel » ; l'absence de toast après reboot était un problème de timing (bascule avant le démarrage du service BOOT_COMPLETED), pas un bug.
- 2026-11-06 : traducteur clic→tap restauré puis remplacé par `sjgam_touchd` en C (uinput direct, drag & drop, chord court annulé) ; décision finale : clic souris seul, tactile désactivé (le tap perturbait MediaBox et le menu DraStic).
- 2026-11-20 : clic droit du daemon converti en `KEYCODE_BACK` par le framework Rockchip ; tentative de patch `android.policy.jar` abandonnée, framework remis à l'origine (`android.policy.jar.orig`, md5 `ff2c543f...`) et reboot.
- 2026-11-20 : OSD corrigé (libellés inversés, toast au premier poll) puis archivé `WORKING/SJGAMMouseOSD-mode-fixed.apk`.
- 2026-11-21 : déplacement fluide (boucle 10 ms, accumulateur sous-pixel) ; correction uinput RK3188 (un seul `input_event` par `write()`, 3 writes REL_X/REL_Y/SYN_REPORT), validation utilisateur « ha oui ! la c'est beau, c'est propre ! ».
- 2026-11-21 : binaire `/system/bin/mouse_daemon` md5 `5fbeaecc3805a2a6de8c863b6916c631`, réglage à chaud `/data/local/tmp/sjgam_mouse_speed` (facteur 1/128, bornes 32..320), sauvegarde `/mnt/internal_sd/mouse_daemon.pre-smooth`.
- 2026-11-21 : toasts OSD passés en anglais par patch DEX en place (tailles égales, signature/checksum DEX recalculés, même clé debug), `WORKING/SJGAMMouseOsd-i18n.apk` md5 `927623418e6ec2dc413e42e220726579`.
- 2026-11-23 : combo START+SELECT 3 s greffé sur `sjgam_touchd` pour force-stop de l'app au premier plan, avec toast OSD `App stopped: <pkg>` ; binaire md5 `0c1784a2e446abf59f1c5effb9343380`, APK md5 `7dc0df3a847c975746acd66a7bd0cc1e`.
- 2026-11-24 : la souris firmware (rk29-mouse + `mouse_daemon` + Mouse OSD) est confirmée comme seul mécanisme : aucun chord dans le port MAX et aucun pont tactile ajouté.

### Cores libretro et jalon GBA/mGBA (2026-11-02 → 2026-11-04)
- 2026-11-02 : sources mGBA disponibles, build Android NDK r15c (API 19, armeabi-v7a) préparé en secours.
- 2026-11-03 : les cores buildbot officiels (NDK r29, API 21) échouent sur Android 4.4 (`__strncpy_chk2` etc.) ; les cores SJGAM de la SD ne sont pas chiffrés, seuls les 32 premiers octets de l'en-tête ELF sont remplacés et reconstructibles.
- 2026-11-03 : core mGBA SJGAM récupéré installé par défaut dans `/data/data/com.retroarch.ra32/cores/mgba_libretro_android.so` (md5 `790f7ef47f732fd96a2012c8cdc01a51`) ; secours r15c md5 `db27aec1f45d4d67362b3a665d0cc352`.
- 2026-11-03 : validation utilisateur sur 4 roms GBA (sans modification des dumps/roms), géométrie 240×160, audio OpenSL actif, ~83 Mo RSS ; Snes9x SJGAM (md5 `e1d8df49fa53fdc15166c172ed69bc6a`) également validé en jeu.
- 2026-11-03 : `com.emu` (cible d'origine du frontend SJGAM) reste bloqué avant affichage ; ne pas y consacrer de temps, passer par `com.retroarch.ra32`.
- 2026-11-04 : 23 cores ARM32 utiles installés dans le dossier privé RetroArch (UID 10042, modes 755), ~126 Mio, MD5 identiques aux originaux SJGAM ; validation en jeu limitée à mGBA, Snes9x et MAME 2003 Plus, les autres restant vérifiés statiquement.
- Doute : les deux cores Desmume récupérés sont AArch64 et incompatibles avec la M27.

### Frontend SJGAM et menu APKS (2026-11-02 → 2026-11-24)
- 2026-11-02 : frontend hybride (base MediaBox + navigateur de roms) retenu, avec conservation du scanner de plateformes et des lanceurs.
- 2026-11-09 : v0.2 : accueil refait en mosaïque 1280×720 adaptée à 1024×600, tuiles RetroArch/Mes applications/Paramètres/Total Commander/Clean RAM, bandeau de plateformes depuis `/mnt/external_sd/iroms` ; v0.3 : passage à `com.retroarch.ra32` + cores internes ; v0.4 : frise façon carousel.
- 2026-11-12 → 13 : v0.5/0.6 (jeux récents, scan corrigé, miniatures), puis v0.7 → 0.36 (plates-formes manquantes, thème UI, icônes insensibles à la casse, fonds par plateforme, réglages de la frise et de LAST GAMES).
- 2026-11-15 : v0.37 → 0.95 : habillage (boîtier DVD, transparences, ombres, jaquette inclinée, `NoPreview.png`), renommage GBA, direct intent DraStic (v0.83, `ACTION_VIEW` + `Uri.fromFile`), touche Retour ignorée (v0.95).
- 2026-11-16 : v1.0 → 1.24 : intégration de SJGAM Control, contrôle CPU ajouté puis retiré, interface anglaise, tuiles et panneaux ajustés, 8 colonnes dans « Mes applications », orientation paysage forcée.
- 2026-11-17 : v1.05 : `RomLauncher.standaloneIntent()` étendu à tous les émulateurs APK autonomes (DraStic, Mupen64Plus AE, Flycast, PPSSPP) ; validation NDS/N64/PSP, Flycast v2.5 patché (`Uri.getPath()`), BIOS Dreamcast ajoutés et réglages performance appliqués.
- Doute : la numérotation n'est pas monotone dans les sources (v1.24 le 16/11, v1.05 le 17/11) ; les deux versions coexistent dans les journaux, sans explication explicite.
- 2026-11-19 : MediaBox Launcher désinstallé après sauvegarde `WORKING/mediaboxlauncher-installed-before-removal.apk` ; le frontend SJGAM devient le seul launcher HOME.
- 2026-11-20 : frontend et Mouse OSD déplacés vers `/system/app/SJGAMFrontend.apk` et `/system/app/SJGAMMouseOsd.apk` pour être masqués de la grille ; préférences restaurées (`recent_roms.xml`, `rom_inventory.xml`), grille « Apps » réduite à 10 entrées ; mise à jour future = remplacer l'APK système + reboot, pas `adb install -r`.
- 2026-11-23 : service système `search` restauré dans `services.jar` (patch `ServerThread`) pour corriger le force-close de MX Player ; sauvegardes `BACKUPS/2026-11-23-search-service/`.
- 2026-11-24 : confusion 6/7 colonnes dans le menu APKS (`MyAppsActivity`) : le build déployé `d02dbe9e...` (versionCode 122) avait 6 colonnes, le build de travail `a1641ac1...` (versionCode 124) en avait 7.
- 2026-11-24 : après correction de l'interprétation par l'utilisateur (« c'est la 7 colonnes qui est le correctif »), le build 7 colonnes est réinstallé dans `/data/app` et validé visuellement dans le menu APKS ; l'archive firmware reste sur 6 colonnes par décision de reporter la reconstruction.

### Stubs Google/Vending et billing Riptide (2026-11-20 → 2026-11-24)
- 2026-11-20 : Riptide GP2 plantait sur `IabHelper.startSetup` car `com.android.vending` absent ; stub AIDL `com.android.vending.billing.InAppBillingService.BIND` (13 Ko) répondant `BILLING_UNAVAILABLE` (code 3), sans patcher le jeu.
- 2026-11-20 : stub installé en `/system/app/VendingStub.apk` (root:root 644) pour être invisible dans la grille du frontend ; reboot vérifié, jeu démarré, `/system` remis en lecture seule, archive `WORKING/VendingBillingStub.apk`.
- 2026-11-23 : 20 stubs de présence fabriqués (`STUBS/`, DEX minimal obligatoire sinon dexopt échoue) puis, à la demande de l'utilisateur, 14 stubs réseaux sociaux/navigateurs/Amazon supprimés.
- 2026-11-23 : 6 stubs conservés : VendingStub, GmsStub, GsfStub, PlayGamesStub, YouTubeStub, GoogleSearchStub ; limites documentées : présence uniquement, pas de sign-in/achats/SafetyNet (microG nécessaire pour de vraies API).
- 2026-11-24 : 5 stubs désactivés par `pm disable` (GMS, GSF, Play Games, YouTube, Google Search), VendingStub conservé car il fait fonctionner Riptide GP2 ; sauvegardes `DUMPS & SAVES/stubs-cleanup-20261124/`, suppression physique non appliquée.
- 2026-11-24 : Drift Zone 2 affiche « Google Play Store signature invalid » (stub signé avec la clé debug) : aucun stub ne peut corriger cela sans patcher le jeu ou installer un vrai Play Store.

### DraStic : mappings A/B, L2, joycon gauche, souris firmware (2026-11-24)
- 2026-11-24 : A et B inversés dans DraStic (`_KeyMapConfigs_0_2/0_3` dans `/data/data/com.dsemu.drastic/shared_prefs/_Dra$t1c_Pref$_.xml`) ; échange 4↔23 appliqué après arrêt de l'app, validation « c'est mieux », sauvegarde `DUMPS & SAVES/drastic-ab-fix-20261124/prefs-avant.xml`.
- 2026-11-24 : `.ui.Settings` s'ouvre sans crash ; le plantage survient seulement lors d'une navigation dans le panneau, non reproduit dans la séquence journalisée.
- 2026-11-24 : passage dans `.ui.KeyMapper` réécrit le XML et rétablit l'ancien mapping A/B ; correctif réappliqué après `force-stop` ; avertissement : toute passe dans le KeyMapper UI écrase le correctif.
- 2026-11-24 : L2 = Open Menu (index 20 = keycode 104 `BUTTON_L2`) écrit directement dans les préférences, validation « impec ca marche », cohérent avec L1=102/R1=103 et le mapping du port MAX.
- 2026-11-24 : joycon gauche assigné à Screen-swap (index 16 = 198), Fast forward (index 17 = 196) et 1/2 Screen Swap (index 25 = 199), identification par corrélation `getevent -l -t` / logcat, validation « c'est nikel ».
- 2026-11-24 : aucun keylayout système modifié pour ces réglages ; la souris firmware permet les appuis à l'écran, aucune souris virtuelle supplémentaire ajoutée.

### Packaging de l'archive firmware (2026-11-24)
- 2026-11-24 : création du dossier `PACKAGING/` (`MANIFEST.md`, `stage/`, 30 fichiers, `FILES.txt`, `MD5SUMS.txt`) ; règle retenue : la source de vérité est l'état réellement déployé et validé sur la console, pas les copies de travail.
- 2026-11-24 : divergences corrigées côté Linux : code frontend resynchronisé sur la version déployée (à ce moment 6 colonnes), `TOOLS/sjgam_mouse.sh` (md5 `e9b6bd761db9ee845f0e64eb153d4081`) et `TOOLS/sjgam_touchd.arm` (md5 `0c1784a2e446abf59f1c5effb9343380`).
- 2026-11-24 : décision utilisateur : archive flashable Rockchip avec « tout le valide, hors données de jeu » ; création de `PACKAGING/rockchip-m27-custom/` et `PACKAGING/sjgam-m27-custom-20261124.zip`.
- 2026-11-24 : archive de 231 399 853 octets, md5 `4ac17e4c7c14d56ae206dceb86361140`, `unzip -t` OK ; sources du Mouse OSD incluses (`rebuild-mouseosd.sh` produisant un DEX identique md5 `98838ca8...`).
- 2026-11-24 : images packées : `parameter` (1024 o), `kernel.img` (M38 rempli à 12 Mo), `boot.img` usine (md5 `5da0f206ca49f13bb5b7165ea73f82c2`), `recovery.img` (md5 `7aac2177...`), `misc.img` (md5 `cf4825bb...`), `system.img` ext4 brut (805 306 368 o, md5 `af895a12...`).
- 2026-11-24 : `system.img` = dump usine + 19 fichiers validés écrits via `debugfs` ; `e2fsck -fn` propre, 1188 fichiers identiques à la console, un seul APK de jeu ajouté dans `/system/app` ; delta = 6 fichiers modifiés + 13 ajoutés, 75 liens symboliques identiques.
- 2026-11-24 : `flash.sh` (bash -n OK) : reconnaissance + backup par défaut, `--write` (kernel/boot/recovery/system puis relecture et `cmp`), `--restore`, contrôle des frontières `KRNL` à 0x4000 et 0xa000 ; pas de formatage de `userdata`/`cache`, pas d'écriture SD, aucun loader distribué.
- 2026-11-24 : rien n'a été flashé ni modifié sur la console à ce stade ; points ouverts avant fabrication : format de distribution, périmètre exact, script de reconstruction du Mouse OSD.

### Nettoyage de la console et fin de session (2026-11-24)
- 2026-11-24 : sauvegarde complète de `/data/local/tmp` avant nettoyage (`DUMPS & SAVES/console-cleanup-20261124/data-local-tmp/`, 135 fichiers, 25 Mo, `MD5SUMS.txt` md5 `86793897f1aa7438b6a868287a17211a`).
- 2026-11-24 : 125 fichiers temporaires supprimés (captures framebuffer ~21 Mo, copies de stubs, anciens OSD, diagnostics, vidéos de test, anciens binaires/scripts) ; `/data/local/tmp` passe de 24,7 Mo à 1,5 Mo.
- 2026-11-24 : conservés : `sjgam_mouse_speed`, `sjgam_touchd.pid/.log`, `mouse_daemon.log`, `mouse_daemon_autostart.log`, `settings.ini.bak-italian`, `services_patched.jar`, `sd_boot.bin`, `usbmark_hook`, `ctsScreenShot/`.
- 2026-11-24 : vérifications post-nettoyage : noyau #38, keylayout `29fb5f55...`, apps présentes, `mouse_daemon` (pid 757) et `sjgam_touchd` (pid 758) actifs ; aucune donnée de jeu/SD/rom touchée.
- 2026-11-24 : tentative de remplacement du frontend 7 colonnes par le 6 colonnes puis correction de l'interprétation ; retour au build `a1641ac1...` via `adb install -r` (upgrade), sauvegarde de la version 6 colonnes dans `DUMPS & SAVES/frontend-7col-20261124/`.
- 2026-11-24 : fin de session « on stop la » : aucun flash, aucun reboot forcé, aucune autre modification ; console en frontend 7 colonnes, DraStic corrigé, 5 stubs désactivés, souris firmware intacte.
- 2026-11-24 : archive inchangée (6 colonnes, 6 stubs) ; à reprendre : tests Riptide GP2 (billing), Drift Zone 2, MX Player (recherche) et menu APKS, puis une seule reconstruction de l'archive après validation.
- 2026-11-24 : nouveau dépôt GitHub accessible en écriture `git@github.com:wisittheunknow-sys/Sjgam-m27-Unclock.git`, clone local `/var/mnt/stockage/sjgamM27/Sjgam-m27-Unclock`, clé `~/.ssh/id_ed25519_m27`, commit `ab370a6` (README + .gitignore, sans données de jeu ni binaires).
- 2026-11-24 : tâche HISTORY.md demandée mais reportée au lendemain ; trois agents d'extraction ont produit `/tmp/opencode/history-{kernel,max,global}.md` (sorties périssables, à régénérer si absentes) ; ne pas copier les journaux bruts dans le dépôt (mot de passe de keystore local).
- Périmètre : le port M.A.X. est détaillé en Annexe A et le noyau/pilotes en Partie I ; cette partie se limite au frontend, à l'émulation, aux stubs et au packaging. Les deux chantiers système sont développés en Partie II.

## Annexe A — Port de M.A.X. sur SJgam M27 (22/11 → 24/11)

Annexe : travail annexe à la console, conservé ici pour mémoire et reproductibilité. La console reste le sujet principal (Parties I à III).


Source unique : `/var/mnt/stockage/sjgamM27/projet MAX/STATE.MD` (1280 lignes, lu intégralement). Aucune donnée de jeu ni contenu binaire n'est repris ici : seuls les chemins, les quantités, les décisions et les résultats sont conservés. Les doutes sont signalés explicitement en fin de document.

### Phase 0 — Cadrage et inventaire (date non précisée, avant le 22/11)

- Objectif : APK installable sur SJgam M27 (Android 4.4 / API 19, RK3188 ARMv7-A 32 bits, Mali-400), jeu M.A.X. (1996, Interplay), IA adverse jouable en solo, aucun multijoueur (pas de Wi-Fi sur la console).
- Contraintes d'entrée : 24 touches physiques vues comme un clavier Android + souris virtuelle ; pas de dalle tactile ; scaling letterbox à prévoir.
- Inventaire initial `C:\MAX` : `MAX.iso` (638 740 480 octets), `MAX/` installé (532 fichiers, 388 200 894 octets), `max-master.zip` (M.A.X. Port, SDL3, klei1984), `maxr-develop.zip` (M.A.X. Reloaded, SDL2/C++20, client-serveur).
- Les deux archives ont été extraites et aplaties dans `C:\MAX\max-master` et `C:\MAX\maxr-develop` ; archives `.zip` et ISO conservées intactes.
- Aucun outil de build au départ : ni cmake, ninja, git, Java, gradle, adb, ni compilateur C/C++ ; python = stub WindowsApps ; `ANDROID_HOME` / `ANDROID_SDK_ROOT` / `NDK_HOME` non définis.
- Audit `maxr-develop` : IA adverse inexistante (stub UI, `isHuman() → true`, `data/ABOUT` « features still missing », seul le déplacement automatique des prospecteurs existe), solo local à un seul joueur, C++20 + SDL2_net + OggVorbis + `resinstaller` obligatoires, aucun code Android/JNI ; points favorables : rendu 2D vers texture ARGB8888, 640×480 minimum, remappage clavier via `keys.json`.
- Audit de l'IA : `MAXRUN.EXE` contient les chaînes d'assertions des sources d'origine (`ai_*.cpp`) qui correspondent un à un aux tâches de `max-master` ; l'IA de `max-master` est donc l'IA d'origine reconstituée, sous licence MIT.
- Périmètre IA mesuré dans `max-master` : façade `src/ai.hpp`, stratégie `src/aiplayer.hpp`, 51 types de tâches dans `src/task_manager.hpp`, 67 `.cpp` (~22 100 lignes) + ~7 000 lignes de gestionnaires, activation dans `src/game_manager.cpp` ; coût IA = 0 si l'on garde ce moteur.
- Comparaison des architectures : greffage de l'IA vers MAXR jugé illusoire (~30 kLOC à traduire + adaptateur d'intentions vers `cAction`) ; rétro-ingénierie de `MAXRUN.EXE` redondante et coûteuse.
- Conflit de versions vérifié : SDL3 (utilisé par le master) exige API 21+ / NDK r28c / SDK 36+, SDL2 accepte l'API 19 ; NDK r25c = dernier compatible KitKat, r26+ impose API 21+.
- Master `max-master` : 53 fichiers SDL3 à rétro-porter et 59 occurrences de `std::format` incompatibles avec LLVM 14 ; deux empilements cohérents identifiés : A = API 19 + SDL2 + NDK r25c, B = ROM ≥ Android 5.0 + SDL3 tel quel.
- Décision utilisateur : rester en Android 4.4 / API 19 (option A) ; l'option B est écartée mais le code reste compatible si une ROM apparaît.
- Décision utilisateur : aucune donnée originale du jeu ne sera embarquée dans l'APK (M.A.X. n'est pas un abandonware) ; l'utilisateur final fournit sa copie originale sur la carte SD.
- Audit des données : arbre installé = 13 `.MSC` (190,5 Mo), 3 `.MVE` (100,1 Mo), 24 `.WRL` (29,4 Mo), 272 `.WAV` (25,9 Mo), `MAX.RES` (17,8 Mo), 71 `.FLC` (10,0 Mo), scénarios et sauvegardes ; `MAX.RES` = 1 828 enregistrements avec 721 références à des médias externes, dont des absences à tolérer.
- `MAX.INI` ne contient aucune clé de résolution : le 640×480 vient du port, pas de la configuration d'origine.
- Assets du port dans `C:\MAX\max-master\assets` = 1 859 676 octets (`PATCHES.RES`, `language.json`, `help.json`, `maxint.json`, `settings.json`) ; `PATCHES.RES` chargé avant `MAX.RES` et prioritaire en cas de collision de tag.
- Contraintes dérivées : permission `READ_EXTERNAL_STORAGE` suffisante en API 19, écran maison de saisie du chemin SD à la place du sélecteur SDL3, chemins audio conservés tels quels (miniaudio compilé sans Vorbis/Opus/FLAC/MP3).

### Phase 0bis — Revision majeure de la base de code (22/11)

- Découverte : le dépôt officiel a migré de SDL2 vers SDL3 le 26 novembre 2025 ; le tag `v0.7.2` est la dernière version SDL2 (SDL2 2.32.6, API 19 officielle, NDK r15c+).
- Comparaison auditée : `master` = SDL3 + 59 `std::format` + 12 dépendances ; `v0.7.2` = SDL2 natif, 0 `std::format`, 9 dépendances, C++20 limité à `concept`/`requires` (LLVM 14 OK).
- L'IA adverse est complète dans `v0.7.2` (110 fichiers `ai*`/`task*`, `Ai_BeginTurn` et `Ai_CheckReactions` dans `game_manager.cpp`) ; seuls les correctifs de gameplay sont moins récents (~defect 250 contre ~306).
- Décision : partir de `v0.7.2` extrait dans `C:\MAX\max-android`, pas du master SDL3 ; évite 53 réécritures SDL, 59 `std::format` et 3 dépendances sans perdre l'IA.
- `max-master` et les archives d'origine ne sont pas modifiés ; le master reste disponible pour récupérer ponctuellement des correctifs.
- Points bloquants propres à v0.7.2 identifiés : `GetBasePath()`/`GetPrefPath()`/`GetGameDataPath()` dans `src/resource_manager.cpp` (branches Windows/Linux seulement), `TestMouse()` qui quitte sans souris, `SDL_Init` sans `SDL_INIT_AUDIO`, table de mapping fixe dans `src/input.cpp`.

### Phase 1 — Chaîne d'outils (22/11)

- L'utilisateur fournit `C:\MAX\android-19_r04.zip` (63 871 092 octets) : SDK Platform Android 4.4.2 / API 19, extrait vers `C:\MAX\sdk\platforms\android-19` ; archive conservée.
- Précision : ce paquet fournit `android.jar` et les skins, mais ni compilateur C/C++, ni `aapt`, `dx` ou `zipalign`.
- Build-tools 19.1 installés dans `C:\MAX\sdk\build-tools\19.1` (`aapt`, `dx`, `zipalign` opérationnels) ; platform-tools dans `C:\MAX\sdk\platform-tools` (adb 1.0.41 / 37.0.3).
- NDK r25c (25.2.9519653) installé dans `C:\MAX\sdk\ndk\25.2.9519653` (archive 446 Mo) ; dernier NDK compatible API 19.
- JDK : l'asset Adoptium renvoyé derrière le proxy était un XML de 1867 octets au lieu d'un ZIP ; remplacé par Amazon Corretto 8 (1.8.0_514) dans `C:\MAX\sdk\jdk8`.
- CMake 3.28.6 et Ninja 1.12.1 installés dans `C:\MAX\sdk\...` ; archives conservées dans `C:\MAX`.
- Chaîne complète validée le 22/11 : compilation croisée `armeabi-v7a` possible ; `adb` utilisable immédiatement.

### Phase 2 — Noyau natif et squelette APK (22/11)

- CMake/dépendances : cible `SHARED` sous `ANDROID`, pas de `-static`/debuglink, `SDL_STATIC ON` + `SDL2main` ; `cmake/enet.cmake` et `cmake/sha2.cmake` utilisent les variables `_SOURCE_DIR` de FetchContent ; `cmake/sdl2.cmake` active joystick et haptic sous Android pour éviter des symboles JNI non résolus.
- Dépendances liées statiquement (9 sources locales dans `C:\MAX\deps-src`) ; correctifs : `path::u8string()` de nlohmann/json 3.12 converti octet par octet pour libc++ 25, `lua_getlocaledecpoint()` forcé à `'.'` sous Android (`localeconv()` absent avant API 21), includes `<filesystem>/<sstream>/<system_error>` ajoutés dans `resource_manager.cpp`.
- Port Android : `GetBasePath()`/`GetPrefPath()` passent par `SDL_GetPrefPath()` (répertoire privé `getFilesDir()`) où le lanceur Java copie les assets ; `GetGameDataPath()` lit `game_data.txt` écrit par le lanceur (pas de sélecteur de dossier natif en API 19) ; `TestMouse()` contourné sous Android.
- Java (`C:\MAX\android`) : `MAXLauncher` copie `PATCHES.RES`, `settings.ini` et les 6 `lang_*.ini` vers le répertoire privé, navigue dans la SD, valide la présence de `MAX.RES` (ou `MAX/MAX.RES`) et persiste le chemin dans `game_data.txt`.
- `MAXActivity` étend `SDLActivity`, `getLibraries() = {"max"}`, remappe optionnellement les keycodes via `keymap.txt` ; `SDLSurface` fait le pont touch→souris ; `SDLAudioManager` remplace les streams/lambdas Java 8 par des boucles (dx 19.1) ; `HIDDeviceManager` utilise `BLUETOOTH_CONNECT` en chaîne littérale (constante API 31).
- Compilation Java contre `android.jar` API 28 (guards `SDK_INT` conservés), package et resources avec `android.jar` API 19.
- Scripts reproductibles : `C:\MAX\build_android.ps1` (configure/build/strip) et `C:\MAX\build_apk.ps1` (R.java, javac, dx, aapt, signature, zipalign).
- Résultat : `C:\MAX\build-android\libmax.so`, ELF32 ARM, `SDL_main` exporté, 4,6 Mo après strip ; `C:\MAX\android\bin\MAX.apk`, 12,5 Mo, signé, `minSdk=19`, `armeabi-v7a` ; vérification `aapt list` : aucune donnée originale embarquée.
- Ajouts « hors ligne » (console non encore branchée) : saisie manuelle d'un chemin SD dans `MAXLauncher` avec résolution du parent contenant `MAX/MAX.RES` ; priorités de lecture du keymap `/sdcard/max_keymap.txt`, `/sdcard1/max_keymap.txt`, `/storage/sdcard1/max_keymap.txt`, `<game_data>/keymap.txt`, fichier privé.
- `C:\MAX\keymap_example.txt` (modèle commenté), `C:\MAX\console_capture.ps1` (relevé ADB automatisé), `C:\MAX\install_console.ps1` (install, push keymap, lancement via `monkey`, option `-Log`) ; APK reconstruit après ces modifications.
- Précision utilisateur : la souris virtuelle du firmware s'active par le chord R1+R2 et peut produire un vrai `SOURCE_MOUSE` ou un touch simulé ; implémentation initialement ajoutée (`vm_r1=`/`vm_r2=`, `sVirtualMouseActive`, pont touch→souris) puis retirée le 23/11 au profit de la souris firmware.

### Phase 3 — Premier démarrage sur console : signature, JNI et chemins (23/11)

- La console est branchée ; objectif de la séance : démarrer le moteur sur M27 (1024×600) sans embarquer la moindre donnée du jeu, IA d'origine intacte.
- Signature APK : `-sigalg SHA1withRSA -digestalg SHA1` ; le SHA-384 était refusé par Android 4.4 (`INSTALL_PARSE_FAILED_NO_CERTIFICATES`).
- JNI/chemins : garde `Android_JNI_GetEnv() == NULL` dans `SDL_android.c` (retour `/data/data/com.max.port/files`) ; `HIDDeviceManager.acquire()` forcé à `null` car `libmax.so` n'a pas les symboles JNI HID.
- Linkage : relink sous Linux avec `ld.lld` 22 (objets et bibliothèques du NDK r25c Windows réutilisés) via `WORKING/max/rebuild-sdl-android.sh`.
- Fausse piste : un crash EGL diagnostiqué comme `SDL_EGL_LoadLibrary` était en réalité un mauvais `load bias` ; la vraie pile était `memcpy` ← `SDL_memcpy` ← `Svga_Blit` ← `Svga_SetPaletteColor` ← `Color_Init`.
- Cause réelle : framebuffer annoncé 24 bits (pitch 2458 octets pour 819 pixels) alors que `Svga_Blit` copiait `bounds.w * 4` octets par ligne ; lecture hors carte.
- Correctifs dans `max-android/src/svga.cpp` : forcer `SDL_PIXELFORMAT_ARGB8888` si la profondeur annoncée n'est pas 32 bits ; remplacer `SDL_LockTexture` + `memcpy` manuel par `SDL_UpdateTexture()` avec le pitch réel.
- Diagnostic temporaire retiré avant livraison (`usleep(10s)` pour gdbserver).
- Résultat : APK reconstruit et installé, cinématique d'intro visible (validation utilisateur), aucun SIGSEGV, process stable, fichiers `/mnt/external_sd/MAX/MAX.RES` et `MAXINT.MVE` ouverts, audio 44,1 kHz actif.
- Outils conservés : `rebuild-sdl-android.sh` (recompile `SDL_android.c` et `svga.cpp`, relie et pousse dans `/data/app-lib/com.max.port-*/`), `rebuild-apk-linux.sh` (dex + APK signé SHA1), `diag-gdb-stack.sh` ; captures dans `WORKING/max/capture-2026-11-23/`.
- Suite immédiate : utilisateur confirme le menu principal (« on a le menu !!!! ») ; rendu 8-bit → texture 32-bit valable aussi pour l'interface, ~25 % CPU, aucun crash.

### Phase 4 — Affichage plein écran 640×480 → 1024×600 (23/11)

- Problème : `WindowManager_GetScale() = screen_h/480 = 1,25` centre une interface 800×600 dans le buffer, d'où un rectangle entouré de noir avec `window_width=1024`, `window_height=600`.
- Tentative 1 : `settings.ini` à 800×600 + `disable_ar_correction=1` et nouvelle `Svga_ApplyStretch()` (`SDL_RenderSetScale` + `SDL_RenderSetViewport`) appelée dans `Svga_Init()` et `Svga_Flush()`/`Svga_Flip()` ; vérification framebuffer 98,4 % de pixels non noirs.
- Tentative 2 : bandes noires autour du fond de menu (640×480 centré dans 800×600) ; ajout du paramètre `rescale=true` aux 3 appels `MAINPIC` dans `menu.cpp` ; vérification 95,6 % de pixels non noirs, validation « magnifiquuueeee ! ».
- Retour arrière : `rescale=true` décalait le menu hors du cadre noir car `ScaleUlx/Uly` n'applique qu'un offset alors que le scaler générique multiplie par 1,25 ; correctif annulé, `menu.cpp` rendu identique à l'upstream.
- État final verrouillé : `settings.ini` en `window_width=640`, `window_height=480`, `disable_ar_correction=1` ; `WindowManager_GetScale()=1,0` ; `Svga_ApplyStretch()` étire la texture 640×480 vers 1024×600 avec `scale=(1,6 ; 1,25)` et viewport plein écran.
- SDL2 local convertit les événements souris/touch selon le renderer scale, donc les clics restent cohérents avec l'image étirée.
- Vérification finale : process stable, capture `/dev/graphics/fb0` 98,6 % de pixels non noirs, aucune bande noire ; validation utilisateur « Oui, parfait ! » ; consigne de ne plus toucher à `svga.cpp`, `settings.ini` ni `menu.cpp` pour cette séance.

### Phase 5 — Souris firmware, suppression de la souris virtuelle et pointeur (23/11)

- Demande utilisateur : supprimer la souris virtuelle intégrée au jeu, qui fait doublon avec celle du custom firmware.
- État firmware relevé : `/system/bin/mouse_daemon` (PID 776, root), `com.sjgam.mouseosd` (PID 730), périphérique `rk29-mouse` = event4.
- Java : suppression du chord R1+R2 et du Toast dans `MAXActivity`, du pont `forwardTouchAsMouse()` et de ses appels dans `SDLSurface`, du champ `sVirtualMouseActive` dans `SDLActivity` ; les événements `SOURCE_MOUSE` réels vers `onNativeMouse()` sont conservés ; le parsing ignore désormais les clés `vm_*`.
- Piège corrigé : `rebuild-apk-linux.sh` ne recompilait que `HIDDeviceManager.java` ; il compile désormais les 4 fichiers Java et copie toutes les classes dans `obj/` avant dex.
- Vérification : `strings classes.dex` sans trace de la souris virtuelle ; APK installé, PID stable, aucun crash ; point de reprise = confirmer un clic unique (plus de double clic).
- Dérive du pointeur signalée (nulle en haut/gauche, croissante vers le bas/droite) : première cause écartée (mode souris relative de `SDL_WINDOW_FULLSCREEN`, désactivé dans `GNW_mouse_init()`), cause réelle = `SDL_GetMouseState()` renvoie des coordonnées physiques 1024×600 non converties par `SDL_RendererEventWatch`.
- Correctif : `Svga_MouseToLogical()` dans `svga.h/cpp` via `SDL_RenderWindowToLogical()` (compte du viewport et du scale), `Svga_WarpMouse()` via `SDL_RenderLogicalToWindow()`, appel dans `mouse.c` `mouse_info()` ; `rebuild-sdl-android.sh` compile aussi `mouse.c` avec `-D_FORTIFY_SOURCE=0`.
- Validation utilisateur : « impec ça suit la souris » ; souris verrouillée.

### Phase 6 — Stabilisation : training, FLIC, curseur et journal (23/11)

- Assertion `CLANATRB` au lancement du training : des tokens français des données d'origine déclenchaient l'assert alors que `settings.ini` était en anglais ; `AttribTokenMatches()` accepte désormais anglais et français, sans modifier les données du jeu.
- FLIC/bandes horizontales : `char psize` signé sur ARM interprétait les runs littéraires comme des tailles énormes, d'où corruption de tas ; corrigé en `int8_t psize` dans `flicsmgr.cpp` ; validation « ho ! impec on a l'image et c'est stable sur la map !!! ».
- Journal persistant : `ResourceManager_LogOutputHandler()` termine par `std::endl` + `flush()` ; `/data/data/com.max.port/files/stdout.txt` récupérable après crash.
- Crash « joycon droit haut » : SIGSEGV dans `Cursor_DrawAttackPowerCursorHelper()` (`cursor.cpp`) dû à un descripteur de ressource mis en cache puis périmé après `ResourceManager_Realloc()` ; ajout de `Cursor_RefreshDescriptor()` avant les dessins dans `game_manager.cpp` (barre d'attaque et chance furtive), avec saut de l'overlay si ressource absente/invalide ; `cursor.cpp` ajouté au script de rebuild.
- Validation : « Oui, plus de crash » ; APK reconstruit et installé (`Success`).

### Phase 7 — Calibration des 24 boutons et mapping (23/11)

- Capture brute `getevent -l /dev/input/event0` → `/tmp/opencode/calib-keypad.txt` : croix 103/105/106/108, stick gauche 183..186 (Android 196..199), A/B/X/Y déclarés dans `/system/usr/keylayout/rk29-keypad.kl` = 290/289/291/288 → Android 23/4/99/100, épaules 294/292/293/295, Select/Start 296/297.
- Actions parasites du stick droit : `/sdcard/max_keymap.txt` contenait encore `183=8`, `184=9`, `185=10`, `186=11` (PROG_RED..BLUE en plus des REL_X/REL_Y de `rk29-mouse`) ; lignes et clés `vm_*` supprimées, backups `-20261123`, copie poussée avec `chmod 644` / `chown 1000:1015`, application redémarrée ; validation « Oui, corrigé ».
- Première déduction ABXY via `getevent` corrigée par un log temporaire `MAXKEY` dans `MAXActivity.dispatchKeyEvent()` : A=23 (scan 290), B=4 (scan 289), X=100 (scan 288), Y=99 (scan 291) ; log retiré, APK reconstruit, `MAXKEY` absent de `classes.dex`.
- Le fichier `.kl` Android reste inchangé (« pas le kl android, dans max ») : seule la table lue par le jeu est modifiée (`WORKING/max/max_keymap.txt`, `keymap_example.txt`, backups SD `-20261123{, -2, -3}`).
- Calibration définitive du losange à jeu arrêté (les actions dans le jeu faussaient les logs) : haut = Y/fin de tour, droite = B/retour, bas = A/valider, gauche = X/centrer ; confirmation utilisateur de la disposition physique A=bas, B=droite, X=gauche, Y=haut.
- Cause profonde du mapping chaotique : les commentaires de la table SD avaient décalé les numéros Linux de +1 (BTN_THUMB2 = 290 et non 291) ; l'entrée `104=34` envoyait F=centrer depuis L2 ; table corrigée à `23=66`, `4=111`, `99=34`, `100=33`, `102=69`, `104=70`, backup `-6`.
- Contrôle du fichier système : mtime inchangé (2026-11-18 17:36:22), md5 `29fb5f55eba641d01de7a11fd6120396` ; aucun APK à reconstruire (table lue au lancement).
- Swap A/Y demandé pour test (`23=33`, `100=66`, backup `-7`) puis retour à l'état antérieur (`23=66`, `4=111`, `99=34`, `100=33`, `102=69`, `104=70`, backup `-9`) ; validation « Oui ».
- Raccourcis Start/Select : ajout de `injectKey()` dans `MAXActivity.java` envoyant un KeyEvent synthétique à SDL (Start 108 → Alt+F, Select 109 → Alt+P, release en ordre inverse), d'après les raccourcis natifs de `game_manager.cpp` ; les anciens mappages Tab/Espace ne déclenchent rien.
- APK reconstruit (`apk-shortcuts.log`), lignes `108=61`/`109=62` retirées de la table SD (backup `-8`) ; validation : Start ouvre FILES/SAVE, Select met en pause (choix utilisateur : garder Select = Pause, préférences accessibles à la souris).
- Tous les logs diagnostics retirés de `MAXActivity.java` ; APK reconstruit (`apk-clean-final.log`), installé, `MAXKEY` absent de `classes.dex`.
- Mapping final figé : A=Entrée/fin de tour (23=66), B=retour (4=111), X=centrer (99=34), Y=changer de curseur (100=33), L1 zoom arrière (102=69), L2 zoom avant (104=70), stick gauche = flèches, stick droit = souris firmware non remappé ; `.kl` intact.

### Phase 8 — Tests fonctionnels et clôture du portage (23/11)

- Validation utilisateur du mapping final : « Oui » (A=Entrée, B=Échap, X=F/chercher, Y=E/fin de tour dans la table alors en vigueur).
- Save/load : bouton `WINDOW_FILES_BUTTON` (rect logique 17,4–83,21, soit ~27,5–133,26 sur 1024×600), FILES → SAVE puis FILES → LOAD : OK (validation utilisateur).
- Test final IA : partie lancée avec 4 adversaires « computer » en parallèle ; retour « stable, ça attaque, ça lance des constructions », `pidof com.max.port` actif, `stdout.txt` sans log FATAL ni tombstone.
- Conformité : les 4 sources `src/ai*.cpp` n'ont jamais été modifiées ; IA adverse d'origine conservée de façon absolue.
- Observation de gameplay pendant le test 4 IA : « les rouges se font défoncer par les verts » ; notée comme observation, pas comme bug.
- Archives : `WORKING/max/logs/stdout-final-20261123.txt` et `WORKING/max/logs/tombstones-list-20261123.txt`.
- Bilan du 23/11 : portage terminé et validé sur M27 (Android 4.4 / API 19, RK3188, Mali-400, 1024×600) : rendu natif 640×480 étiré, souris firmware conservée, mapping sur SD calibré, training stable, save/load OK, IA d'origine fonctionnelle ; aucune donnée originale dans l'APK (copie du joueur sur `/mnt/external_sd/MAX`).
- Son : perte ponctuelle juste après un redémarrage, logs `AudioFlinger: write blocked` (366/398 msecs) et avertissements API 19 de `SDLAudioManager` ; volumes normaux, pas d'erreur d'initialisation ; le son est revenu de lui-même (« là c'est bon j'ai du son dans max »), sans modification de code.

### Phase 9 — Langue des textes (23/11 → 24/11)

- `src/localization.cpp` : `GetLanguage()` lit `SETUP/language` dans `<prefpath>/settings.ini` puis charge `lang_<langue>.ini` depuis les assets de l'APK ; les 6 fichiers de langue sont embarqués ; Voice/Sfx/Music pointent vers la copie MAX du joueur (aucun réglage de langue audio dans le moteur).
- 23/11 : `language=french` écrit dans `/data/data/com.max.port/files/settings.ini` et dans `android/assets/settings.ini` (modèle) ; CRLF conservés, `chown 10041:10041`, `chmod 600` (busybox `sed -i` avait repassé le fichier en root:root) ; redémarrage via `am force-stop` + `monkey` ; validation « Oui, tout est français ».
- 23/11 : sélecteur de langue au premier démarrage ajouté dans `MAXLauncher.java` (`LANGUAGES[]`, spinner, `readLanguage()`/`writeLanguage()` par regex `(?m)^language=`, écriture ISO-8859-1 pour préserver inode, droits et CRLF), appelé avant `writeDataPath()` + `startGame()`.
- Piège corrigé : `rebuild-apk-linux.sh` compilait seulement 4 fichiers Java et réutilisait un ancien `MAXLauncher.class` ; `MAXLauncher.java` ajouté à la compilation et import `FileInputStream` ajouté ; nouvelle APK vérifiée dans `classes.dex`.
- Test : langue italienne choisie dans le lanceur avec le dossier `/mnt/external_sd/MAX` → textes italiens, `language=italian`, CRLF et permissions intacts ; cinématique restée française, attendu car lue depuis la copie du joueur.
- Vérification en lecture seule : `MAXINT.MVE` unique (71 202 530 octets), `MAXMVE1.MVE` et `MAXMVE2.MVE` = autres cinématiques, `INTRO1.CAM` = campagne ; une seule occurrence de `INTROFLC` dans `MAX.RES`, `Movie_PlayIntro()` → `movie_run(INTROFLC)` ; pas de suffixe ni de dossier de langue : intro monolingue, aucun choix audio/vidéo à implémenter.
- 24/11 : retour à `language=french` dans `settings.ini` privé (backup `/data/local/tmp/settings.ini.bak-italian`), CRLF et droits préservés ; `lang_french.ini` déjà présent ; validation visuelle des textes par l'utilisateur.
- 24/11 : textes français confirmés, port considéré terminé (son, rendu, icône, commandes, langue). Une évolution ultérieure possible : fichier `/sdcard/max_language.txt` appliqué par `MAXActivity` avant `super.onCreate()` (sans changer le natif) ou vrai sélecteur dans Préférences (C++).

### Phase 10 — Export visuel et icône (24/11)

- Export « backgrounds » à la demande de l'utilisateur : extraction en lecture seule de 29 images 640×480 de `MAX.RES`, palette 6 bits de `COLOR.PAL` convertie en 8 bits (×4), décodage RLE conforme à `WindowManager_DecodeBigImage` (flux à partir de l'octet 776 = couleur transparente).
- Résultats dans `EXPORT/tank-sprites/backgrounds/` (miniatures `thumbs/`, planche `ALL_BACKGROUNDS_sheet.png`) et `EXPORT/presentation/` (4 composites, fonds bruts, portraits, planche des 29 fonds, `LISEZ-MOI.txt`) ; écrans de production reconstitués `FACBUILD`/`CONBUILD` + sprite de char composite en (11,13), position relevée dans `buildmenu.cpp`, vérification pixel par pixel OK.
- Rappel : l'utilisateur voulait l'écran de production, ni l'animation `TANK03.FLC`/`ALNTANK.FLC` ni les sprites RLE ; aucune donnée MAX, source `ai*.cpp` ou APK modifiée.
- Icône : source `icon_final.png` fournie par l'utilisateur (144×144 RGBA, fond transparent), aucune image d'origine utilisée ; `install-icon.sh` génère 48/72/96/144, met à jour le manifest, reconstruit et installe l'APK ; ancienne APK sauvegardée dans `BACKUPS/2026-11-24-MAX-pre-icon.apk`.
- Vérifications : `aapt dump badging`, `unzip -l` (4 densités), `pm path` → `/data/app/com.max.port-1.apk`, `settings.ini` toujours en `language=italian` à ce moment ; md5 de l'APK à l'icône = `571e1b3ae3fbe415fc820a1f27c4b762`.

### Phase 11 — Son intermittent : diagnostic, watchdog et rendu batché (24/11)

- Symptôme : son absent partout au premier lancement, présent après « quitter + relancer » ; non reproduit sur 6 cycles force-stop + `monkey` (tous `ma_engine_init()` réussis) ; volumes 100/100/50 et `disable_music/fx/voice=0`, aucun mute système.
- Séquence reproductible : Burger au premier plan, quitter, lancer M.A.X. → silence ; captures dans `/tmp/opencode/audio-bad/`.
- Diagnostic : Burger garde 43 threads, 2 `AudioTrack`, 26,5 Mo PSS ; au démarrage de M.A.X., `AudioFlinger` signale `write blocked for 395 msecs, 3 delayed writes` ; piste M.A.X. en état `S`, flags `0x601`, `UndFrmCnt=2230`, absente d'`active tracks` ; après arrêt de Burger : état `A`, flags `0x000`, présente dans `active tracks`.
- RAM écartée : 512 Mo, 274 Mo libres/cache, pas de swap, kswapd0 inactif ; cause retenue = underrun/scheduling OpenSL ES lié aux `AudioTrack` d'une appli en fond.
- Décision intermédiaire utilisateur : combo START + SELECT 3 s sur l'appli au premier plan avant de lancer M.A.X. (force-stop déjà en place) ; pas de watchdog natif retenu à ce stade, aucune recompilation.
- Revue d'optimisation (analyse seule) : 1 `SDL_RenderPresent` par `Svga_Blit` (dirty rects) → batcher ; `SDL_LowerBlit` plein écran 8→32 à chaque changement de palette (~30 Hz, color cycling) ; déplacer `SDL_HINT_RENDER_SCALE_QUALITY` avant `SDL_CreateRenderer`, retirer `SVGA_NO_TEXTURE_UPDATE` (code mort), ne rejouer `Svga_ApplyStretch()` que sur changement de taille ; réglages `scale_quality=1`, `enhanced_graphics=1`, films désactivés ; RAM largement suffisante.
- NDK r25c Linux installé, script `WORKING/max/build-native-linux.sh` créé, baseline compilée, backup complet avant modification dans `BACKUPS/2026-11-24-MAX-pre-audio-render/`.
- Watchdog audio v2 dans `sound_manager.cpp` : contrôle toutes les 500 ms, `ma_device_start()` si `ma_device_get_state() == ma_device_state_stopped`, log limité à 5 reprises.
- Rendu batché dans `svga.cpp` : `Svga_Flush()` = 1 seul `SDL_RenderCopy`/`SDL_RenderPresent` par frame, `Svga_Blit()` ne fait plus que l'upload + flag dirty, fallback 25 ms ; hint de qualité déplacé avant le renderer, constante morte retirée ; flush en tête de `GNW_do_bk_process()` dans `input.c`.
- Vérification : `Svga_Flush` présent dans le `.so` (`llvm-nm`), `.so` stripé 4,8 Mo, APK de 2 856 567 octets installée, IA intacte ; test automatique : démarrage propre, piste active, pas de crash.
- Validation utilisateur : son OK après Burger, rendu « vachement fluide », FLV/intro activable, watchdog non déclenché lors du test mais présent et loggué ; aucune régression visuelle ni tombstone ; version installée `/data/app/com.max.port-2.apk`, 2 856 567 octets.
- Retour final : navigation sur la carte également fluide, gain de FPS global confirmé, correctif de rendu définitif.
- Watchdog v3 : le silence a été reproduit avec le device en état « started » (v2 aveugle) et la piste AudioFlinger bloquée ; détection par horloge audio dans `sound_manager.cpp` : si `ma_engine_get_time_in_pcm_frames()` n'avance plus pendant 1500 ms (vérification 500 ms), forcer stop+start du device OpenSL et logger (5 fois max) ; champs `audio_stall_time`, `audio_last_engine_time`, constante `SOUND_MANAGER_STALL_MS=1500`.
- APK v3 : 2 856 905 octets, md5 `064460b9...` (valeur tronquée dans la source), `.so` `fb2a1f86...` (tronqué), installée ; test avec Burger en fond : son OK, piste active, aucune reprise logguée ; diagnostics à collecter dans `DIAG/2026-11-24-audio-watchdog-v3/` si le silence revient.

### Phase 12 — Actions console hors périmètre MAX (24/11)

- Audit console : CPU sain (~64 % idle), RAM non saturée (245 Mo de cache réclamable) mais LMK agressif (14–36 Mo) ; Drift Zone 2 à ~91 Mo de PSS fait tuer MAX et Burger au lancement ; erreur Drift = signature du stub Vending invalide (clé debug) ; rien changé sur la console.
- Nettoyage `/data/local/tmp` après backup complet dans `DUMPS & SAVES/console-cleanup-20261124/` : 125 fichiers supprimés, 24,7 Mo → 1,5 Mo ; aucune donnée MAX sur SD touchée, aucun `ai*.cpp` modifié ; `settings.ini.bak-italian` et `services_patched.jar` conservés.
- Après nettoyage : `com.max.port` (`/data/app/com.max.port-1.apk`, md5 `064460b9...`) et la souris restent opérationnels.
- Frontend (hors MAX) : oscillation 7 → 6 → 7 colonnes dans le menu APKS ; rectification validée « la c'est bon » = build 7 colonnes `a1641ac1...` (versionCode 124) réinstallé via `adb install -r`, chemin final `/data/app/com.sjgam.frontend-2.apk`, backup 6 colonnes dans `DUMPS & SAVES/frontend-7col-20261124/` ; l'archive firmware contient encore le build 6 colonnes dans `/system/app` (décision de resynchronisation à prendre).
- Drastic (hors MAX) : inversion A/B corrigée dans `_KeyMapConfigs_0_2 = 23` / `_0_3 = 4` (backup `prefs-avant.xml`), mapping spécial L2 = Open Menu (`_0_20 = 104`), screen-swap = joycon G gauche (198), fast forward = haut (196), 1/2 screen swap = droite (199) ; validation utilisateur « c'est nikel » ; `.ui.Settings` plante encore après ouverture du panneau, logcat prêt ; aucun lien avec le port MAX ni avec la souris firmware d'origine.

### Anomalies résolues — récapitulatif

- Signature SHA-384 refusée par Android 4.4 → signature SHA1withRSA/SHA1.
- Crash natif au démarrage (pile EGL trompeuse) → framebuffer 24 bits et pitch erroné dans `Svga_Blit` → format ARGB8888 forcé + `SDL_UpdateTexture`.
- Symboles JNI HID non résolus → `HIDDeviceManager.acquire()` à null et backends joystick/haptic activés dans SDL2.
- Sélecteur de dossier SDL3 absent en API 19 → lanceur Java avec navigation SD, saisie manuelle et chemin persisté.
- Absence de souris détectée au démarrage → `TestMouse()` contourné sous Android.
- Double clic / doublon de souris → suppression complète de la souris virtuelle du port au profit de `rk29-mouse`.
- Dérive du curseur (coordonnées physiques vs logiques) → `Svga_MouseToLogical()` / `Svga_WarpMouse()`.
- Bandes noires et décalage du menu → buffer natif 640×480 + `Svga_ApplyStretch()` ; `menu.cpp` restauré à l'upstream.
- Assertion `CLANATRB` sur tokens français → `AttribTokenMatches()` bilingue, données intactes.
- Bandes/corruption FLIC sur ARM → `psize` en `int8_t`.
- Crash du curseur après réallocation de ressource → `Cursor_RefreshDescriptor()`.
- Actions parasites du stick droit et mapping décalé par les commentaires de la table SD → keymap corrigé, `.kl` intact.
- Recentrement inattendu par L2 → suppression de `104=34`, table corrigée.
- Tab/Espace sans effet pour FILES/Pause → raccourcis synthétiques Alt+F / Alt+P.
- Build Java obsolète (`rebuild-apk-linux.sh` ne recompilait qu'un fichier) → script étendu aux 5 sources Java.
- `path::u8string()` / `localeconv()` / includes libc++ → correctifs de compatibilité NDK r25c.
- Perte de son intermittente → watchdog v2 puis v3 par horloge audio, rendu batché (fluidité validée).
- Droits et CRLF de `settings.ini` altérés par `sed -i` → restauration `chown`/`chmod`.
- Langue italienne puis française → `language=` piloté par le lanceur et le fichier privé.

### Checksums et identifiants d'APK reportés dans la source

- Table de clavier système (`/system/usr/keylayout/rk29-keypad.kl`, jamais modifiée) : md5 `29fb5f55eba641d01de7a11fd6120396`, mtime 2026-11-18 17:36:22.
- APK avec icône (avant les modifications audio/rendu) : md5 `571e1b3ae3fbe415fc820a1f27c4b762`.
- APK watchdog v3 / dernière version MAX signalée : 2 856 905 octets, md5 `064460b9...` (tronqué dans `STATE.MD`) ; `.so` associé `fb2a1f86...` (tronqué).
- Version intermédiaire batch/watchdog v2 : 2 856 567 octets, installée sous `/data/app/com.max.port-2.apk`.
- Frontend hors MAX : build 7 colonnes `a1641ac1...` (versionCode 124, retenu) ; build 6 colonnes `d02dbe9e...` (versionCode 122).

### Doutes et points à ne pas inventer

- `STATE.MD` porte en tête « dernière mise à jour : 2026-11-22 », mais son contenu couvre jusqu'au 24/11 : la date d'en-tête n'est pas fiable pour la fin du journal.
- Plusieurs « mapping final » se succèdent le 23/11 (calibration losange, décalage de +1 découvert ensuite, swap A/Y, retour arrière) ; la table retenue est celle de la phase 7 (A=Entrée, B=retour, X=centrer, Y=changer de curseur), mais le compte rendu intermédiaire des tests fonctionnels évoque encore X=F et Y=E : incohérence de nomenclature à ne pas lisser artificiellement.
- Les chemins d'installation de l'APK divergent selon les passages : `/data/app/com.max.port-1.apk` (icône, après nettoyage) et `/data/app/com.max.port-2.apk` (version 2 856 567 octets) ; le lien entre ces chemins et l'APK v3 n'est pas explicité.
- Les md5 de l'APK v3 et du `.so` sont tronqués dans la source (« ... ») : aucun checksum complet de la toute dernière APK n'y est consigné ; la valeur `571e...` concerne l'APK à l'icône, pas forcément la version finale.
- Le watchdog v2 n'a jamais été déclenché lors des tests et le silence d'origine n'a pas été reproduit après le v3 : efficacité définitive non prouvée, surveillance recommandée.
- La résolution du problème du son a d'abord été tranchée en faveur du combo START+SELECT sans watchdog, puis un watchdog natif a été implémenté le même jour : décisions successives, à ne pas présenter comme un choix unique simultané.
- La calibration « définitive » du losange du 23/11 utilisait des numéros Linux décalés par les commentaires de la table SD ; elle a été corrigée ensuite. Les codes intermédiaires (X=104, etc.) ne doivent pas être présentés comme finaux.
- L'ancien inventaire ISO (« 111 fichiers / 110,7 Mo ») est explicitement faux et remplacé par 1 075 fichiers / 637 053 313 octets ; ne pas conserver l'ancien chiffre.
- Plusieurs actions du 24/11 sont hors périmètre MAX (frontend, Drastic, LMK, nettoyage) et certaines ont été annulées/reprises : elles sont relatées pour contexte, sans lien avec l'APK du port ni avec les mappings.

## Annexe B — 25/11/2026 : assertion `corruption_detected` dans M.A.X.

- Signalement utilisateur : `assertion failed at saveloadmenu_save (.../max-android/src/saveloadmenu.cpp:7) triggered 1 time : 'corruption_detected'`. Le chemin affiché est une chaîne compilée dans l'APK (`__FILE__` de la macro `SDL_assert`) : la console n'a jamais tenté de lire ce chemin sur le PC.
- Vérification : le code est identique au tag amont `klei1984/max` `v0.7.2` (`saveloadmenu.cpp:699-716`) ; cette assertion n'a pas été ajoutée par le port. Le build local active les assertions avec `SDL_ASSERT_LEVEL=2`, d'où l'affichage ; « triggered 1 time » est le comportement one-shot de SDL.
- Déclencheur identifié : `SaveLoadMenu_Save(..., play_voice=false, backup=true)` appelé depuis `game_manager.cpp:2737` = autosave de fin de tour. Les sauvegardes manuelles du menu passent `play_voice=true` et sautent le test.
- Mécanisme d'origine : `SaveLoadMenu_RunPlausibilityTests()` enchaîne `Defect11` (dépôt/entrepôt), `Defect183` (hash de carte) et `Defect151` (connecteur chevauchant une unité). Si un correctif s'applique, l'état est réparé en mémoire, `corruption_detected` devient vrai et la fonction `return` avant d'écrire le fichier : **cet autosave précis n'est pas écrit**, sans crash.
- Le `stdout.txt` de la console confirme le rapport SDL et `always ignore: yes` : l'utilisateur a choisi « Always ignore » dans la boîte de dialogue. Les futurs messages identiques ne s'afficheront plus, mais le `return` reste actif.
- Fichiers observés : `SAVE10.BAK` 00:04 et `SAVE10.DTA` 00:08 (un autosave a réussi à 00:08) ; aucun `ai_log.txt` présent ; `settings.ini` mesuré à 0 octet puis réécrit à 1 196 octets au lancement. L'utilisateur confirme que les textes restent en français : rien à modifier.
- Décision utilisateur : « diagnostiquer plus fin ». Logs temporaires `MAXDBG` ajoutés **uniquement** dans `saveloadchecks.cpp` (7 points : Defect11, Defect151, quatre branches Defect183) et un récapitulatif dans `SaveLoadMenu_RunPlausibilityTests()` de `saveloadmenu.cpp`. Aucune modification des `ai*.cpp` (md5 vérifiés avant/après).
- Backup préalable : `BACKUPS/2026-11-25-MAX-corruption-diag/` (sources avant, APK installé `064460b913819ea3c0c214081dfa7e43`, `stdout.txt.avant`, inventaire md5 des 4 `ai*.cpp`).
- Rebuild natif via `WORKING/max/build-native-linux.sh`, strip `llvm-strip` : `.so` `556ae81cc97191f2a0b3e567ead2d2ee` (4 806 040 octets) contenant les 7 chaînes `MAXDBG`. APK reconstruit via `WORKING/max/rebuild-apk-linux.sh` : `b15c8f34a5cd07ab0e6a28b94d8eec5f` (2 857 125 octets, versionName 0.7.2, API 19, signature SHA1).
- Installation autorisée via `adb install -r` : chemin final `/data/app/com.max.port-2.apk`, md5 installé vérifié identique, bibliothèque embarquée vérifiée. Données conservées.
- Résultats pendant la partie à 4 IA : les 10 premiers autosaves sont propres (`Defect11=0 Defect183=0 Defect151=0 result=0`), puis le 11e déclenche `MAXDBG Defect151 connector at [78,81] overlaps unit type=35 at [78,81]` et `MAXDBG plausibility summary: Defect11=0 Defect183=0 Defect151=1 result=1`. C'est donc bien le contrôle d'origine `Defect151` qui est responsable : un connecteur chevauchait une unité stationnaire de type 35 en [78,81]. Le code amont détruit ce connecteur en mémoire (`UnitsManager_DestroyUnit()` dans `saveloadchecks.cpp:223`) puis saute l'écriture de cet autosave.
- Aucun crash, la partie continue ; l'assertion ne s'est pas affichée car l'utilisateur avait choisi « Always ignore » dans la boîte SDL. Aucune modification des `ai*.cpp`, aucune donnée de jeu touchée, aucun flash/reboot. L'APK de diagnostic est **temporaire** : revenir à l'APK validé `064460b9...` une fois la partie terminée, et ne pas reconstruire l'archive firmware avant la fin des tests utilisateur.
- Cause racine identifiée par analyse lecture seule : le moteur générique autorise une unité stationnaire amie sur son propre connecteur (`access.cpp:208-218` : seul un autre connecteur ou un connecteur ennemi est bloqué) et le pipeline de construction finalise l'`ANTIAIR` sur la case de l'utilité sans retirer le connecteur (`unitinfo.cpp:4022-4027`), alors que ce nettoyage existe déjà dans `UnitsManager_BuildNext()` (`units_manager.cpp:5687-5691`) pour les unités construites avec un chemin. `UnitInfo::SetPosition()` ajoute l'unité au hash sans test d'occupation (`unitinfo.cpp:1962`). L'IA n'est que déclencheur de scénario via des appels génériques (`taskmanagebuildings.cpp`, `builder.cpp`) : aucun `ai*.cpp` n'est en cause.
- Correctif appliqué à la demande de l'utilisateur (qui a explicitement autorisé d'arrêter la partie en cours) : backup de l'état dans `BACKUPS/2026-11-25-antiair-fix/`, `am force-stop com.max.port`, puis deux modifications sans toucher aux `ai*.cpp` ni aux données de jeu :
  - `unitinfo.cpp` (`SpawnNewUnit()`) : juste avant `UnitsManager_DeployUnit()`, si l'unité construite est stationnaire et n'est pas `CNCT_4W`/`ROAD`/`BRIDGE`/`WTRPLTFM`, retirer le connecteur présent sur la case de l'utilité via `UnitsManager_RemoveConnections()` puis `UnitsManager_DestroyUnit()`. Les cônes/rubans restent gérés comme avant, et le nettoyage ne dépend plus de l'existence d'un chemin.
  - `saveloadchecks.cpp` (`Defect151`) : appeler `UnitsManager_RemoveConnections()` avant la destruction du connecteur, afin que les anciennes sauvegardes réparées ne conservent pas des liens réseau obsolètes.
  - Aucune garde ajoutée dans `Access_IsAccessible()` : elle risquerait d'invalider des sites de construction déjà planifiés par l'IA. L'IA n'a pas été modifiée (md5 des quatre `ai*.cpp` vérifiés avant/après).
- Build intermédiaire de vérification via `WORKING/max/build-native-linux.sh` : `.so` stripé `c8352e25a911585c3e4f5e604641a079`, APK `df211558a87fc904c9e51e9cb161b412` (`/data/app/com.max.port-1.apk`). Relance sans `FATAL` : cinq bilans consécutifs `MAXDBG plausibility summary ... Defect151=0 result=0`. L'utilisateur confirme que la DCA mobile a le bon sprite ; aucun autre symptôme n'est signalé.
- APK final propre après confirmation : backup des sources avec logs (`sources-correctif-avec-logs/`) et de l'APK intermédiaire (`APK-correctif-avec-logs.apk`) dans `BACKUPS/2026-11-25-antiair-fix/`, puis retrait des huit blocs temporaires `MAXDBG` / `TEMP DIAG` sans toucher au correctif. `.so` `5b6fb42467b4d324f0854f1494592bb7`, APK `06fe08f34689e129927f34e9dbf4ca57` installé via `adb install -r` -> `/data/app/com.max.port-2.apk` (md5 vérifié), données conservées (`settings.ini` 1 196 octets, `SAVE10.DTA` 459 843 octets). Relance sans `FATAL` : `stdout.txt` ne contient plus que la ligne de format de pixel. C'est l'APK à retenir pour la reconstruction finale de l'archive.
- Aucune donnée de jeu touchée, aucun flash ni reboot. L'archive `PACKAGING/sjgam-m27-custom-20261124.zip` reste inchangée pour l'instant : elle contient encore l'ancien APK `064460b9...` et devra être reconstruite une seule fois avec `06fe08f3...` après les tests utilisateur.

## Notes sur les divergences de sources

- Deux md5 sont associés au packaging du 24/11 : `4f85e77197d10c722355fa7cfec632c1` dans le journal noyau (vraisemblablement le `kernel.img` rempli à 12 582 912 octets) et `4ac17e4c7c14d56ae206dceb86361140` dans le journal global (archive ZIP finale, 231 399 853 octets, `unzip -t` OK). C'est ce dernier qui identifie l'archive retenue ; ne pas les confondre.
- Les journaux MAX mentionnent plusieurs « mapping final » successifs le 23/11 et des chemins d'APK (`com.max.port-1.apk` / `com.max.port-2.apk`) qui reflètent les réinstallations. La table retenue est celle de la phase 7 (A=Entrée, B=Retour, X=Centrer, Y=Changer de curseur, L1/L2=Zoom) et le chemin courant après l'APK final du 25/11 est `-2.apk` (le build intermédiaire était en `-1.apk`, l'APK diagnostic initial en `-2.apk`).
- Les dates d'en-tête de certains journaux sont obsolètes par rapport à leur contenu ; les dates indiquées ici sont celles des entrées elles-mêmes, pas celles des en-têtes.
- Le premier accès ADB/root, avant le 07/10, n'est pas relaté dans les journaux : seul l'état ultérieur est documenté.
