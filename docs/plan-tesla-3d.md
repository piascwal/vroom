# Vroom — gamification de conduite, sur téléphone et écran Tesla

> Document de cadrage, révision 5. Une application web 3D réaliste dont le monde est piloté
> par la conduite réelle, sur **deux écrans traités à égalité** : le téléphone d'un passager
> et l'écran central d'une Tesla Model Y 2026, pendant la conduite. Aucun serveur, aucune
> dépendance à l'API Tesla.
>
> Ce n'est pas un jeu vidéo : personne ne pilote. Le conducteur conduit sa vraie voiture ;
> l'application observe, représente et récompense.
>
> Révisions précédentes dans l'historique git : `eff3905` (plateforme UGC),
> `5cefd1d` (jeu de conduite classique).

---

## 0. Le cadre

**Les deux écrans sont des cibles de premier rang.** Aucun n'est un portage de l'autre, aucun
n'attend que l'autre soit fini. Chaque phase se termine en état de marche sur les deux, et
l'intégration continue vérifie les deux. C'est une contrainte de méthode autant que
d'architecture, et elle irrigue tout ce document.

L'affichage sur l'écran central pendant la conduite reste au programme, comme décidé. Le
point réglementaire (article R412-6-2 : appareil en fonctionnement dans le champ de vision du
conducteur) a été signalé une fois et ne sera pas rediscuté ici.

Une seule distinction subsiste entre les deux, et ce n'est pas une hiérarchie — ce sont deux
contextes de lecture différents, chacun conçu pour ce qu'il est :

- **Dans la voiture**, la scène est ambiante. Belle en périphérie du regard, rien à décoder,
  une seule information lisible.
- **Sur téléphone**, le passager regarde vraiment. L'interface peut être dense, explorable,
  détaillée.

Deux interfaces conçues séparément, sur un même moteur. Pas une interface et sa version
réduite.

---

## 1. Les deux cibles

|  | **Téléphone** | **Écran Tesla** |
|---|---|---|
| **Contexte de lecture** | passager, regard soutenu | ambiant, coup d'œil |
| **Orientation** | portrait d'abord, paysage géré | paysage, 15,4" ou 16" |
| **Résolution** | ~390×844 pixels CSS, densité 3 | ~2,5K selon finition, densité variable |
| **GPU** | mobile, bride thermiquement vite | AMD RDNA 2, confortable |
| **Navigateur** | Safari / Chrome récents | **Chromium 109** |
| **Capteurs** | **GPS + accéléromètre + gyroscope** | GPS seul |
| **Réseau** | forfait du téléphone | Premium Connectivity |
| **Boucle de test** | immédiate, à chaque enregistrement | il faut aller dans la voiture |

Trois conséquences, toutes structurantes :

1. **WebGL2 est le dénominateur commun**, et pour deux raisons désormais : Chromium 109 n'a
   pas WebGPU (arrivé en Chrome 113), et le parc mobile n'est pas homogène. Décision confirmée,
   `WebGLRenderer` classique — un seul chemin de rendu pour les deux écrans.
2. **Les deux écrans n'ont pas les mêmes capteurs.** Le téléphone a une centrale inertielle,
   la voiture non. C'est la seule asymétrie irréductible du projet, et elle est absorbée par
   une abstraction (§2.2) plutôt que par un chemin dégradé — le reste de l'application ignore
   sur quel écran elle tourne.
3. **La boucle de test est asymétrique, pas le produit.** Développer se fait au téléphone et
   sur traces rejouables parce que c'est cent fois plus rapide ; mais **chaque phase se
   valide dans la voiture avant d'être déclarée finie**, et l'intégration continue mesure les
   deux profils. Sans cette discipline, l'écran Tesla dériverait en silence.

> Les caractéristiques de la colonne Tesla sont à revérifier dans la voiture. Elles varient
> par finition et par firmware, et aucune source publique ne remplace un test réel.

---

## 2. Les données : les capteurs du navigateur, et rien d'autre

