## 1. Objectif et philosophie

Piloter les paramètres d'un synthé VST depuis un Akai Fire
Un contrôleur qui oblige à naviguer dans des menus, ou à regarder l'écran pour savoir où
on en est, est plus lent que la souris et donc inutile. C'est le défaut de la majorité des
contrôleurs génériques.

L'intérêt principal du concept est la **représentation physique simultanée** de plusieurs
paramètres. La souris donne de la précision, mais pas une vue d'ensemble. Exemple ADSR :

```
ATTACK   ● ● ● ● ● ● ◐ ○ ○ ○ ○ ○
DECAY    ● ● ● ● ○ ○ ○ ○ ○ ○ ○ ○
SUSTAIN  ● ● ● ● ● ● ● ● ● ◐ ○ ○
RELEASE  ● ● ◐ ○ ○ ○ ○ ○ ○ ○ ○ ○
```

La forme de l'enveloppe est lisible d'un coup d'œil, sans lire quatre nombres.

--installation--
1 copier le dossier dans le dossier HARDWARE de fl studio ou il y a les scripts
2 lancer FL studio Et démarrer le FIRE
3 selectionner le script dans les options midi  
4 sélectionner le vst que vous voulez mapper 
5 un vst qui a jamais été mapper a la led BROWSER du akai fire qui clignote, il faut impérativement cliquer sur BROWSER une seule et unique fois pour chaque nouveau VST pour créer un fichier de config, quand un VST a deja un fichier de config, la led BROWSER est allumé en fixe
6 une fois le fichier créer et la led BROWSER fixe, vous pouvez mapper le vst comme vous le voulez, ca sera sauvegarder automatiquement à chaque changement 


### Vue d'ensemble

Le Akai Fire VST Controller transforme votre Akai Fire en surface de contrôle dédiée aux **plugins VST** de FL Studio. Les 64 pads, les 4 encodeurs et l'écran OLED sont utilisés pour piloter en temps réel les paramètres de n'importe quel synthétiseur 
ou effet VST, mémoriser des presets personnalisés, et générer automatiquement un mapping quand le plugin n'en a pas.

### Sélection du plugin cible

Le contrôleur suit automatiquement le **channel sélectionné** dans FL Studio. Dès que vous cliquez sur un channel contenant un plugin VST, il devient la cible active. L'OLED affiche son nom à côté de la page courante.

---

### Les 4 encodeurs

Les 4 encodeurs crantés contrôlent les **4 paramètres continus** de la page de rangées courante. Un cran = un pas de 1/127 

#### Bouton MODE (vitesse)

Le bouton **MODE** (sous les encodeurs) bascule entre deux vitesses :

| État | LEDs MODE | Vitesse |
|------|-----------|---------|
| Normal | 2 LEDs allumées | ×1 (1 pas par cran) |
| Rapide | 4 LEDs allumées | ×2 (2 pas par cran) |

La vitesse est remise à ×1 au redémarrage. L'OLED affiche brièvement `Encodeurs x1` ou `Encodeurs x2`.

---

### Les pages de rangées (bloc gauche)

Les 6 boutons de gauche (**STEP SEQ**, **NOTE**, **DRUM**, **PERFORM**, **SHIFT**, **ALT**) sélectionnent les pages des 4 rangées de pads.

#### Pages A et B

Chaque bouton porte **deux sous-pages** : A et B. Cela donne **12 pages** au total.

- **1er clic** sur un bouton : sélectionne la page, restaure la sous-page mémorisée (A ou B).
- **2e clic** sur le bouton déjà actif : bascule A ↔ B.
- La sous-page est **mémorisée** par bouton. Quitter une page B et y revenir restaure B.

#### Code couleur des LEDs

| Sous-page | Couleur LED du bouton |
|-----------|----------------------|
| A | Jaune / orange / vert (selon la page) |
| B | Rouge |

#### Particularité du bouton ALT

