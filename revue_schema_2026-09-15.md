# Revue du schéma — z906_hardware.kicad_sch

**Date :** 2026-09-15
**Portée :** revue électrique du schéma uniquement (pas le PCB), comparaison avec les datasheets officielles des composants. **Aucune modification n'a été apportée au projet** — revue en lecture seule, conformément à la demande.

**Méthode :** 127 composants inspectés via l'API KiCad (broches, connexions filaires point-à-point, propriétés des symboles) + consultation des datasheets officielles (TI, Espressif, USB-IF) pour chaque circuit critique. ERC KiCad lancé en complément. Les outils de calcul de netlist globale (`list_schematic_nets`, `get_nets_list`) plantent sur ce fichier (trop volumineux, 21 000 lignes) — la vérification s'est donc faite broche par broche, ce qui est plus lent mais fiable.

---

## Résumé

| Sous-système | Verdict global |
|---|---|
| Alimentation USB-C + régulateurs (U2 buck, U6 LDO) | ✅ Conforme, aucun défaut trouvé |
| ESP32-S3-WROOM-1 (U1) | ✅ Conforme, 1 remarque mineure |
| PCM1860 — ADC audio (U7) | ⚠️ 1 écart réel identifié (voir ci-dessous) |
| PCM5102A — DAC audio (U3) | ✅ Conforme, correspond exactement aux circuits de référence TI |
| Connecteurs J2/J3/J4 | ❌ **Footprint manquant** — bloquant pour la fabrication |
| ERC KiCad | 25 violations, quasi toutes bénignes (expliquées ci-dessous) |

---

## 1. Alimentation USB-C et régulateurs

**J1 (USB-C receptacle) :**
- CC1 → R2 (5.1k) → GND, CC2 → R1 (5.1k) → GND, indépendamment l'un de l'autre. C'est exactement la configuration requise par la spec USB Type-C (§4.5.1.2) pour qu'un appareil soit reconnu comme périphérique (UFP/sink). **OK.**
- VBUS (4 broches) rejoint F1 (fusible réarmable 0ZLM0075FF2G, 0.75 A hold) avant d'alimenter tout le reste (+5V) — le fusible est bien en série avant toute charge. **OK.**
- D+/D- passent par U4 (TPD2EUSB30, protection ESD 2 canaux, topologie shunt vers GND) avant de rejoindre l'ESP32. Placement conforme aux recommandations TI (protection au plus près du connecteur). **OK.**

**U2 — TPS62A02A (buck 5V→3.3V numérique) :**
- L1 (1.0 µH) / C2 entrée (10 µF) / C4 sortie (22 µF) : correspond exactement au circuit type TI pour une application 2A. **OK.**
- Diviseur de contre-réaction : R3 (453k, côté haut) / R8 (100k, côté bas). Calcul : Vout = 0.6 V × (1 + 453/100) = **3.318 V**, soit à moins de 0.5 % de la cible 3.3 V. **OK.**
- EN relié directement à VIN (toujours actif). PG (power-good) non connecté — sortie optionnelle en drain ouvert, laisser flottant est autorisé par la datasheet, mais génère un avertissement ERC cosmétique.

**U6 — TPS7A4533 (LDO 5V→3.3VA, rail analogique propre) :**
- SENSE relié directement à OUT (exigence de la datasheet pour ce variant "sense-enabled"). **OK.**
- Cin/Cout = 10 µF chacun, largement au-dessus du minimum 2.2 µF requis. **OK.**