Sans serveur, Fleet Telemetry est hors de portée — elle exige un endpoint HTTPS public avec
une configuration TLS imposée. Il reste ce que le navigateur expose lui-même. Sur téléphone
c'est confortable ; sur l'écran Tesla c'est plus maigre. **C'est la seule asymétrie
irréductible entre les deux écrans**, et tout le §2.2 consiste à faire en sorte qu'elle ne se
voie pas.

### 2.1 Ce qu'on lit

**GPS — les deux écrans.** `navigator.geolocation.watchPosition()`, environ 1 Hz :

| Champ | Usage |
|---|---|
| `coords.speed` | vitesse instantanée (m/s) — l'entrée principale |
| `coords.heading` | cap — donne la courbure de la route |
| `coords.latitude` / `longitude` | position du soleil (§5.2), graine du monde (§4) |
| `coords.altitude` | dénivelé — peu fiable, à traiter comme indicatif |
| `timestamp` | base de temps réelle, plus fiable que `performance.now()` pour dériver |

**Centrale inertielle — téléphone seulement.** `DeviceMotionEvent` à 60 Hz :
accélération sur trois axes (avec et sans gravité) et vitesses de rotation. C'est soixante
fois la cadence du GPS, et c'est ce qui change tout.

*Deux pièges à traiter tôt* : iOS exige `DeviceMotionEvent.requestPermission()` déclenché par
un geste utilisateur — même contrainte que l'`AudioContext` du §5.5, donc **un seul écran
d'accueil qui demande les deux d'un coup**. Et le téléphone a une orientation quelconque dans
la poche ou la main : il faut estimer le repère du véhicule sur les premières secondes de
roulage plutôt que de supposer que l'axe Y du téléphone pointe vers l'avant.

### 2.2 Le vrai sujet : passer de 1 Hz à 30 images par seconde

Une position par seconde, une scène à 30 images par seconde. Sans traitement, le monde avance
par à-coups une fois par seconde — c'est le défaut le plus visible qu'on puisse livrer.

**L'abstraction d'abord.** Une seule interface, `MotionSource`, qui expose à chaque image un
état continu — vitesse, cap, accélérations, taux de rotation, plus un indicateur de confiance.
Deux implémentations derrière, et **rien d'autre dans l'application ne sait laquelle tourne** :
ni le corridor, ni la caméra, ni le score, ni le son. C'est ce qui permet de tenir les deux
écrans à égalité sans dupliquer une ligne de logique, et de tester les deux chemins sur les
mêmes traces enregistrées.

**Implémentation A — fusion de capteurs (téléphone).** Le GPS donne une vérité absolue mais
lente et bruitée ; l'accéléromètre donne du relatif, rapide et précis, mais qui dérive. On les
complète l'un par l'autre — filtre complémentaire ou petit Kalman :

- entre deux relevés GPS, l'inertie fait avancer le monde à 60 Hz ;
- à chaque relevé GPS, la dérive accumulée se corrige en douceur, sur ~300 ms, jamais d'un saut ;
- résultat : un mouvement continu **et** juste, y compris dans un tunnel où le GPS disparaît
  mais où l'inertie continue de mesurer.

**Implémentation B — prédiction seule (écran Tesla).** Pas de centrale inertielle : on avance
sur la dernière vitesse et la dernière accélération connues, avec le même recalage progressif
et le même contrat de sortie. C'est intrinsèquement moins fin — l'indicateur de confiance le
dit — mais ce n'est pas un chemin de repli négligé : il a ses propres tests, ses propres
traces, et il doit produire un mouvement aussi *fluide*, à défaut d'être aussi *juste*.

Le raffinement qui compte ici : en l'absence d'inertie, on exploite davantage le **modèle** —
une voiture ne change pas de vitesse arbitrairement entre deux relevés. Borner l'accélération
plausible et lisser sur la dynamique attendue d'un véhicule récupère une bonne part de l'écart
avec le téléphone.

