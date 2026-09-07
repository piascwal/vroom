# Vroom — gamification de conduite, sur téléphone et écran Tesla

> Document de cadrage, révision 4. Une application web 3D réaliste dont le monde est piloté
> par la conduite réelle. **Cible n°1 : un passager sur son téléphone.** Cible n°2 : l'écran
> central d'une Tesla Model Y 2026, pendant la conduite. Aucun serveur, aucune dépendance à
> l'API Tesla.
>
> Ce n'est pas un jeu vidéo : personne ne pilote. Le conducteur conduit sa vraie voiture ;
> l'application observe, représente et récompense.
>
> Révisions précédentes dans l'historique git : `eff3905` (plateforme UGC),
> `5cefd1d` (jeu de conduite classique).

---

## 0. Le cadre

Le passager sur son téléphone est l'usage principal, et il ne pose aucune question
particulière — c'est un passager qui regarde son écran.

L'affichage sur l'écran central pendant la conduite reste au programme, comme décidé. Le
point réglementaire (article R412-6-2 : appareil en fonctionnement dans le champ de vision du
conducteur) a été signalé une fois et ne sera pas rediscuté ici.

Une seule conséquence de conception en est tirée, et elle ne coûte rien puisqu'elle ne
s'applique qu'à la cible n°2 : **sur l'écran de la voiture, la scène est ambiante, pas
informative.** Belle en périphérie du regard, rien à décoder. Sur téléphone, où le passager
regarde vraiment, cette retenue tombe et l'interface peut être plus riche.

---

## 1. Les deux cibles

|  | **Téléphone** — cible n°1 | **Écran Tesla** — cible n°2 |
|---|---|---|
| **Usage** | passager, regard soutenu | ambiant, coup d'œil |
| **Orientation** | portrait d'abord, paysage géré | paysage, 15,4" ou 16" |
| **Résolution** | ~390×844 pixels CSS, densité 3 | ~2,5K selon finition, densité variable |
| **GPU** | mobile, bride thermiquement vite | AMD RDNA 2, confortable |
| **Navigateur** | Safari / Chrome récents | **Chromium 109** |
| **Capteurs** | **GPS + accéléromètre + gyroscope** | GPS seul |
| **Réseau** | forfait du téléphone | Premium Connectivity |
| **Test** | immédiat, à chaque enregistrement | nécessite d'aller dans la voiture |

Trois conséquences, toutes structurantes :

1. **WebGL2 est le dénominateur commun**, et pour deux raisons désormais : Chromium 109 n'a
   pas WebGPU (arrivé en Chrome 113), et le parc mobile n'est pas homogène. Décision confirmée,
   `WebGLRenderer` classique.
2. **Le téléphone a des capteurs que la voiture n'a pas.** C'est ce qui résout le problème
   central du §2 — et ça rend la cible n°1 techniquement *meilleure* que la n°2, pas dégradée.
3. **On construit pour le téléphone d'abord.** Boucle de test en secondes au lieu d'un trajet
   en voiture, meilleurs capteurs, pas de Premium Connectivity. L'écran Tesla est un portage,
   fait ensuite — pas l'inverse.

> Les caractéristiques de la colonne Tesla sont à revérifier dans la voiture. Elles varient
> par finition et par firmware, et aucune source publique ne remplace un test réel.

---

## 2. Les données : les capteurs du navigateur, et rien d'autre

Sans serveur, Fleet Telemetry est hors de portée — elle exige un endpoint HTTPS public avec
une configuration TLS imposée. Il reste ce que le navigateur expose lui-même. Sur téléphone,
c'est largement suffisant ; sur l'écran Tesla, c'est plus maigre, et c'est le seul endroit où
la cible n°2 est en retrait.

### 2.1 Ce qu'on lit

**GPS — les deux cibles.** `navigator.geolocation.watchPosition()`, environ 1 Hz :

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

**Sur téléphone : fusion de capteurs.** Le GPS donne une vérité absolue mais lente et
bruitée ; l'accéléromètre donne du relatif, rapide et précis, mais qui dérive. On les
complète l'un par l'autre — filtre complémentaire ou petit Kalman :

- entre deux relevés GPS, l'inertie fait avancer le monde à 60 Hz ;
- à chaque relevé GPS, la dérive accumulée se corrige en douceur, sur ~300 ms, jamais d'un saut ;
- résultat : un mouvement continu **et** juste, y compris dans un tunnel où le GPS disparaît
  mais où l'inertie continue de mesurer.