Le bouton **ALT** (bouton 6) est monochrome : il n'a qu'**une seule page** (pas de sous-page B).

#### Noms des 12 pages

| Bouton | Sous-page A | Sous-page B |
|--------|-------------|-------------|
| STEP SEQ | ENV AMP | ENV FILT |
| NOTE | OSC 1 | OSC 2 |
| DRUM | FILT A | FILT B |
| PERFORM | MOD A | MOD B |
| SHIFT | LFO A | LFO B |
| ALT | MISC A | — |

---

### Les pads de rangées (colonnes 0–11)

Les 12 colonnes de gauche forment un **bargraphe** pour les 4 paramètres continus :

- Chaque rangée (0–3) correspond à un encodeur.
- Les 12 pads affichent la **valeur** du paramètre sous forme de barre lumineuse.
- **Cliquer un pad** ancre la valeur à la position correspondante.
- La couleur reflète la page active (chaque page a sa teinte).

---

### Les pages de grille (bloc droit)

Les 4 boutons de droite (**PATTERN/SONG**, **PLAY**, **STOP**, **REC**) sélectionnent les pages de la grille discrète (colonnes 12–15). Ces pages sont **indépendantes** des pages de rangées.

Comme pour les rangées, chaque bouton porte deux sous-pages A/B, sauf exceptions. Cela donne **8 pages** au total.

#### Noms des 8 pages

| Bouton | Sous-page A | Sous-page B |
|--------|-------------|-------------|
| PATTERN/SONG | OSC A | OSC B |
| PLAY | LFO A | LFO B |
| STOP | OTHER A (toggle) | — |
| REC | MISC A | MISC B |

#### Format dual (pages normales)

Les pages OSC, LFO et MISC utilisent le format **dual** :

- 8 paramètres par page, chacun sur **2 pads verticaux**.
- Pad **haut** = valeur 1 (off par défaut), pad **bas** = valeur 2 (on par défaut).
- La moitié haute (rangées 0–1) et la moitié basse (rangées 2–3) ont des couleurs distinctes.

#### Format toggle (page STOP)

La page STOP est un **toggle 16 pads** :

- 16 pads indépendants (grille 4×4).
- Un appui = **on** (1.0), un autre appui = **off** (0.0).
- Pad allumé = paramètre actif, pad éteint = paramètre inactif.

#### Inverser le sens dual (SELECT)

La rotation du **SELECT** permet d'inverser le sens des pads dual :

| Mode | Pad haut | Pad bas |
|------|----------|---------|
| UP=OFF (défaut) | off | on |
| UP=ON | on | off |

Ce réglage est mémorisé dans le preset via `DUAL_DEFAULT`.

---

### Mappage des paramètres (Learn)

Le mode Learn permet d'assigner manuellement un paramètre du VST à un contrôle du Fire.

#### Mappage d'une rangée (MUTE 1–4)

1. Appuyez sur un bouton **MUTE** (1 à 4) correspondant à la rangée à mapper.
2. La LED MUTE et la rangée de pads **clignotent**.
3. Dans FL Studio, **touchez le paramètre** du VST souhaité (clic sur le knob du plugin).
4. Le paramètre est mappé automatiquement, le preset est sauvegardé.
5. L'OLED affiche `MAPPE : <nom du paramètre>`.

Ré-appuyer sur le même MUTE annule le Learn.

#### Mappage d'une entrée de grille (GRID ◀ ▶)

1. Appuyez sur **GRID ◀** ou **GRID ▶** pour armer le Learn sur la page de grille courante.
2. L'emplacement actif **clignote**. L'OLED indique le numéro de colonne.
3. Touchez le paramètre du VST : il est mappé sur l'emplacement qui clignote.
4. GRID ▶/◀ défile les emplacements pour enchaîner plusieurs mappings rapidement.

#### Effacer un mappage (SELECT push)

Pendant un Learn (rangée ou grille), **appuyer sur le SELECT** (le bouton cranté) efface le mappage courant :

