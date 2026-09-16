# Akai Fire VST Controler — Spécification d'implémentation

Script MIDI FL Studio (Python) pour contrôleur Akai Fire.
Ce document est autonome : il contient tout le nécessaire pour coder la V1 sans autre contexte.

**Statut : v1.0 implémentée (2026-09-08, refactor grille orthogonale 2026-09-09).**
Phases 1 à 6 codées et smoke-testées hors FL sur stubs des modules FL (57 vérifications :
protocole pads, bargraph, encodeurs, pages de rangées, pages de grille globales,
orthogonalité des deux blocs, couleurs par colonne, preset générique/réel/malformé,
scanner, caches anti-spam, pertes de cible). Critères de sortie restant à vérifier
**sur le matériel réel**. Phase 0 : validée (§6.1, §7).

**Modèle orthogonal (décision 2026-09-09).** Les 6 pages de gauche (rangées continus)
et les 4 pages de droite (grille discrète) sont **totalement indépendantes** : changer
une page de rangées ne change pas la grille, et inversement. La grille est **globale**
(4 pages × 4 colonnes = 16 mappings discrets), pas imbriquée par page de rangées
(l'ancien modèle 6×4×4=96 est abandonné). Chaque colonne de la grille a une couleur
fixe propre (`GridColors`), indépendante de la page de rangées courante.

---

## 1. Objectif et philosophie

Piloter les paramètres d'un synthé VST depuis un Akai Fire, en gardant sous les doigts
**les 20-30 paramètres réellement manipulés** plutôt qu'en reproduisant l'intégralité de
l'interface du plugin.

Règle de conception unique, à appliquer à chaque décision :

> **Une action physique doit faire gagner du temps par rapport à la souris.**

Un contrôleur qui oblige à naviguer dans des menus, ou à regarder l'écran pour savoir où
on en est, est plus lent que la souris et donc inutile. C'est le défaut de la majorité des
contrôleurs génériques.

Corollaires :

- **Aucun état invisible.** À tout instant, l'état complet du contrôleur doit être lisible
  sur les pads et les LED, sans consulter l'écran.
- **Pas de navigation.** Toute page est atteignable en un appui.
- **L'écran informe, il ne navigue pas.**
- **Refuser toute fonction simplement parce qu'elle est techniquement possible.**

L'intérêt principal du concept est la **représentation physique simultanée** de plusieurs
paramètres. La souris donne de la précision, mais pas une vue d'ensemble. Exemple ADSR :

```
ATTACK   ● ● ● ● ● ● ◐ ○ ○ ○ ○ ○
DECAY    ● ● ● ● ○ ○ ○ ○ ○ ○ ○ ○
SUSTAIN  ● ● ● ● ● ● ● ● ● ◐ ○ ○
RELEASE  ● ● ◐ ○ ○ ○ ○ ○ ○ ○ ○ ○
```

La forme de l'enveloppe est lisible d'un coup d'œil, sans lire quatre nombres.

### Contexte matériel de l'utilisateur

Deux unités Akai Fire :
- Unité A : script `Akai Fire EndGAme v1.5` (séquenceur, contrôle FL)
- Unité B : ce script

L'attribution d'un script à une unité se fait dans les réglages MIDI de FL Studio.

---

## 2. Périmètre

### Dans la V1

- 6 pages de mapping (boutons de gauche), chacune exposant 4 paramètres continus
- 4 pages de grille globales (boutons de droite), indépendantes des pages de mapping,
  chacune exposant 4 paramètres discrets — **16 mappings discrets au total**, pas 96
- Bargraph 12 pads par paramètre continu, avec pad partiel en luminosité
- Clic sur un pad du bargraph = pose la valeur correspondant à sa position
- Encodeur = ajustement fin relatif depuis la valeur courante
- Bloc 4×4 de paramètres discrets, paginé sur 4 pages globales, une couleur fixe par colonne
- Un fichier preset par VST, définissant l'intégralité du mapping
- Affichage OLED du nom et de la valeur du paramètre au toucher d'un encodeur
- Outil de scan des paramètres d'un plugin, pour construire les presets

### Explicitement hors V1

Ne pas implémenter, ne pas préparer, ne pas ajouter de champ « au cas où » :

min/max par paramètre · courbes de réponse · macros · automation / enregistrement ·
effets du mixer · accélération des encodeurs · configuration depuis le boîtier ·
GUI d'édition · multi-device · transport FL.

**Rationale.** Le concept doit être testé avec le minimum de logique. Les fonctions
manquantes se révéleront à l'usage, et seront alors implémentées correctement parce qu'on
saura ce qu'elles doivent faire. L'inverse produit une usine à gaz avant même d'avoir
essayé l'interface.

### Seule concession au futur

Deux choix structurels, et rien d'autre :

1. **Tous les accès au plugin passent par `param_bridge.py`.** Aucun autre fichier
   n'importe `plugins`. Ajouter min/max ou une courbe plus tard = modifier ce seul fichier.
2. **Les presets utilisent des `dict`, pas des tuples.** Ajouter une clé ne cassera aucun
   preset existant.

Ne jamais coder en dur l'hypothèse « il y a exactement 6 pages ». Le nombre de pages doit
venir de `len(PAGES)`. Extension prévisible : `SELECT` + bouton de page = pages 7 à 12.

---

## 3. Faits matériels vérifiés

Ces valeurs sont confirmées par le protocole relevé dans
`../Akai Fire EndGAme V1.5/analysis.md` et le code de la v1.5. Les utiliser telles quelles.

### 3.1 Pads

64 pads, 4 rangées × 16 colonnes. Note MIDI Note On / Note Off.

```
note   = PadFirst + row * 16 + col        # PadFirst = 0x36 (54), PadLast = 0x75 (117)
index  = event.data1 - PadFirst           # 0..63
row    = index // PadsStride              # PadsStride = 16
col    = index %  PadsStride
```

**Sections physiques.** La grille est matériellement divisée en 4 sections de 4 colonnes,
séparées par un espace visible sur le boîtier :

| Section | Colonnes (1-based) | Colonnes (0-based) |
|---|---|---|
| A | 1-4 | 0-3 |
| B | 5-8 | 4-7 |
| C | 9-12 | 8-11 |
| D | 13-16 | 12-15 |

Le découpage fonctionnel de ce script tombe exactement sur ces frontières :
**bargraph = A+B+C (colonnes 0-11)**, **bloc discret = D (colonnes 12-15)**. Le boîtier
sépare donc visuellement les deux zones sans effort de coloration. Les deux gaps internes
du bargraph tombent à 33 % et 66 %, ce qui fournit des graduations gratuites.

Vélocité des pads : 32-127 (le Fire a un seuil élevé). Ne pas s'en servir en V1 :
tout clic vaut clic.

### 3.2 LED des pads

SysEx, RGB **7 bits** (0x00-0x7F par composante) :

```python
SendMessageToDevice(MsgIDSetRGBPadLedState, len(data), data)
# data = bytearray, 4 octets par pad : [padIndex, R, G, B]
```

Les couleurs sont manipulées en `0xRRGGBB` 24 bits dans le code et divisées par 2 au
moment de l'envoi (voir `AddPadDataCol` de la v1.5, à reprendre tel quel).

**`BtnMap[64]` est obligatoire.** C'est un cache de la dernière couleur envoyée par pad.
Sans lui, le rafraîchissement continu pendant qu'un encodeur tourne produit un flickering
visible et sature le port MIDI. Ne jamais envoyer une couleur identique à la précédente.

### 3.3 LED des boutons

CC via `SendCC(ID, value)`.

- Boutons monochromes : `SingleColorOff` = 0, `SingleColorHalfBright` = 1, `SingleColorFull` = 2
- Boutons bicolores : `DualColorOff` = 0, `DualColorHalfBright1/2` = 1/2, `DualColorFull1/2` = 3/4
  (`DualColorFull1` = rouge, `DualColorFull2` = vert)
- Les boutons `MUTE 1-4` n'ont **qu'une LED verte**, pas de rouge.
- `IDTrackSel1-4` (0x28-0x2B) ne sont **pas des boutons** : ce sont les 4 bandeaux LED
  au-dessus des boutons Mute. Sortie uniquement.

### 3.4 Encodeurs — point critique

Les 5 encodeurs du Fire sont **relatifs sans fin**. Ils n'envoient jamais de position
absolue. Décodage du delta :

```python
delta = event.data2
if delta >= 0x40:          # v1.5 utilise 0x7F // 2
    delta = -(0x80 - delta)
```

Les 4 encodeurs de gauche sont **tactiles** : ils envoient Note On au contact du capuchon
et Note Off au relâchement, sur la même valeur que leur CC.

| Encodeur | CC rotation | Note toucher |
|---|---|---|
| 1 (VOLUME) | 0x10 | 0x10 |
| 2 (PAN) | 0x11 | 0x11 |
| 3 (FILTER) | 0x12 | 0x12 |
| 4 (RESONANCE) | 0x13 | 0x13 |
| SELECT | 0x76 | **0x19** (appui, pas toucher) |

**Attention :** `Akai Fire EndGAme - Contexte IA.md` annonçait `0x77` pour l'appui SELECT.
C'était faux, la valeur est `0x19`. Le bug a été corrigé dans ce fichier.

L'appui SELECT envoie Note On à l'appui et Note Off au relâchement, il peut donc servir de
modificateur maintenu (`SelectHeld`), exactement comme `ShiftHeld` dans la v1.5.

**Conséquences pour le design, à ne pas contourner :**

- Il n'y a pas de position de potentiomètre à lire. La valeur vit dans le plugin.
- Aucun problème de saut de valeur ni de pickup au changement de page ou de VST.
- Le clic sur un pad et la rotation de l'encodeur sont deux gestes complémentaires :
  le pad pose une valeur absolue, l'encodeur la fait dériver depuis là.

### 3.5 Boutons

| Bouton | Constante | CC |
|---|---|---|
| PATTERN ▲ | `IDPatternUp` | 0x1F |
| PATTERN ▼ | `IDPatternDown` | 0x20 |
| BROWSER | `IDBrowser` | 0x21 |
| GRID ◀ | `IDBankL` | 0x22 |
| GRID ▶ | `IDBankR` | 0x23 |
| MUTE 1-4 | `IDMute1..4` | 0x24-0x27 |
| STEP | `IDStepSeq` | 0x2C |
| NOTE | `IDNote` | 0x2D |
| DRUM | `IDDrum` | 0x2E |
| PERFORM | `IDPerform` | 0x2F |
| SHIFT | `IDShift` | 0x30 |
| ALT | `IDAlt` | 0x31 |
| PATTERN/SONG | `IDPatternSong` | 0x32 |
| PLAY | `IDPlay` | 0x33 |
| STOP | `IDStop` | 0x34 |
| REC | `IDRec` | 0x35 |

La rangée du bas compte **10 boutons physiques**, correspondant aux CC contigus 0x2C-0x35 :
6 à gauche (STEP → ALT) et 4 à droite (PATTERN/SONG → REC).

### 3.6 OLED

128 × 64. Piloté par le module `screen` de FL (non documenté publiquement).
Reprendre `display.py` de la v1.5 sans modification : `InitScreen`, `DeInitScreen`,
`DisplayText`, `DisplayBar`, `DisplayTimedText`, `ClearDisplay`.

---

## 4. Mapping des contrôles

```
┌─ SELECT ─┬─ 4 encodeurs ─┬──── OLED ────┬─ PATTERN ▲▼ · BROWSER · GRID ◀▶ ─┐
│  = Shift │  4 params     │  info param  │       réservés (voir §4.4)        │
└──────────┴───────────────┴──────────────┴───────────────────────────────────┘

 MUTE1 │ ●●●●●●◐○○○○○ │ ○○○○ │   rangée 0 : param continu 0  +  4 discrets (page grille courante)
 MUTE2 │ ●●●●○○○○○○○○ │ ●○○○ │   rangée 1 : param continu 1  +  4 discrets (page grille courante)
 MUTE3 │ ●●●●●●●●●◐○○ │ ○●○○ │   rangée 2 : param continu 2  +  4 discrets (page grille courante)
 MUTE4 │ ●●◐○○○○○○○○○ │ ○○○● │   rangée 3 : param continu 3  +  4 discrets (page grille courante)
       └── A + B + C ──┘ └─ D ─┘
          cols 0-11      cols 12-15

[ STEP  NOTE  DRUM  PERFORM  SHIFT  ALT ]   [ PAT/SONG  PLAY  STOP  REC ]
   6 pages de RANGÉES (continu, bloc G)        4 pages de GRILLE (discret, bloc D)
        indépendantes des pages de grille          indépendantes des pages de rangées
```

### 4.1 Paramètres continus — 4 rangées × 12 pads

- La rangée `r` est associée à l'encodeur `r`.
- Colonnes 0-11 : bargraph de la valeur du paramètre continu de la rangée.
- **Clic sur le pad de colonne `c`** → écrit la valeur `c / 11.0` (donc 0.0 à 1.0).
- **Rotation de l'encodeur `r`** → `write(clamp(read() + delta_step))`, avec
  `delta_step = signof(delta) / 127.0`. Un cran = un pas de 1/127.
  **Ne pas implémenter l'accélération en V1** : forcer `signof`, sinon le réglage fin
  devient impossible.
- **Toucher l'encodeur `r`** (Note On sans rotation) → mémorise le paramètre de la
  rangée pour l'affichage OLED persistant (voir §4.6), **sans rien modifier**.

Total : **6 pages × 4 rangées = 24 paramètres continus.**

### 4.2 Paramètres discrets — bloc D, 4 pages globales, 2 formats

- La section D porte des **paramètres discrets** : le pad allumé vif indique la
  valeur active.
- **Pages 1-3 : format DUAL — 8 params × 2 pads.** La plupart des synths ont des
  boutons à 2 états (formes d'onde, on/off) ; 4 pads par paramètre en était trop.
  La grille 4×4 est donc découpée en 8 colonnes verticales de 2 pads :
  - entrées 0-3 : moitié HAUTE (rangées 0-1), couleur `GridColors[page]`
  - entrées 4-7 : moitié BASSE (rangées 2-3), couleur `GridColors2[page]`
  - pad HAUT d'une entrée = **activé (on)** = `values[0]`, pad BAS = **off/normal** = `values[1]` (défaut `[1.0, 0.0]`)
  - les 2 couleurs sont de la même famille de teinte mais clairement distinctes :
    elles séparent visuellement les 2 groupes de 4 params
- **Page 4 : format QUAD** (inchangé) — 4 colonnes × 4 pads, jusqu'à 4 valeurs
  par colonne (ex. type de filtre LP12/LP24/HP/BP).
- Une entrée peut n'utiliser qu'une partie de ses pads (champ `count`) : les pads
  au-delà restent éteints et inertes.
- Les 4 boutons de droite de la rangée du bas sélectionnent une des 4 **pages de
  grille globales**. Cette page de grille est **unique et globale** — elle est
  **indépendante** de la page de rangées courante : changer de page de rangées
  (boutons de gauche) ne change ni la page de grille, ni les mappings de la
  grille. Inversement, changer de page de grille ne change rien aux rangées.
- **Couleur par page, pas par colonne.** La moitié haute a la couleur
  `GridColors[page]`, la moitié basse `GridColors2[page]` — uniformes sur toute
  la moitié. La couleur identifie la page active d'un coup d'œil. Une entrée de
  preset peut surcharger via le champ `'color'`.

Total : **pages 1-3 : 8 × 3 = 24 params 2 états + page 4 : 4 × 4 = 4 params
multi-valeurs = 28 paramètres discrets.**

Cette capacité est volontairement dimensionnée pour les paramètres 2 états
(formes d'onde, on/off, sync) qui dominent sur un synthé. Une page de grille sans
mapping affiche des pads éteints, sans code supplémentaire.

### 4.3 Boutons MUTE 1-4 — mode LEARN (mapping continu)

Le toucher d'encodeur fait déjà la consultation OLED (§4.6) — les boutons
MUTE sont **libérés pour le mode LEARN** :

1. **Appui MUTE r** → le learn est **armé sur la rangée r** de la page courante :
   - la LED MUTE r **clignote** (2 Hz),
   - la rangée de pads correspondante **clignote** (fantôme atténué si la
     rangée est vide, pour rester repérable),
   - l'OLED affiche `LEARN R<r> : touchez le VST`.
2. On touche un paramètre **dans le VST** (potard ou bouton) : le paramètre
   nommé qui change est **mappé automatiquement** sur la rangée — nom affiché
   sur l'OLED (`MAPPE : <LABEL>`), **fichier preset réécrit** et rechargé à
   chaud, sortie du mode.
3. **Ré-appui MUTE r** → annulation. Timeout 30 s → sortie automatique.

Détection : snapshot de tous les params nommés à l'armement, relecture à
chaque tick idle. **Exactement un** paramètre changeant = candidat ; plusieurs
changés (LFO, automation) = dérive absorbée, on attend un geste isolé.
Param sans nom : non mappable (pas de label). Pendant le learn, **les pads et
encodeurs du Fire sont verrouillés** (pas d'auto-mappage accidentel) ainsi que
les boutons de page, BROWSER et PATTERN ▲▼.

Sur un preset **générique**, le learn **crée** `presets/<vst>.py`
(PLUGIN_MATCH = nom du plugin) — c'est la voie rapide pour se construire un
preset paramètre par paramètre. La réécriture du fichier perd les commentaires
manuels ; les mappings sont conservés intégralement.

### 4.3b GRID ◀ ▶ — mode LEARN (mapping discret)

`GRID ▶` (emplacement suivant) / `GRID ◀` (précédent) arment le learn **discret**
et font **défiler + clignoter** les emplacements de la page de grille courante
(8 slots dual / 4 quad, y compris vides — fantôme atténué). Les deux LEDs GRID
restent allumées pendant le mode. Un clic sur un bouton du VST mappe le paramètre
sur l'emplacement qui clignote (dual : `count: 2, values = DUAL_DEFAULT` — pad HAUT = on, pad BAS = off ; quad :
`count: 4` ; si l'entrée existait, le `count`/`values` est adapté au type de la page), puis **sortie
du mode** — les LEDs GRID s'éteignent. Même détection/verrouillage/timeout que
§4.3. Si la page de grille est vide, une page dual de 8 slots est créée.
Après un mapping réussi, la position mémorisée avance de **+1** (GRID droite) ou
**-1** (GRID gauche) selon la direction utilisée pour armer le learn — permet de
mapper rapidement param 1, GRID droite, param 2, GRID droite, etc.

### 4.4 Contrôles réservés et inactifs en V1

À câbler comme no-op explicite, avec LED éteinte :

- `BROWSER` — destiné à changer le type d'une colonne (`linear` / `increment` / `count`).
  **Exception V1 : `BROWSER` (appui simple) déclenche le générateur de preset
  intelligent (§6.5).** Le générateur analyse les noms de paramètres du plugin
  courant, les catégorise par mots-clés (ADSR, filtre, OSC, LFO, etc.), et écrit un
  fichier `presets/<vst>.py` immédiatement utilisable. Le preset est rechargé à
  chaud — l'utilisateur peut jouer tout de suite.
- `SELECT` rotation — **active en V1 : cycle des modes dual par défaut (§4.4c)**

Historiquement réservés, désormais actifs : `GRID ◀ ▶` = LEARN discret (§4.3b) et
`PATTERN ▲ ▼` = changement de preset VST (§4.4b).

### 4.4c SELECT rotation — mode dual par défaut (DUAL_DEFAULT)

La rotation du SELECT (CC `0x76`) cycle entre deux modes dual :

| Mode | Valeurs | Sens |
|------|---------|------|
| `UP=OFF` | `[0.0, 1.0]` | pad HAUT = off, pad BAS = activé (on) (défaut) |
| `UP=ON`  | `[1.0, 0.0]` | pad HAUT = activé (on), pad BAS = off (Repro-5 et VST inversés) |

- Deux modes seulement — pas de système de configuration complexe.
- Le mode actif est affiché sur l'OLED (`DUAL : UP=ON` / `DUAL : UP=OFF`).
- Le mode est **persisté dans le preset** via `DUAL_DEFAULT = [1.0, 0.0]` (ou `[0.0, 1.0]`).
- Les **entrées existantes avec des `values` explicites ne sont pas modifiées** — seul le défaut pour les nouvelles entrées (learn ou génération) change.
- Les **pages quad (page 4) ne sont pas affectées** par ce réglage.
- Utile pour les VST où la convention de valeur diffère (Repro-1 vs Repro-5 par exemple).
- Si `DUAL_DEFAULT` est absent du preset, le défaut `[0.0, 1.0]` est utilisé.

**La configuration depuis le boîtier n'est pas dans la V1.** Elle demanderait un mode
d'édition, une UI OLED, une persistance disque et un moyen d'annuler — soit la plus grosse
pièce de machinerie du projet, pour un réglage effectué une fois par VST. De plus les
index de paramètres doivent de toute façon être écrits dans un fichier preset, puisque
c'est le scanner qui les fournit ; y mettre le type de colonne ne coûte rien de plus.

`SELECT` **appui** est en revanche actif dès la V1 : c'est le modificateur `SelectHeld`.
Il n'a aucune fonction attribuée en V1, mais l'état doit être suivi et exposé, car c'est
le point d'extension prévu (pages 7-12).

### 4.4b PATTERN ▲ ▼ — changement de preset du VST

`PATTERN ▲` / `PATTERN ▼` chargent le preset **suivant / précédent** du VST courant.
La mécanique dépend entièrement du VST (sysex, program change, flèches, CC) — le
preset déclare donc comment faire via :

```python
PRESET_CHANGE = {'param': 47, 'step': 0.03125}   # 1/32 = sélecteur 32 positions
```

`param` = index du paramètre sélecteur de preset du VST, `step` = pas entre deux
presets. `PATTERN ▲` : valeur courante arrondie au pas le plus proche, puis + step
(clampé 0.0-1.0) ; `PATTERN ▼` : - step. Le snap tolère une valeur hors grille.
Sans `PRESET_CHANGE` dans le preset : no-op avec message OLED.

Le générateur (§6.5) détecte automatiquement un paramètre nommé `preset`,
`program` ou `patch` et écrit `PRESET_CHANGE` dans le preset généré.

**Limitation assumée** : ce mécanisme couvre les VST dont le sélecteur de preset est
un paramètre normal (le cas le plus courant). Program change MIDI et sysex ne sont
pas câblés en V1 — l'API script FL n'offre pas d'envoi MIDI direct à un plugin.

### 4.5 Pas de transport

`PLAY` / `STOP` / `REC` sont des sélecteurs de **page de grille** (bloc D, droit), pas
le transport FL. Décision assumée : l'utilisateur dispose d'une seconde unité Fire sous
EndGame pour le transport, et en usage solo la souris reste dans l'autre main.
Ne pas ajouter de transport, même sous modificateur.

### 4.6 OLED — valeur temps réel persistante

L'OLED affiche deux lignes persistantes, mises à jour à chaque tick idle (~100 ms) :

- **Ligne 1** (Font6x16, y=0) : `TopText` = nom de la page de rangées + nom du plugin.
- **Ligne 2** (Font6x8, y=40) : `ParamText` = `label = valeur` du **dernier paramètre
  touché**, en **temps réel**.

Le « dernier paramètre touché » est mémorisé dès qu'on :
- touche un encodeur (Note On),
- tourne un encodeur (CC),
- clique un pad du bargraph,
- clique un pad de la grille,
- presse un bouton MUTE.

La valeur est **relue directement à chaque tick idle** depuis le plugin — elle suit
donc l'encodeur en temps réel, et **ne disparaît pas** (pas de timeout). Si on bouge
le même paramètre à la souris dans la fenêtre du VST, l'OLED se met à jour aussi.

Les **notifications transitoires** (changement de page, scan, perte de cible)
remplacent temporairement `ParamText` pendant ~2 s, puis `ParamText` revient au
dernier paramètre touché.

### 4.7 Rafraîchissement périodique des pads

Les valeurs de tous les paramètres affichés sont **relues à chaque tick idle**
(~100 ms) et les pads ne sont renvoyés que si une couleur change (caches
`BtnMap` / `_ledCache`). De plus, **chaque seconde** (`FullRefreshPeriod`), le
script force le renvoi complet des 64 pads (reset du cache `BtnMap`).

Pourquoi : quand on change de preset **sur le synthé** (et non depuis le
contrôleur), FL ne pousse pas toujours les nouvelles valeurs vers le script ;
le renvoi périodique resynchronise l'affichage dès que les valeurs redeviennent
lisibles, et répare aussi un sysex perdu. Coût : un sysex de 256 octets par
seconde — négligeable.

### 4.8 LEDs TrackSel 1-4 — suivi du beat et de la mesure

Les 4 LEDs d'état à côté des boutons MUTE (sorties CC 0x28-0x2B, jamais des
boutons) suivent le **beat et la mesure** pendant le play — logique copiée
d'EndGame (`drum_mode.py`) :

- remplissage progressif : temps 1 → 1 LED, temps 2 → 2 LEDs, etc. (vert),
- **rouge** (`SingleColorHalfBright`) chaque **4e mesure**,
- tout éteint à l'arrêt.

Callback FL : `OnUpdateBeatIndicator(value)` — 0 = arrêt, 1 = temps fort,
2 = autres temps. Implémenté dans `fire_modules/beat_leds.py`, seul module qui
importe `transport` (même logique d'isolation que `param_bridge` pour `plugins`).

---

## 5. Architecture

```
Akai Fire VST Controler/
├── SPEC.md                        # ce document
├── README.md                      # installation et usage
├── device_FireVST.py              # script principal, classe TFireVST
├── Akai Fire VST Controler Spike/ # spike phase 0 — jetable, garder jusqu'au
│   ├── device_SpikeVST.py         # test optionnel sur plugin natif
│   └── spike_helpers.py
└── fire_modules/
    ├── __init__.py
    ├── constants.py               # adapté de la v1.5, purgé du step seq
    ├── display.py                 # copie de la v1.5, sans modification
    ├── fire_utils.py              # conversions HSV (Python pur, sans API FL)
    ├── param_bridge.py            # SEUL fichier qui importe `plugins`
    ├── generic_preset.py          # mapping automatique quand il n'y a pas de preset
    ├── vst_page.py                # rendu et interaction d'une page
    ├── vst_mode.py                # état global : pages de rangées, page de grille globale, dispatch
    ├── vst_scanner.py             # dump des params + génération preset (BROWSER)
    ├── preset_generator.py        # catégorisation intelligente par mots-clés
    ├── learn.py                   # mode Learn : snapshot, polling, détection
    ├── beat_leds.py               # LEDs TrackSel = beat/mesure (importe transport)
    ├── scan_<plugin>.txt          # sortie du scanner (généré)
    └── presets/
        ├── __init__.py            # registre + résolution par nom de plugin
        ├── exemple.py             # modèle commenté — ne correspond à aucun plugin
        └── <nom_vst>.py           # un fichier par VST
```

### Réutilisation depuis la v1.5

Le dossier `../Akai Fire EndGAme V1.5/` est la référence. Reprendre **par copie**, pas par
import — les deux projets doivent rester indépendants.

À copier tel quel :
- `fire_modules/display.py` (entier)
- `fire_modules/constants.py` (en retirant tout ce qui concerne step seq, gammes,
  harmonicScales, modes Note/Drum/Perf, params de step)
- De `device_Fire.py` : `SendCC`, `SendMessageToDevice`, `AddPadDataCol`, `ScaleColor`,
  `ClearAllPads`, `ClearAllButtons`, `ClearBtnMap`, `ClearDisplayText`, et la structure
  générale de `OnInit` / `OnDeInit` / `OnIdle`.

À **ne pas** copier, sous aucun prétexte :
- Tout le multi-device : `MultiDeviceMode`, `SetAsMasterDevice`, `SetAsSlaveDevice`,
  `CheckForMasterDevice`, `DispatchMessageToDeviceScripts`, les handlers `SM_*`
- Les modes StepSeq, StepEdit, Note, Drum, Perf, ChordSelect
- `key_sender.py` et le watcher externe
- `fl_control_mode.py` / `fl_control_config.py`

### En-tête du script — critique

```python
#   name=Akai Fire VST Controler
# url=
```

**Aucune ligne `receiveFrom`.** C'est ce qui isole ce script d'EndGame.

Raison : EndGame diffuse `SM_SetAsSlave` en broadcast (`device.dispatch(-1, ...)`) lorsque
l'utilisateur maintient `SHIFT`+`ALT` environ 2 secondes. Sans `receiveFrom`, FL ne compte
pas ce script comme destinataire : il n'apparaît pas dans `dispatchReceiverCount()`, ne
reçoit pas le broadcast, et ne peut donc pas être asservi par accident. Comme le mapping
de ce script utilise `SHIFT` et `ALT` comme pages 5 et 6, le geste déclencheur existe et
peut survenir.

---

## 6. Spécification des modules

### 6.1 `param_bridge.py`

Seule couche en contact avec le module `plugins` de FL. Interface volontairement étroite.

```python
import plugins
import channels
import midi

def resolve_target():
    """Détermine le plugin cible.
    Retourne (index, plugin_name) ou None.
    Cible V1 : générateur du channel rack sélectionné, slotIndex = -1.
    """

def read(index, param):
    """Valeur du paramètre, float 0.0-1.0."""

def write(index, param, value):
    """Écrit une valeur 0.0-1.0. Clamp obligatoire avant écriture.
    pickupMode = 0 (pas de pickup).
    """

def label(index, param):
    """Nom du paramètre selon le plugin (plugins.getParamName)."""

def text_value(index, param):
    """Valeur formatée par le plugin ('42 %', '2.4 kHz').
    Via plugins.getName(index, -1, midi.FPN_ParamValue, param).
    Fallback sur un pourcentage calculé si indisponible ou vide.
    """

def named_params(target):
    """[(paramIndex, nom), ...] — params nommés du plugin, banc complet scanné.
    Utilisé par le scanner (§6.5) et le preset générique.
    """
```

Appels FL sous-jacents :

| Fonction | Signature FL |
|---|---|
| validité | `plugins.isValid(index, slotIndex=-1)` |
| nom plugin | `plugins.getPluginName(index, slotIndex=-1, userName=False)` |
| nb params | `plugins.getParamCount(index, slotIndex=-1)` |
| nom param | `plugins.getParamName(paramIndex, index, slotIndex=-1)` |
| lecture | `plugins.getParamValue(paramIndex, index, slotIndex=-1)` → float 0.0-1.0 |
| écriture | `plugins.setParamValue(value, paramIndex, index, slotIndex=-1, pickupMode=0)` |
| valeur texte | `plugins.getName(index, -1, midi.FPN_ParamValue, paramIndex)` |

**Résultats phase 0 (2026-09-08, FL 40, Python 3.12.1, cible Repro-1 — VST)** :

- **Index** : `channels.channelNumber()` avec `useGlobalIndex=False` (index groupé,
  défaut de la doc) identifie le plugin. Les deux interprétations (groupé/global)
  retournaient le même index dans le projet de test ; on retient l'index groupé par
  défaut et on **ne passe jamais `useGlobalIndex` explicitement**.
- **Lecture** : `getParamValue` renvoie des valeurs réelles, non nulles et variées
  (0.5, 0.7333, 0.8). Le bug historique « 0.0 partout » est **absent** de cette version.
- **Écriture** : `setParamValue` aller-retour exact — écrit 0.25, relu 0.25 (écart 0.0),
  restauration exacte. La valeur relue est un float IEEE (ex. 0.7333333492279053) :
  comparer avec tolérance (piège n°12), arrondir pour l'affichage.
- **Valeur formatée** : `getName(FPN_ParamValue)` validé, et **mieux que prévu** — il
  renvoie les libellés d'énumération ("Velocity", "Aftertouch") et des nombres formatés
  ("100.00 "). Deux obligations : `strip()` (espace final systématique) et fallback si
  chaîne vide.
- **`getParamValueString` : écartée.** Signature incompatible dans FL 40 (4 arguments
  max). Ne pas l'utiliser — `getName(FPN_ParamValue)` couvre le besoin.
- **Coût** : scan de 4240 `getParamName` en 1,5 ms (~0,4 µs/appel). Les lectures sont
  quasi gratuites → lecture directe dans `OnIdle`, pas de miroir (voir §6.3).

Règle de résolution de la cible, à implémenter explicitement :

1. `channels.channelNumber()` → si `-1`, retourner `None`
2. `plugins.isValid(index)` → si faux (sampler, audio clip, layer), retourner `None`
3. Sinon retourner `(index, plugins.getPluginName(index))`

Toutes les fonctions doivent être tolérantes aux exceptions : l'API `plugins` peut lever
sur des plugins exotiques. En cas d'échec, `read` retourne `0.0`, `label` et `text_value`
retournent une chaîne vide, `write` ne fait rien. **Ne pas laisser une exception remonter
jusqu'à `OnIdle`** : cela tuerait le rafraîchissement.

**Comportement sans preset (implémenté en v1.0, complète la règle ci-dessus)** : un
plugin valide sans preset reçoit un **preset générique** construit par
`generic_preset.py` — les 24 premiers params nommés en continu (6 pages × 4 rangées),
les **16 suivants** en discret linéaire (4 pages de grille × 4 colonnes, valeurs
0, 1/3, 2/3, 1). Les pads ne sont donc éteints que s'il n'y a **pas de cible plugin**.
Intérêt : la V1 est utilisable et testable immédiatement sur n'importe quel VST, sans
écrire de preset ; l'ordre du banc
de params sert de point de départ avant d'écrire le preset réel. Les presets malformés
(entrée sans `'param'`) sont traités en entrées inertes, pas en erreur.

### 6.2 `vst_page.py`

Rendu et interaction d'une page. **Sans aucun état persistant** — tout l'état vit dans
`vst_mode` (page de rangées courante, page de grille globale).

Responsabilités :
- calculer la couleur des 64 pads à partir des valeurs lues
- traduire un clic de pad en écriture
- traduire un delta d'encodeur en écriture

Algorithme du bargraph (colonnes 0-11) :

```python
PADS = 12
LEVELS = 8                      # niveaux de luminosité intermédiaires

def bargraph_colors(value, base_color):
    """value : float 0.0-1.0. Retourne une liste de 12 couleurs 0xRRGGBB."""
    pos = value * (PADS - 1)    # position en pads, 0.0 .. 11.0
    full = int(pos)             # nb de pads pleins
    frac = pos - full           # fraction du pad courant
    out = []
    for c in range(PADS):
        if c < full:
            out.append(base_color)                      # plein
        elif c == full:
            lvl = max(1, int(frac * LEVELS)) / LEVELS
            out.append(scale(base_color, lvl))          # partiel
        else:
            out.append(DIM)                             # éteint / très faible
    return out
```

`scale()` réduit la luminosité en conservant la teinte. Réutiliser `ScaleColor` de la v1.5
(conversion HSV) plutôt que de multiplier naïvement les composantes RGB.

Résolution perçue : 12 pads × 8 niveaux ≈ 96 crans, pour une valeur MIDI de 128. C'est
volontaire : on conserve la lecture instantanée de la position tout en gardant assez de
finesse pour que le bargraph bouge dès qu'on tourne l'encodeur d'un cran.

`DIM` doit être une valeur très faible non nulle plutôt que 0x000000, pour que
l'utilisateur voie l'étendue de la rangée même à valeur nulle. À régler à l'œil.

Bloc discret (colonnes 12-15) — grille globale, 4 pages × 4 colonnes :

```python
# colonne = un paramètre discret, jusqu'à 4 valeurs
# pad row r de la colonne = valeur r du paramètre
# actif  -> couleur pleine de la colonne (GridColors[gc], ou 'color' si surcharge)
# inactif-> même couleur très atténuée
# r >= count -> éteint et inerte
# la page de grille est sélectionnée par les 4 boutons de droite, indépendamment
# de la page de rangées ; les couleurs des colonnes sont fixes sur les 4 pages
```

Comparaison de la valeur active en flottant : utiliser une tolérance
(`abs(read() - v) < 0.01`), jamais `==`.

### 6.3 `vst_mode.py`

État global et dispatch.

État à maintenir :

```python
current_page      # 0..len(PAGES)-1  — page de RANGÉES (boutons de gauche)
grid_page         # 0..len(GRID)-1   — page de GRILLE globale (boutons de droite)
selected_row      # 0..3, pour les boutons MUTE
select_held       # bool, appui SELECT
target            # (index, plugin_name) ou None
pages             # 6 pages de rangées (preset ou générique)
grid              # 4 pages de grille, 4 colonnes chacune (preset ou générique)
preset_name       # 'preset' | '(generic)' | ''
last_param        # (param_index, label) du dernier param touché — pour l'OLED (§4.6)
_notify_text      # notification transitoire (changement de page, scan...)
_notify_until     # timestamp d'expiration de la notification
preset_change     # PRESET_CHANGE du preset : {'param', 'step'} ou None (PATTERN ▲▼)
learn             # mode Learn actif : dict de learn.py (kind/row/entry/snapshot) ou None
_last_full_refresh # timestamp du dernier renvoi complet des pads (§4.7)
```

**Orthogonalité (décision 2026-09-09).** `current_page` et `grid_page` sont deux
variables indépendantes. Les boutons de gauche (`OnPageButton`) ne modifient que
`current_page`. Les boutons de droite (`OnSubPageButton`) ne modifient que `grid_page`.
Aucune fonction ne lit l'une pour écrire l'autre.

Dispatch dans `OnMidiMsg` :

| Événement | Action |
|---|---|
| Note On/Off pad, col 0-11 | `OnPadPress(row, col)` → bargraph click sur Note On uniquement |
| Note On/Off pad, col 12-15 | `OnPadPress(row, col)` → `_grid_click(row, col-12)` sur Note On uniquement |
| CC 0x10-0x13 | `OnEncoder(row, delta)` |
| Note On 0x10-0x13 | `OnKnobTouch(row)` — afficher info du paramètre de la rangée |
| Note Off 0x10-0x13 | rien |
| Note On/Off 0x2C-0x31 | `OnPageButton(0..5)` sur Note On — change `current_page` uniquement |
| Note On/Off 0x32-0x35 | `OnSubPageButton(0..3)` sur Note On — change `grid_page` uniquement |
| Note On/Off 0x24-0x27 | `OnMuteButton(0..3)` sur Note On — arme/annule le LEARN rangée (§4.3) |
| Note On/Off 0x22-0x23 | `OnGridLearnStep(suivant?)` sur Note On — LEARN discret, défilement (§4.3b) |
| Note On/Off 0x19 | `OnSelectPush(is_on)` — `select_held = (Note On)` |
| Note On 0x21 | `OnScanRequest()` — génération preset + dump (§6.5) |
| Note On 0x1F / 0x20 | `OnPresetChange(up/down)` — preset suivant/précédent du VST (§4.4b) |
| CC 0x76, 0x22, 0x23 | no-op, réservé |

Toujours poser `event.handled = True` sur les événements consommés, pour que FL ne les
interprète pas.

Rafraîchissement dans `OnIdle` :

1. Résoudre la cible et le preset. Si la cible a changé depuis le dernier tour,
   `ClearBtnMap()` puis afficher le nom du plugin sur l'OLED. Le preset fournit
   `PAGES` (rangées) et `GRID` (grille globale) ; sans `GRID`, `EmptyGrid` (4 pages
   de colonnes `None`) est utilisé.
2. Si pas de cible : tous les pads éteints, message OLED, sortir. (Un plugin valide
   sans preset reçoit le **preset générique** — voir §6.1 — donc l'absence de preset
   n'éteint plus les pads.)
3. Sinon, recalculer les 64 couleurs : bargraph des 4 rangées (page de rangées
   courante) sur les colonnes 0-11, grille (page de grille globale courante) sur les
   colonnes 12-15. Les deux calculs sont indépendants. Envoyer via `AddPadDataCol`
   (qui filtre par `BtnMap`).
4. Mettre à jour les LED des boutons : page de rangées courante (6 boutons de gauche),
   page de grille courante (4 boutons de droite — reste allumée quel que soit
   `current_page`), rangée sélectionnée (MUTE).

**Fréquence.** La v1.5 tourne à `Idle_Interval = 100`. Résolu en phase 0 : la lecture est
quasi gratuite (~0,4 µs par appel — 4240 appels mesurés en 1,5 ms). On relit donc les
valeurs **directement à chaque tick**, sans miroir interne.
La lecture directe garantit la synchronisation avec la souris, ce qui a plus de valeur
que l'économie d'appels.

### 6.4 Format des presets

Les deux blocs sont **séparés** dans le preset : `PAGES` (rangées continus, boutons de
gauche) et `GRID` (grille discrète globale, boutons de droite). La grille n'est plus
imbriquée dans chaque page de rangées.

```python
#   Akai Fire VST Controler - preset
#   <Nom du VST>

PLUGIN_MATCH = 'Serum'        # comparé à plugins.getPluginName(), insensible à la casse

PAGES = [                              # 6 pages de RANGÉES max (boutons de gauche)
    {
        'name': 'AMP',                 # affiché sur l'OLED au changement de page
        'color': 0x00FF88,             # couleur par défaut des rangées de la page
        'rows': [                      # exactement 4 entrées, ou moins (le reste = None)
            {'param': 12, 'label': 'ATTACK'},
            {'param': 13, 'label': 'DECAY'},
            {'param': 14, 'label': 'SUSTAIN'},
            {'param': 15, 'label': 'RELEASE'},
        ],
    },
    # ... jusqu'à 6 pages. Pas de clé 'grid' ici.
]

GRID = [                               # 4 pages de GRILLE globales (boutons de droite)
    [                                  # page DUAL : 8 entrées (8 params x 2 pads)
        {                              # entrée 0 — pad HAUT (rangée 0) = values[0],
                                       # pad BAS (rangée 1) = values[1]
            'param': 3,
            'label': 'OSC1 SAW',
            'type':  'linear',
            'count': 2,                # 2 pads en dual
            'values': [1.0, 0.0],      # pad HAUT = on, pad BAS = off (ou [0.75, 0.25] si le VST l'exige)
        },
        {                              # entrée 1 — colonne suivante, mêmes pads
            'param': 4, 'label': 'OSC1 SQR', 'count': 2, 'values': [1.0, 0.0],
        },
        # ... 8 entrées max. Les entrées 0-3 = moitié HAUTE (rangées 0-1,
        # couleur GridColors[page]), entrées 4-7 = moitié BASSE (rangées 2-3,
        # couleur GridColors2[page]). None = pads éteints et inertes.
        None,
        None,
        None,
        None,
        None,
    ],
    [                                  # page QUAD : 4 colonnes, 1 param/colonne,
        {                              # jusqu'à 4 valeurs — le pad r écrit values[r]
            'param': 30,
            'label': 'FILT TYPE',
            'type':  'linear',         # 'linear' | 'increment'
            'count': 4,                # nb de pads utilisés, 1..4
            'values': [0.0, 0.3333, 0.6667, 1.0],
            'names':  ['LP12', 'LP24', 'HP', 'BP'],
            # 'color': 0xFF6600,        # optionnel : surcharge GridColors[page]
        },
        None,
        None,
        None,
    ],
    None,                              # page de grille 2 vide (None entier)
    None,                              # page de grille 3 vide
]

# PATTERN ▲/▼ : preset suivant/précédent du VST (facultatif)
PRESET_CHANGE = {'param': 47, 'step': 0.03125}   # sélecteur de preset + pas
```

Règles :

- `PAGES` et `GRID` sont deux listes indépendantes. `len(PAGES)` pilote les boutons
  de gauche, `len(GRID)` pilote les boutons de droite. Ne jamais coder en dur 6 ou 4.
- Une page de grille est **DUAL** (liste de 8 entrées : 8 params × 2 pads, format
  on/off) ou **QUAD** (liste de 4 entrées : 4 colonnes × 4 pads). Le format est
  déduit de la longueur de la liste : 8 entrées = dual, 4 = quad.
- En dual : pad HAUT = **activé (on)** = `values[0]`, pad BAS = **off** = `values[1]` (défaut `[1.0, 0.0]`). Les entrées
  0-3 occupent les rangées 0-1 (couleur `GridColors[page]`), les entrées 4-7 les
  rangées 2-3 (couleur `GridColors2[page]`).
- Toute entrée manquante ou `None` = pads éteints et inertes. Aucune erreur, aucun
  message. Une page de grille entière peut être `None` (tous pads éteints).
- Si `GRID` est absent du preset, `vst_mode` utilise `EmptyGrid` (4 pages de colonnes
  `None`) — la grille est alors inerte, sans erreur.
- `'values'` doit contenir `count` éléments. Si `'values'` est absent, les répartir
  linéairement sur `count` positions.
- `'names'` est facultatif, utilisé pour l'OLED. Fallback sur `'label'` + index.
- `'color'` est facultatif dans une entrée de grille : par défaut la couleur vient de
  `GridColors[grid_page]` (moitié haute) / `GridColors2[grid_page]` (moitié basse).
  Surcharger `'color'` ne change que cette entrée, sur cette page.
- `PRESET_CHANGE` est facultatif : sans lui, PATTERN ▲▼ est un no-op documenté.
- `type='linear'` : le pad `r` écrit `values[r]`.
- `type='increment'` : à définir précisément lors de la phase 3. Comportement pressenti —
  les pads agissent comme des pas relatifs plutôt que des valeurs absolues. **Si le
  comportement n'est pas clair au moment de coder, n'implémenter que `linear` et laisser
  `increment` lever un no-op silencieux.** Ne pas inventer.
- Le registre `presets/__init__.py` expose `resolve(plugin_name)` qui parcourt les modules
  presets et retourne le premier dont `PLUGIN_MATCH` est contenu dans `plugin_name`
  (comparaison insensible à la casse), sinon `None`.

### 6.5 `vst_scanner.py` + `preset_generator.py`

Module utilitaire, déclenché depuis le script principal par **`BROWSER`** (appui
simple) sur le plugin courant. Intégré plutôt que script séparé : avec deux unités
Fire occupées (EndGame + ce script), un script séparé imposerait de réattribuer un
contrôleur à chaque scan. Le code reste un module à part, hors de la logique du mode.

Un VST déclare 4240 paramètres, dont l'immense majorité ont un nom vide (4096 paramètres
natifs + 128 CC MIDI en 4096-4223 + 16 aftertouch en 4224-4239). Sans cet outil, écrire un
preset est impraticable.

**Deux fichiers sont produits à chaque scan :**

1. **`presets/<vst>.py`** — preset cohérent auto-généré par `preset_generator.py`.
   Les paramètres sont catégorisés par mots-clés pondérés :
   - Page 1 (AMP) : Amp Attack/Decay/Sustain/Release, volume
   - Page 2 (FILT ENV) : Filter Attack/Decay/Sustain/Release, Env 2
   - Page 3 (OSC) : Osc Level/Mix, Pulse Width, Tune, Detune
   - Page 4 (FILTER) : Cutoff, Resonance, Filter Amount, Drive
   - Page 5 (LFO/MOD) : LFO Rate/Depth, Mod Amount
   - Page 6 (MISC) : Glide, Noise, Unison, Pitch Bend
   - Grille 1 (OSC WAVE, dual) : Osc Wave/Shape — 2 pads par param (off/on)
   - Grille 2 (LFO WAVE, dual) : LFO Wave/Shape/Type
   - Grille 3 (MODE, dual) : Play Mode, Mono/Poly, Unison, Sync
   - Grille 4 (TYPE, quad) : Filter Type, etc. — 4 colonnes × 4 valeurs

   Les 3 premières grilles sont **dual** (8 params × 2 pads : la plupart des
   synths ont des boutons à 2 états), la 4e reste **quad**. Le générateur
   détecte aussi un paramètre nommé `preset`/`program`/`patch` et écrit
   `PRESET_CHANGE` (PATTERN ▲▼, §4.4b) — ce paramètre est exclu du mapping.

   Le preset est rechargé à chaud après génération — l'utilisateur peut jouer
   immédiatement. Valeurs dual par défaut `[1, 0]` — pad HAUT = on, pad BAS = off ; quad `[0, 1/3, 2/3, 1]`.

   **Nomenclature testée Repro-1 (u-he)** — le VST de référence du projet :
   `Env1 Attack..Release` → FILT ENV, `Env2 Attack..Release` → AMP (convention
   Pro-One : ENV 1 = filtre, ENV 2 = ampli — inversible à la main), `Env1 Amount`
   → FILTER, `Osc1 PW/Level/Semi` → OSC, `Key Track` → FILTER, `Spread` → MISC,
   `Osc1 Wave`/`LFO1 Wave`/`Filter Mode`/`Glide Mode`/`Unison` → grilles.

2. **`scan_<vst>.txt`** — dump brut de tous les params nommés (référence pour
   ajuster le preset à la main) :

```
# Serum  —  4240 params déclarés, 312 nommés
{'param':   12, 'label': 'A ATTACK'},        # valeur actuelle : 0.043  "4.3 %"
{'param':   13, 'label': 'A DECAY'},         # valeur actuelle : 0.500  "500 ms"
...
```

Pas de GUI en V1. Le preset auto-généré + le dump texte suffisent. Le preset est
éditable à la main pour affiner les catégories.

**Contrainte d'implémentation** (piège n°14) : le chemin du fichier de dump doit être
résolu dans un module importé (`spike_helpers.py` du spike en est le modèle), jamais
dans le script d'entrée.

**Constat phase 0** : les noms de paramètres peuvent être dupliqués — Repro-1 expose
deux `Source` et deux `Depth #1`. L'index est donc la seule clé fiable : le dump doit
toujours l'inclure (le format ci-dessus le fait déjà), et un preset ne doit jamais
référencer un paramètre par son nom seul.

### 6.6 `learn.py` + `beat_leds.py`

**`learn.py`** — logique pure du mode Learn (§4.3/§4.3b), sans rendu ni disque :

- `start(kind, target, row/entry)` → dict d'état : snapshot des valeurs de
  tous les params **nommés** (un param sans nom n'est pas mappable), phase de
  clignotement 2 Hz, timestamp d'armement.
- `poll(learn, target)` → `(param, label)` si exactement UN paramètre nommé a
  changé (tolérance `MatchTol`) ; `'timeout'` après `LearnTimeout` ; `None`
  sinon. Plusieurs changés = dérive/automation : re-snapshot, on attend.

`vst_mode` orchestre : armement (`OnMuteButton`/`OnGridLearnStep`), polling
dans `OnIdle`, clignotement (`_apply_learn_blink`), application du mapping
(`_learn_apply`) puis **`_save_and_reload`** — `preset_generator.save_preset`
réécrit le fichier depuis la mémoire (preset réel : son fichier ; générique :
création de `presets/<safe>.py`), puis rechargement du registre.

**Important** : `presets/__init__.reload()` **purge `sys.modules`** pour chaque
fichier preset avant réimport — sans elle, réimporter un preset déjà chargé
retournerait le module en cache et les changements (re-scan, learn) seraient
invisibles jusqu'au redémarrage de FL.

**`beat_leds.py`** — LEDs TrackSel 1-4 = beat/mesure (§4.8), copie de la
logique EndGame. **Seul module qui importe `transport`** (isolation, même
principe que `param_bridge`/`plugins`). Le callback FL
`OnUpdateBeatIndicator(value)` est câblé dans `device_FireVST.py`.

---

## 7. Phases de développement

Chaque phase a un critère de sortie vérifiable. Ne pas passer à la suivante avant.

**État au 2026-09-08 (refactor grille orthogonale 2026-09-09) : les phases 1 à 6 sont
implémentées (v1.0) et smoke-testées hors FL sur stubs des modules FL — 57 vérifications
couvrant le protocole pads, le bargraph, les encodeurs, les pages de rangées, les pages
de grille globales, l'orthogonalité des deux blocs, les couleurs par colonne, les
presets générique/réel/malformé/sans-GRID, le scanner, les caches anti-spam et les
pertes de cible. Les critères de sortie ci-dessous restent à vérifier sur le matériel
réel : c'est la première session de test qui décide.**

### Phase 0 — Spike de validation de l'API `plugins`

**C'est la seule phase qui peut invalider le projet. Rien d'autre ne commence avant.**

Script jetable, une trentaine de lignes, sans structure ni module. Sur le channel
sélectionné :

1. `plugins.isValid()` et `plugins.getPluginName()` → identifie-t-on le VST ?
2. `plugins.getParamCount()` → attendu 4240 pour un VST
3. Boucle sur les 300 premiers `getParamName()`, imprimer les non vides
4. `getParamValue()` sur 4 paramètres → obtient-on autre chose que `0.0` ?
5. `setParamValue(0.25, ...)` puis relecture → le VST bouge-t-il, et la relecture concorde-t-elle ?
6. `getName(index, -1, midi.FPN_ParamValue, param)` → obtient-on `"25 %"` ?

À exécuter sur **deux plugins** : un VST tiers et un plugin FL natif (3xOsc ou Sytrus).
Le comportement diverge entre les deux.

**Risque connu.** Un bug historique de FL Studio faisait que `getParamValue` /
`setParamValue` ne fonctionnaient que sur Insert 1 / Slot 1 pour les effets du mixer, et
renvoyaient `0.0` ailleurs. Officiellement corrigé, mais dépendant de la version de FL.
C'est pourquoi la V1 cible uniquement le channel rack (`slotIndex = -1`), cas le plus
fiable.

**Critère de sortie :** les points 4 et 5 fonctionnent.

**Si échec :** arrêt et rediscussion. Le plan B consisterait à sortir des CC MIDI et à les
lier manuellement dans FL (`Ctrl` + clic droit sur un paramètre), avec relecture via
`device.getLinkedValue()`. Cela fonctionne partout mais supprime les noms de paramètres,
impose un mapping manuel par VST, et change complètement l'ergonomie. Ne pas s'y engager
sans en discuter.

Le point 6 est un bonus : s'il échoue, l'OLED affichera un pourcentage calculé au lieu
d'une valeur formatée. Non bloquant.

**Résultat : VALIDÉE le 2026-09-08** (FL 40, Python 3.12.1, cible Repro-1 — VST tiers) :

- Identité, noms (440 params nommés sur 4240 déclarés), lectures non nulles et variées.
- Écriture aller-retour **exacte** : écrit 0.25 → relu 0.25 (écart 0.0), restauration
  exacte.
- Valeurs formatées via `getName(FPN_ParamValue)` : validées, avec libellés
  d'énumération en prime ("Velocity", "Aftertouch").
- Le bug historique « 0.0 partout » est absent de cette version.
- Chiffres détaillés en §6.1.

Reste optionnel et non bloquant : la même vérification sur un plugin FL natif
(Sytrus, 3xOsc) — les presets V1 visant des VST. Le spike reste installé pour cela :
sélectionner le channel, appuyer le pad 1.

### Phase 1 — Une rangée, un encodeur, 12 pads

Créer `device_FireVST.py`, `constants.py`, `display.py`, `param_bridge.py`.

Rangée 0 seulement. Bargraph sur les colonnes 0-11, clic = ancrage, encodeur 1 = ±1 cran.
Paramètre cible codé en dur, pas encore de preset.

**Critère de sortie :** les LED suivent l'encodeur sans latence ni flickering, **et**
suivent également la souris quand on bouge le paramètre dans la fenêtre du VST.

C'est ici qu'on mesure le coût CPU de la lecture continue. Noter le résultat.

### Phase 2 — Les 4 rangées et l'OLED

`vst_page.py`. 4 rangées, 4 encodeurs, une couleur par rangée. Toucher un encodeur affiche
nom et valeur formatée sans modifier.

**Critère de sortie :** un ADSR réglable et lisible d'un coup d'œil, conforme au schéma du §1.

### Phase 3 — Le bloc discret

Section D, 4 colonnes × 4 pads, pad allumé = valeur active, respect de `count`.
Types de colonne lus depuis le preset.

**Critère de sortie :** changer une forme d'onde en un clic, avec retour visuel de la
sélection.

### Phase 4 — Les pages

6 pages de rangées sur les 6 boutons de gauche, 4 pages de grille globales sur les 4
boutons de droite, **indépendantes** : changer une page de rangées ne change ni la page
de grille, ni les mappings de la grille, ni les couleurs des colonnes. LED de page de
rangées et de page de grille. Boutons MUTE. Suivi de `select_held`. No-op explicites
sur les contrôles réservés.

**Critère de sortie :** 6 pages de rangées en accès direct, 4 pages de grille en accès
direct, aucun état invisible, et l'état complet du contrôleur lisible sans regarder
l'OLED. Vérifier en particulier qu'appuyer sur un bouton de gauche ne change rien à la
grille, et qu'appuyer sur un bouton de droite ne change rien aux rangées.

### Phase 5 — Presets et résolution du VST

Format `dict`, registre par `PLUGIN_MATCH`, règle de fallback explicite à trois niveaux
avec message OLED distinct pour chaque cas. Un preset complet pour un synthé réel.

**Critère de sortie :** brancher un VST connu charge son mapping ; brancher un VST
inconnu affiche son nom et propose le **preset générique** (pads actifs — voir §6.1),
sans erreur. Seule l'absence de cible plugin laisse les pads éteints.

### Phase 6 — Scanner

`vst_scanner.py`.

**Critère de sortie :** produire un preset pour un nouveau VST en 15 minutes.

---

## 8. Pièges connus

Repris de l'expérience de la v1.5, plus les spécificités de ce projet.

1. **Les encodeurs sont relatifs, pas absolus.** Décoder le delta avec le test `>= 0x40`.
   Ne jamais traiter `data2` comme une position.
2. **`BtnMap` est obligatoire.** Sans le cache, flickering et saturation du port MIDI.
3. **`event.data1 - PadFirst`** pour obtenir l'index de pad. Ne pas oublier la soustraction.
4. **`channels.channelNumber()` peut retourner `-1`.** Toujours vérifier.
5. **`channels.selectedChannel()` retourne un index de groupe**, `channelNumber()` l'index
   absolu. Ne pas les confondre.
6. **Aucune exception ne doit remonter jusqu'à `OnIdle`.** Elle tuerait le rafraîchissement
   sans message clair. Envelopper les accès à `plugins`.
7. **Les constantes `FPN_*`, `FPT_*` viennent du module `midi`**, pas de `plugins` ni de
   `transport`. Faire `from midi import *` ou `import midi`. Ne jamais recourir à
   `getattr(module, 'CONST', fallback)` : les valeurs de repli seraient fausses.
8. **`# receiveFrom` doit être absent** de l'en-tête. Voir §5.
9. **Les couleurs sont 24 bits dans le code, 7 bits par composante sur le fil.** La
   division par 2 est faite dans `AddPadDataCol`.
10. **Les LED des boutons MUTE sont vertes uniquement.** Ne pas espérer une seconde couleur.
11. **`IDTrackSel1-4` sont des sorties**, pas des boutons. Utilisables comme indicateurs.
12. **Comparer des valeurs de paramètre avec une tolérance**, jamais avec `==`.
13. **Ne pas coder en dur le nombre de pages.** Utiliser `len(PAGES)`.
14. **`__file__` n'est pas défini dans le script d'entrée** exécuté par FL Studio
    (`NameError`, constaté en phase 0 sur l'installation cible). Il est en revanche
    défini normalement dans les **modules importés**. Toute écriture de fichier à côté
    du script (rapport, dump du scanner, état persistant) doit donc vivre dans un module
    importé — pattern `fire_modules/key_sender.py` de la v1.5. Le script d'entrée
    (`device_FireVST.py`) ne doit jamais référencer `__file__`.

---

## 9. Points ouverts, à trancher en beta

À implémenter au plus simple, et à réévaluer après quelques heures d'usage réel.
Marquer chacun d'un commentaire dans le code.

- **Boutons MUTE 1-4.** Que signifie « sélectionner » une rangée, une fois que le toucher
  d'encodeur affiche déjà l'information ? Redondance probable.
- **PATTERN ▲ ▼.** Non attribués. Piste envisagée : changer de channel / de VST, qui sera
  le geste le plus fréquent après le changement de page.
- **24 paramètres continus suffisent-ils ?** C'est le vrai plafond du design. Si non,
  l'extension est `SELECT` + bouton de page pour les pages 7-12 — pas de bouton
  supplémentaire, pas de menu.
- **La configuration depuis le boîtier est-elle réellement nécessaire ?** Les contrôles
  sont réservés pour cela mais inactifs.
- **`type='increment'`** : comportement à définir précisément.

---

## 10. Références API FL Studio

- MIDI Scripting : https://www.image-line.com/fl-studio-learning/fl-studio-online-manual/html/midi_scripting.htm
- Référence Python (stubs, plus complète que le manuel officiel) : https://il-group.github.io/FL-Studio-API-Stubs/
- Module `plugins` : https://il-group.github.io/FL-Studio-API-Stubs/midi_controller_scripting/plugins/
- Flags `plugins.getName` : https://il-group.github.io/FL-Studio-API-Stubs/midi_controller_scripting/midi/plugin%20get%20name%20flags/
- Module `channels` : https://www.image-line.com/fl-studio-learning/fl-studio-online-manual/html/midi_scripting_channels.htm
- Module `device` : https://www.image-line.com/fl-studio-learning/fl-studio-online-manual/html/midi_scripting_device.htm
- Le module `screen` (OLED du Fire) n'a pas de documentation publique. Se référer à
  `../Akai Fire EndGAme V1.5/fire_modules/display.py`.

### Fichiers de référence dans le projet voisin

- `../Akai Fire EndGAme V1.5/analysis.md` — protocole matériel du Fire, relevé exhaustif
  des notes et CC. **Source la plus fiable.**
- `../Akai Fire EndGAme V1.5/Akai Fire EndGAme - Contexte IA.md` — architecture, pièges,
  API les plus utilisées.
- `../Akai Fire EndGAme V1.5/device_Fire.py` — implémentation de référence pour la
  communication avec le device.
