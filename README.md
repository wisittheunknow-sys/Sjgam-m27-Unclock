# Sjgam-m27-Unclock

Unlock and make the SJgam M27 more user-friendly.

## Objectif

Consolider le travail fait sur la console SJgam M27 (Android 4.4 / rk3188) :

- noyau M38 (gadget USB, stick droit = souris firmware, correctif reboot/extinction) ;
- frontend SJgam (menu APKS, grille d'icônes) ;
- port M.A.X. (JNI, chemins SD, rendu, son, IA d'origine intacte) ;
- Mouse OSD, daemons souris et touch firmware ;
- stubs Google/Vending (billing conservé pour Riptide GP2) ;
- Drastic (mappings clavier, hors port MAX).

## Règles du dépôt

- **Aucune donnée originale de jeu** ne doit être versionnée. Le joueur fournit sa propre copie sur SD.
- Les sources `ai*.cpp` du port MAX ne sont pas modifiées (IA d'origine préservée).
- Ne pas inclure de loader Rockchip (distribution non autorisée).
- Les images binaires, dumps et APK ne sont pas versionnés : seules les sources, scripts, notes et checksums sont suivis.

## Organisation prévue

- `docs/` : journaux d'état (`STATE.md`), décisions, procédures.
- `tools/` : scripts reproductibles (flash, rebuild Mouse OSD, etc.).
- `kernel/` : config et patches du noyau M38.
- `frontend/` : sources du frontend.
- `max/` : sources du port (hors données de jeu).
- `packaging/` : manifeste et procédures (sans images ni zip).

## État

Travail en cours sur console ; archive firmware de test : `sjgam-m27-custom-20261124.zip` (non versionnée).
