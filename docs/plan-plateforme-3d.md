# Vroom 3D — plan de passage à une plateforme de jeu

> Document de cadrage. Objectif : passer du mono-fichier `moteur-sim.html` à un vrai jeu 3D
> temps réel, doublé d'une plateforme de création de mondes façon Roblox / Fortnite Creative.
> Rien n'est jeté : le fichier actuel reste jouable, et sa pièce maîtresse — la synthèse
> moteur — devient le cœur audio de la nouvelle version.

---

## 1. Où on en est

`moteur-sim.html` est un mono-fichier de ~4 900 lignes, sans aucune dépendance, qui contient
en réalité quatre logiciels empilés :

| Brique | Ce que c'est | Verdict |
|---|---|---|
| **Synthèse moteur** (Web Audio) | harmoniques, `rumble`, calendrier d'allumage, pétarade, turbo, antilag, coupure de rupteur, résonance d'échappement | **La valeur du projet.** Indépendante du rendu. À extraire telle quelle. |
| **Atelier moteur** | ~20 réglages, presets iconiques, sauvegarde `localStorage`, `sanitizeEngine()` | **Déjà une fonction de création de contenu.** C'est la graine de la plateforme. |
| **Modèle physique** | régime ↔ vitesse par rapports de boîte, seuils de passage en fraction de plage | À garder comme *couche transmission*, à poser sur une vraie physique de châssis. |
| **Rendu** | rasteriseur logiciel maison sur canvas 2D : projection sommet par sommet, faces peintes, tri en profondeur, pixellisation, vignettage | Remarquable, mais c'est le plafond. À remplacer. |

Le commentaire ligne 2248 explique pourquoi il n'y a pas de WebGL : *« une dépendance WebGL
coûterait l'autonomie du fichier et supposerait un GPU dont on ne sait rien sur l'écran de la
voiture »*. Cette contrainte reste vraie — elle ne disparaît pas, elle change de forme : on
la traite désormais avec un **repli WebGL2 automatique** et un **système de paliers de
qualité**, pas en renonçant au GPU.

Ce qui bloque aujourd'hui :

- **Le remplissage de pixels est le mur.** Le profil cité dans le code dit que le JS ne pèse
  que 7 % du temps ; tout le reste, c'est le CPU qui peint des triangles. Aucune optimisation
  JS ne débloquera ça, seul le GPU le peut.
- **Pas d'éclairage, pas de matières.** Les couleurs sont figées par face. Pas d'ombres, pas
  de reflets, pas de normal maps : le réalisme demandé est hors d'atteinte par construction.
- **Le monde est un ruban, pas un espace.** La route est une pile de tranches horizontales, la
  caméra est sur des rails. On ne peut pas sortir de la route, tourner, faire un circuit
  fermé — donc on ne peut pas *créer un monde*.
- **Un seul fichier de 254 ko** : pas de tests, pas de types, pas de découpage, pas de
  chargement progressif.

---

## 2. Ce qu'on veut construire

Trois produits qui partagent un moteur :

1. **Rouler** — un jeu de conduite 3D réaliste, ≥ 30 FPS garanti, qui utilise la synthèse
   moteur existante.
2. **Créer** — un atelier très « jeu vidéo » : dessiner une route, poser du décor, régler la
   lumière et la météo, choisir/composer un moteur et un véhicule.
3. **Partager** — une galerie de mondes publiés, jouables en un clic, remixables.

Et une contrainte transverse qui prime sur tout : **le plancher de 30 FPS n'est pas
négociable, y compris sur les mondes créés par les joueurs.** C'est cette contrainte qui
dicte l'essentiel de l'architecture ci-dessous.

---

## 3. Choix de stack

### 3.1 Le moteur de rendu — **three.js**