**Sur l'écran Tesla : prédiction seule.** Pas de centrale inertielle, donc on avance sur la
dernière vitesse et la dernière accélération connues, avec le même recalage progressif. C'est
moins bon, et il faut l'accepter : le téléphone aura le meilleur ressenti.

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
  aussi l'application installable sur l'écran d'accueil, ce qui est le bon geste pour la
  cible n°1.
- **Aucune tuile de carte, aucune API météo, aucune police distante.** Tout est embarqué ou
  calculé.
- **Persistance en IndexedDB** : historique, score, moteurs débloqués. Rien ne sort de l'appareil.
- Budget de premier chargement : **sous 15 Mo**, une fois pour toutes.

### 3.5 Une mise en page, deux formats

Portrait de téléphone et paysage 16" n'ont ni le même ratio, ni la même densité, ni la même
distance de lecture. Ce n'est pas une adaptation cosmétique :

- **Le canvas 3D remplit toujours l'écran** ; seul le champ de vision s'ajuste au ratio, pour
  qu'on voie la même chose de la route dans les deux formats.
- **Le HUD est repositionné, pas redimensionné.** En portrait il vit en bas, sous le pouce ;
  en paysage il se range dans un coin. Deux dispositions, pas une mise à l'échelle.
- **Rien en dur** : ni résolution, ni ratio, ni densité de pixels. La mise à jour Tesla
  2026.26 a déjà changé la densité et cassé des applications web.
