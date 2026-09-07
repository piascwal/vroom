# Vroom — le cockpit en 3D réaliste

> Document de cadrage, révision 6. Prendre le cockpit qui existe déjà dans
> `moteur-sim.html` et le rendre nettement plus réaliste, en 3D, sur **deux écrans traités à
> égalité** : le téléphone d'un passager et l'écran central d'une Tesla Model Y 2026, pendant
> la conduite. La route et le moteur suivent la conduite réelle. Aucun serveur.
>
> **Un seul objectif : le réalisme.** Pas de score, pas de progression, pas de récompense,
> pas de contenu créé par les joueurs. Ces pistes ont été explorées dans les révisions
> précédentes et sont abandonnées — elles restent dans l'historique git (`eff3905`,
> `5cefd1d`, `9eb75fc`).

---

## 0. Le cadre

**Les deux écrans sont des cibles de premier rang.** Aucun n'est un portage de l'autre, aucun
n'attend que l'autre soit fini. Chaque phase se termine en état de marche sur les deux, et
l'intégration continue vérifie les deux.

L'affichage sur l'écran central pendant la conduite est décidé. Le point réglementaire
(article R412-6-2) a été signalé une fois et ne sera pas rediscuté ici.

Une seule distinction subsiste, et ce n'est pas une hiérarchie — ce sont deux distances de
lecture. Dans la voiture, l'écran est à 80 cm et se regarde de biais ; sur téléphone, à 40 cm
et de face. Ça change le champ de vision, la taille du texte et la densité de l'habillage, pas
la nature de ce qui est montré.

---

## 1. Ce qu'on a, et ce qu'on améliore

`moteur-sim.html` est un mono-fichier de ~4 900 lignes, sans aucune dépendance. Il fait déjà
tout ce qu'il faut — il le fait juste dans les limites d'un rasteriseur logiciel sur canvas 2D.

| Brique existante | État | Ce qu'on en fait |
|---|---|---|
| **Synthèse moteur** (Web Audio) | excellente | On l'extrait telle quelle. On l'alimente avec la vitesse réelle au lieu d'une pédale à l'écran, et on lui ajoute ce qui manque autour du moteur (§6.2). |
| **Atelier moteur** | 18 préréglages, ~20 réglages, banc d'essai | On le garde tel quel et on le porte. C'est là qu'on choisit le moteur qu'on va entendre en roulant. |
| **Transmission** | rapports, seuils de passage | On la porte. C'est elle qui transforme la vitesse réelle en régime. |
| **Compteurs du cockpit** | aiguilles, chiffres | Ils deviennent réels : ils affichent ta vitesse et ton régime, pas ceux d'une simulation. |
| **Route et décor** | tranches horizontales, faces peintes, pas d'éclairage | **C'est le seul morceau remplacé.** Route 3D, matières PBR, ciel physique, ombres. |

Le plafond actuel, en trois points :

- **Le remplissage de pixels est le mur.** Le profil cité dans le code dit que le JS ne pèse
  que 7 % du temps ; tout le reste, c'est le CPU qui peint des triangles. Aucune optimisation
  JS ne débloquera ça, seul le GPU le peut.
- **Pas d'éclairage, pas de matières.** Les couleurs sont figées par face. Pas d'ombres, pas
  de reflets, pas de normal maps : le réalisme demandé est hors d'atteinte par construction.
- **La route est un ruban sur des rails.** Une pile de tranches horizontales, une caméra qui
  ne peut pas en sortir. Pas de dénivelé réel, pas de courbure vraie.

Le commentaire ligne 2248 explique pourquoi il n'y a pas de WebGL : *« une dépendance WebGL
coûterait l'autonomie du fichier et supposerait un GPU dont on ne sait rien sur l'écran de la
voiture »*. On sait désormais de quel GPU il s'agit (§2), et l'autonomie du fichier est
remplacée par une application entièrement hors ligne (§4.4).

---

## 2. Les deux écrans

