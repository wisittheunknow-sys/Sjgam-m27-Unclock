# Sjgam-m27-Unclock

Unlock and make the SJgam M27 more user-friendly.

## Constat

- fraude au remarking, Chip avec la mention RK3568B2 --> la réalité : RK3188
- Pas de chip wifi, mais antenne présente sur la board, surement pour des manettes type ps2 sans fils 
- Console étudiée avec board R10 rev 2.3 avec chipset RK3188, 512 mb de ram, Android 4.4 AOSP inscrit en totalité dans la Nand.
- Luncher propre au fabricant embarquant PPSSPP(gametemplate.apk), se servant de cores Libretro pour l'émulation bas niveau (jusqu'a PS1), émulation en APK natif pour N64, dreamcast, PPSSPP
- Drastic (NDS) absent, pourtant le frontend Sjgam y fait appel pour la NDS (categorie NDS volontairement désactivée)
- Core Dosbox présent, mais désactivé lui aussi
- com.emu présent (retroarch), mais quick menu minimal
- Inventaire des roms via des fichiers gamelist.xml, lus et compilé en cachegames.db (sql) par le frontend, la suppression de celui-ci provoque la création d'une nouvelle cachegames.db
- Mapping du joycon droit jumelé à ABXY
- Les joycons sont des boutons 4 appuis pas des axes. 24 inputs au total (rk29-keyboard)
- impossibilité de quitter le frontend pour android, et verrou si la carte sd n'a pas les fichiers attendus.
- ADB non actif, recovery condamné par le fabricant, accès au mode LOADER/maskrom rockchip en maintenant VOL- avec cable usb data branché au pc.

## Objectif

- ADB actif sur système live
- retrouver les Settings Android
- Retrouver Android et la possibilité d'installer des APKs
- Retrouver un quick menu rétroarch fourni et paramétrable
- Avoir un luncher qui prend en compte les roms ajoutées ou supprimées de manière automatique
- Implémenter un clavier virtuel navigable au D-pad
- retrouver l'émulation NDS et DOSBOX
- implémenter une souris virtuelle pour compenser l'absence de dalle tactile 

## Règles du dépôt

- **Aucune donnée originale de jeu** ne doit être versionnée. Le joueur fournit ses propres copies sur SD.
- Les images binaires, dumps et APK ne sont pas versionnés : seules les sources, scripts, notes et checksums sont suivis.

## Organisation prévue

- `docs/` : journaux d'état (`STATE.md`), décisions, procédures.
- `tools/` : scripts reproductibles (flash, rebuild Mouse OSD, etc.).
- `kernel/` : config et patches du noyau M38.
- `frontend/` : sources du frontend.
- `max/` : sources du port (hors données de jeu).
- `packaging/` : manifeste et procédures (sans images ni zip).

## État

- Kernel ver.m38 Android recomplilé et configuré grace aux sources : https://github.com/bbelos/rk3188-kernel/tree/master, (gadget USB ajouté, ADB actif, séparation du joycon droit de ABXY, patch du du crash du a la veille)
- Conservation du System Android 4.4 (meilleur compromis pour ce chipset (rapide, éfficace))
- Installation d'un settings.apk issu d'une rom tablette kingston rk3188, apk patché car erreur concernant l'interrogation de la connectivité réseau de la console
- Clavier virtuel implémenté, controllable au D-pad
- Souris virtuelle implémentée sur joycon droit (appui sur r1+r2 pendant 3 secondes, maintenir SELECT pendant 2 secondes pour changer de mode CLIC à TOUCH)
- Création d'un nouveau frontend hybrid (android/emulation) avec actualisation des roms au lancement, customisable via le dossier UI sur la carte SD
- Mise en place de Rétroarch et quickmenu complet (accès via combo SELECT+START)
- Patch des cores Libretro Sjgam originaux (ils ont une entête qui les rends inutilisable par Rétroarch)
- Conservation des APK Sjgam originaux (flycast, mupen64plus) car l'émulation de "haut niveau" n'est pas possible via des cores (trop lent), apk obligatoires
- intégration au frontend des GPIO des RGB joycons et du réglage du BACKLIGHT
- installation de Drastic pour la NDS et configuration du mapping
- installation de PPSSPP, qui était intégré au frontend original
- Ajout de MX PLAYER pour lecture de vidéos et lecture MP3 (problème avec VLC)

  
- Projet M.A.X. Port (https://github.com/klei1984/max) pour android 4.4 dédié Sjgam m27 pour le fun  
