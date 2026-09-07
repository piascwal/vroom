# Vroom — refaire l'interface, en qualité jeu vidéo

> Document de cadrage, révision 7. Reprendre `moteur-sim.html` et en refaire **l'interface et
> le rendu de la route** à un niveau de finition professionnel, sur deux écrans traités à
> égalité : le téléphone et l'écran central d'une Tesla Model Y 2026.
>
> **« Réalisme » veut dire ici : la qualité visuelle.** Les cadrans et l'interface d'un côté,
> la route, ses côtés et l'arrière-plan de l'autre — les deux au même niveau d'exigence. Pas de
> simulation, pas de physique, et **aucun calcul d'ambiance depuis l'heure réelle** : la lumière
> de chaque scène reste un réglage artistique. La demande d'origine était « quelque chose d'un
> peu plus jeu vidéo professionnel » — c'est exactement ça.
>
> Les révisions précédentes (plateforme UGC, jeu de course, gamification, réalisme graphique)
> sont abandonnées et restent dans l'historique git.

---

## 1. Ce qui existe déjà — y compris ce que je croyais à construire

En relisant le code, une bonne partie de ce que les révisions précédentes plaçaient en
« phases fondatrices » est **déjà écrite et fonctionne**.

| Déjà fait | Où | Ce que j'avais planifié à tort |
|---|---|---|
| **Vitesse GPS réelle** | `watchPosition`, `gpsSpeedKmh`, `smoothedGpsSpeed` | « brancher la vraie voiture » — c'est fait, `debugMode` faux |
| **Cap et courbure de route** | `applyHeadingSample`, trois sources : GPS, trajet, boussole | idem |
| **Rythme irrégulier de `watchPosition`** | cap suivi **par image** à vitesse plafonnée | toute l'abstraction `MotionSource` et sa « fusion de capteurs » |
| **Repli quand le GPS ne donne pas de cap** | relèvement entre deux points, seuil de 4 m | « dégradation propre » |
| **Accélération dérivée** | `prevSpeedForAccel` | idem |
| **Mode pédales** | `debugMode`, avec régulateur et nitro | — |
| **Coup de gaz à l'arrêt** | pédale d'accélérateur du cockpit | j'avais proposé de la supprimer — elle a sa raison d'être |

Le commentaire ligne 2017 documente même l'approche qui échouait (asservir la courbure au
delta entre deux relevés) et pourquoi le suivi par image la remplace. **Il n'y a rien à
refaire là-dedans.**

Ce qui reste vrai de l'audit : la synthèse moteur, l'atelier, la transmission et les compteurs
sont bons. **Le seul morceau qui plafonne est le rendu**, et l'interface mérite un cran de
finition supplémentaire.

---

## 2. Ce qu'il reste vraiment à faire

Trois chantiers, et rien d'autre.

1. **Les cadrans et l'interface, refaits en qualité jeu vidéo** (§3).
2. **La scène en WebGL** — route, côtés, arrière-plan — au même niveau d'exigence (§4).
3. **Les deux écrans tenus proprement** — portrait téléphone et paysage Tesla (§5).

Les deux premiers sont le projet, à parts égales. C'est l'ensemble qui doit avoir l'air d'un
jeu fini : un beau cadran sur une route pauvre ne suffit pas, l'inverse non plus.

Ce qui est explicitement hors périmètre : physique de véhicule, serveur, score, contenu créé
par les joueurs, habitacle modélisé, météo, et **toute ambiance calculée depuis l'heure ou la
position réelles** — l'éclairage appartient à la scène, pas à l'horloge.

---

## 3. Le réalisme des cadrans et de l'interface

Premier des deux fronts. Voici ce que « professionnel » veut dire, concrètement.

### 3.1 Les matières

Aujourd'hui les compteurs sont dessinés avec des dégradés. Un instrument crédible a de la
matière :

- **Lunettes en métal usiné** avec réflexion anisotrope — la lumière file le long du tournage,
  elle ne fait pas un dégradé uniforme.
- **Verre** avec un reflet propre, une légère parallaxe entre le verre et le cadran, et un
  assombrissement sur les bords.
- **Cadran texturé** — grain fin, pas un aplat.
- **Profondeur réelle** : le cadran est en creux sous la lunette, les boutons ont une
  épaisseur, et **la lumière vient d'une direction unique et constante** sur tout l'écran.
  C'est la règle qui fait le plus pour la cohérence, et celle qu'on enfreint le plus souvent.