|  | **Téléphone** | **Écran Tesla** |
|---|---|---|
| **Distance de lecture** | ~40 cm, de face | ~80 cm, de biais |
| **Orientation** | portrait d'abord, paysage géré | paysage, 15,4" ou 16" |
| **Résolution** | ~390×844 pixels CSS, densité 3 | ~2,5K selon finition, densité variable |
| **GPU** | mobile, bride thermiquement vite | AMD RDNA 2, confortable |
| **Navigateur** | Safari / Chrome récents | **Chromium 109** |
| **Capteurs** | **GPS + accéléromètre + gyroscope** | GPS seul |
| **Réseau** | forfait du téléphone | Premium Connectivity |
| **Boucle de test** | immédiate | il faut aller dans la voiture |

Trois conséquences :

1. **WebGL2 est le dénominateur commun.** Chromium 109 n'a pas WebGPU (arrivé en Chrome 113),
   et le parc mobile n'est pas homogène. Un seul chemin de rendu pour les deux écrans.
2. **Les deux écrans n'ont pas les mêmes capteurs.** Seule asymétrie irréductible du projet,
   absorbée par une abstraction (§3.2) plutôt que par un chemin dégradé.
3. **La boucle de test est asymétrique, pas le produit.** On développe au téléphone et sur
   traces rejouables parce que c'est cent fois plus rapide, mais **chaque phase se valide dans
   la voiture avant d'être déclarée finie**.

> Les caractéristiques de la colonne Tesla sont à revérifier dans la voiture. Elles varient
> par finition et par firmware, et aucune source publique ne remplace un test réel.

---

## 3. Ce qui pilote la simulation

Aujourd'hui, la vitesse vient d'une pédale dessinée à l'écran. Demain elle vient de la voiture.
C'est le seul changement de fond dans la logique existante — et c'est un gain de réalisme, pas
une fonctionnalité en plus : le régime qu'on entend est celui qui correspond à la vitesse à
laquelle on roule vraiment.

### 3.1 Ce qu'on lit

Sans serveur, on s'en tient à ce que le navigateur expose.

**GPS — les deux écrans.** `watchPosition()`, environ 1 Hz : `coords.speed` (l'entrée
principale), `coords.heading` (la courbure de la route), `latitude`/`longitude` (position du
soleil, graine du monde), `altitude` (peu fiable), `timestamp`.

**Centrale inertielle — téléphone seulement.** `DeviceMotionEvent` à 60 Hz : accélération sur
trois axes et vitesses de rotation. Soixante fois la cadence du GPS.

*Deux pièges* : iOS exige `DeviceMotionEvent.requestPermission()` déclenché par un geste
utilisateur — même contrainte que l'`AudioContext`, donc **un seul écran d'accueil qui demande
les deux d'un coup**. Et le téléphone a une orientation quelconque dans la main : il faut
estimer le repère du véhicule sur les premières secondes de roulage.

*Ce qu'on ne peut pas savoir* : pas d'accès au CAN, pas de niveau de batterie, pas de
régénération, pas de position de pédale. Le GPS et la centrale inertielle sont la totalité de
l'entrée.

### 3.2 `MotionSource` — passer de 1 Hz à 30 images par seconde

Une position par seconde, une scène à 30 images par seconde. Sans traitement, le monde avance
par à-coups une fois par seconde. C'est le défaut le plus visible qu'on puisse livrer, et
c'est aussi celui qui ruine le plus sûrement le réalisme.

**L'abstraction d'abord.** Une seule interface qui expose à chaque image un état continu —
vitesse, cap, accélérations, taux de rotation, plus un indicateur de confiance. Deux
implémentations derrière, et **rien d'autre dans l'application ne sait laquelle tourne** : ni
la route, ni la caméra, ni les compteurs, ni le son.

**Implémentation A — fusion de capteurs (téléphone).** Le GPS donne une vérité absolue mais
lente et bruitée ; l'accéléromètre donne du relatif, rapide, qui dérive. On les complète l'un
par l'autre — filtre complémentaire ou petit Kalman. Entre deux relevés GPS, l'inertie fait
avancer le monde à 60 Hz ; à chaque relevé, la dérive accumulée se corrige sur ~300 ms, jamais
d'un saut. Ça continue de fonctionner en tunnel.

**Implémentation B — prédiction seule (écran Tesla).** Pas de centrale inertielle : on avance
sur la dernière vitesse et la dernière accélération connues, avec le même contrat de sortie.
Intrinsèquement moins fin, mais ce n'est pas un chemin négligé — il a ses propres tests et ses
propres traces. En l'absence d'inertie on exploite davantage le **modèle** : une voiture ne
change pas de vitesse arbitrairement entre deux relevés, donc borner l'accélération plausible
récupère une bonne part de l'écart.