**Dans les deux cas, une dégradation propre.** Tunnel, parking souterrain, perte de signal,
permission refusée : le monde continue sur son erre et ralentit doucement. Il ne se fige pas,
il ne plante pas. Un bug connu du navigateur Tesla remonte parfois un refus de permission de
façon inattendue — ce chemin se teste, il ne se suppose pas.

### 2.3 Ce qu'on dérive

- **accélération longitudinale** — directe sur téléphone, dérivée de la vitesse sur Tesla ;
- **accélération latérale** — directe sur téléphone, cap × vitesse sur Tesla ;
- **à-coup (jerk)** — dérivée de l'accélération, *la* mesure de douceur de conduite ;
- **anticipation** — corrélation entre le relâchement et l'arrivée d'un virage.

### 2.4 Ce qu'on ne peut pas savoir

Pas d'accès au CAN, pas de niveau de batterie, pas de régénération, pas de position de
pédale, pas d'essuie-glaces, pas de météo. Toute mécanique qui suppose autre chose que le GPS
et la centrale inertielle est à écarter dès maintenant.

---

## 3. La stack

### 3.1 Rendu — three.js sur WebGL2

`WebGLRenderer` classique, pas `three/webgpu`. Chromium 109 n'a pas WebGPU et le parc mobile
est hétérogène : embarquer le chemin WebGPU ne ferait qu'alourdir le bundle pour rien. Les
quelques shaders nécessaires s'écrivent en GLSL.

### 3.2 Physique — **aucune**

C'est la simplification la plus nette de ce plan. Personne ne pilote : la caméra avance le
long d'un corridor à la vitesse réelle et tourne selon le cap réel. Il n'y a rien à simuler.

**Rapier disparaît** — et avec lui ~1 Mo de WASM, le pas de temps fixe et toute la phase
« modèle de pneu » de la révision 2. Le seul « moteur physique » restant est le filtre du
§2.2, qui tient en un fichier.

### 3.3 Langage et outillage

TypeScript + Vite. Vitest pour le filtre de fusion, la dérivation des métriques, le calcul
solaire et la génération procédurale — tout ça est du code pur, testable sans GPU, et rejouable
sur des traces enregistrées. Aucun framework d'interface : le HUD est du DOM nu, comme dans le
fichier actuel.

### 3.4 Pas de serveur — et donc, hors ligne pour de bon

Le navigateur Tesla exige Premium Connectivity, un téléphone perd le réseau en tunnel, et une
coupure au milieu d'un trajet ne doit rien casser. Donc :

- **Service worker** qui met en cache toute l'application au premier chargement — code,
  textures, sons. Ensuite, plus une seule requête de tout le trajet. Sur téléphone, ça rend
  aussi l'application installable sur l'écran d'accueil ; dans la voiture, ça la rend
  indépendante de Premium Connectivity une fois le premier chargement fait.
- **Aucune tuile de carte, aucune API météo, aucune police distante.** Tout est embarqué ou
  calculé.
- **Persistance en IndexedDB** : historique, score, moteurs débloqués. Rien ne sort de l'appareil.
- Budget de premier chargement : **sous 15 Mo**, une fois pour toutes.

### 3.5 Deux mises en page, pas une mise à l'échelle

Portrait de téléphone tenu à 40 cm et paysage 16" vu à 80 cm n'ont ni le même ratio, ni la
même densité, ni la même distance de lecture, ni le même temps de regard disponible. Étirer
l'un pour obtenir l'autre donne deux interfaces médiocres.

- **Le canvas 3D remplit toujours l'écran**, et c'est la seule chose vraiment commune. Le
  champ de vision s'ajuste au ratio pour qu'on voie la même portion de route dans les deux
  formats — sinon le paysage paraît écrasé sur l'un des deux.
- **Deux HUD distincts, écrits séparément.** En portrait : dense, sous le pouce, explorable au
  doigt. En paysage voiture : un chiffre, une couleur, dans un coin, taille de texte calculée
  pour la distance de lecture. Ce ne sont pas deux dispositions du même composant.
- **Rien en dur** : ni résolution, ni ratio, ni densité de pixels. La mise à jour Tesla
  2026.26 a déjà changé la densité et cassé des applications web.