### 3.2 Les aiguilles et le mouvement

- **Ombre portée de l'aiguille sur le cadran**, décalée selon la même source de lumière.
- **Inertie** : l'aiguille ne suit pas la valeur, elle la rattrape, avec un léger dépassement
  et un rebond en butée.
- **Aucune transition linéaire.** Chaque animation a une courbe choisie. C'est ce qui sépare
  une interface web d'une interface de jeu.
- Tout à **60 images par seconde** — l'interface n'est pas soumise au budget de la 3D (§6.2).

### 3.3 La typographie

- Une vraie fonte d'instrumentation, pas la police système. Chiffres **tabulaires** partout où
  une valeur change, sinon les chiffres dansent.
- **Graduations dessinées**, pas approximées : longueurs majeures et mineures distinctes,
  numérotation alignée sur l'arc, épaisseurs cohérentes.
- Une échelle typographique tenue, et des tailles calculées pour la distance de lecture de
  chaque écran (§5).

### 3.4 Les états

Chaque état est **dessiné**, pas obtenu en changeant une couleur :

- allumage — les aiguilles balayent leur course, comme un vrai tableau de bord ;
- veille, marche, zone rouge, alerte ;
- pression d'un bouton, maintien, désactivé ;
- perte du signal GPS — un état visuel propre, pas un chiffre qui se fige.

### 3.5 Un seul système

Mêmes rayons, mêmes ombres, même source de lumière, même palette, mêmes durées d'animation —
du cockpit à l'atelier en passant par les menus. L'atelier moteur en particulier doit devenir
un vrai **banc de réglage** : c'est l'écran le plus dense du produit et celui où la finition
se voit le plus.

La palette actuelle (ambre, cyan, métal, fond presque noir) est bonne et se garde. Ce n'est
pas une refonte de direction artistique, c'est un cran de finition.

---

## 4. Le réalisme de la scène

Second front, à parts égales avec le premier. Le rasteriseur logiciel sur canvas 2D est le
plafond — le profil cité dans le code dit que le JS ne pèse que 7 % du temps, tout le reste
est du remplissage de pixels au CPU. WebGL débloque l'éclairage, les ombres, la profondeur et
la densité.

Trois plans, chacun avec son travail propre.

### 4.1 La route

C'est ce qu'on regarde 90 % du temps, donc c'est là qu'il faut mettre le plus de soin.

- **Revêtement** avec normal map et **rugosité variable** : les traces de passage sont plus
  lisses que le reste. C'est ce qui distingue une route d'un ruban gris.
- **Marquages au sol correctement filtrés** — mipmaps et filtrage anisotrope. Sans ça, les
  bandes scintillent à distance : c'est le défaut le plus visible d'une route en 3D, et le
  plus facile à éviter si on y pense dès le début.
- **La route a des bords** : bas-côté, bordures, glissières, joints de chaussée, raccords
  d'enrobé. C'est le détail qui donne l'épaisseur.
- **Variantes par scène** : enrobé neuf, béton, piste de terre, route mouillée avec ses
  reflets étirés.

### 4.2 Les côtés

Végétation, mobilier urbain, bâti, rochers — selon la scène. **Instanciés**, avec LOD et
fondu, en densité suffisante pour que ça défile vraiment : c'est la densité qui donne la
sensation de vitesse, plus que la vitesse elle-même.

### 4.3 L'arrière-plan

Horizon en plusieurs couches, perspective atmosphérique — plus c'est loin, plus ça se fond
dans le ciel. C'est ce qui crée la profondeur, et c'est peu coûteux.

### 4.4 L'éclairage

Une directionnelle avec ombres en cascade, plus un éclairage d'ambiance dérivé du ciel de la
scène.

**La direction du soleil est un réglage de scène, choisi artistiquement — jamais calculé
depuis l'heure ni la position.** Les huit `SCENES` actuelles ont déjà de bons réglages
d'ambiance (couleurs de ciel, brume, teintes de décor) : elles servent de base, et gagnent un
angle de lumière et une intensité.

### 4.5 Le nez du véhicule

Au lieu d'un habitacle complet, un élément d'avant-plan par scène : le nez d'une F1 sur le
circuit, un capot ailleurs. C'est la vue d'un jeu de course en caméra avant, ça donne une
identité immédiate à chaque scène, et ça coûte quelques centaines de triangles fixes.

