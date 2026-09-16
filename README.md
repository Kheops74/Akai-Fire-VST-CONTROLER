Vue d'ensemble
Le VST Controller transforme l'Akai Fire en surface de contrôle dédiée aux plugins VST de FL Studio. Il suit automatiquement le channel sélectionné et affiche ses paramètres sur les pads et l'OLED.

La matrice de pads : deux zones
Les 64 pads (16 × 4) sont divisés en deux zones indépendantes :

Zone gauche — 4 × 12 pads
Bargraph des 4 paramètres continus. Chaque rangée correspond à un encodeur. Les 12 colonnes affichent la valeur du paramètre sous forme de barre lumineuse. Cliquer un pad ancre la valeur.

Zone droite — 4 × 4 pads
Grille discrète. Deux formats possibles : dual (8 paramètres × 2 pads verticaux) ou toggle (16 pads on/off indépendants). La couleur identifie la page active.

Akai Fire
Les deux blocs de boutons
Bloc gauche — Pages de rangées
6 boutons (STEP SEQ → ALT) sélectionnent les pages des 4 rangées. Chaque bouton porte 2 sous-pages A/B = 12 pages.

Bloc droit — Pages de grille
4 boutons (PATTERN/SONG → REC) sélectionnent les pages de la grille 4×4. 2 sous-pages A/B par bouton = 8 pages.

Akai Fire VST Controller
2
Fonction de chaque touche
Akai Fire
Encodeurs & bouton MODE
Contrôle	Fonction
Encodeurs 1–4	Règlent les 4 paramètres continus de la page courante (1 pas = 1/127)
Bouton MODE	Toggle vitesse : 2 LEDs = ×1 (normal), 4 LEDs = ×2 (rapide)
Toucher encodeur	Sélectionne la rangée, affiche le paramètre sur l'OLED
Boutons de page (gauche)
Bouton	Fonction
STEP SEQ	Page 1 : ENV AMP (A) / ENV FILT (B)
NOTE	Page 2 : OSC 1 (A) / OSC 2 (B)
DRUM	Page 3 : FILT A (A) / FILT B (B)
PERFORM	Page 4 : MOD A (A) / MOD B (B)
SHIFT	Page 5 : LFO A (A) / LFO B (B)
ALT	Page 6 : MISC A (une seule page, pas de B)
2e clic sur le bouton actif = bascule A ↔ B. La sous-page est mémorisée par bouton.
Akai Fire VST Controller
3
Fonction de chaque touche (suite)
Boutons de grille (droite)
Bouton	Fonction
PATTERN/SONG	Grille 1 : OSC A (A) / OSC B (B) — 8 params × 2 pads
PLAY	Grille 2 : LFO A (A) / LFO B (B) — 8 params × 2 pads
STOP	Grille 3 : OTHER A — 16 pads on/off (une seule page)
REC	Grille 4 : MISC A (A) / MISC B (B) — 4 params × 4 pads
Boutons MUTE & GRID
Bouton	Fonction
MUTE 1–4	Learn rangée : arme le mappage, clignote, puis mappe le paramètre touché dans le VST
GRID ◀ ▶	Learn grille : arme et défile les emplacements de la page de grille courante
SELECT & BROWSER
Contrôle	Fonction
SELECT (rotation)	Inverse le sens des pads dual (UP=OFF / UP=ON)
SELECT (push)	Pendant un Learn : efface le mappage courant
BROWSER	Scanner : analyse le plugin et génère un preset automatique
PATTERN ▲ ▼	Change le preset du VST (si PRESET_CHANGE défini)
LEDs TrackSel 1–4
Les 4 bandeaux LED affichent le beat et la mesure pendant la lecture.

Akai Fire VST Controller
4
Pages A / B
Chaque bouton de page porte deux sous-pages : A et B. Un 2e clic sur le bouton déjà sélectionné bascule entre les deux. La sous-page active est mémorisée par bouton : quitter une page B et y revenir restaure B.

Code couleur
Sous-page	LED bouton
A	 Jaune / orange / vert (selon la page)
B	 Rouge
Les 12 pages de rangées
Bouton	A	B	Couleur A	Couleur B
STEP SEQ	ENV AMP	ENV FILT	Jaune	Rouge-orange
NOTE	OSC 1	OSC 2	Rouge	Magenta
DRUM	FILT A	FILT B	Violet	Bleu-violet
PERFORM	MOD A	MOD B	Bleu	Bleu clair
SHIFT	LFO A	LFO B	Cyan	Vert-menthe
ALT	MISC A	—	Rouge	—
Les 8 pages de grille
Bouton	A	B	Couleur A	Couleur B	Format
PATTERN/SONG	OSC A	OSC B	Jaune	Rouge-orange	8 × 2 pads
PLAY	LFO A	LFO B	Magenta	Violet	8 × 2 pads
STOP	OTHER A	—	16 couleurs vives (1 par pad)	16 pads on/off
REC	MISC A	MISC B	Bleu	Bleu clair	4 × 4 pads
Akai Fire VST Controller
5
Grille des Boutons (4 × 4)
Les 4 colonnes de droite forment la grille pour les boutons et commutateurs (switch). Trois formats selon les pages :

PATTERN/SONG et PLAY — 8 paramètres × 2 pads
Chaque paramètre occupe 2 pads verticaux :

Pad haut = valeur 1 (off par défaut)
Pad bas = valeur 2 (on par défaut)
Moitié haute (rangées 0–1) et basse (rangées 2–3) : couleurs distinctes
Couleurs par page
Page	Sous-page A	Sous-page B
PATTERN/SONG	Jaune	Rouge-orange
PLAY	Magenta	Violet
Inverser le sens — tournez SELECT on/off
Mode	Pad haut	Pad bas
UP=OFF (défaut)	off	on
UP=ON	on	off
Mémorisé dans le preset via DUAL_DEFAULT.

STOP — 16 pads on/off
Page STOP : 16 pads indépendants en grille 4×4. Chaque pad a sa propre couleur vive (rouge, orange, jaune, vert, menthe, bleu, violet, magenta...) pour les distinguer au premier coup d'œil.

Un appui = on (1.0)
Un autre appui = off (0.0)
Pad allumé = couleur du pad, pad éteint = inactif
REC — 4 paramètres × 4 pads
Chaque paramètre occupe 4 pads verticaux, pour les boutons qui utilisent 3 ou 4 états différents. Couleur unique par sous-page.

Sous-page	Couleur
A	Bleu
B	Bleu clair
Akai Fire VST Controller
6
Mappage des paramètres (Learn)
Mapper une rangée — MUTE 1–4
Appuyez sur MUTE (1 à 4) pour la rangée à mapper.
La LED MUTE et la rangée de pads clignotent.
Touchez le paramètre dans le VST (clic sur le knob du plugin).
Le paramètre est mappé, le preset est sauvegardé.
L'OLED affiche MAPPE : .
Ré-appuyer sur le même MUTE annule le Learn.
Mapper une entrée de grille — GRID ◀ ▶
Appuyez sur GRID ◀ ou GRID ▶.
L'emplacement actif clignote. L'OLED indique la colonne.
Touchez le paramètre du VST : il est mappé sur l'emplacement.
GRID ▶ / ◀ défile les emplacements pour enchaîner les mappings.
Effacer un mappage — SELECT push
Pendant un Learn (rangée ou grille), appuyer sur le SELECT efface le mappage courant :

Les pads s'éteignent, le preset est sauvegardé.
L'OLED affiche R1 effacee ou G col 3 effacee.
Si l'emplacement est vide, le Learn est annulé.