- **Zones sûres** respectées (encoche, barre d'accueil) via `env(safe-area-inset-*)`, et
  cibles tactiles dimensionnées pour un doigt en mouvement dans une voiture — nettement plus
  larges que sur un bureau, sur les deux écrans.

---

## 4. Le monde : procédural, pas cartographique

Sans serveur, reproduire la géographie réelle supposerait des tuiles vectorielles distantes —
donc une dépendance réseau, une clé d'API et un coût par requête. On prend l'autre chemin, et
il se trouve qu'il est meilleur :

**Un corridor routier généré procéduralement, dont la forme suit la conduite réelle.**

- La route se construit devant soi, tuile par tuile, en reprenant **la courbure réelle**
  mesurée. Quand la voiture tourne à droite, la route tourne à droite.
- Le décor, le relief et la végétation sont semés depuis une graine dérivée de la position :
  **le même endroit produit toujours le même paysage.** Le trajet domicile-travail a sa
  physionomie, reconnaissable, sans qu'on ait jamais chargé une carte.
- Le biome suit la latitude et l'altitude — alpin, méditerranéen, plaine — donc le paysage
  reste plausible sans être une reproduction.

Ce que ça gagne : ça marche en tunnel, ça marche sans réseau, ça ne coûte rien, ça ne peut pas
afficher une carte fausse, et un paysage cohérent est plus beau qu'un modèle bas de gamme
extrudé depuis des données ouvertes.

Ce que ça perd, et il faut l'assumer : **ce n'est pas la vraie route.** C'est une route qui a
la forme, la lumière et le rythme de la vraie. Si la reconnaissance géographique littérale
devient un besoin produit, il faudra rouvrir la question du réseau.

---

## 5. Le réalisme

Priorité affirmée. Voici où le mettre, dans l'ordre de rendement.

### 5.1 La lumière avant tout

Un ciel physique et un bon étalonnage font davantage que n'importe quelle densité de
géométrie, et coûtent peu — y compris sur un GPU de téléphone.

- **Ciel procédural physique** (Hosek-Wilkie ou Preetham) calculé dans un shader : aucun HDRI
  à télécharger, et il est juste à toute heure.
- **Éclairage d'environnement** dérivé de ce ciel, régénéré seulement quand le soleil a bougé
  de façon significative — quelques fois par trajet, pas à chaque image.
- **Tone mapping ACES**, exposition adaptée à l'heure. C'est ce qui sépare « scène 3D » de
  « photo ».
- **Une seule directionnelle** avec ombres en cascade — deux cascades sur Tesla, une seule sur
  téléphone.

### 5.2 Le détail qui porte tout le concept

**Le soleil de la scène est le vrai soleil.**

Azimut et élévation se calculent depuis latitude, longitude et heure UTC : de l'astronomie,
une quarantaine de lignes, aucun réseau. Résultat — en roulant plein ouest un soir de
novembre, on a le soleil couchant dans le pare-brise **et** dans la scène, au même endroit, à
la même hauteur, avec la même couleur.

Pour le passager qui alterne entre la vitre et son écran, c'est exactement le détail qui fait
basculer la perception de « décor » à « fenêtre ». Pour un coût dérisoire.

*Limite honnête* : le soleil est juste, la météo ne l'est pas. Sans serveur, pas d'API météo.
La couverture nuageuse reste soit un réglage, soit une variation lente et neutre. À trancher.

### 5.3 Les matières

Nombre d'objets réduit, qualité par objet élevée — le bon arbitrage quand la caméra est en
mouvement constant et que rien n'est jamais examiné de près.

- Asphalte avec normal map et **rugosité variable** : les traces de passage sont plus lisses
  que le reste. C'est ce qui distingue une route d'un ruban gris.
- Végétation en cartes croisées avec vent léger, pas en géométrie dense.
- **Éclairage cuit impossible ici** — le monde est généré à la volée. On compense avec de
  l'occlusion ambiante approchée dans le shader et un ciel de qualité.

### 5.4 Le mouvement

Le plus grand levier de réalisme quand la caméra ne s'arrête jamais.

- **Flou cinétique par vecteurs de vitesse**, indexé sur la vitesse réelle. À 130 km/h les
  bas-côtés doivent filer. Sur téléphone, version simplifiée si le budget le demande.
- **Champ de vision qui s'ouvre avec la vitesse**, très légèrement.
- **Aucune secousse de caméra.** C'est l'inverse d'un jeu : l'occupant ressent déjà les vraies
  secousses dans son corps. En ajouter à l'image crée un conflit sensoriel désagréable — et,
  sur un téléphone tenu en main dans une voiture en mouvement, un vrai risque de nausée. La
  caméra doit être d'une stabilité absolue.
- Vignettage et aberration chromatique très légère à haute vitesse.

### 5.5 Le son — la plus forte réutilisation du projet

La synthèse moteur existante est branchée sur la **conduite réelle** : vitesse et accélération
entrent dans le modèle de transmission déjà écrit, qui produit un régime, qui pilote le
synthé. Les dix-huit préréglages fonctionnent tels quels.

**Une voiture électrique qui sonne comme un V8 au rythme de la vraie accélération.** C'est
déjà construit, ça ne coûte presque rien en calcul, et c'est probablement le premier argument
du produit — surtout au casque, pour un passager.

À y ajouter : vent filtré par la vitesse, roulement, et un `ConvolverNode` dont la réponse
change en tunnel. Trois nœuds Web Audio, un effet immédiat.

**Contrainte à traiter tôt** : tous les navigateurs exigent un geste utilisateur pour démarrer
l'`AudioContext`. Comme iOS exige le même geste pour la centrale inertielle (§2.1), **un seul
écran d'accueil demande les deux permissions d'un coup** — sans quoi le produit est muet, ou
saccadé, et personne ne comprend pourquoi.

---

## 6. La fluidité : 30 FPS verrouillés, volontairement

Les deux cibles pourraient viser 60 images par seconde. **On ne le fera sur aucune des deux**,
pour trois raisons qui vont toutes dans le même sens.

1. **L'énergie.** Sur téléphone, une application 3D qui tourne tout un trajet vide la
   batterie du passager — le plus sûr moyen de ne pas être relancé. Sur Tesla, l'APU sous
   charge coûte ~22 km d'autonomie. Diviser la charge par deux est un vrai service rendu.
2. **La chaleur.** Un téléphone qui chauffe se bride tout seul, et une chute de 60 à 35 se
   voit bien plus qu'un 30 stable. La régularité est perçue comme de la fluidité ; le nombre
   brut, non.
3. **L'image.** Les 16 ms récupérées vont dans la lumière, les ombres et le flou cinétique.
   C'est exactement l'arbitrage demandé : du réalisme, en restant fluide.

**30 FPS verrouillés, jamais dépassés, jamais manqués.**

### 6.1 Budget d'image — 33,3 ms, identique sur les deux cibles

| Poste | Budget |
|---|---|
| Télémétrie, fusion, score | 2 ms |
| Audio | 1 ms |
| Génération procédurale (amortie sur plusieurs images) | 2 ms |
| Soumission du rendu (JS) | 4 ms |
| GPU | 20 ms |
| Marge | 4,3 ms |

Le budget ne change pas d'une cible à l'autre. **Ce qui change, c'est ce qu'on fait tenir dans
les 20 ms de GPU** — c'est tout l'objet des deux profils ci-dessous.

### 6.2 Deux profils, tenus tous les deux

| | **Téléphone** | **Écran Tesla** |
|---|---|---|
| Échelle de rendu | 0,6 · plancher 0,45 | 0,7 · plancher 0,55 |
| Appels de dessin | ≤ 90 | ≤ 200 |
| Triangles visibles | ≤ 200 k | ≤ 700 k |
| Ombres | 1 cascade 1024 | 2 cascades 2048 |
| Post-traitement | tone mapping, vignettage, flou simplifié | + flou par vecteurs, SMAA |
| Distance de vue | 350 m | 700 m |
| Végétation | densité réduite, LOD agressif | densité pleine |

Détection au premier lancement : on mesure le temps d'image trois secondes sur une scène
étalon et on choisit. Réglable à la main ensuite.

Les deux profils sont des cibles de livraison, pas un profil et sa version dégradée : chacun
a son jeu de réglages accordé pour être **beau à son échelle**, et les deux passent les mêmes
tests de performance en intégration continue. Un profil qui casse bloque la livraison, quel
qu'il soit.

### 6.3 Les techniques

1. **Génération amortie.** Jamais une tuile entière dans une seule image : le travail est
   découpé et étalé. Un à-coup de génération est plus visible que dix images moins belles.
2. **`InstancedMesh` pour tout ce qui se répète.** Le corridor entier doit tenir sous une
   centaine d'appels de dessin.
3. **Échelle de rendu adaptative.** On rend nettement sous la résolution native et on remonte.
   Invisible en mouvement, décisif pour le budget — surtout à 2,5K.
4. **Culling par tuiles** le long du corridor.
5. **Objets recyclés** : les tuiles sortantes reviennent dans un pool. Aucune allocation en
   régime établi — un ramasse-miettes au mauvais moment, c'est une saccade.
6. **Distance de vue courte, brouillard atmosphérique généreux.** Cohérent avec le ciel
   physique, et ça supprime la moitié de la géométrie.

### 6.4 Vérification

Un test Playwright rejoue trois traces enregistrées (ville, autoroute, montagne) **dans les
deux formats d'écran et sur les deux profils**, avec les deux implémentations de
`MotionSource`. Il mesure le p95 du temps d'image et **échoue au-delà de 33,3 ms** — sur
l'une ou l'autre des configurations, indifféremment. Six combinaisons, une seule barre.

C'est le garde-fou qui empêche l'écran Tesla de dériver pendant qu'on travaille au téléphone.
Mais il ne remplace rien : **les seuls chiffres qui comptent sont mesurés sur un vrai
téléphone et dans la vraie voiture**, et chaque phase se termine par ces deux mesures.

---

## 7. La boucle de gamification

Le monde *est* le retour d'information. Rien à lire pour comprendre comment on conduit.

| Mesure | Ce qu'elle vaut | Comment le monde réagit |
|---|---|---|
| **Douceur** (à-coup faible) | le cœur du score | ciel qui se dégage, lumière qui se réchauffe, végétation plus dense |
| **Anticipation** (relâcher avant le virage) | forte | la route s'ouvre, l'horizon recule |
| **Régularité de vitesse** | moyenne | ambiance stable, couleurs saturées |
| **Freinages brusques, accélérations dures** | pénalisant | lumière qui se ternit, ciel qui se couvre |

**Sur téléphone**, le passager regarde vraiment : on peut afficher le score en continu, le
détail des métriques, la carte de chaleur du trajet, et laisser explorer.

**Sur l'écran Tesla**, une seule information lisible : **un chiffre, gros, avec une couleur.**
Rien d'autre. Pas de menu, pas de notification, pas d'animation qui appelle le regard.

**Et à l'arrêt, la récompense :** bilan du trajet, progression, et **l'atelier moteur** — déjà
construit, avec ses dix-huit préréglages et son banc d'essai — où les kilomètres bien conduits
débloquent des pièces et des moteurs. La boucle est complète : bien conduire fait un plus beau
monde et un meilleur son.

---

## 8. Phases

**Une règle traverse tout le découpage : une phase n'est finie que quand elle tourne sur les
deux écrans.** Pas « ça marche au téléphone, on verra la voiture plus tard » — c'est
exactement ainsi qu'un des deux finit par accumuler six semaines de dette invisible.

Le développement quotidien se fait au téléphone et sur traces rejouables, parce que la boucle
y est cent fois plus rapide. Mais chaque fin de phase passe par la voiture, et l'intégration
continue mesure les deux profils à chaque commit.

### Phase 0 — Fondations *(~1 semaine)*
- Vite + TypeScript ; `legacy/moteur-sim.html` déplacé et toujours servi ; `index.html`
  devient le hub.
- **Extraction de la synthèse moteur** et de la transmission dans `src/`, avec tests de
  non-régression.
- Squelette de déploiement HTTPS — nécessaire dès la phase suivante pour ouvrir la page dans
  la voiture.
- ✅ *Le hub, et le jeu actuel intact derrière.*

### Phase 1 — Vérité terrain, sur les deux écrans *(~1 semaine)* — **avant tout le reste**
- Une page de mesure, ouverte **sur le téléphone** puis **dans la voiture**, qui répond aux
  mêmes questions des deux côtés : cadence réelle du GPS, `coords.speed` renseigné ou non,
  cadence de `DeviceMotion` là où elle existe, comportement des permissions, densité de pixels
  réelle, et combien de triangles un WebGL2 nu tient à 30 FPS.
- **Enregistrement de traces** GPS + inertie sur trois trajets types. Elles deviennent le jeu
  de test de tout le projet, pour les deux implémentations de `MotionSource`.
- ✅ *Deux jeux de mesures, un par écran, et trois traces rejouables.*
- 🚦 **Aucune ligne de moteur 3D avant ces chiffres.** Et surtout : les chiffres de la voiture
  sont pris maintenant, pas dans deux mois. C'est le seul moyen de ne pas concevoir une
  architecture que l'écran Tesla ne pourra pas tenir.

### Phase 2 — Le mouvement juste *(~2–3 semaines)*
- `MotionSource` et **ses deux implémentations**, développées ensemble et testées sur les
  mêmes traces : fusion GPS + inertie d'un côté, prédiction contrainte par le modèle véhicule
  de l'autre.
- Estimation du repère du véhicule (téléphone), dégradation propre en tunnel (les deux).
- Corridor procédural, semis par graine de position, recyclage des tuiles.
- ✅ *Le monde avance à la vraie vitesse, sans un à-coup, sur les deux écrans.*

### Phase 3 — La lumière et les deux formats *(~2–3 semaines)*
- Ciel physique, soleil réel, ACES, ombres en cascade.
- Verrouillage 30 FPS, échelle de rendu adaptative, **les deux profils accordés séparément**.
- **Les deux mises en page**, écrites séparément : portrait dense, paysage voiture épuré.
- ✅ *C'est beau et stable des deux côtés — et c'est ici qu'on le vérifie, pas plus tard.*

### Phase 4 — Le son et le score *(~2 semaines)*
- Synthèse moteur branchée sur la télémétrie réelle ; vent, roulement, réverbération.
- Écran d'accueil unique : audio + capteurs en un geste (téléphone), audio seul (voiture).
- Métriques, score, réaction du monde du §7, **et les deux HUD** — l'explorable et l'ambiant.
- ✅ *La conduite s'entend et se voit, dans les deux contextes de lecture.*

### Phase 5 — Matières et profondeur *(~2 semaines)*
- Asphalte à rugosité variable, végétation, flou cinétique, brouillard atmosphérique.
- Biomes par latitude et altitude, cycle jour/nuit complet, éclairage nocturne.
- Bilan de trajet, historique, atelier moteur porté, déblocages.
- ✅ *La boucle est fermée.*

### Phase 6 — Finition *(continu)*
- Météo réglable, pluie et route mouillée, davantage de biomes et de moteurs.

---

## 9. Risques

| Risque | Gravité | Réponse |
|---|---|---|
| **L'écran Tesla dérive pendant qu'on développe au téléphone** | Élevée | Le risque propre au double écran. Réponse : mesures dans la voiture dès la phase 1, validation des deux écrans à chaque fin de phase, et les six combinaisons testées en intégration continue. C'est une discipline, elle ne s'improvise pas en fin de projet. |
| **Les capteurs ne donnent pas ce qu'on croit** — cadence GPS, `speed` nul, permission refusée, `DeviceMotion` bridé | Élevée | Tout l'objet de la phase 1, sur les deux écrans. Une semaine, avant tout engagement d'architecture. |
| **Le monde avance par à-coups** | Élevée | `MotionSource` et ses deux implémentations sont la phase 2, avant le rendu, testées sur traces rejouables. C'est le défaut le plus visible possible. |
| **L'écran Tesla ne peut pas tenir ce que le téléphone tient** (ou l'inverse) | Moyenne | L'abstraction `MotionSource` et les deux profils isolent les différences. Si un écart de qualité s'avère irréductible, il se constate en phase 1 — quand il est encore temps de revoir la cible visuelle des deux. |
| **Nausée du passager** | Moyenne | Caméra absolument stable, aucune secousse ajoutée, champ de vision contenu. À tester tôt sur de vrais passagers, pas au bureau. |
| **Batterie et chauffe du téléphone** | Moyenne | 30 FPS verrouillés, échelle de rendu basse, veille dès l'arrêt. Mesurer sur un trajet d'une heure en phase 3. |
| **Le procédural ne « ressemble » pas assez à la vraie route** | Moyenne | Assumé au §4. Si la reconnaissance littérale devient un besoin, rouvrir la question du réseau — et donc du sans-serveur. |
| **Le firmware Tesla casse l'appli** (2026.26 a déjà changé la densité de pixels) | Faible | Rien en dur : ni résolution, ni ratio, ni densité. Retester dans la voiture à chaque mise à jour majeure. |
| **Régression de la synthèse audio à l'extraction** | Faible | Tests de non-régression ; le legacy reste jouable côte à côte pour comparer à l'oreille. |