---

## 5. Les deux écrans

|  | **Téléphone** | **Écran Tesla** |
|---|---|---|
| **Distance de lecture** | ~40 cm, de face | ~80 cm, de biais |
| **Orientation** | portrait d'abord, paysage géré | paysage, 15,4" ou 16" |
| **Résolution** | ~390×844 pixels CSS, densité 3 | ~2,5K selon finition, densité variable |
| **GPU** | mobile, bride thermiquement vite | AMD RDNA 2, confortable |
| **Navigateur** | Safari / Chrome récents | **Chromium 109 — WebGL2, pas WebGPU** |

**Deux mises en page écrites séparément**, pas une mise à l'échelle. Un cadran lisible à 40 cm
sur un écran de téléphone et le même cadran lisible à 80 cm de biais n'ont ni la même taille
relative, ni la même épaisseur de trait, ni la même densité d'information.

Rien en dur : ni résolution, ni ratio, ni densité de pixels — la mise à jour Tesla 2026.26 a
déjà changé la densité et cassé des applications web.

**Une phase n'est finie que quand elle tourne sur les deux écrans.** On développe au téléphone
parce que la boucle est plus rapide, mais chaque fin de phase passe par la voiture.

---

## 6. La stack

### 6.1 Ce qui change, ce qui ne change pas

- **three.js sur WebGL2** pour la route et le décor. `WebGLRenderer` classique : Chromium 109
  n'a pas WebGPU, inutile d'embarquer ce chemin.
- **TypeScript + Vite**, découpage du mono-fichier en modules. Vitest sur la transmission et
  la synthèse.
- **Aucune physique, aucun serveur, aucun framework d'interface.** Le code GPS existant est
  porté tel quel.
- Service worker pour que tout fonctionne hors ligne après le premier chargement — ça préserve
  l'autonomie du mono-fichier, autrement.

### 6.2 L'interface n'est pas dans la scène 3D

Décision importante : **les compteurs restent en canvas 2D et en DOM par-dessus le canvas
WebGL**, comme aujourd'hui.

- Le texte reste net à n'importe quelle densité de pixels — un cadran rendu en 3D est
  systématiquement plus flou.
- L'interface tourne à 60 images par seconde même si la scène est à 30.
- Elle ne consomme pas le budget GPU de la route.
- Et c'est bien plus rapide à itérer, ce qui compte quand c'est le cœur du projet.

---

## 7. La fluidité

**Scène 3D à 30 images par seconde, verrouillées. Interface à 60.** Les deux sont découplées.

Trois raisons de plafonner la scène : sur téléphone, une application 3D qui tourne tout un
trajet vide la batterie ; dans la voiture, l'APU sous charge coûte ~22 km d'autonomie ; et un
téléphone qui chauffe se bride tout seul, or une chute de 60 à 35 se voit bien plus qu'un 30
stable. La régularité est perçue comme de la fluidité, le nombre brut non.

Budget de la scène, à 33,3 ms : ~4 ms de logique et de soumission JS, ~26 ms de GPU, le reste
en marge. L'interface a son propre budget de 16,7 ms, dont elle n'utilisera qu'une fraction.

**Les techniques qui comptent** : `InstancedMesh` pour tout ce qui se répète, échelle de rendu
adaptative (on rend sous la résolution native et on remonte — décisif à 2,5K), culling par
tuiles, objets recyclés pour n'allouer rien en régime établi, distance de vue courte avec
brouillard généreux.

---

## 8. Phases

### Phase 0 — Fondations *(~1 semaine)*
- Vite + TypeScript ; `legacy/moteur-sim.html` déplacé et toujours servi ; `index.html`
  devient le hub à deux entrées.
- Découpage du mono-fichier : synthèse moteur, transmission, GPS, rendu — chacun dans son
  module, avec tests de non-régression sur l'audio.
- Squelette de déploiement HTTPS (la Geolocation API exige un contexte sécurisé).
- **Sonde WebGL** : une page qui mesure, dans la voiture et sur le téléphone, combien de
  triangles tiennent à 30 FPS. Une demi-journée, et c'est la seule inconnue matérielle qui
  reste.
- ✅ *Le hub, le jeu actuel intact derrière, et un chiffre de budget géométrique.*