| Option | Pour | Contre | Décision |
|---|---|---|---|
| **three.js** | Écosystème de loin le plus large ; `three/webgpu` production-ready depuis r171 avec **repli WebGL2 automatique** ; TSL (shaders écrits une fois, compilés en WGSL *et* GLSL) ; on reste 100 % dans la plateforme web, ce qui compte quand hub + galerie + atelier sont des pages web | Rien n'est fourni : pas d'éditeur, pas d'inspecteur, il faut assembler | ✅ **Retenu** |
| Babylon.js | Piles incluses : physique Havok, inspecteur, Node Material Editor, GUI, pipeline glTF | Bundle plus lourd, communauté plus petite, moins de recettes de perf publiées | Sérieux second choix |
| PlayCanvas | Meilleure réputation de perf, éditeur cloud existant | L'éditeur est leur SaaS : on ne peut pas en faire *notre* atelier, or l'atelier est le produit | ❌ |
| Godot 4 (export web) | Vrai moteur de jeu, éditeur complet | Build WASM de 30–60 Mo, premier chargement lent, web mobile fragile, et greffer une plateforme UGC web par-dessus est pénible | ❌ |
| Unity WebGL | — | Lourd, mobile web médiocre, licence | ❌ |

**Le point décisif** : `import * as THREE from 'three/webgpu'` donne WebGPU quand il est là et
WebGL2 sinon, sans branche dans notre code. C'est exactement la couverture qu'il faut pour
« un GPU dont on ne sait rien ».

### 3.2 La physique — **Rapier**

`@dimforge/rapier3d-compat` (Rust → WASM, avec SIMD). C'est le plus rapide en navigateur
aujourd'hui, et il fournit `DynamicRayCastVehicleController` : le modèle *raycast vehicle*
(4 rayons de suspension, pas de roues rigides) — c'est-à-dire exactement ce qu'utilisent la
quasi-totalité des jeux de course arcade-simu.

- Pas de repli : Rapier tourne partout où WASM tourne.
- Jolt (`JoltPhysics.js`) est l'alternative si on veut un jour du corps mou / du tissu.
- Havok n'a de sens qu'avec Babylon.

**Pas touche à la transmission existante.** Rapier calcule le châssis, les suspensions,
l'adhérence. Le couple, les rapports, les seuils de passage et le régime restent le code
actuel, qui est bon et qui alimente déjà l'audio. Rapier fournit la vitesse aux roues,
la transmission en déduit le régime, le régime pilote la synthèse. La boucle est propre.

### 3.3 Le langage et l'outillage — **TypeScript + Vite**

- **TypeScript** non négociable dès qu'il y a des formats de données partagés (mondes,
  véhicules, moteurs) qui transitent par du stockage et du réseau.
- **Vite** pour le dev et le build. Sortie statique → l'hébergement reste bête et méchant,
  dans l'esprit du `scripts/serve.js` actuel.
- **Zod** pour valider tout ce qui vient de l'extérieur. `sanitizeEngine()` fait déjà
  exactement ça à la main — on généralise le principe à tous les formats.
- **Vitest** pour les tests unitaires (transmission, splines, migrations de schéma),
  **Playwright** pour les tests de performance en intégration continue.

### 3.4 L'interface — **React pour le 2D, three.js impératif pour le 3D**

C'est un arbitrage à assumer, alors je le pose clairement.

- **Toute l'UI en React** : hub, galerie, panneaux de l'atelier, HUD, réglages. C'est du DOM,
  c'est ce que React fait le mieux, et l'atelier est une grosse application à état.
- **La scène 3D en three.js impératif**, pas en React Three Fiber. Raison : la boucle de jeu
  a un budget d'image serré et des centaines d'objets instanciés qui bougent ; le passage
  par un réconciliateur ajoute une couche d'imprévisibilité là où on a justement besoin de
  contrôle. Par ailleurs le support WebGPU de R3F est en retard sur celui de three.
- **Le viewport de l'atelier est le moteur du jeu**, pas un aperçu séparé. Ce qu'on voit en
  édition est littéralement ce qu'on jouera. Ça supprime toute une classe de bugs et c'est
  ce qui donne la sensation « pro » recherchée.

> R3F reste un excellent outil ; ce n'est simplement pas le bon ici, parce qu'on a un seul
> canvas 3D très chargé et beaucoup d'UI 2D autour, plutôt que l'inverse.

### 3.5 Les assets — glTF/GLB + CC0 + `gltf-transform`