- **Zones sûres** respectées (encoche, barre d'accueil) via `env(safe-area-inset-*)`.

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

### 6.2 Deux profils, détectés au lancement

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

Un test Playwright qui rejoue trois traces enregistrées (ville, autoroute, montagne), mesure
le p95 du temps d'image et **échoue au-delà de 33,3 ms**. Utile en garde-fou — mais il ne
remplace rien : **les seuls chiffres qui comptent sont mesurés sur un vrai téléphone et dans
la vraie voiture.**

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

Tout se construit sur téléphone. L'écran Tesla est un portage, fait une fois que le produit
tient debout.

### Phase 0 — Fondations *(~1 semaine)*
- Vite + TypeScript ; `legacy/moteur-sim.html` déplacé et toujours servi ; `index.html`
  devient le hub.
- **Extraction de la synthèse moteur** et de la transmission dans `src/`, avec tests de
  non-régression.
- ✅ *Le hub, et le jeu actuel intact derrière.*

### Phase 1 — Vérité terrain *(~2–3 jours)* — **avant tout le reste**
- Une page de mesure, en HTTPS, ouverte **sur ton téléphone, en voiture** : cadence réelle du
  GPS, `coords.speed` renseigné ou non, cadence de `DeviceMotion`, le geste de permission iOS,
  et combien de triangles un WebGL2 nu tient à 30 FPS.
- **Enregistrement de traces** GPS + inertie sur trois trajets types. Elles deviennent le jeu
  de test de tout le reste du projet — on développe ensuite au bureau, sans reprendre la voiture.
- ✅ *Cinq réponses mesurées et trois traces rejouables.*
- 🚦 **Aucune ligne de moteur 3D avant ces chiffres.** Deux jours ici évitent deux mois
  d'architecture posée sur des suppositions.

### Phase 2 — Le mouvement juste *(~2 semaines)*
- Fusion GPS + inertie du §2.2, testée sur les traces enregistrées. La partie la plus délicate
  du projet, et la première à faire.
- Estimation du repère du véhicule, dégradation propre en tunnel.
- Corridor procédural, semis par graine de position, recyclage des tuiles.
- ✅ *Le monde avance à la vraie vitesse, sans un à-coup.*

### Phase 3 — La lumière *(~2 semaines)*
- Ciel physique, soleil réel, ACES, ombres en cascade.
- Verrouillage 30 FPS, échelle de rendu adaptative, les deux profils.
- Mise en page portrait et paysage.
- ✅ *C'est beau, c'est stable, ça tourne sur ton téléphone.*

### Phase 4 — Le son et le score *(~2 semaines)*
- Synthèse moteur branchée sur la télémétrie réelle ; vent, roulement, réverbération.
- Écran d'accueil unique : audio + capteurs en un geste.
- Métriques, score, réaction du monde du §7, HUD des deux cibles.
- ✅ *La conduite s'entend et se voit.*

### Phase 5 — Le portage Tesla *(~1 semaine)*
- Chemin sans centrale inertielle, mise en page paysage 2,5K, profil Tesla.
- Service worker et fonctionnement hors Premium Connectivity.
- ✅ *Ça tourne sur l'écran de la voiture.*

### Phase 6 — Matières et profondeur *(~2 semaines)*
- Asphalte, végétation, flou cinétique, brouillard atmosphérique, biomes, cycle jour/nuit.
- Bilan de trajet, historique, atelier moteur porté, déblocages.
- ✅ *La boucle est fermée.*

---

## 9. Risques

| Risque | Gravité | Réponse |
|---|---|---|
| **Les capteurs ne donnent pas ce qu'on croit** — cadence GPS, `speed` nul, permission refusée, `DeviceMotion` bridé | Élevée | Tout l'objet de la phase 1. Deux jours, avant tout engagement d'architecture. |
| **Le monde avance par à-coups** | Élevée | La fusion est la phase 2, avant le rendu, testée sur traces rejouables. C'est le défaut le plus visible possible. |
| **Nausée du passager** | Moyenne | Caméra absolument stable, aucune secousse ajoutée, champ de vision contenu. À tester tôt sur de vrais passagers, pas au bureau. |
| **Batterie et chauffe du téléphone** | Moyenne | 30 FPS verrouillés, échelle de rendu basse, veille dès l'arrêt. Mesurer la consommation réelle sur un trajet d'une heure en phase 3. |
| **Le procédural ne « ressemble » pas assez à la vraie route** | Moyenne | Assumé au §4. Si la reconnaissance littérale devient un besoin, rouvrir la question du réseau — et donc du sans-serveur. |
| **Le navigateur Tesla est plus limité que ne le disent les sources** | Moyenne | Le portage est isolé en phase 5 : si la voiture déçoit, le produit téléphone tient debout tout seul. |
| **Régression de la synthèse audio à l'extraction** | Faible | Tests de non-régression ; le legacy reste jouable côte à côte pour comparer à l'oreille. |

---

## 10. Ce que je ferais en premier

1. **La phase 1, cette semaine.** Une page de mesure sur ton téléphone, un trajet, trois
   traces enregistrées. Cinq chiffres.
2. **Extraire la synthèse moteur** en parallèle — travail sûr, indépendant du reste, et c'est
   la pièce irremplaçable du projet.
3. **Puis la fusion de capteurs**, avec ses tests sur les traces, avant la moindre ligne de rendu.
4. **Ensuite seulement**, le corridor et la lumière.

Le plan tient en ~10 semaines. Il est nettement moins risqué que les révisions précédentes :
aucune physique, aucun serveur, aucune modération, une boucle de test qui tient dans une poche.
La seule vraie inconnue est le comportement des capteurs en voiture — et elle se lève en deux
jours.

---

## Annexe — sources consultées

- Navigateur Tesla : [passage à Chromium](https://www.teslarati.com/tesla-chromium-in-car-web-browser/) · [Geolocation API](https://forums.tesla.com/forum/forums/web-browser-59-supports-geolocation-api) · [bug de permission connu](https://github.com/Leaflet/Leaflet/issues/7157) · [optimiser un site pour le navigateur Tesla](https://webhostingbuddy.com/blog/5-tips-to-optimize-your-website-for-teslas-in-car-web-browser-model-s-3-x-y/) · [mise à jour été 2026](https://codriver.io/guides/tesla-browser-summer-2026-update)
- Matériel : [MCU2 vs MCU3](https://www.notateslaapp.com/news/2417/tesla-intel-atom-mcu-2-and-amd-ryzen-mcu-3-feature-differences-and-how-to-tell-what-you-have) · [APU Ryzen et autonomie](https://insideevs.com/news/588007/tesla-showdown-old-intel-gpu-vs-new-amd-apu/) · [écran Model Y 2026](https://tslablog.com/vehicles/2026-model-y-juniper-premium-awd/)
- three.js : [état 2026](https://www.utsubo.com/blog/threejs-2026-what-changed) · [100 astuces de performance](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
- Cadre : [article R412-6-2 — Légifrance](https://www.legifrance.gouv.fr/codes/article_lc/LEGIARTI000025111520)