### Phase 1 — Le langage visuel *(~1–2 semaines)*
- Maquettes de l'écran principal, **dans les deux formats**, jusqu'à ce que ce soit juste.
- Le système : matières, source de lumière, rayons, ombres, typographie, échelle, durées
  d'animation, états.
- Un cadran de référence implémenté pour de bon, pas une image — c'est lui qui valide le système.
- ✅ *On sait à quoi ça ressemble, et on l'a vu sur les deux écrans.*
- 🚦 **C'est ici que le projet se joue.** Si le langage visuel n'est pas au niveau, tout le
  reste n'y changera rien.

### Phase 2 — L'interface refaite *(~2 semaines)*
- Tous les compteurs, les commandes, les menus, le HUD, aux deux formats.
- L'atelier moteur en vrai banc de réglage.
- États complets : allumage, veille, alerte, perte de GPS.
- ✅ *L'interface complète, au niveau de finition visé.*

### Phase 3 — La scène en WebGL *(~2 semaines)*
- Socle : route, véhicules et décor portés depuis le rasteriseur canvas, éclairage
  directionnel et ombres en cascade, atmosphère.
- Les huit scènes existantes reportées avec leurs réglages d'ambiance, plus un angle et une
  intensité de lumière par scène.
- Verrouillage 30 FPS, échelle de rendu adaptative, découplage d'avec l'interface.
- ✅ *On roule en WebGL, à budget tenu.*

### Phase 4 — La scène au niveau *(~2 semaines)*
- Revêtement, rugosité variable, marquages correctement filtrés, bords de chaussée.
- Densité et LOD des côtés, arrière-plan en couches, perspective atmosphérique.
- Nez de véhicule en avant-plan par scène.
- Variantes de revêtement, route mouillée.
- ✅ *L'écart avec le rendu actuel est flagrant, des deux côtés.*

### Phase 5 — Finition *(~1 semaine)*
- Passe de perf dans la voiture et sur téléphone, mesure de consommation sur un trajet long.
- Ajustements de lisibilité à 80 cm.
- ✅ *Livrable.*

**~8 semaines**, contre 11 dans la révision précédente — parce qu'une bonne partie de ce que
j'avais planifié est déjà dans le fichier.

---

## 9. Risques

| Risque | Gravité | Réponse |
|---|---|---|
| **Le langage visuel n'atteint pas le niveau visé** | Élevée | C'est le risque principal, et c'est un risque de design, pas de technique. D'où une phase 1 dédiée, avec des maquettes avant tout code, et un cadran de référence réellement implémenté pour valider. |
| **Illisible à 80 cm dans la voiture** | Moyenne | Les deux formats sont maquettés dès la phase 1, et vérifiés dans la voiture — pas sur un navigateur redimensionné. |
| **La scène n'atteint pas le niveau visé** | Élevée | Même nature que le risque précédent, sur l'autre front. La route se traite en premier (§4.1) : c'est 90 % de ce qu'on regarde, et le reste peut se densifier progressivement. |
| **Le GPU Tesla ne tient pas la scène** | Moyenne | La sonde WebGL est en phase 0, avant tout engagement. La densité des côtés et la distance de vue sont les deux variables d'ajustement, sans toucher à la route elle-même. |
| **Scintillement des marquages au sol** | Moyenne | Mipmaps et filtrage anisotrope dès le premier jet de la route, pas en correctif. C'est le défaut le plus visible d'une route en 3D. |
| **Régression de la synthèse audio au découpage** | Moyenne | Tests de non-régression sur la sortie ; le legacy reste jouable côte à côte pour comparer à l'oreille. |
| **Batterie et chauffe du téléphone** | Faible | 30 FPS verrouillés sur la scène, échelle de rendu basse, veille dès l'arrêt. |

---

## 10. Par où commencer

1. **Phase 0**, qui est du travail sûr : découpage, hub, et la sonde WebGL pour connaître le
   budget géométrique.
2. **Puis les maquettes de la phase 1** — cadrans *et* une image de la route visée — dans les
   deux formats. C'est là que le projet se décide, et ça ne demande pas une ligne de moteur 3D.

Une question ouverte avant de maquetter : **quelle référence visuelle ?** Un tableau de bord
d'hypercar moderne, un HUD de jeu de course type Forza / Gran Turismo, ou un registre plus
instrument d'aviation. La palette actuelle (ambre, cyan, métal, fond presque noir) marche pour
les trois, mais le dessin des cadrans et le traitement de la route changent complètement selon
la réponse.
