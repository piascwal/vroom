# Vroom 3D — plan de passage à un vrai jeu 3D

> Document de cadrage, révision 2. Objectif : faire de `moteur-sim.html` un jeu de conduite 3D
> réaliste, avec une architecture propre. **Pas de plateforme, pas d'éditeur livré aux joueurs,
> pas d'upload, pas de serveur.** Le contenu est fait par nous, et il est bon.
>
> La version précédente de ce plan, qui incluait l'étage plateforme façon Roblox, reste
> consultable dans l'historique git au commit `eff3905`.

---

## 1. Où on en est

`moteur-sim.html` est un mono-fichier de ~4 900 lignes, sans aucune dépendance, qui contient
en réalité quatre logiciels empilés :

| Brique | Ce que c'est | Verdict |
|---|---|---|
| **Synthèse moteur** (Web Audio) | harmoniques, `rumble`, calendrier d'allumage, pétarade, turbo, antilag, coupure de rupteur, résonance d'échappement | **La valeur du projet.** Indépendante du rendu. À extraire telle quelle, puis à étendre. |
| **Atelier moteur** | ~20 réglages, presets iconiques, banc d'essai plein écran, sauvegarde `localStorage` | **Une vraie fonction de jeu.** On la garde et on l'étend au châssis. |
| **Modèle de transmission** | régime ↔ vitesse par rapports de boîte, seuils de passage en fraction de plage | À garder comme *couche transmission*, sous une vraie physique de châssis. |
| **Rendu** | rasteriseur logiciel sur canvas 2D : projection sommet par sommet, faces peintes, tri en profondeur, pixellisation, vignettage | Remarquable, mais c'est le plafond. À remplacer. |

Le commentaire ligne 2248 explique pourquoi il n'y a pas de WebGL : *« une dépendance WebGL
coûterait l'autonomie du fichier et supposerait un GPU dont on ne sait rien sur l'écran de la
voiture »*. Cette contrainte reste vraie — elle ne disparaît pas, elle change de forme : on
la traite avec un **repli WebGL2 automatique** et des **paliers de qualité**, pas en
renonçant au GPU. Et sans serveur, le projet garde son autonomie : c'est toujours un tas de
fichiers statiques qu'on pose n'importe où.

Ce qui bloque aujourd'hui :

- **Le remplissage de pixels est le mur.** Le profil cité dans le code dit que le JS ne pèse
  que 7 % du temps ; tout le reste, c'est le CPU qui peint des triangles. Aucune optimisation
  JS ne débloquera ça, seul le GPU le peut.
- **Pas d'éclairage, pas de matières.** Les couleurs sont figées par face. Pas d'ombres, pas
  de reflets, pas de normal maps : le réalisme demandé est hors d'atteinte par construction.
- **Pas de physique de véhicule.** Il y a une transmission, pas un châssis : pas de transfert
  de charge, pas de glisse, pas de suspension. Or c'est *ça*, le réalisme d'un jeu de conduite.
- **Le monde est un ruban.** La route est une pile de tranches horizontales, la caméra est sur
  des rails. Pas de circuit fermé, pas de dénivelé réel, pas de trajectoire libre.
- **Un seul fichier de 254 ko** : pas de tests, pas de types, pas de découpage, pas de
  chargement progressif.

---

## 2. Ce qu'on vise — et ce qu'on abandonne

**Un seul produit** : un jeu de conduite 3D, réaliste, qui tourne partout, avec l'atelier
moteur en profondeur de jeu.

Ce qu'on **abandonne explicitement**, et qui n'est plus jamais mentionné dans ce plan :

- éditeur de monde livré aux joueurs, catalogue d'assets, semis au pinceau ;
- publication, galerie, remix, comptes utilisateurs, classements en ligne ;
- upload de maillages, modération, obligations DSA, porte d'âge ;
- backend, base de données, stockage objet, coûts d'infrastructure récurrents.

Ce que ça change vraiment — et ce n'est pas surtout du calendrier :