**Dans les deux cas, une dégradation propre.** Tunnel, parking, perte de signal, permission
refusée : le monde continue sur son erre et ralentit doucement. Il ne se fige pas, il ne plante
pas. Un bug connu du navigateur Tesla remonte parfois un refus de permission de façon
inattendue — ce chemin se teste, il ne se suppose pas.

### 3.3 Et la transmission existante prend le relais

Vitesse réelle → rapports de boîte → régime → synthèse moteur. C'est exactement la chaîne
déjà écrite dans `moteur-sim.html`, avec une seule entrée changée. Les seuils de passage, les
rétrogradations, la coupure au rupteur : tout fonctionne tel quel.

---

## 4. La stack

### 4.1 Rendu — three.js sur WebGL2

`WebGLRenderer` classique, pas `three/webgpu` : la cible n'a pas WebGPU, embarquer ce chemin
alourdirait le bundle pour rien. Les quelques shaders nécessaires s'écrivent en GLSL.

### 4.2 Physique — aucune

Personne ne pilote : la caméra avance à la vitesse réelle et tourne selon le cap réel. Il n'y
a rien à simuler. **Pas de Rapier, pas de WASM, pas de pas de temps fixe.** Le seul filtre est
celui du §3.2.

### 4.3 Langage et outillage

TypeScript + Vite. Vitest pour le filtre de fusion, la transmission, le calcul solaire et la
génération procédurale — du code pur, testable sans GPU et **rejouable sur les traces
enregistrées**. Aucun framework d'interface : le cockpit est du DOM nu, comme aujourd'hui.

### 4.4 Pas de serveur

Service worker qui met toute l'application en cache au premier chargement — ensuite, plus une
seule requête de tout le trajet. Aucune tuile de carte, aucune API météo, aucune police
distante. Réglages et moteurs créés en `localStorage`/IndexedDB, comme aujourd'hui. Sur
téléphone, l'application s'installe sur l'écran d'accueil ; dans la voiture, elle devient
indépendante de Premium Connectivity. **L'autonomie du mono-fichier est préservée, autrement.**

### 4.5 Deux mises en page, pas une mise à l'échelle

Le canvas 3D remplit toujours l'écran, et le champ de vision s'ajuste au ratio pour qu'on voie
la même portion de route des deux côtés. Le cockpit, lui, est **écrit deux fois** : dense et
au pouce en portrait, plus épuré et dimensionné pour 80 cm en paysage voiture. Rien en dur —
ni résolution, ni ratio, ni densité : la mise à jour Tesla 2026.26 a déjà changé la densité et
cassé des applications web.

---

## 5. Le monde

Sans serveur, reproduire la géographie réelle supposerait des tuiles vectorielles distantes —
dépendance réseau, clé d'API, coût par requête. On génère à la place, et c'est plus robuste :

**Un corridor routier procédural dont la forme suit la conduite réelle.** La route se construit
devant soi en reprenant la courbure mesurée : quand la voiture tourne à droite, la route tourne
à droite. Décor, relief et végétation sont semés depuis une graine dérivée de la position, donc
**le même endroit produit toujours le même paysage** — le trajet domicile-travail est
reconnaissable sans qu'on ait jamais chargé une carte. Le biome suit la latitude et l'altitude.

Ça marche en tunnel, sans réseau, sans coût. **Ce n'est pas la vraie route** : c'en est une qui
a sa forme, sa lumière et son rythme. Si la reconnaissance géographique littérale devenait un
besoin, il faudrait rouvrir la question du réseau.

C'est aussi un net gain sur l'existant : les huit `SCENES` actuelles sont de belles ambiances
figées ; là, le paysage est cohérent avec l'endroit où l'on se trouve.

---

## 6. Le réalisme

Le seul axe du projet. Voici où le mettre, dans l'ordre de rendement.

### 6.1 L'image

1. **Ciel procédural physique** (Hosek-Wilkie ou Preetham) calculé dans un shader : aucun HDRI
   à télécharger, juste à toute heure. **Éclairage d'environnement dérivé de ce ciel**,
   régénéré seulement quand le soleil a bougé sensiblement.
