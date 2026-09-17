# Cahier des charges — Application de suivi personnel (nom provisoire : "Suivi")

## 1. Contexte et objectif du projet

Application mobile personnelle de suivi de vie, organisée en **blocs
modulaires** (sommeil, habitudes, poids en V1, extensible ensuite). L'objectif
n'est pas de dupliquer des trackers existants, mais de combiner plusieurs
domaines dans un seul outil léger, sans publicité, sans abonnement, avec les
données stockées **100% en local** et récupérables intégralement à tout moment.

Le projet naît d'un constat : les apps de suivi combinées existantes qui font de
la mise en relation entre données (sommeil, humeur, habitudes...) sont payantes
par abonnement. L'objectif ici est d'obtenir un outil équivalent, sur mesure,
sans coût récurrent.

## 2. Utilisateur cible

Un seul utilisateur (usage personnel), pas de gestion multi-compte, pas de
synchronisation cloud obligatoire. L'app doit rester rapide à utiliser au
quotidien (saisie en quelques secondes par bloc).

## 3. Principes de conception

Ces principes s'appliquent à l'ensemble de l'app et priment sur toute décision
de détail :

- **Sans publicité, sans abonnement** : aucune fonctionnalité ne doit dépendre
  d'un service payant tiers.
- **Local-first** : aucune donnée n'est envoyée sur un serveur externe ; tout
  est stocké sur l'appareil.
- **Saisie minimale** : chaque entrée manuelle doit pouvoir se faire en quelques
  taps, jamais de formulaire long à remplir.
- **Modularité** : chaque domaine de suivi (sommeil, habitudes, poids, futurs
  blocs) est un module indépendant partageant la même structure de page, pour
  permettre d'en ajouter facilement sans réécrire l'architecture.
- **Récupérabilité totale** : l'utilisateur doit pouvoir exporter puis
  réimporter 100% de ses données à tout moment (changement de téléphone,
  sauvegarde manuelle, etc.), prévoir a chaque nouvelle version la possibilité
  d'importer les données d'anciennes versions.
- **Palette de couleurs unique** : toute l'interface s'appuie sur une palette de
  couleurs définie à l'avance ; aucune couleur en dehors de cette palette n'est
  utilisée où que ce soit dans l'app, y compris dans les graphiques (les codes
  couleur fonctionnels, comme le rouge-vert du bloc sommeil, doivent eux aussi
  être choisis dans cette palette et non ajoutés à part).

## 4. Périmètre du MVP

### 4.1 Inclus dans le MVP

- Écran de démarrage (splash screen) avec message de bienvenue personnalisé
- Bloc Sommeil (saisie, calculs, graphique calendrier, statistiques)
- Bloc Habitudes (CRUD des habitudes, checklist, calendrier par habitude,
  streaks)
- Bloc Poids (saisie, graphique d'évolution, objectif de poids avec barre de
  progression)
- Export / import complet des données (JSON)
- Page d'accueil listant les blocs

### 4.2 Exclus du MVP (pistes V2, voir section 13)

- Bloc Humeur / énergie
- Bloc corrélations entre blocs
- Bloc Sport/musculation détaillé (charges, volume, records)
- Toute fonctionnalité de nutrition, finances ou temps d'écran

## 5. Architecture générale de l'application

### 5.1 Stack technique

| Élément        | Choix                                                                                            | Justification                                                                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Framework      | Flutter (Dart)                                                                                   | Un seul code iOS/Android, UI 100% custom sans limite de widgets natifs, nécessaire pour le calendrier heatmap et les barres de progression personnalisées |
| Stockage local | SQLite (via `drift` ou `sqflite`)                                                                | Permet les requêtes de calcul (moyennes, filtres par date) directement en base, plus fiable qu'un JSON brut pour ce volume de requêtes                    |
| Export/Import  | Sérialisation de toutes les tables vers un fichier `.json` unique, et lecture inverse            | Répond à l'exigence de récupération à 100% des données                                                                                                    |
| Graphiques     | Package `fl_chart` (ou équivalent) pour le linéaire, composant custom pour le calendrier heatmap | Pas de solution standard pour le calendrier coloré, à développer sur mesure                                                                               |

### 5.2 Modèle de page générique (composant `BlockPage`)

Tous les blocs avec graphique + infos + saisie manuelle partagent **un seul
composant réutilisable**, paramétré par bloc, plutôt que trois pages codées
séparément. Structure fixe de la page :

1. Graphique en grand, en haut de la page.
2. En dessous, sur la même page (sans scroll pour accéder aux infos
   principales), des blocs d'information organisés par 2 (grille 2 colonnes).
3. Bouton d'ajout flottant en bas à droite, ouvrant le formulaire de saisie
   manuelle du bloc.