| | Avec plateforme | Sans plateforme |
|---|---|---|
| Surface technique | jeu + éditeur + backend + modération | **jeu** |
| Qui garantit les 30 FPS | des garde-fous à l'exécution, sur du contenu inconnu | **nous, au moment de fabriquer le contenu** |
| Qualité par circuit | moyenne, imposée par le budget UGC | **élevée : éclairage cuit, décor placé à la main** |
| Risque juridique | réel (UGC, mineurs, droits) | **nul** |
| Coût d'exploitation | serveur + stockage + modération | **hébergement statique** |

Le budget libéré par l'éditeur et la plateforme (~9 semaines) **n'est pas économisé : il est
réinvesti** dans la dynamique du véhicule, l'image, le son et le contenu. Le calendrier total
bouge peu ; le jeu, lui, est nettement meilleur à ce qu'il fait. C'est un arbitrage, pas une
réduction de voilure — à toi de dire si tu veux plutôt récupérer le temps.

---

## 3. La stack

### 3.1 Le moteur de rendu — **three.js**

| Option | Pour | Contre | Décision |
|---|---|---|---|
| **three.js** | Écosystème de loin le plus large ; `three/webgpu` production-ready depuis r171 avec **repli WebGL2 automatique** ; TSL (shaders écrits une fois, compilés en WGSL *et* GLSL) ; sortie statique, aucune infrastructure | Rien n'est fourni : il faut assembler | ✅ **Retenu** |
| Babylon.js | Piles incluses : physique Havok, inspecteur, Node Material Editor, pipeline glTF | Bundle plus lourd, communauté plus petite, moins de recettes de perf publiées | Sérieux second choix |
| PlayCanvas | Excellente réputation de performance | Modèle tourné vers leur éditeur cloud, moins d'intérêt ici | ❌ |
| Godot 4 (export web) | Vrai moteur de jeu, éditeur complet | Build WASM de 30–60 Mo, premier chargement lent, web mobile fragile — incompatible avec « ça s'ouvre dans un onglet » | ❌ |
| Unity WebGL | — | Lourd, mobile web médiocre, licence | ❌ |

**Le point décisif** : `import * as THREE from 'three/webgpu'` donne WebGPU quand il est là et
WebGL2 sinon, sans branche dans notre code. C'est exactement la couverture qu'il faut pour
« un GPU dont on ne sait rien ».

### 3.2 La physique — **Rapier**

`@dimforge/rapier3d-compat` (Rust → WASM, avec SIMD). Le plus rapide en navigateur
aujourd'hui, et il fournit `DynamicRayCastVehicleController` : le modèle *raycast vehicle*
— quatre rayons de suspension, pas de roues rigides — qu'utilise la quasi-totalité des jeux
de course.

**On ne touche pas à la transmission existante.** Rapier calcule le châssis, les suspensions
et le contact au sol ; la couche pneu (§5.2) calcule les forces ; le couple, les rapports et
les seuils de passage restent le code actuel, qui est bon et qui alimente déjà l'audio.

### 3.3 Langage et outillage — **TypeScript + Vite**

- **TypeScript** dès le départ : c'est un moteur temps réel avec des formats de données
  persistés, on ne le tient pas en JS nu à cette taille.
- **Vite** pour le dev et le build. Sortie statique → l'hébergement reste bête et méchant,
  dans l'esprit du `scripts/serve.js` actuel.
- **Zod** pour valider ce qui vient du `localStorage`. `sanitizeEngine()` fait déjà exactement
  ça à la main — on généralise le principe.
- **Vitest** pour les tests unitaires (transmission, pneu, splines), **Playwright** pour les
  tests de performance en intégration continue.

### 3.4 L'interface — **pas de framework**

C'est le changement le plus net par rapport à la version précédente du plan. **Sans éditeur,
React ne se justifie plus.** Ce qu'il reste d'UI, c'est : un hub, des menus, un HUD, et
l'atelier moteur — que le fichier actuel réalise déjà très bien en DOM nu.

- **TypeScript + DOM direct**, avec un mini-utilitaire de rendu de gabarits (~50 lignes) pour
  les panneaux à état. Zéro dépendance d'UI, bundle plus léger, et on reste dans le caractère
  du projet.
- **Le HUD reste du DOM**, superposé au canvas, comme aujourd'hui : c'est plus net que du
  texte dessiné dans la scène, et c'est accessible.
- Si les menus devenaient un jour une vraie application à état, **Lit** ou **Preact** sont
  l'échappatoire — 5 à 10 ko, pas 45.