2. **Tone mapping ACES**, exposition adaptée à l'heure. C'est ce qui sépare « scène 3D » de
   « photo », et c'est le plus gros écart avec le rendu actuel.
3. **Une seule directionnelle** avec ombres en cascade — deux cascades dans la voiture, une
   sur téléphone.
4. **Matières PBR.** Asphalte avec normal map et **rugosité variable** — les traces de passage
   sont plus lisses que le reste. C'est ce qui distingue une route d'un ruban gris.
5. **Post-traitement** : SMAA, bloom discret, flou cinétique par vecteurs de vitesse,
   vignettage — déjà présent dans le fichier actuel, l'auteur avait raison.

**Le détail qui porte tout** : azimut et élévation du soleil se calculent depuis latitude,
longitude et heure UTC. Quarante lignes d'astronomie, aucun réseau. En roulant plein ouest un
soir de novembre, on a le soleil couchant dans le pare-brise **et** dans la scène, au même
endroit, à la même hauteur. Pour quelqu'un qui alterne entre la vitre et l'écran, c'est ce qui
fait basculer la perception de « décor » à « fenêtre ».

Pas de météo : le ciel varie avec l'heure, un point c'est tout.

### 6.2 Le son

La synthèse existante est déjà le point fort du projet. Ce qui lui manque, c'est tout ce qui
n'est pas le moteur :

- **Roulement** : bruit filtré dont le timbre dépend du revêtement et le volume de la vitesse.
- **Vent** : bruit rose filtré, ouvert avec la vitesse.
- **Sifflement de transmission** indexé sur le régime de sortie de boîte.
- **Réverbération contextuelle** : un `ConvolverNode` dont la réponse change en tunnel, en
  forêt, en espace ouvert. Spectaculaire pour un coût dérisoire.

Et surtout, le régime est désormais **juste** : il correspond à la vitesse à laquelle on roule
réellement. Le moteur choisi à l'atelier s'entend au bon régime, au bon moment, avec les vraies
montées de rapport.

**Contrainte à traiter tôt** : tous les navigateurs exigent un geste utilisateur pour démarrer
l'`AudioContext`.

### 6.3 Le cockpit

C'est l'identité du projet et il faut le porter proprement en 3D :

- **Habitacle modélisé**, volant, montants de pare-brise, tableau de bord — la route est vue
  *depuis* une voiture, pas d'une caméra flottante.
- **Aiguilles physiques** qui oscillent, avec l'inertie d'une vraie aiguille.
- **Compteurs justes** : la vitesse affichée est la vraie vitesse, le régime est celui de la
  transmission alimentée par elle.
- **Ombres portées de l'habitacle** sur le tableau de bord, qui bougent avec le soleil réel.
- L'atelier moteur reste ce qu'il est aujourd'hui : plein écran, avec son banc d'essai.

### 6.4 Le mouvement

- **Flou cinétique par vecteurs de vitesse**, indexé sur la vitesse réelle.
- **Champ de vision qui s'ouvre avec la vitesse**, très légèrement.
- **Aucune secousse de caméra.** L'occupant ressent déjà les vraies secousses dans son corps ;
  en ajouter à l'image crée un conflit sensoriel désagréable, et sur un téléphone tenu en main
  dans une voiture qui bouge, un vrai risque de nausée. Stabilité absolue.

---

## 7. La fluidité : 30 FPS verrouillés

Les deux écrans pourraient viser 60 images par seconde. On ne le fera sur aucun des deux.

1. **L'énergie.** Sur téléphone, une application 3D qui tourne tout un trajet vide la batterie.
   Dans la voiture, l'APU sous charge coûte ~22 km d'autonomie.
2. **La chaleur.** Un téléphone qui chauffe se bride tout seul, et une chute de 60 à 35 se voit
   bien plus qu'un 30 stable. La régularité est perçue comme de la fluidité ; le nombre brut, non.
3. **L'image.** Les 16 ms récupérées vont dans la lumière, les ombres et le flou cinétique.
   C'est l'arbitrage demandé : du réalisme, en restant fluide.

### 7.1 Budget d'image — 33,3 ms, identique sur les deux écrans

