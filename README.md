# Akai Fire VST Controler

Contrôle des paramètres d'un VST depuis un Akai Fire : 6 pages de rangées (boutons de
gauche) pour 4 paramètres continus sous les encodeurs avec bargraph 12 pads, et 4 pages
de grille globales (boutons de droite) pour un bloc discret 4×4 — les deux blocs sont
**totalement indépendants**. Un preset par VST. Conçu pour être plus rapide que la
souris sur les 20-30 paramètres réellement utilisés — voir `SPEC.md` pour la
philosophie, l'architecture et les limites de la V1.

## Installation

1. Copier le dossier `Akai Fire VST Controler` **en entier** dans
   `<données FL>\Settings\Hardware\` (le chemin du dossier de données se voit dans
   Options → Fichier).
2. Options → Configuration MIDI → entrée du second Fire → Type de contrôleur →
   **Akai Fire VST Controler**.
3. L'autre unité Fire peut rester sous EndGame : ce script ne déclare aucun
   `receiveFrom`, il ne peut pas être asservi par le mode multi-device d'EndGame.

| Contrôle | Fonction |
|---|---|
| 4 encodeurs | 4 params continus de la page de rangées — ±1/127 par cran, sans accélération |
| Toucher un encodeur | Mémorise le paramètre → OLED affiche nom + valeur en temps réel (persistant) |
| Pads colonnes 1-12 | Bargraph du paramètre de la rangée — clic = pose la valeur |
| Pads colonnes 13-16 | Bloc discret (voir ci-dessous) |
| `STEP` `NOTE` `DRUM` `PERFORM` `SHIFT` `ALT` | 6 **pages de rangées** (continu, bloc gauche) |
| `PATTERN/SONG` `PLAY` `STOP` `REC` | 4 **pages de grille** (discret, bloc droit) — **globales et indépendantes** des pages de rangées |
| `PATTERN ▲` `PATTERN ▼` | Preset **suivant / précédent** du VST (si `PRESET_CHANGE` dans le preset) |
| `MUTE 1-4` | **Mode LEARN** : arme la rangée — le paramètre touché dans le VST est mappé automatiquement, preset sauvegardé. Ré-appui = annulation |
| `GRID ◀` `GRID ▶` | **LEARN discret** : fait clignoter + défiler les emplacements de la page de grille — un bouton du VST mappe l'emplacement qui clignote |
| `SELECT` (appui) | Modificateur — réservé (aucune fonction en V1) |
| `SELECT` (rotation) | **Cycle des modes dual par défaut** — `UP=OFF` `[0.0, 1.0]` (défaut) ↔ `UP=ON` `[1.0, 0.0]` (persisté dans le preset via `DUAL_DEFAULT`) |
| `BROWSER` | Génère un preset auto pour le VST courant (catégorisation intelligente) |

**Orthogonalité des pages.** Les 6 boutons de gauche ne changent que les 4 rangées
(bargraph + encodeurs). Les 4 boutons de droite ne changent que la grille (bloc D).
Changer d'un côté n'affecte pas l'autre.

**Bloc discret — deux formats de page.** La plupart des synths ont des boutons à
2 états (formes d'onde, on/off) : les **pages 1-3 sont dual** — 8 paramètres par
page, chacun sur 2 pads verticaux (pad **HAUT = activé (on)**, pad **BAS = off/normal**,
valeurs `[1.0, 0.0]` par défaut). La **page 4 reste quad** : 4 colonnes, jusqu'à 4 valeurs par
paramètre (types de filtre, etc.).

**Orientation dual par VST (SELECT rotation).** Certains VST inversent la convention
(Repro-5 : `1.0` = état actif en haut). La **rotation du SELECT** cycle entre `UP=OFF`
`[0.0, 1.0]` (défaut : pad BAS = on) et `UP=ON` `[1.0, 0.0]` (pad HAUT = on). Le mode
est **persisté dans le preset** (`DUAL_DEFAULT`) et s'applique aux **nouvelles** entrées
(learn ou génération) — les entrées existantes avec des `values` explicites ne sont pas
modifiées. Les pages quad ne sont pas affectées.

**Couleurs de la grille.** La couleur identifie la page active et le groupe :
moitié **haute** de la grille (rangées 1-2) = `GridColors[page]`, moitié **basse**
(rangées 3-4) = `GridColors2[page]` — même famille de teinte, clairement distincte.
Page 1 = orange/rouge, page 2 = jaune/lime, page 3 = vert/menthe, page 4 = bleu/violet.

**OLED.** Ligne 1 : page + nom du plugin. Ligne 2 : nom + valeur du dernier paramètre
touché, **en temps réel et persistant** (ne disparaît pas, suit l'encodeur et la souris).

**Suivi des presets du synthé.** Les valeurs sont relues en continu et tous les pads
sont renvoyés **chaque seconde** : changer de preset sur le synthé se répercute sur
les pads sans toucher aux contrôles.

**LEDs de beat.** Les 4 LEDs à côté des boutons MUTE suivent le **beat et la mesure**
pendant le play (remplissage progressif, rouge chaque 4e mesure) — même comportement
que le script EndGame sur l'autre Fire.

Pas de transport en V1 : les boutons de droite sont des pages de grille, la lecture se
pilote depuis l'autre Fire (EndGame) ou à la souris.

## Capacité

- Continu : **6 pages × 4 rangées = 24 paramètres continus.**
- Discret : **3 pages dual × 8 params = 24 params on/off + 1 page quad × 4 colonnes
  = 4 paramètres multi-valeurs**, soit 28 discrets (la grille n'est pas multipliée
  par les pages de rangées).

## Sans preset

Un VST sans preset reçoit un **mapping générique** automatique : les 24 premiers
paramètres nommés en continu (6 pages × 4 rangées), les **16 suivants** en discret
(4 pages de grille × 4 colonnes, valeurs linéaires 0, 1/3, 2/3, 1). Utilisable
immédiatement, mais pas nécessairement musical — c'est un point de départ.

Les pads ne sont éteints que si le channel sélectionné n'a pas de plugin (sampler,
layer, audio clip, aucun channel) — l'OLED l'indique.

## Créer un preset

### Méthode Learn (la plus intuitive)

1. Sélectionner la page de rangées voulue (boutons de gauche) ou la page de
   grille (boutons de droite).
2. **MUTE r** pour mapper la rangée r, ou **GRID ◀/▶** pour défiler les
   emplacements de la grille.
3. La LED MUTE / l'emplacement **clignote** → tourner un potard ou cliquer un
   bouton **dans le VST** → mappé automatiquement, nom sur l'OLED, preset
   sauvegardé. Ré-appui = annulation, timeout 30 s.
4. Sans preset existant, le learn **crée** `presets/<vst>.py` au premier
   mapping — façon rapide de construire un preset paramètre par paramètre.

### Méthode automatique (BROWSER)

1. Sélectionner le channel du VST dans FL.
2. Appuyer sur `BROWSER` sur le Fire.
3. Le script analyse les noms de paramètres, les catégorise par mots-clés
   (ADSR, filtre, OSC, LFO, etc.) et écrit `fire_modules/presets/<vst>.py`.
   S'il trouve un paramètre `preset`/`program`/`patch`, il ajoute aussi
   `PRESET_CHANGE` — `PATTERN ▲▼` change alors de preset sur le VST.
   La nomenclature **Repro-1 (u-he)** est couverte : `Env1/Env2`, `Osc1 PW`,
   `Env1 Amount`, `Key Track`, `Osc1 Wave`, `LFO1 Wave`, `Filter Mode`, etc.
4. Le preset est **rechargé à chaud** — vous pouvez jouer immédiatement.
5. Pour ajuster : éditer le fichier `.py` généré (les valeurs 2 états valent
   `[1.0, 0.0]` par défaut — pad HAUT = on, pad BAS = off ; mettre `[0.75, 0.25]`
   si le VST attend 25/75 %), puis recharger le script.

### Méthode manuelle

1. Sélectionner le channel du VST dans FL, puis `BROWSER` sur le Fire.
2. Ouvrir `fire_modules/scan_<vst>.txt` : chaque paramètre nommé y est listé avec son
   index, sa valeur courante et sa valeur formatée.
3. Copier `fire_modules/presets/exemple.py` en `fire_modules/presets/<vst>.py`,
   remplir `PLUGIN_MATCH` (contenu dans le nom du plugin), puis `PAGES` (rangées
   continus) et `GRID` (4 pages de grille globales) avec les index relevés dans le
   scan. Les deux blocs sont séparés dans le preset.
4. Recharger le script : réattribuer le type de contrôleur ou redémarrer FL.

Les noms de paramètres peuvent être dupliqués (deux `Source` sur Repro-1) : l'index
est la seule clé fiable, ne jamais référencer un paramètre par son nom seul.

## Structure

```
device_FireVST.py        script principal (classe TFireVST)
fire_modules/
  constants.py           IDs matériels, protocole, constantes
  display.py             OLED (copie v1.5)
  fire_utils.py          conversions HSV
  param_bridge.py        SEULE couche qui parle à l'API plugins
  generic_preset.py      mapping automatique sans preset
  preset_generator.py    catégorisation intelligente (BROWSER)
  vst_page.py            rendu bargraph + bloc discret
  vst_mode.py            état, dispatch, boucle idle
  vst_scanner.py         dump des paramètres (BROWSER)
  presets/               un fichier par VST + exemple.py (modèle)
Akai Fire VST Controler Spike/   spike de la phase 0 (jetable)
```

Toute modification : lire `SPEC.md` d'abord — notamment §2 (périmètre), §8 (pièges)
et §9 (points ouverts à trancher en beta).