### 3.5 Les assets — glTF/GLB + CC0 + `gltf-transform`

- **Format** : `.glb`, géométrie compressée en **Meshopt**, textures en **KTX2/Basis**
  (compression GPU native : moins de mémoire vidéo, pas seulement moins de bande passante —
  c'est ça qui compte sur mobile).
- **Sources CC0** : *Poly Haven* (HDRI et matières photo-scannées — l'essentiel du réalisme
  vient de là), *AmbientCG* (asphalte, béton, terre, glissières), *Quaternius / KayKit /
  Kenney* (décor, souvent en atlas partagé donc très peu d'appels de dessin).
- **Pipeline** : un script `tools/build-assets.ts` qui passe tout au `gltf-transform` CLI
  (dedup, prune, weld, `simplify` pour générer les LOD, meshopt, ktx2). Les GLB du dépôt sont
  les sources ; les GLB servis sont produits par le build.

### 3.6 Le dos — **il n'y en a pas**

Réglages, moteurs créés et meilleurs temps vivent en `localStorage` / IndexedDB, comme
aujourd'hui. Le déploiement est un dossier de fichiers statiques. Pas de compte, pas de coût,
pas de RGPD.

---

## 4. Architecture

### 4.1 Arborescence

```
vroom/
├─ index.html                 # HUB : écran de sélection (Classic / 3D)
├─ legacy/
│  └─ moteur-sim.html         # le jeu actuel, déplacé et intact
├─ src/
│  ├─ core/                   # zéro dépendance de rendu, entièrement testable
│  │  ├─ transmission.ts      # boîte, rapports, seuils — porté de moteur-sim.html
│  │  ├─ tyre.ts              # courbes de glissement, charge, adhérence (§5.2)
│  │  ├─ spline.ts            # tracé : Catmull-Rom, largeur, dévers, altitude
│  │  └─ schema/              # EngineSpec, CarSpec, TrackSpec (zod) + migrations
│  ├─ audio/
│  │  ├─ engine-synth.ts      # LA synthèse, portée telle quelle
│  │  └─ soundscape.ts        # roulement, vent, transmission, réverbération (§5.3)
│  ├─ runtime/
│  │  ├─ renderer.ts          # three/webgpu, paliers de qualité, post-traitement
│  │  ├─ physics.ts           # Rapier, pas de temps fixe 60 Hz
│  │  ├─ car.ts               # châssis + pneu + transmission + audio, en un objet
│  │  ├─ camera.ts            # cockpit, capot, poursuite — et le ressenti (§5.4)
│  │  ├─ track-build.ts       # TrackSpec → maillages, colliders, instances
│  │  └─ budget.ts            # mesure du temps d'image, palier adaptatif
│  ├─ ui/                     # hub, menus, HUD, atelier moteur (DOM nu)
│  └─ content/                # les circuits et les voitures, en données
├─ assets/                    # GLB + HDRI + textures, optimisés au build
├─ tools/
│  ├─ build-assets.ts         # pipeline gltf-transform
│  └─ track-editor/           # outil INTERNE de tracé (§7.3), non livré
└─ docs/
```

Une seule application Vite. Pas de monorepo : il n'y a qu'un consommateur.

### 4.2 Le hub

`index.html` cesse d'être une redirection et devient l'écran d'accueil, à deux entrées :
**Cockpit Classic** (le jeu actuel, inchangé, hors ligne) et **Vroom 3D**. L'ancienne URL
`/moteur-sim.html` continue de répondre, pour ne casser aucun lien existant.

### 4.3 Les formats de données

Trois schémas versionnés et validés par zod. Ce sont des **formats internes** : ils décrivent
notre contenu, ils ne sont pas une API publique. On peut donc les faire évoluer librement —
sauf `EngineSpec`, qui est persisté chez les joueurs et mérite ses migrations.

**`EngineSpec`** — existe déjà. On reprend les champs de `sanitizeEngine()` tels quels
(`cylinders`, `idleRPM`, `maxRPM`, `redline`, `harmonics`, `roughness`, `brightness`,
`crackle`, `turbo`, `growl`, `mechanical`, `rumble`, `pipeHz`, `popLength`, `limiterCutMs`,
`antilagWeight`, `firingPattern`, `gearRatios`, `shiftUpFrac`, …). **Les moteurs déjà
enregistrés par les joueurs sont importés automatiquement** au premier lancement de la 3D.

**`CarSpec`** — nouveau.
```jsonc
{
  "name": "Berline 3.0",
  "chassis": { "asset": "sedan_a", "mass": 1420, "com": [0, -0.2, 0],
               "inertia": [1900, 2100, 500], "drag": 0.31, "frontalArea": 2.2 },
  "wheels": {
    "positions": [[0.78, -0.25, 1.32], [-0.78, -0.25, 1.32], [0.78, -0.25, -1.28], [-0.78, -0.25, -1.28]],
    "radius": 0.34, "width": 0.22,
    "suspension": { "rest": 0.32, "stiffness": 32, "damping": 4.2, "travel": 0.18, "antiRoll": 0.5 },
    "tyre": { "peakSlip": 0.14, "peakGrip": 1.15, "falloff": 0.72, "loadSensitivity": 0.85 }
  },
  "engine": "v8crackle",
  "drivetrain": { "layout": "rwd", "diffLock": 0.35, "brakeBias": 0.62 },
  "aids": { "abs": true, "tc": "soft" },
  "cockpit": { "asset": "sedan_a_interior", "eye": [0.37, 1.14, 0.28], "wheelAngle": 900 },
  "livery": { "body": "#d33a3a", "trim": "#141924", "wheel": "#8d949c" }
}
```

**`TrackSpec`** — le circuit, décrit comme un tracé, pas comme une géométrie.
```jsonc
{
  "name": "Col de Vence",
  "env": { "hdri": "kloofendal_dawn", "sun": { "azimuth": 112, "elevation": 14 },
           "weather": "clear", "fog": { "color": "#cfdde8", "density": 0.004 } },
  "path": { "closed": true, "profile": "route-montagne",
            "nodes": [{ "p": [0,0,0], "width": 7.5, "bank": 0, "surface": "asphalt" }, …] },
  "terrain": { "heightmap": "vence.ktx2", "amplitude": 90, "material": "alpine" },
  "scatter": [{ "asset": "pine_a", "density": 0.4, "band": [8, 60], "seed": 42 }],
  "props":   [{ "asset": "barrier_a", "p": [...], "r": [...] }],
  "lightmap": "vence_lm.ktx2",
  "rules": { "mode": "timeattack", "checkpoints": [...], "laps": 3 }
}
```

Deux choix qui restent bons même sans plateforme :

- **Le tracé est une spline, pas une géométrie.** Les nœuds portent position, largeur, dévers
  et revêtement ; le maillage, le collider, les glissières, les marquages et le semis en sont
  **dérivés**. Un circuit se retouche en déplaçant un nœud, et se régénère automatiquement en
  meilleure qualité quand le moteur progresse — au lieu d'être un `.glb` figé à réexporter.
- **`scatter` est déclaratif** : une règle de semis, pas dix mille positions d'arbres. Le
  dépôt reste léger et le semis reproductible par sa graine.

### 4.4 La boucle

```
                    ┌── entrées (clavier / manette / tactile) ───┐
                    ▼                                            │
  pas fixe 60 Hz ─► transmission (rapport, régime) ──────────────┼──► synthèse audio
                    │            ▲                               │    (fil Web Audio)
                    │            │ vitesse roue                  │
                    └─► Rapier ──┴─► pneu (glissement → force) ──┘
                           │
  image variable ──────────┴──► interpolation d'état ──► three.js ──► écran
```

- **Physique à pas fixe** (60 Hz, accumulateur, 2 sous-pas maximum) : la tenue de route ne
  doit pas dépendre de la fréquence d'affichage. Règle numéro un d'un jeu de course.
- **Rendu à pas variable**, avec interpolation entre les deux derniers états physiques.
- **L'audio ne dépend d'aucun des deux** : Web Audio a son propre fil. On lui pousse le
  régime et la charge ; il continue de jouer proprement même si une image saute.

---

## 5. Le réalisme

C'est désormais le cœur du projet, alors il faut être précis sur ce qui produit du réalisme
et dans quel ordre. Voici mon classement par rapport qualité/coût réel.

### 5.1 L'image

Un bon éclairage bat une géométrie dense, toujours.

1. **HDRI en éclairage d'environnement + tone mapping ACES.** C'est 80 % de l'écart entre
   « scène 3D » et « photo ». Un HDRI Poly Haven, un `PMREMGenerator`, et tout ce qui est
   métallique ou vernis devient crédible d'un coup.
2. **Une seule directionnelle, avec ombres en cascade (CSM).** Pas dix lumières : une, bien
   réglée, avec des cascades dimensionnées pour le champ de vision de conduite.
3. **Matières PBR honnêtes.** Asphalte avec normal map et rugosité *variable* (les traces de
   passage sont plus lisses que le reste), carrosserie en `clearcoat`, vitres avec
   transmission. C'est ce qui distingue une route d'un ruban gris.
4. **Éclairage cuit pour le statique** (lightmaps + AO), calculé au build. Sans UGC, on peut
   se le permettre — c'est *le* privilège de faire son contenu soi-même.
5. **Post-traitement** : SMAA, bloom discret, flou cinétique par vecteurs de vitesse,
   aberration chromatique très légère à haute vitesse, vignettage (déjà présent dans le
   fichier actuel — l'auteur avait raison).
6. **Enfin seulement** la densité de géométrie.

### 5.2 La dynamique du véhicule — le gros morceau

Le raycast vehicle de Rapier donne un châssis qui tient debout. Il ne donne pas un jeu de
conduite. Ce qu'il faut ajouter par-dessus, dans `core/tyre.ts` :

- **Une courbe de glissement par pneu** (Pacejka simplifiée : montée linéaire, pic, décroissance).
  C'est elle qui produit la perte d'adhérence progressive, le sous-virage, le survirage, et le
  fait qu'on *sente* la limite avant de la franchir. Sans elle, la voiture colle puis part
  d'un coup — le défaut le plus reconnaissable d'un jeu de course amateur.
- **Le transfert de charge** longitudinal et latéral : au freinage l'avant s'écrase et gagne
  de l'adhérence, l'arrière s'allège. C'est ce qui rend le freinage-dégressif possible.
- **La sensibilité à la charge** : un pneu chargé au double n'adhère pas au double. Deux
  lignes de code, et le comportement en appui devient juste.
- **Un différentiel** avec taux de blocage réglable, et un **répartiteur de freinage**.
- **ABS et antipatinage** optionnels, désactivables — ils font partie du réalisme d'une
  voiture moderne, et leur absence fait partie de celui d'une ancienne.
- **Aérodynamique** : traînée et appui en fonction de la vitesse.

Tout ça est du code `core/`, testable sans rendu : on peut écrire des tests qui vérifient
qu'un freinage à 100 km/h s'arrête en ~40 m, qu'une courbe se prend à telle vitesse, que la
voiture ne dérive pas à l'arrêt. **C'est la partie qui décide si le jeu est bon.**

### 5.3 Le son — étendre ce qui existe déjà

La synthèse moteur est excellente et le restera. Ce qui lui manque, c'est tout ce qui n'est
pas le moteur :

- **Roulement** : bruit filtré dont le timbre dépend du revêtement (asphalte, gravier, herbe,
  bande rugueuse) et le volume de la vitesse.
- **Crissement de pneu** piloté par le taux de glissement calculé en §5.2 — donc juste par
  construction, pas déclenché par un seuil arbitraire.
- **Vent** : bruit rose filtré, ouvert avec la vitesse. C'est ce qui donne la sensation de
  vitesse quand l'image ne suffit plus.
- **Sifflement de transmission** indexé sur le régime de sortie de boîte.
- **Réverbération contextuelle** : un `ConvolverNode` dont la réponse change en tunnel, en
  forêt, en espace ouvert. Effet spectaculaire pour un coût dérisoire.
- **Spatialisation** des autres véhicules par `PannerNode`, avec effet Doppler au dépassement.

### 5.4 Le ressenti — gratuit, et sous-estimé

- **Vue cockpit véritable** : c'est l'identité du projet, il faut la porter en 3D. Habitacle
  modélisé, volant qui tourne réellement, aiguilles physiques qui oscillent, rétroviseurs
  (rendus en basse résolution, une passe supplémentaire à budget serré), ombres portées de
  l'habitacle sur le tableau de bord.
- **Champ de vision qui s'ouvre avec la vitesse**, léger retard de la caméra sur les
  accélérations, roulis en virage.
- **Secousses corrélées à la suspension**, pas un bruit aléatoire : quand la roue avant droite
  passe un nid-de-poule, la caméra bouge de ce côté-là.
- **Retour haptique de manette** (`GamepadHapticActuator`) indexé sur le glissement des pneus.
- **Une bonne courbe de réponse d'entrée** : la direction au clavier a besoin d'une rampe et
  d'un rappel dépendant de la vitesse, sinon le jeu est injouable — c'est un détail qui prend
  une demi-journée et qui change tout.

---

## 6. La performance

Le plancher de **30 FPS** reste la contrainte maîtresse. La différence, sans plateforme :
**c'est nous qui fabriquons le contenu, donc les budgets s'appliquent au moment de le
fabriquer**, pas à l'exécution sur du contenu inconnu. Plus de jauge ni de verrou de
publication : un simple rapport de build qui échoue si un circuit dépasse son palier.

### 6.1 Les paliers

Détectés automatiquement au premier lancement (temps d'image mesuré 3 secondes sur une scène
étalon), puis ajustables à la main.

| | **T0 — Fluide** | **T1 — Équilibré** | **T2 — Élevé** |
|---|---|---|---|
| Cible | mobile d'entrée de gamme, écran embarqué | mobile récent / portable | desktop avec GPU dédié |
| Résolution | 720p × 0,7 | 1080p × 0,85 | native |
| Appels de dessin | ≤ 120 | ≤ 300 | ≤ 600 |
| Triangles visibles | ≤ 250 k | ≤ 900 k | ≤ 2,5 M |
| Textures | 512 px | 1024 px | 2048 px |
| Ombres | lightmap seule + ombre au sol | 1 cascade 1024 | 3 cascades 2048 |
| Post-traitement | tone mapping seul | + SMAA, bloom | + flou cinétique, SSAO |
| Rétroviseurs | désactivés | 1 passe, 256 px | 1 passe, 512 px |
| Distance de vue | 400 m | 900 m | 1 600 m |
| **Plancher** | **30 FPS** | **30 FPS** (cible 60) | **60 FPS** |

### 6.2 Le budget d'image, à 30 FPS (33,3 ms)

| Poste | Budget |
|---|---|
| Physique + dynamique véhicule (pas fixe) | 5 ms |
| Logique de jeu + graphe de scène | 3 ms |
| Ordonnancement audio | 1 ms |
| Soumission du rendu (côté JS) | 5 ms |
| GPU | 17 ms |
| Marge | 2,3 ms |

Par rapport à la version plateforme du plan, la physique gagne 1 ms (modèle de pneu plus
riche) et le GPU 1 ms (plus d'effets), pris sur la logique de jeu — il n'y a plus d'état
d'éditeur à tenir.

### 6.3 Les techniques, dans l'ordre où elles rapportent

1. **`InstancedMesh` pour tout ce qui se répète** — arbres, glissières, poteaux, bâtiments.
   Un circuit de 10 000 objets doit tenir en une trentaine d'appels de dessin. C'est *la*
   technique qui décide de tout.
2. **LOD** générés au build par `gltf-transform simplify`, trois niveaux, avec fondu.
3. **Éclairage cuit** pour le statique. Le dynamique se limite au véhicule et à ses feux.
4. **Culling par tuiles le long de la spline** : on ne teste que les tuiles proches du
   véhicule, pas les 10 000 objets.
5. **Résolution adaptative** en dernier recours : au-delà de 30 ms sur 30 images consécutives,
   on descend l'échelle de rendu par pas de 5 % (plancher 0,6). Le joueur ne le voit presque
   pas ; une saccade, si.
6. **Colliders séparés des maillages visuels** : sol en heightfield, route en trimesh
   simplifié, props en boîtes. Jamais de collider sur la géométrie de rendu.

### 6.4 Surveillance en intégration continue

Un test Playwright charge chaque circuit sur les trois paliers, mesure le p95 du temps d'image
et **échoue si le budget du palier est dépassé, ou si une régression dépasse 10 %**. Chromium
est déjà disponible dans l'environnement — c'est peu de travail pour beaucoup de sûreté.

---

## 7. Le contenu

### 7.1 Peu, et très soigné

Sans UGC, la règle change : **trois à cinq circuits excellents valent mieux que vingt
médiocres.** Chacun a son éclairage cuit, son HDRI, son décor placé à la main aux endroits
qui comptent, et son ambiance sonore.

Les huit `SCENES` existantes sont un excellent point de départ, mais ce sont des *ambiances*,
pas des circuits. Je les traiterais comme un vivier de directions artistiques et j'en ferais
de vrais tracés :

| Circuit | Depuis | Ce qu'il montre |
|---|---|---|
| **Col de montagne** | `montagne` | dénivelé, épingles, neige, transfert de charge |
| **Circuit fermé** | `circuit` | vitesse pure, appui, freinages tardifs, tribunes |
| **Route côtière au couchant** | `sunset` | contre-jour, reflets, ambiance — la carte postale |
| **Ville de nuit** | `ville` | éclairage artificiel, reflets mouillés, ambiance sonore |
| **Piste de désert** | `desert` | gravier, glisse, poussière, adhérence dégradée |

### 7.2 Les voitures

Trois à quatre châssis, chacun avec un comportement franchement distinct (propulsion lourde,
traction agile, quatre roues motrices), et non trois variantes de la même. Les moteurs, eux,
sont déjà là — dix-huit presets et un atelier.

### 7.3 L'outil de tracé — **interne, jamais livré**

On a quand même besoin de dessiner des routes. La différence avec un éditeur produit est
énorme :

- il tourne en `npm run track-editor`, en local, pour nous ;
- pas d'UX à polir, pas d'annuler/refaire sophistiqué, pas de tutoriel, pas de sauvegarde
  cloud, pas de garde-fous contre les erreurs d'un inconnu ;
- il écrit un fichier `TrackSpec` dans `src/content/`, qui est ensuite versionné comme du code.

C'est ~1 semaine de travail au lieu de 4. C'est le seul reste de l'idée d'atelier, et il est
au bon endroit.

---

## 8. Phases

Chaque phase se termine sur quelque chose de jouable. Pas de phase « refactoring » sans
livrable visible.

### Phase 0 — Fondations *(~1 semaine)*
- Vite + TypeScript + ESLint ; `legacy/moteur-sim.html` déplacé et toujours servi.
- `index.html` devient le hub à deux entrées.
- **Extraction de la synthèse moteur** dans `src/audio/engine-synth.ts`, sans dépendance, avec
  tests de non-régression. Le fichier legacy continue de tourner sur son code d'origine.
- Portage de la transmission dans `src/core/transmission.ts`, avec tests.
- ✅ *Livrable : le hub, et le jeu actuel toujours intact derrière.*

### Phase 1 — Tranche verticale *(~2–3 semaines)* — **la phase qui décide de tout**
- three/webgpu + Rapier, un tracé en spline en dur, un terrain simple, une voiture jouable.
- Transmission branchée sur la synthèse extraite.
- HDRI + tone mapping ACES, une directionnelle avec cascades.
- Compteur de performance, paliers T0/T1/T2, résolution adaptative.
- ✅ *Livrable : on conduit, ça sonne comme avant, et on tient 30 FPS sur téléphone.*
- 🚦 **Point de décision : si la tranche verticale ne tient pas le plancher de 30 FPS avec un
  circuit vide, tout le reste du plan est à revoir.** On le découvre ici, pas au mois 3.

### Phase 2 — La conduite *(~2–3 semaines)*
- `core/tyre.ts` : courbes de glissement, transfert de charge, sensibilité à la charge,
  différentiel, répartition de freinage, aéro. **Avec sa batterie de tests.**
- ABS / antipatinage, courbes de réponse d'entrée, manette et retour haptique.
- Caméras (cockpit, capot, poursuite) et tout le ressenti du §5.4.
- Extension sonore : roulement, crissement piloté par le glissement, vent, réverbération.
- ✅ *Livrable : ça se conduit vraiment. C'est ici que le jeu devient bon ou pas.*

### Phase 3 — L'image *(~2–3 semaines)*
- Matières PBR, asphalte à rugosité variable, carrosserie `clearcoat`, vitres.
- Pipeline de lightmaps cuites au build ; LOD automatiques ; instanciation ; culling par tuiles.
- Post-traitement complet par palier ; météo et cycle jour/nuit.
- Habitacle modélisé, volant animé, aiguilles physiques, rétroviseurs.
- ✅ *Livrable : ça ressemble à un jeu commercial.*

### Phase 4 — Le contenu et le tour de jeu *(~3–4 semaines)*
- Outil de tracé interne, puis 3 à 5 circuits construits avec.
- 3 à 4 voitures avec des comportements distincts.
- Atelier moteur porté en 3D, étendu au châssis, avec son banc d'essai.
- HUD, menus, contre-la-montre, meilleurs temps locaux, fantôme personnel.
- ✅ *Livrable : un jeu fini.*

### Phase 5 — Finition *(continu)*
- Route mouillée et reflets, effets de particules (poussière, gomme, éclaboussures).
- Dégâts visuels, trafic, circuits et voitures supplémentaires.
- Shaders TSL personnalisés, SSR sur T2.

---

## 9. Risques

| Risque | Gravité | Réponse |
|---|---|---|
| **La conduite ne « sent » rien** — le risque principal maintenant | Élevée | La phase 2 est une phase à part entière, avec un modèle de pneu testé unitairement plutôt qu'un réglage à l'aveugle. Ne pas la comprimer. |
| **Le réalisme visé est incompatible avec 30 FPS sur mobile** | Élevée | Paliers contractuels, réalisme par l'éclairage et le cuit plutôt que par la densité. Vérifié dès la phase 1. |
| **Le contenu prend deux fois le temps prévu** | Élevée | Le vrai coût d'un jeu sans UGC, c'est le contenu. D'où : peu de circuits, un outil de tracé fait tôt, et un semis déclaratif. Réduire le nombre de circuits, jamais leur qualité. |
| **Régression de la synthèse audio à l'extraction** | Moyenne | Tests de non-régression sur la sortie du synthé ; le legacy reste jouable côte à côte pour comparer à l'oreille. |
| **Poids de chargement** | Moyenne | KTX2 + meshopt, chargement par circuit, budget de premier affichage sous 5 Mo. |
| **Les moteurs enregistrés des joueurs sont perdus** | Faible | Import automatique du `localStorage` legacy au premier lancement, avec les migrations zod. À faire en phase 0, pas après. |

---

## 10. Ce que je ferais en premier

1. **Extraire la synthèse moteur** dans un module testé. C'est la pièce irremplaçable du
   projet, et elle n'a aucune raison d'être couplée au rendu.
2. **Construire la tranche verticale** : une voiture, une route, three + Rapier, l'audio
   branché. Rien d'autre — pas de menu, pas de HUD, pas de deuxième circuit.
3. **Mesurer.** Sur un téléphone d'entrée de gamme, pas sur la machine de développement.
4. **Puis passer trois semaines sur le pneu.** C'est contre-intuitif — ça ne se voit pas sur
   une capture d'écran — mais c'est ce qui sépare un jeu de conduite d'une démo technique.

Si les 30 FPS tiennent sur un circuit vide, on a un jeu. S'ils ne tiennent pas, on l'aura
appris pour trois semaines de travail au lieu de trois mois — et le plan se réoriente vers une
direction artistique plus stylisée, qui reste un très bon jeu.

---

## Annexe — sources consultées

- three.js WebGPU : [What's New in Three.js (2026)](https://www.utsubo.com/blog/threejs-2026-what-changed) · [guide de migration](https://www.utsubo.com/blog/webgpu-threejs-migration-guide)
- Physique : [Rapier](https://rapier.rs/) · [comparatif moteurs physiques web](https://www.abratabia.com/game-physics/best-web-physics-engine.php)
- Frameworks 3D : [three.js vs R3F vs Babylon (2026)](https://www.pkgpulse.com/guides/threejs-vs-react-three-fiber-vs-babylonjs-3d-webgl-2026)
- Assets & compression : [KTX 2.0 / Khronos](https://www.khronos.org/news/press/khronos-ktx-2-0-textures-enable-compact-visually-rich-gltf-3d-assets) · [sources d'assets libres 2026](https://app.cinevva.com/guides/free-3d-model-sites)
- Perf three.js : [100 tips (2026)](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