- **Format** : `.glb`, géométrie compressée en **Meshopt** (ou Draco), textures en **KTX2 /
  Basis** (compression GPU native : moins de mémoire vidéo, pas seulement moins de bande
  passante — c'est ça qui compte sur mobile).
- **Sources CC0** (usage commercial, sans attribution obligatoire) :
  - *Poly Haven* — HDRI et matières PBR photo-scannées. C'est de là que viendra le plus gros
    du réalisme : un bon HDRI + du tone mapping ACES font 80 % du travail.
  - *AmbientCG* — matières (asphalte, béton, terre, glissières).
  - *Kenney*, *Quaternius*, *KayKit* — kits de décor et de véhicules, déjà en glTF, souvent
    en atlas partagé donc très peu d'appels de dessin.
- **Pipeline** : un script `tools/build-assets.ts` qui passe tout au `gltf-transform` CLI
  (dedup, prune, weld, simplify pour les LOD, meshopt, ktx2) et produit un
  `assets/catalog.json` — le catalogue que l'atelier présentera aux joueurs.

### 3.6 Le dos — d'abord rien, puis Supabase

- **Phase 1 : pas de serveur.** Les mondes vivent en **IndexedDB**, s'exportent/s'importent
  en `.json`, et se partagent par **URL compressée** (le monde entier tient dans le fragment
  d'URL, en LZ + base64url, tant qu'il reste sous ~30 ko). On garde le déploiement statique
  et zéro coût, et on valide le produit avant de payer une infra.
- **Phase 2 : Supabase.** Postgres + Auth + Storage + Row Level Security. C'est le chemin le
  plus court entre « ça marche en local » et « il y a des comptes et une galerie publique ».
  Alternative si on veut rester chez Cloudflare : Pages + Workers + D1 + R2.

---

## 4. Architecture

### 4.1 Arborescence

```
vroom/
├─ index.html                 # HUB : écran de sélection (Classic / Studio)
├─ legacy/
│  └─ moteur-sim.html         # le jeu actuel, déplacé et intact
├─ src/
│  ├─ core/                   # zéro dépendance de rendu
│  │  ├─ schema/              # EngineSpec, VehicleSpec, WorldSpec (zod) + migrations
│  │  ├─ transmission.ts      # boîte, rapports, seuils — porté de moteur-sim.html
│  │  └─ spline.ts            # route : Catmull-Rom, largeur, dévers, altitude
│  ├─ audio/
│  │  └─ engine-synth.ts      # LA synthèse, portée telle quelle, sans dépendance
│  ├─ runtime/                # le moteur de jeu
│  │  ├─ renderer.ts          # three/webgpu, paliers de qualité, post-traitement
│  │  ├─ physics.ts           # Rapier, pas de temps fixe 60 Hz
│  │  ├─ vehicle.ts           # raycast vehicle + branchement transmission/audio
│  │  ├─ world-build.ts       # WorldSpec → maillages, colliders, instances
│  │  ├─ scatter.ts           # semis de décor instancié le long de la route
│  │  └─ budget.ts            # mesure et applique les budgets d'image
│  ├─ studio/                 # l'atelier (React)
│  │  ├─ tools/               # outils route, décor, matière, lumière, règles
│  │  └─ panels/              # inspecteur, catalogue, jauge de budget
│  ├─ hub/                    # accueil + galerie (React)
│  └─ platform/               # stockage local, export/import, client API
├─ assets/                    # GLB optimisés + catalog.json
├─ tools/                     # pipeline d'assets, tests de perf
└─ docs/
```

Une seule application Vite, plusieurs routes. Pas de monorepo tant qu'il n'y a pas deux
consommateurs réels d'un même paquet — ce serait de la cérémonie pure aujourd'hui.

### 4.2 Le hub

`index.html` cesse d'être une redirection et devient l'écran d'accueil :

- **Cockpit Classic** → `legacy/moteur-sim.html` (le jeu actuel, inchangé, hors ligne)
- **Vroom Studio** → le nouveau jeu + l'atelier + la galerie

La redirection actuelle est conservée sous forme de règle : `/moteur-sim.html` continue de
répondre, pour ne casser aucun lien existant.

### 4.3 Les formats de données

Trois schémas versionnés, validés par zod, migrables. C'est le contrat de la plateforme —
c'est la partie qu'il faut concevoir avec le plus de soin, parce que c'est celle qu'on ne
pourra plus casser une fois que des joueurs auront publié.

**`EngineSpec`** — existe déjà. On reprend les champs de `sanitizeEngine()` tels quels
(`cylinders`, `idleRPM`, `maxRPM`, `redline`, `harmonics`, `roughness`, `brightness`,
`crackle`, `turbo`, `growl`, `mechanical`, `rumble`, `pipeHz`, `popLength`, `limiterCutMs`,
`antilagWeight`, `firingPattern`, `gearRatios`, `shiftUpFrac`, …). Les moteurs déjà
enregistrés dans le `localStorage` des joueurs sont **importés automatiquement** au premier
lancement du Studio.

**`VehicleSpec`** — nouveau.
```jsonc
{
  "schemaVersion": 1,
  "name": "Berline 3.0",
  "chassis": { "asset": "sedan_a", "mass": 1420, "com": [0, -0.2, 0] },
  "wheels": {
    "positions": [[0.78, -0.25, 1.32], [-0.78, -0.25, 1.32], …],
    "radius": 0.34, "width": 0.22,
    "suspension": { "rest": 0.32, "stiffness": 32, "damping": 4.2, "travel": 0.18 },
    "grip": { "lateral": 1.15, "longitudinal": 1.0 }
  },
  "engine": "user:v8-atelier-3",       // référence à un EngineSpec
  "drivetrain": "rwd",
  "livery": { "body": "#d33a3a", "trim": "#141924", "wheel": "#8d949c", "decals": [] }
}
```

**`WorldSpec`** — nouveau, le cœur de la création.
```jsonc
{
  "schemaVersion": 1,
  "name": "Col de Vence",
  "env": {
    "hdri": "kloofendal_dawn",         // du catalogue Poly Haven
    "sun": { "azimuth": 112, "elevation": 14 },
    "weather": "clear" | "rain" | "snow" | "storm",
    "fog": { "color": "#cfdde8", "density": 0.004 }
  },
  "road": {
    "closed": true,
    "profile": "route-2-voies",         // gabarit : largeur, bas-côté, glissière
    "nodes": [{ "p": [0,0,0], "width": 7.5, "bank": 0, "surface": "asphalt" }, …]
  },
  "terrain": { "biome": "alpine", "seed": 42, "amplitude": 90 },
  "scatter": [{ "asset": "pine_a", "density": 0.4, "band": [8, 60], "jitter": 0.7 }],
  "props":   [{ "asset": "barrier_a", "p": [...], "r": [...], "s": 1, "tint": "#d33a3a" }],
  "rules": { "mode": "freeride" | "timeattack" | "sprint", "checkpoints": [...], "laps": 3 },
  "tier": "T1"                          // palier de qualité visé par l'auteur
}
```

Deux choses importantes dans ce format :

- **La route est une spline, pas une géométrie.** Les nœuds portent position, largeur, dévers
  et type de revêtement ; le maillage, le collider, les glissières, les marquages et le semis
  de décor en sont **dérivés** au chargement. Un monde entier pèse donc quelques kilo-octets,
  se remixe facilement, et se régénère en meilleure qualité quand le moteur progresse.
- **`scatter` est déclaratif** : on stocke une règle de semis, pas dix mille positions
  d'arbres. Même bénéfice de poids, et c'est ce qui permet à l'atelier de rester réactif.

### 4.4 La boucle

```
                    ┌── entrées (clavier / manette / tactile) ───┐
                    ▼                                            │
  pas fixe 60 Hz ─► transmission (rapport, régime)  ─────────────┼──► synthèse audio
                    │                                            │    (fil Web Audio)
                    └─► Rapier (châssis, suspensions, adhérence) ┘
                                     │
  image variable ──────────────────► interpolation d'état ──► three.js ──► écran
```

- **Physique à pas fixe** (60 Hz, accumulateur, 2 sous-pas maximum) : la tenue de route ne
  doit pas dépendre de la fréquence d'affichage. C'est la règle numéro un d'un jeu de course.
- **Rendu à pas variable**, avec interpolation entre les deux derniers états physiques.
- **L'audio ne dépend d'aucun des deux** : Web Audio a son propre fil. On lui pousse le régime
  et la charge ; il continue de jouer proprement même si une image saute.

---

## 5. Tenir le réalisme *et* les 30 FPS

Les deux demandes sont en tension. Voilà comment je les concilie.

### 5.1 D'abord : le réalisme d'un jeu de conduite ne vient pas du nombre de triangles

Par ordre de rapport qualité/coût réel :

1. **L'audio.** Déjà fait, et c'est le levier le plus fort. Un jeu qui sonne juste est perçu
   comme réaliste avant même qu'on regarde l'image.
2. **La caméra.** Champ de vision qui s'ouvre avec la vitesse, léger retard sur les
   accélérations, secousses corrélées à la suspension, roulis en virage. Coût : ~0.
3. **L'éclairage.** Un bon HDRI en éclairage d'environnement + du tone mapping ACES + une
   seule lumière directionnelle avec ombres en cascade. Coût : faible, gain : énorme.
4. **Les matières.** Asphalte avec normal map et rugosité variable, reflet mouillé sous la
   pluie. C'est ce qui distingue une route d'un ruban gris.
5. **Le mouvement.** Flou cinétique par vecteurs de vitesse, aberration chromatique légère à
   haute vitesse, vignettage (déjà présent dans le code actuel, l'auteur avait raison).
6. **Enfin seulement** : la densité de géométrie.

L'orientation artistique visée est donc le **réalisme photographique cohérent**, pas la course
au polygone : une image bien éclairée et bien étalonnée avec 300 k triangles bat une image
plate avec 3 M.

### 5.2 Les paliers de qualité

Détectés automatiquement au premier lancement (on mesure le temps d'image pendant 3 secondes
sur une scène étalon), puis ajustables à la main. Chaque palier est un **contrat** :

| | **T0 — Fluide** | **T1 — Équilibré** | **T2 — Élevé** |
|---|---|---|---|
| Cible | mobile d'entrée de gamme, écran embarqué | mobile récent / portable | desktop avec GPU dédié |
| Résolution | 720p × 0.7 | 1080p × 0.85 | native |
| Appels de dessin | ≤ 120 | ≤ 300 | ≤ 600 |
| Triangles visibles | ≤ 250 k | ≤ 800 k | ≤ 2 M |
| Textures | 512 px | 1024 px | 2048 px |
| Ombres | aucune (AO cuit + ombre au sol) | 1 cascade 1024 | 3 cascades 2048 |
| Post-traitement | tone mapping seul | + SMAA | + SSAO, bloom, flou cinétique |
| Distance de vue | 400 m | 900 m | 1600 m |
| **Plancher** | **30 FPS** | **30 FPS** (cible 60) | **60 FPS** |

### 5.3 Le budget d'image, à 30 FPS (33,3 ms)

| Poste | Budget |
|---|---|
| Physique (Rapier, pas fixe) | 4 ms |
| Logique de jeu + mise à jour du graphe de scène | 4 ms |
| Ordonnancement audio | 1 ms |
| Soumission du rendu (côté JS) | 5 ms |
| GPU | 16 ms |
| Marge | 3 ms |

### 5.4 Les techniques, dans l'ordre où elles rapportent

1. **`InstancedMesh` pour tout ce qui se répète** — arbres, glissières, poteaux, bâtiments.
   Un monde de 10 000 objets doit tenir en ~30 appels de dessin. C'est *la* technique qui
   décide de tout.
2. **LOD** générés automatiquement par `gltf-transform simplify` (3 niveaux), avec fondu.
3. **Éclairage cuit pour le statique** : AO et ombres cuites dans les GLB au moment du build.
   Le dynamique se limite au véhicule et à ses feux.
4. **Culling par tuiles le long de la spline** — on ne teste que les tuiles proches du
   véhicule, pas les 10 000 objets.
5. **Résolution adaptative** en dernier recours : si le temps d'image dépasse 30 ms sur 30
   images consécutives, on descend l'échelle de rendu par pas de 5 % (plancher 0.6). Le
   joueur ne le voit presque pas ; une saccade, si.
6. **Colliders séparés des maillages visuels** : le sol reste un heightfield, la route un
   trimesh simplifié, les props des boîtes. Jamais de collider sur la géométrie de rendu.

### 5.5 Faire respecter le budget par les créateurs — le point critique

C'est là que les plateformes UGC vivent ou meurent : sans garde-fou, les joueurs publient des
mondes injouables et c'est le jeu qui a l'air cassé.

- **Une jauge de budget permanente dans l'atelier** : appels de dessin, triangles, mémoire de
  texture, colliders, lumières dynamiques — en direct, avec le palier visé. Verte, orange,
  rouge. Le créateur voit le coût de chaque objet qu'il pose, immédiatement.
- **Un verrou à la publication** : un monde qui explose le budget T0 ne peut pas être publié
  comme « compatible mobile ». Il peut l'être en T2, mais il sera étiqueté comme tel.
- **Un test automatique avant publication** : on lance le monde à vide pendant 10 secondes,
  on mesure le p95 du temps d'image, on l'affiche dans la galerie.

### 5.6 Surveillance en intégration continue

Un test Playwright qui charge trois mondes de référence (léger / moyen / saturé), enregistre
le p95 du temps d'image et **échoue si une régression dépasse 10 %**. Chromium est déjà
disponible dans l'environnement — c'est peu de travail pour beaucoup de sûreté.

---

## 6. L'atelier

L'exigence est « très UX, très jeu vidéo ». Concrètement, ça veut dire :

**Les principes**
- **Pas de modal, pas de formulaire.** On manipule dans la vue 3D, l'inspecteur suit.
- **Retour immédiat.** Chaque geste se voit dans la même image. Pas de bouton « appliquer ».
- **Annuler/refaire sur tout**, avec une pile d'actions nommées et un raccourci franc.
- **Un seul viewport** : c'est le jeu. Une touche bascule entre édition et conduite, sans
  chargement — on teste sa route en trois secondes et on revient.
- **Manette et tactile pris en charge** dès le départ, pas après coup.

**Les outils, par ordre de construction**

| Outil | Ce qu'il fait |
|---|---|
| **Route** | Poser/déplacer des nœuds de spline sur le terrain, régler largeur, dévers et altitude par nœud, choisir un gabarit (2 voies, circuit, piste de terre, tunnel). Le maillage, les glissières, les marquages et le bas-côté se régénèrent en direct. |
| **Terrain** | Biome, seed, amplitude ; pinceaux d'élévation et d'aplanissement autour de la route. |
| **Décor** | Poser à la main, ou **peindre au pinceau de semis** le long de la route (densité, bande de distance, dispersion). Le pinceau est ce qui fait la différence entre 20 minutes et 3 heures pour habiller un circuit. |
| **Matières & couleurs** | Teinter les props, choisir le revêtement par segment de route, palettes prédéfinies. |
| **Ambiance** | Heure du jour (curseur soleil), HDRI, météo, brouillard. Le plus gratifiant à utiliser : on change tout l'aspect du monde avec un curseur. |
| **Règles** | Mode (balade / contre-la-montre / sprint), placement des checkpoints le long de la spline, nombre de tours. |
| **Véhicule & moteur** | L'atelier moteur actuel, repris et étendu au châssis : masse, suspension, adhérence, transmission, livrée. Avec un **banc d'essai** — le mode plein écran existant, conservé. |

**Ce qu'on ne fait pas en v1**, volontairement :
- Pas d'import de maillage arbitraire (voir §7).
- Pas de scripting libre. Un système **événement → action** (« au passage du checkpoint 3 →
  ouvrir la barrière ») couvre 90 % des besoins sans ouvrir un moteur de scripts.
- Pas de modélisation 3D dans le navigateur. On compose à partir d'un catalogue, comme les
  débuts de Fortnite Creative. C'est ce qui garde le budget d'image tenable.

---

## 7. La plateforme

### 7.1 Progression

| | Stockage | Partage | Comptes |
|---|---|---|---|
| **Étape 1** | IndexedDB | export `.json` + URL compressée | aucun |
| **Étape 2** | Supabase Postgres | galerie publique, lien court | OAuth (Google / GitHub) |
| **Étape 3** | + Storage R2/S3 | remix (fork), vignettes, compteurs | profils, collections |

Commencer sans serveur n'est pas une demi-mesure : ça permet de sortir un produit complet et
partageable **sans compte, sans coût, sans RGPD**, et de ne payer l'infra que quand l'usage
la justifie.

### 7.2 Import d'assets par les joueurs

À terme oui, mais pas naïvement. Un GLB uploadé, c'est trois risques d'un coup : la
performance, la modération, et la propriété intellectuelle. Le jour où on l'ouvre :

- validation stricte côté serveur (limites de triangles, de textures, de matériaux) ;
- passage automatique au `gltf-transform` (simplify, meshopt, ktx2) — l'asset importé est
  **réécrit**, jamais servi tel quel ;
- file de modération, signalement, et attestation de droits à l'upload.

### 7.3 Modération et cadre légal

Le fichier actuel a déjà un écran d'avertissement et des CGU — le réflexe est bon, il faut le
garder et l'étendre :

- filtrage des noms et descriptions de mondes ;
- bouton de signalement sur chaque monde publié ;
- dépublication immédiate possible ;
- si le public inclut des mineurs : porte d'âge, et obligations DSA côté UE à regarder
  sérieusement avant l'ouverture publique.

### 7.4 Multijoueur — hors périmètre, et voici pourquoi

Le temps réel synchronisé (serveur autoritatif, prédiction côté client, réconciliation,
anti-triche) est un projet entier à lui seul, plus gros que tout le reste de ce plan.

**L'alternative à 1 % du coût : les fantômes.** On enregistre les entrées du joueur (quelques
octets par image, très compressibles), on rejoue la trajectoire. Le joueur court contre le
meilleur temps du monde, voit la voiture, sent la présence des autres — sans une ligne de
netcode. C'est ce que font Trackmania et Mario Kart depuis toujours, et c'est ce qu'il faut
faire ici.

---

## 8. Phases

Chaque phase se termine sur quelque chose de jouable. Pas de phase « refactoring » sans
livrable visible.

### Phase 0 — Fondations *(~1 semaine)*
- Vite + TypeScript + ESLint, `legacy/moteur-sim.html` déplacé et toujours servi.
- `index.html` devient le hub à deux entrées.
- **Extraction de la synthèse moteur** dans `src/audio/engine-synth.ts`, sans dépendance,
  avec tests. Le fichier legacy continue de tourner sur son code d'origine ; on ne le touche
  pas.
- ✅ *Livrable : le hub, et le jeu actuel toujours intact derrière.*

### Phase 1 — Tranche verticale *(~2–3 semaines)* — **la phase qui décide de tout**
- three/webgpu + Rapier, une route en spline en dur, un terrain simple, une voiture jouable.
- Transmission portée depuis le legacy, branchée sur la synthèse extraite.
- Caméra de jeu (FOV dynamique, secousses, roulis), HDRI + tone mapping ACES.
- Compteur de performance, paliers de qualité T0/T1/T2.
- ✅ *Livrable : on conduit, ça sonne comme avant, et on tient 30 FPS sur téléphone.*
- 🚦 **Point de décision : si la tranche verticale ne tient pas le plancher de 30 FPS avec un
  monde vide, tout le reste du plan est à revoir.** C'est ici qu'on le découvre, pas au
  mois 4.

### Phase 2 — Le monde comme donnée *(~2 semaines)*
- Schémas `WorldSpec` / `VehicleSpec` / `EngineSpec` en zod, avec migrations.
- `world-build.ts` : spline → maillage de route + collider + glissières + marquages.
- `scatter.ts` : semis instancié, LOD, culling par tuiles.
- 3 biomes portés depuis les `SCENES` existantes (montagne, désert, circuit).
- ✅ *Livrable : on charge un monde depuis un JSON. Les mondes sont éditables à la main.*

### Phase 3 — L'atelier *(~4 semaines)* — **la plus grosse**
- Coquille React, viewport partagé, bascule édition ↔ conduite instantanée.
- Outils route, terrain, décor, pinceau de semis, matières, ambiance.
- Jauge de budget en direct.
- Atelier moteur/véhicule repris du legacy, en React, avec le banc d'essai.
- Annuler/refaire, sauvegarde locale, export/import.
- ✅ *Livrable : on crée un circuit complet et on le conduit, sans quitter la page.*

### Phase 4 — Partage *(~2 semaines)*
- URL compressée, galerie locale, vignettes générées depuis le viewport.
- Import automatique des moteurs déjà présents dans le `localStorage` des joueurs.
- Fantômes : enregistrement et rejeu des entrées, meilleur temps par monde.
- ✅ *Livrable : on envoie un lien, l'autre joue le monde et bat le temps.*

### Phase 5 — La plateforme *(~3 semaines)*
- Supabase : auth, publication, galerie publique, remix, compteurs, signalement.
- Classements par monde.
- ✅ *Livrable : une vraie plateforme.*

### Phase 6 — Réalisme et profondeur *(continu)*
- Route mouillée et reflets, météo dynamique, cycle jour/nuit.
- Véhicules et biomes supplémentaires, dégâts visuels, poussière et gomme.
- Shaders TSL personnalisés, éventuellement SSR sur T2.

---

## 9. Risques

| Risque | Gravité | Réponse |
|---|---|---|
| **Les mondes créés par les joueurs tuent la performance** | Élevée | Jauge de budget + verrou à la publication + test automatique avant publication. Conçu dès la phase 3, pas rajouté après. |
| **Le réalisme demandé est incompatible avec 30 FPS sur mobile** | Élevée | Paliers de qualité contractuels ; réalisme par l'éclairage et l'étalonnage plutôt que par la densité. Vérifié dès la phase 1. |
| **L'atelier n'est jamais fini** (c'est le piège classique) | Élevée | Périmètre v1 verrouillé : catalogue fermé, pas de scripting, pas d'import de maillage. On coupe dans les outils, jamais dans la fluidité. |
| **Régression de la synthèse audio à l'extraction** | Moyenne | Tests de non-régression sur la sortie du synthé ; le legacy reste jouable côté à côté pour comparer à l'oreille. |
| **Le format de monde se fige mal** | Moyenne | `schemaVersion` + migrations dès la phase 2, avant qu'un seul monde soit publié. |
| **Poids de chargement** | Moyenne | KTX2 + meshopt, chargement progressif par tuiles, budget de premier affichage < 5 Mo. |
| **Modération / droits sur les assets** | Moyenne | Catalogue CC0 uniquement en v1. L'import utilisateur n'ouvre qu'avec modération. |

---

## 10. Ce que je ferais en premier

Dans l'ordre, et sans rien anticiper d'autre :

1. **Extraire la synthèse moteur** dans un module testé. C'est la pièce irremplaçable du
   projet, et elle n'a aucune raison d'être couplée au rendu.
2. **Construire la tranche verticale** : une voiture, une route, three + Rapier, l'audio
   branché. Rien d'autre. Pas d'atelier, pas de menu, pas de galerie.
3. **Mesurer.** Sur un téléphone d'entrée de gamme, pas sur la machine de développement.

Si les 30 FPS tiennent sur un monde vide, on a un jeu et on peut construire la plateforme
par-dessus en confiance. S'ils ne tiennent pas, on l'aura appris pour trois semaines de
travail au lieu de quatre mois — et le plan se réoriente vers une direction artistique plus
stylisée, qui reste un très bon jeu.

---

## Annexe — sources consultées

- three.js WebGPU : [What's New in Three.js (2026)](https://www.utsubo.com/blog/threejs-2026-what-changed) · [guide de migration](https://www.utsubo.com/blog/webgpu-threejs-migration-guide)
- Physique : [Rapier](https://rapier.rs/) · [comparatif moteurs physiques web](https://www.abratabia.com/game-physics/best-web-physics-engine.php)
- R3F vs three.js : [PkgPulse 2026](https://www.pkgpulse.com/guides/threejs-vs-react-three-fiber-vs-babylonjs-3d-webgl-2026)
- Assets & compression : [KTX 2.0 / Khronos](https://www.khronos.org/news/press/khronos-ktx-2-0-textures-enable-compact-visually-rich-gltf-3d-assets) · [sources d'assets libres 2026](https://app.cinevva.com/guides/free-3d-model-sites)
- Perf three.js : [100 tips (2026)](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