Ce composant prend en paramètres : le type de graphique, la liste des
statistiques à afficher, et le formulaire de saisie associé. Toute évolution
future (nouveau bloc) doit réutiliser ce composant plutôt qu'en créer un
nouveau.

### 5.3 Stockage des données

Toutes les données sont stockées dans une base SQLite locale à l'appareil.
Aucune donnée ne transite par un serveur. La base est accessible en
lecture/écriture uniquement par l'app.

### 5.4 Charte graphique

L'app repose sur une palette de couleurs unique, définie avant le début du
développement de l'interface (couleurs primaires, couleurs neutres/fond, et
couleurs fonctionnelles utilisées pour les codes couleur comme celui du bloc
sommeil). Aucun écran, composant ou graphique ne doit introduire de couleur hors
de cette palette. Cette palette doit être arrêtée en amont de la phase 7 du plan
de développement (section 10).

## 6. Spécifications fonctionnelles détaillées

### 6.1 Écran de démarrage (splash screen)

Au lancement de l'app, un écran de démarrage s'affiche brièvement avant
d'arriver sur la page d'accueil, avec le texte "Bonjour" en gros, suivi à la
ligne du prénom (ou nom) de l'utilisateur. Ce nom est demandé lors du tout
premier lancement de l'app, via un champ de saisie simple affiché une seule fois
(onboarding minimal), sauf si l'utilisateur choisit d'importer une sauvegarde
JSON dès ce premier lancement — auquel cas le nom est récupéré directement
depuis les données importées plutôt que redemandé. Une fois saisi ou importé, le
nom est stocké en base de données comme les autres informations de l'app (table
`account`), ce qui signifie qu'il est lui aussi inclus dans les exports/imports
et ne sera donc jamais reperdu, même en cas de changement d'appareil.

Point ouvert : durée d'affichage de l'écran (fixe, ex. 1-2 secondes, ou
disparition dès que les données sont chargées) et possibilité ou non de modifier
le nom après coup depuis les paramètres (voir section 12).

### 6.2 Bloc Sommeil

**Saisie manuelle** (bouton + en bas à droite) :

- Heure de coucher
- Heure d'endormissement
- Heure de réveil
- Ressenti au réveil (échelle 1 à 5)

**Calculs automatiques :**

- Durée de sommeil = heure de réveil − heure d'endormissement
- Log rattaché au jour concerné, modifiable a posteriori

**Affichage / résultats :**

- Graphique type calendrier : une case par jour, colorée selon la durée de
  sommeil :
  - Rouge : moins de 4h
  - Orange : moins de 6h
  - Jaune : moins de 7h
  - Vert clair : moins de 8h
  - Vert : 8h ou plus
- Statistiques affichées sous le graphique, calculées sur la période
  sélectionnée :
  - Heure de coucher moyenne
  - Temps d'endormissement moyen
  - Heure d'endormissement moyenne
  - Heure de réveil moyenne
  - Ressenti moyen
  - Plus bas de la période (nuit la plus courte)

**Filtres de période :**

- Réglages prédéfinis sans saisie requise : 1 mois / 6 mois / 1 an / tout
- Possibilité de sélectionner une date de départ personnalisée ; la durée est
  alors calculée automatiquement jusqu'au jour même par défaut

**Règles de gestion :**

- Une entrée par jour maximum ; une nouvelle saisie sur un jour déjà rempli
  modifie l'entrée existante plutôt que d'en créer une seconde.
- Cas d'un coucher après minuit : le jour de rattachement du log doit être
  défini clairement (voir point ouvert en section 12).

### 6.3 Bloc Habitudes

**Gestion des habitudes :**

- Ajout, suppression et modification libres d'habitudes (pas de liste figée)
- Exemples : travail mobilité, travail grip

**Saisie manuelle :**

- Liste à cocher, une checklist par jour, pour chaque habitude active

**Affichage / résultats :**

- Pour chaque habitude, un bouton "voir" ouvre un calendrier qui colore les
  jours où l'habitude a été cochée
- Filtres de période : 1 mois / 6 mois / 1 an / tout
- La streak (série en cours) de chaque habitude est affichée directement dans le
  menu principal, sans avoir à ouvrir le calendrier

**Règles de gestion :**

- La suppression d'une habitude doit préserver l'historique déjà enregistré
  (désactivation plutôt que suppression physique — voir point ouvert en section
  12).

### 6.4 Bloc Poids

**Saisie manuelle :**

- Poids (kg)
- Date de pesée (automatique au jour même, modifiable)

**Affichage / résultats :**

- Graphique linéaire de l'évolution du poids, filtrable : 1 mois / 6 mois / 1 an
  / tout
- Calcul de la différence entre le poids de départ et le poids d'arrivée sur la
  période sélectionnée