- Les pads correspondants s'éteignent.
- Le preset est sauvegardé immédiatement.
- L'OLED affiche `R1 effacee` ou `G col 3 effacee`.

Si l'emplacement est déjà vide, le Learn est simplement annulé.

---

### Bouton BROWSER (génération de preset)

Le bouton **BROWSER** lance le **scanner** : il analyse tous les paramètres du plugin et génère un fichier preset automatiquement.

#### LED BROWSER

| État de la LED | Signification |
|----------------|---------------|
| Éteinte | Pas de plugin cible |
| Clignotante | Plugin sans preset — appuyez sur BROWSER pour générer |
| Allumée fixe | Preset existant (manuel ou généré) |

#### Utilisation

1. Sélectionnez un plugin sans preset (LED BROWSER clignote).
2. Appuyez sur **BROWSER**.
3. Le scanner analyse les paramètres et crée un preset au format 12 pages / 8 grid.
4. L'OLED affiche le nombre de paramètres continus et discrets trouvés.
5. La LED BROWSER passe en fixe.

---

### Changement de preset VST (PATTERN ▲ ▼)

Les boutons **PATTERN ▲** et **PATTERN ▼** changent le preset du plugin VST directement depuis le Fire, si le preset contient une section `PRESET_CHANGE` (paramètre sélecteur + pas). Sans cette section, les boutons sont sans effet.

---

### LEDs TrackSel 1–4 (beat/mesure)

Les 4 bandeaux LED en haut du Fire affichent le **beat et la mesure** pendant la lecture :

- Clignotement synchronisé avec le tempo de FL Studio.
- Indicateur de temps fort et position dans la mesure.

---

### OLED

L'écran OLED affiche en permanence :

- **Ligne 1** : nom de la page courante + nom du plugin (ex. `ENV AMP | Repro-5`).
- **Ligne 2** : nom et valeur du dernier paramètre touché (persistant, mis à jour en temps réel par les encodeurs).
- **Notifications transitoires** : changement de page, Learn, mappage, vitesse encodeurs (affichées 2–3 secondes puis disparaissent).

---

### Structure des fichiers preset

Les presets sont stockés dans `fire_modules/presets/`. Chaque preset est un module Python contenant :

- `PAGES` : 12 pages de rangées (6 boutons × 2 sous-pages), chacune avec 4 entrées de paramètres.
- `GRID` : 8 pages de grille (4 boutons × 2 sous-pages), au format dual ou toggle.
- `DUAL_DEFAULT` : sens par défaut des pads dual (optionnel).
- `PRESET_CHANGE` : paramètre sélecteur pour PATTERN ▲/▼ (optionnel).

Les presets sont générés automatiquement par le scanner (BROWSER) ou créés manuellement. Le fichier `exemple.py` sert de modèle de référence.

---

### Résumé des contrôles

| Contrôle | Fonction |
|----------|---------|
| **Encodeurs 1–4** | Paramètres continus de la page courante |
| **Bouton MODE** | Toggle vitesse ×1 / ×2 des encodeurs |
| **STEP SEQ → ALT** | Pages de rangées (A/B par double-clic) |
| **PATTERN/SONG → REC** | Pages de grille (A/B par double-clic) |
| **Pads colonnes 0–11** | Bargraph + clic pour ancrer la valeur |
| **Pads colonnes 12–15** | Grille discrète (dual / toggle) |
| **MUTE 1–4** | Learn rangée (mapper un paramètre) |
| **GRID ◀ ▶** | Learn grille (mapper + défiler) |
| **SELECT (push)** | Effacer le mappage en cours de Learn |
| **SELECT (rotate)** | Inverser le sens des pads dual |
| **BROWSER** | Scanner + générer un preset |
| **PATTERN ▲ ▼** | Changement de preset VST |
| **TrackSel 1–4** | Indicateur beat/mesure |