---

## 10. Ce que je ferais en premier

1. **La phase 1, cette semaine, sur les deux écrans.** Une page de mesure, un trajet avec le
   téléphone, un trajet avec la page ouverte dans la voiture. Deux jeux de chiffres, trois
   traces enregistrées.
2. **Extraire la synthèse moteur** en parallèle — travail sûr, indépendant du reste, et c'est
   la pièce irremplaçable du projet.
3. **Puis `MotionSource` et ses deux implémentations**, avec leurs tests sur les traces, avant
   la moindre ligne de rendu.
4. **Ensuite seulement**, le corridor et la lumière.

Le plan tient en ~11 semaines : aucune physique, aucun serveur, aucune modération. Tenir deux
écrans à égalité coûte environ une semaine de plus qu'en tenir un — l'abstraction de mouvement,
la seconde mise en page et la double validation — et c'est un prix modeste comparé à un portage
découvert trop tard.

La seule vraie inconnue reste le comportement des capteurs en voiture, et elle se lève en une
semaine.

---

## Annexe — sources consultées

- Navigateur Tesla : [passage à Chromium](https://www.teslarati.com/tesla-chromium-in-car-web-browser/) · [Geolocation API](https://forums.tesla.com/forum/forums/web-browser-59-supports-geolocation-api) · [bug de permission connu](https://github.com/Leaflet/Leaflet/issues/7157) · [optimiser un site pour le navigateur Tesla](https://webhostingbuddy.com/blog/5-tips-to-optimize-your-website-for-teslas-in-car-web-browser-model-s-3-x-y/) · [mise à jour été 2026](https://codriver.io/guides/tesla-browser-summer-2026-update)
- Matériel : [MCU2 vs MCU3](https://www.notateslaapp.com/news/2417/tesla-intel-atom-mcu-2-and-amd-ryzen-mcu-3-feature-differences-and-how-to-tell-what-you-have) · [APU Ryzen et autonomie](https://insideevs.com/news/588007/tesla-showdown-old-intel-gpu-vs-new-amd-apu/) · [écran Model Y 2026](https://tslablog.com/vehicles/2026-model-y-juniper-premium-awd/)
- three.js : [état 2026](https://www.utsubo.com/blog/threejs-2026-what-changed) · [100 astuces de performance](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
- Cadre : [article R412-6-2 — Légifrance](https://www.legifrance.gouv.fr/codes/article_lc/LEGIARTI000025111520)