- Poids actuel affiché au-dessus du graphique
- Barre de progression vers l'objectif, affichant :
  - Poids restant par rapport à l'objectif
  - Progression en %
  - Poids de départ (au moment où l'objectif a été fixé)
  - Poids objectif
- Si aucun objectif n'est défini : bouton pleine largeur "Ajouter un objectif de
  poids" à la place de la barre
- Fonctionne aussi bien pour une prise de poids que pour une perte de poids
  (calcul de progression bidirectionnel)

## 7. Modèle de données (schéma simplifié)

```
account
  name

sleep_entries
  id, date, bedtime, sleep_time, wake_time, feeling (1-5)

habits
  id, name, active (bool), creation_date

habit_logs
  id, habit_id, date, checked (bool)

weight_entries
  id, date, weight_kg

weight_goal
  target_weight, start_date, starting_weight
```

## 8. Exigences non-fonctionnelles

- **Confidentialité** : aucune donnée envoyée hors de l'appareil, aucun compte
  requis.
- **Fonctionnement hors ligne** : l'app doit être 100% utilisable sans connexion
  internet.
- **Plateformes cibles** : iOS et Android via le même code Flutter.
- **Performance** : ouverture d'un bloc et calcul des statistiques en moins
  d'une seconde même avec plusieurs années de données.
- **Cohérence visuelle** : respect strict de la palette de couleurs définie
  (voir section 5.4) sur l'ensemble des écrans.

## 9. Import / Export

- Accessible depuis les paramètres de l'app.
- Export : génère un fichier `.json` unique contenant l'intégralité des tables
  (account, sleep, habits, logs, weight, goal).
- Import : lit un fichier `.json` conforme au même format et restaure les
  données, avec une vérification de structure avant d'écraser les données
  existantes (confirmation demandée à l'utilisateur).

## 10. Plan de développement (phases)

1. Squelette du projet (navigation, page d'accueil listant les blocs)
2. Composant générique `BlockPage`
3. Écran de démarrage (splash screen) et onboarding du nom
4. Bloc Sommeil
5. Bloc Habitudes
6. Bloc Poids
7. Import / export
8. Finitions (cas limites, cohérence visuelle, application définitive de la
   palette de couleurs)

## 11. Critères d'acceptation (Definition of Done) par bloc

**Écran de démarrage** : au premier lancement, le nom est demandé ou récupéré
depuis un import ; aux lancements suivants, "Bonjour [nom]" s'affiche sans
redemander le nom.

**Sommeil** : une saisie complète crée/modifie correctement l'entrée du jour ;
le calendrier reflète le bon code couleur pour chaque durée ; les statistiques
se recalculent correctement lors du changement de filtre de période.

**Habitudes** : une habitude peut être ajoutée, cochée, décochée, et supprimée
sans perte de l'historique déjà enregistré ; la streak affichée correspond bien
au nombre de jours consécutifs cochés jusqu'à aujourd'hui (ou jusqu'au dernier
jour coché).

**Poids** : le graphique reflète l'historique complet ; la barre de progression
fonctionne correctement à la fois en scénario de perte et de prise de poids ;
l'absence d'objectif affiche bien le bouton d'ajout au lieu de la barre.

**Import/export** : un export suivi d'un import sur une base vide restaure 100%
des données à l'identique, y compris le nom de l'utilisateur.

**Charte graphique** : aucune couleur utilisée dans l'app, tous écrans
confondus, n'est absente de la palette définie en section 5.4.

## 12. Points ouverts à trancher avant développement

- **Sommeil à cheval sur minuit** : sur quel jour rattacher l'entrée si le
  coucher a lieu après minuit — jour du coucher ou jour du réveil ?
- **Suppression d'une habitude** : désactivation (historique conservé mais
  habitude non affichée) ou suppression totale avec ses logs ?
- **Fuseau horaire** : comportement si l'utilisateur change de fuseau horaire
  (voyage) — les dates doivent-elles rester en heure locale au moment de la
  saisie ?
- **Unité de poids** : kg uniquement en V1, ou choix kg/lb dans les paramètres
  dès le départ ?
- **Écran de démarrage** : durée d'affichage fixe ou disparition dès chargement
  des données ; possibilité de modifier le nom plus tard depuis les paramètres ?
- **Palette de couleurs** : liste exacte des couleurs (primaires, neutres,
  fonctionnelles) à arrêter avant la phase 8 du plan de développement.

## 13. Évolutions futures (hors MVP)

- Bloc Humeur / énergie (saisie 1-5, calendrier combiné avec le sommeil)
- Bloc Corrélations : mise en relation automatique entre les blocs (ex. lien
  entre nuits courtes et habitudes ratées), sans graphique complexe — sous forme
  de phrases générées
- Bloc Sport/musculation détaillé (charges, volumes, records personnels par
  exercice)