| Poste | Budget |
|---|---|
| Télémétrie, fusion, transmission | 2 ms |
| Audio | 1 ms |
| Génération procédurale (amortie) | 2 ms |
| Soumission du rendu (JS) | 4 ms |
| GPU | 20 ms |
| Marge | 4,3 ms |

Le budget ne change pas d'un écran à l'autre. **Ce qui change, c'est ce qu'on fait tenir dans
les 20 ms de GPU.**

### 7.2 Deux profils, tenus tous les deux

| | **Téléphone** | **Écran Tesla** |
|---|---|---|
| Échelle de rendu | 0,6 · plancher 0,45 | 0,7 · plancher 0,55 |
| Appels de dessin | ≤ 90 | ≤ 200 |
| Triangles visibles | ≤ 200 k | ≤ 700 k |
| Ombres | 1 cascade 1024 | 2 cascades 2048 |
| Post-traitement | tone mapping, vignettage, flou simplifié | + flou par vecteurs, SMAA |
| Distance de vue | 350 m | 700 m |
| **Plancher** | **30 FPS** | **30 FPS** |

Deux cibles de livraison, pas un profil et sa version dégradée : chacun est accordé pour être
beau à son échelle, et un profil qui casse bloque la livraison, quel qu'il soit.

### 7.3 Les techniques

1. **Génération amortie** : jamais une tuile entière dans une seule image. Un à-coup de
   génération est plus visible que dix images moins belles.
2. **`InstancedMesh` pour tout ce qui se répète.** Le corridor entier sous une centaine
   d'appels de dessin.
3. **Échelle de rendu adaptative** : on rend sous la résolution native et on remonte.
4. **Culling par tuiles** le long du corridor.
5. **Objets recyclés** : aucune allocation en régime établi — un ramasse-miettes au mauvais
   moment, c'est une saccade.
6. **Distance de vue courte, brouillard atmosphérique généreux**, cohérent avec le ciel physique.

### 7.4 Vérification

Un test Playwright rejoue trois traces enregistrées **dans les deux formats et sur les deux
profils**, avec les deux implémentations de `MotionSource` : six combinaisons, une seule barre
à 33,3 ms. C'est le garde-fou qui empêche l'écran Tesla de dériver pendant qu'on travaille au
téléphone. Mais les seuls chiffres qui comptent sont mesurés sur un vrai téléphone et dans la
vraie voiture.

---

## 8. Phases

**Une phase n'est finie que quand elle tourne sur les deux écrans.**

### Phase 0 — Fondations *(~1 semaine)*
- Vite + TypeScript ; `legacy/moteur-sim.html` déplacé et toujours servi ; `index.html`
  devient le hub à deux entrées.
- **Extraction de la synthèse moteur** et de la transmission, avec tests de non-régression.
- Squelette de déploiement HTTPS — la Geolocation API exige un contexte sécurisé, donc sans
  ça il n'y a pas de phase 1.
- ✅ *Le hub, et le jeu actuel intact derrière.*

### Phase 1 — Vérité terrain *(~1 semaine)* — 🚦 **avant tout le reste**
- Une page de mesure ouverte **sur le téléphone** puis **dans la voiture** : cadence réelle du
  GPS, `coords.speed` renseigné ou non, cadence de `DeviceMotion`, comportement des
  permissions, densité de pixels réelle, et combien de triangles un WebGL2 nu tient à 30 FPS.
- **Enregistrement de traces** GPS + inertie sur trois trajets types. Elles deviennent le jeu
  de test de tout le projet.
- ✅ *Deux jeux de mesures et trois traces rejouables.*
- 🚦 Aucune ligne de moteur 3D avant ces chiffres.

### Phase 2 — Le mouvement juste *(~2–3 semaines)*
- `MotionSource` et ses deux implémentations, testées sur les traces.
- Transmission existante rebranchée sur la vitesse réelle ; compteurs justes.
- Corridor procédural, semis par graine de position, recyclage des tuiles.
- ✅ *On roule, la route avance à la vraie vitesse, le compte-tours dit vrai.*