**U5 (TPD1E05U06DYA)** protège le rail +5V (VBUS). **U8 (TPD2E2U06DCK)** protège en réalité les lignes Tip/Ring de la prise jack 3.5 mm J2 (et non l'USB ou les connecteurs DE15 comme on aurait pu le supposer au premier abord) — placement cohérent.

**JP1/JP2 :** JP1 court-circuite EN/CHIP_PU de l'ESP32 vers GND (bouton reset manuel), JP2 court-circuite GPIO0 vers GND (bouton boot/flash manuel). Fonctionnement standard.

**R4 (0 Ω)** relie le rail +3.3V général au rail dédié "+3.3V_esp" de l'ESP32 — probablement un pont sécable pour isoler/mesurer la consommation du module. Pas un défaut.

**Aucun défaut de câblage trouvé dans ce sous-système.**

---

## 2. ESP32-S3-WROOM-1 (U1)

- Réseau RC sur EN/CHIP_PU : R5 (10k, pull-up vers +3.3V_esp) + C11 (1 µF vers GND) — correspond exactement aux préconisations Espressif pour un reset propre au démarrage. **OK.**
- GPIO0 : pas de pull-up externe, mais s'appuie sur le pull-up interne faible (WPU) actif par défaut au reset selon la datasheet, combiné au bouton JP2 pour forcer le mode téléchargement. Pratique courante et acceptée. **OK.**
- Toutes les broches GND (1, 40, 41) correctement reliées. **OK.**

**⚠️ Remarque mineure :** la broche 3V3 (pin 2) du module n'a pas de condensateur de découplage local (ex. 100 nF) directement à la broche — elle n'atteint le condensateur de réservoir (C10, 22 µF) que via le pont R4 (0 Ω). Les directives matérielles Espressif recommandent un découplage local au plus près de chaque broche d'alimentation. À corriger si possible (ajouter un 100 nF juste à côté de la broche 2 de U1).

- Plusieurs GPIO (dont GPIO45, GPIO46, GPIO3, et une vingtaine d'autres) sont laissés flottants sans étiquette ni marqueur "No Connect". Fonctionnellement correct (ils s'appuient sur la configuration par défaut au reset), mais c'est la cause de la majorité des avertissements ERC (voir section 5) — recommandé d'ajouter des flags "No Connect" pour documenter l'intention et nettoyer l'ERC.

---

## 3. PCM1860 — ADC audio (U7)

Alimentations et découplage tous conformes point par point à la datasheet TI :
- AVDD (analogique propre, rail +3.3VA) : C28 (10 µF) + C29 (0.1 µF) — **OK**, exactement la valeur recommandée.
- DVDD/IOVDD (numérique, rail +3.3V) : C30 (10 µF) + C31 (0.1 µF) — **OK**.
- Broche LDO interne : C32 (10 µF) + C33 (0.1 µF) vers GND — **OK**.
- VREF : C34 (2.2 µF) vers GND, non chargé par ailleurs — **OK** (minimum datasheet : 1 µF).
- XI relié à GND, XO en No-Connect, horloge externe (MCLK) injectée sur SCKI — mode "horloge externe" valide selon la datasheet, pas besoin de quartz. **OK.**
- Entrées analogiques inutilisées (VIN1M/VIN2M/VIN3P/VIN3M/VIN4P/VIN4M) correctement marquées No-Connect. **OK.**
- MD0–MD6 (broches de configuration matérielle) tous reliés à GND ensemble → configuration valide (entrée single-ended canal 1, format I2S, filtre FIR, mode esclave/autodétection). **OK.**

**❌ Écart réel identifié :** Les entrées analogiques utilisées (VIN1P/VIN2P, alimentées depuis le connecteur jack J2) sont couplées en **continu (DC)** — R9/R12 (100 Ω) + C36/C38 (1 nF) forment un filtre, et R10/R11 (100k) polarisent l'entrée, mais **aucun condensateur de blocage DC en série n'est présent** entre le jack et l'ADC. La datasheet PCM1860 (§9.3.1) exige explicitly un condensateur de blocage DC sur les entrées analogiques, car celles-ci sont conçues pour s'autopolariser autour de AVDD/2 en interne. Sans ce condensateur, tout décalage DC provenant de la source externe (casque/ligne branché sur J2) n'est pas isolé du point de polarisation interne de l'ADC — risque de distorsion, décalage du point de fonctionnement, voire dommage selon la source. **À corriger avant fabrication.**

---

## 4. PCM5102A — DAC audio (U3)

Tout correspond très précisément au circuit d'application de référence TI (datasheet SLAS859C, Fig. 33) :
- Pompe de charge : C22 (2.2 µF) directement entre CAPP/CAPM, sans résistance série — **OK**, valeur et topologie identiques à la référence TI.
- VNEG : C23 (2.2 µF) vers la masse — **OK**.
- Sortie LDO interne (LDOO) : C20 (0.1 µF) + C21 (10 µF), non utilisée pour alimenter autre chose — **OK**, conforme à l'exigence datasheet.
- DEMP/FLT/FMT tous reliés à GND → dé-emphasis désactivée, filtre à latence normale, format I2S — configuration explicitement documentée comme exemple type par TI. **OK.**
- XSMT relié directement au rail numérique +3.3V (mute logiciel désactivé en permanence) — pattern explicitement documenté comme acceptable par TI quand le contrôle de mute externe n'est pas nécessaire. **OK.**
- Filtre de sortie : R6/R7 (470 Ω) + C24/C25 (2.2 nF) sur OUTL/OUTR — **valeurs identiques** au filtre de sortie recommandé par TI. **OK.**
- Découplage AVDD/CPVDD (+3.3VA) et DVDD (+3.3V) tous conformes. **OK.**

**Précision utile :** la sortie audio du DAC (réseau "aux_left"/"aux_right") part vers **J3** (connecteur DE-15 vers l'amplificateur), **pas vers J2** comme on pourrait le supposer. J2 (jack 3.5 mm) est en réalité dédié à l'entrée de l'ADC (U7). Bon à savoir pour la suite de la relecture du schéma.

---

## 5. ERC KiCad — 25 violations

- **5 "Input Power pin not driven by any Output Power pin"** (réseaux +5V, GND, +3.3V, +3.3V_esp, et la broche LDO interne de U7) : après vérification manuelle de chaque réseau (tracé fil par fil), **tous sont correctement câblés électriquement**. Cet avertissement est un artefact de la façon dont KiCad type les broches (aucune broche du schéma n'est explicitement typée « Power Output » sur ces réseaux, car l'alimentation arrive de l'extérieur via l'USB, ou passe par une inductance/un retour Kelvin). Peut être supprimé en ajoutant des symboles `PWR_FLAG` sur ces réseaux si un ERC propre est souhaité — **pas un défaut électrique réel**.
- **20 "Pin not connected"** : 1 correspond à la broche Mic_Bias de U7 (non utilisée, entrée ligne uniquement — normal), les 19 autres correspondent à des GPIO inutilisées de l'ESP32-S3-WROOM-1 (U1), laissées flottantes par conception. Recommandé d'ajouter des marqueurs "No Connect" pour nettoyer l'ERC et documenter l'intention, mais ce ne sont pas des erreurs de câblage.

---

## 6. Connecteurs — footprints manquants (bloquant)

En vérifiant les propriétés de chaque composant, j'ai trouvé que **trois connecteurs n'ont aucun footprint assigné** (champ "Footprint" vide) :

- **J2** (AudioJack3, jack 3.5 mm)
- **J3** ("DE_15 Amplifier", DE-15 femelle haute densité)
- **J4** ("DE15 Console", DE-15 haute densité)

À titre de comparaison, J1 (USB-C) et les points de test (TP1–TP4) ont bien un footprint renseigné. Cette absence empêchera la mise à jour du PCB / la génération de la netlist pour ces trois connecteurs tant qu'un footprint ne leur sera pas assigné manuellement dans KiCad.

---

## Conclusion

Le schéma est globalement solide : l'alimentation (USB-C, buck, LDO) et le DAC audio sont conformes point par point aux datasheets et aux circuits de référence des fabricants. Deux points méritent une correction avant fabrication :

1. **Ajouter un condensateur de blocage DC** sur les entrées analogiques de l'ADC PCM1860 (U7, broches VIN1P/VIN2P côté J2).
2. **Assigner un footprint** à J2, J3 et J4.

Et deux améliorations mineures recommandées (non bloquantes) :
- Ajouter un découplage local (100 nF) directement à la broche 3V3 de l'ESP32-S3-WROOM-1.
- Marquer explicitement en "No Connect" les GPIO inutilisées de l'ESP32 pour nettoyer l'ERC.