### Phase 3 — La lumière *(~2–3 semaines)*
- Ciel physique, soleil réel, ACES, ombres en cascade, matières PBR.
- Verrouillage 30 FPS, échelle de rendu adaptative, les deux profils accordés séparément.
- Les deux mises en page.
- ✅ *L'écart avec le rendu canvas actuel est flagrant.*

### Phase 4 — Le son et le cockpit *(~2 semaines)*
- Roulement, vent, sifflement de transmission, réverbération contextuelle.
- Écran d'accueil unique : audio + capteurs en un geste.
- Habitacle modélisé, aiguilles physiques, ombres portées.
- Atelier moteur porté, avec son banc d'essai.
- ✅ *Le cockpit complet, en 3D, qui sonne juste.*

### Phase 5 — Profondeur *(~2 semaines)*
- Végétation, biomes par latitude et altitude, cycle jour/nuit complet, éclairage nocturne.
- Flou cinétique, brouillard atmosphérique, finitions de matières.
- ✅ *Ça ressemble à une fenêtre, pas à un décor.*

### Phase 6 — Finition *(continu)*
- Davantage de biomes, de moteurs, de variations d'ambiance.

---

## 9. Risques

| Risque | Gravité | Réponse |
|---|---|---|
| **Les capteurs ne donnent pas ce qu'on croit** | Élevée | Tout l'objet de la phase 1, sur les deux écrans. Une semaine, avant tout engagement d'architecture. |
| **Le monde avance par à-coups** | Élevée | `MotionSource` est la phase 2, avant le rendu, testée sur traces rejouables. Le défaut le plus destructeur pour le réalisme. |
| **L'écran Tesla dérive pendant qu'on développe au téléphone** | Élevée | Mesures dans la voiture dès la phase 1, validation des deux à chaque fin de phase, six combinaisons en intégration continue. |
| **Nausée** | Moyenne | Caméra absolument stable, aucune secousse ajoutée. À tester tôt sur de vrais passagers. |
| **Batterie et chauffe du téléphone** | Moyenne | 30 FPS verrouillés, échelle de rendu basse, veille dès l'arrêt. Mesurer sur un trajet d'une heure en phase 3. |
| **Le procédural ne « ressemble » pas assez à la vraie route** | Moyenne | Assumé au §5. Si la reconnaissance littérale devient un besoin, rouvrir la question du réseau. |
| **Régression de la synthèse audio à l'extraction** | Faible | Tests de non-régression ; le legacy reste jouable côte à côte pour comparer à l'oreille. |

---

## 10. Par où commencer

1. **La phase 1, cette semaine, sur les deux écrans.** Une page de mesure, un trajet avec le
   téléphone, un trajet avec la page ouverte dans la voiture.
2. **Extraire la synthèse moteur** en parallèle — travail sûr, indépendant du reste, et c'est
   la pièce irremplaçable du projet.
3. **Puis `MotionSource` et ses deux implémentations**, avec leurs tests, avant la moindre
   ligne de rendu.
4. **Ensuite seulement**, la route et la lumière.

Le plan tient en ~11 semaines. Aucune physique, aucun serveur, aucune fonctionnalité nouvelle
au-delà du réalisme : on remplace le rasteriseur et on branche la vraie voiture à la place des
pédales à l'écran. Tout le reste du fichier actuel survit.

---

## Annexe — sources consultées

- Navigateur Tesla : [passage à Chromium](https://www.teslarati.com/tesla-chromium-in-car-web-browser/) · [Geolocation API](https://forums.tesla.com/forum/forums/web-browser-59-supports-geolocation-api) · [bug de permission connu](https://github.com/Leaflet/Leaflet/issues/7157) · [mise à jour été 2026](https://codriver.io/guides/tesla-browser-summer-2026-update)
- Matériel : [MCU2 vs MCU3](https://www.notateslaapp.com/news/2417/tesla-intel-atom-mcu-2-and-amd-ryzen-mcu-3-feature-differences-and-how-to-tell-what-you-have) · [APU Ryzen et autonomie](https://insideevs.com/news/588007/tesla-showdown-old-intel-gpu-vs-new-amd-apu/) · [écran Model Y 2026](https://tslablog.com/vehicles/2026-model-y-juniper-premium-awd/)
- three.js : [état 2026](https://www.utsubo.com/blog/threejs-2026-what-changed) · [100 astuces de performance](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
