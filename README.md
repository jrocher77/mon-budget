# Mon Budget — Documentation technique exhaustive

> Application web de gestion budgétaire personnelle — PWA mono-fichier `index.html`

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Architecture technique](#2-architecture-technique)
3. [Authentification & sessions](#3-authentification--sessions)
4. [Persistance des données (Firestore)](#4-persistance-des-données-firestore)
5. [Gestion des comptes](#5-gestion-des-comptes)
6. [Transactions](#6-transactions)
7. [Filtres de la vue Historique](#7-filtres-de-la-vue-historique)
8. [Templates (opérations récurrentes)](#8-templates-opérations-récurrentes)
9. [Gestion des cartes à débit différé](#9-gestion-des-cartes-à-débit-différé)
10. [Compte Avantages (Ticket Restaurant & assimilés)](#10-compte-avantages-ticket-restaurant--assimilés)
11. [Batch de reconstitution des transactions Avantages](#11-batch-de-reconstitution-des-transactions-avantages)
12. [Nettoyage automatique](#12-nettoyage-automatique)
13. [Snapshots épargne](#13-snapshots-épargne)
14. [Tableau de bord (Dashboard)](#14-tableau-de-bord-dashboard)
15. [Vue Statistiques épargne](#15-vue-statistiques-épargne)
16. [Paramètres](#16-paramètres)
17. [Mode démo](#17-mode-démo)
18. [Console de debug](#18-console-de-debug)
19. [Thème clair / sombre](#19-thème-clair--sombre)
20. [Gestion de la connexion lente](#20-gestion-de-la-connexion-lente)
21. [Schéma des données Firestore](#21-schéma-des-données-firestore)

---

## 1. Vue d'ensemble

**Mon Budget** est une application budgétaire personnelle accessible depuis n'importe quel navigateur (desktop, mobile, PWA installable). Elle est entièrement contenue dans un seul fichier `index.html` — aucun build system, aucun serveur applicatif.

Caractéristiques principales :

- **Multi-comptes** : comptes courants, épargne (disponible ou bloquée), avantages (TR…).
- **Persistance temps-réel** sur Firebase Firestore avec sauvegarde automatique debounced (1 s).
- **Templates mensuels** par parité (mois pair / impair) pour pré-remplir les opérations récurrentes.
- **Cartes à débit différé** : les dépenses carte sont conservées jusqu'à leur date de prélèvement effective.
- **Compte Avantages** avec gestion des dotations mensuelles, du plafond journalier, du solde reporté et d'un batch de reconstitution automatique.
- **Nettoyage automatique** des anciennes transactions avec vérifications de sécurité avant exécution.
- **Snapshots épargne** automatiques sur des jours configurables pour historiser l'évolution du patrimoine.
- **PWA** : manifest embarqué, icône, thème, mode plein écran sur iOS/Android.

---

## 2. Architecture technique

| Couche | Technologie |
|---|---|
| UI | React 18 (UMD, sans build) via Babel Standalone |
| Styles | CSS-in-JS inline + variables CSS (thème clair/sombre) |
| Auth | Firebase Authentication (email/mot de passe) |
| Base de données | Firebase Firestore (document unique par utilisateur) |
| Polices | DM Serif Display + DM Sans (Google Fonts) |
| Hébergement | Tout hébergeur statique (GitHub Pages, Netlify, Firebase Hosting…) |

L'application démarre ainsi :

1. Le module Firebase est chargé en `type="module"` et expose `window._fb` (auth, db, doc, setDoc, onSnapshot…).
2. Une `Promise` (`fbReadyPromise`) est résolue dès que `window._fb` est disponible.
3. React est chargé en parallèle via CDN (Cloudflare).
4. Une fois les deux prêts, `ReactDOM.createRoot` monte le composant racine `<App />`.

---

## 3. Authentification & sessions

### Connexion
L'écran de connexion accepte un e-mail et un mot de passe via `signInWithEmailAndPassword`. Les erreurs Firebase sont simplifiées en un message générique côté UI.

### Session & inactivité
- La durée maximale d'inactivité est de **8 heures** pour un compte normal et **15 minutes** pour le mode démo.
- L'activité est détectée sur les événements `mousedown`, `keydown`, `touchstart`, `scroll`, `click` et enregistrée dans `localStorage` (`budget_last_activity`).
- À chaque reprise de focus (onglet visible, `visibilitychange`, `focus`, `pageshow`), la durée écoulée est vérifiée. Si elle dépasse la limite, `signOut` est appelé automatiquement.
- Sur iOS Safari, l'événement `pageshow` avec `e.persisted = true` (restauration depuis le bfcache) déclenche également ce contrôle, ce qui évite les sessions zombies après mise en veille longue.

### Déconnexion
Bouton dans les Réglages → `signOut(auth)` + suppression de la clé `budget_last_activity`.

---

## 4. Persistance des données (Firestore)

### Principe
Chaque utilisateur possède un **document unique** dans la collection `budgets`, identifié par son UID Firebase : `budgets/{uid}`.

### Abonnement temps-réel
`onSnapshot` maintient un flux temps-réel. À chaque snapshot entrant :

1. **Filtre d'écho** : si le snapshot arrive moins de 8 secondes après la dernière sauvegarde locale, il est ignoré pour éviter les boucles de mise à jour.
2. **Snapshot non-null** : les données sont chargées dans le state React.
3. **Snapshot null** (document inexistant) avant toute réception de données réelles : un **timer de grâce de 3 secondes** est déclenché. S'il expire sans que de vraies données arrivent, le compte est considéré comme nouveau et initialisé (données de démonstration pour le compte démo, données d'exemple pour un vrai compte). Ce mécanisme protège contre les snapshots `null` du cache offline Firebase sur iOS, qui peuvent précéder les données serveur de quelques millisecondes.
4. **Snapshot null post-chargement** : ignoré silencieusement pour ne jamais écraser des données existantes.

### Sauvegarde debounced
Toute modification du state (transactions, comptes, templates, paramètres, épargne…) déclenche un timer debounce de **1 seconde**. Au terme de ce délai, `setDoc(..., { merge: true })` est appelé avec l'état complet nettoyé des valeurs `undefined`.

Un indicateur visuel d'erreur de sauvegarde s'affiche si Firestore retourne une erreur (réseau coupé, quota…). Il disparaît automatiquement après 8 secondes ou sur clic.

### Garde de sécurité (`trustedRef`)
La sauvegarde est **bloquée** tant que `trustedRef` est `false`. Ce flag ne passe à `true` que lorsqu'un snapshot non-null a été reçu **ou** que le timer de grâce a confirmé un nouveau compte. Cela empêche d'écraser des données server-side avec des données par défaut lors d'un rechargement à froid sur iOS PWA.

---

## 5. Gestion des comptes

### Types de comptes

| Type | Valeur interne | Description |
|---|---|---|
| Compte courant | `checking` | Compte principal avec suivi des transactions, templates, cartes. |
| Épargne disponible | `savings` + `savingsSubtype: "available"` | Livret, compte épargne — solde saisi manuellement, inclus dans les snapshots. |
| Épargne bloquée | `savings` + `savingsSubtype: "blocked"` | PEL, assurance-vie — idem mais catégorisé séparément dans les statistiques. |
| Avantages | `benefit` | Compte dédié Ticket Restaurant ou avantage similaire, sans transactions classiques. |

### Capacité
Maximum **15 comptes** au total.

### Attributs d'un compte courant
- `name` : libellé affiché.
- `cardName` (optionnel) : nom de la carte associée (active le mode "carte à débit différé").
- `balance` : non utilisé pour les comptes courants (calculé dynamiquement depuis les transactions).

### Attributs d'un compte épargne
- `balance` : solde saisi manuellement via un champ inline cliquable dans le Dashboard.

### Attributs d'un compte Avantages
- `dailyCap` : plafond journalier d'utilisation (ex. 19 €/jour pour un TR).
- `overflowAccountId` : identifiant du compte courant vers lequel l'excédent doit être imputé.

### Réordonnancement
Dans les Réglages, en mode édition, des flèches ↑/↓ permettent de réordonner les comptes au sein de chaque groupe de type. L'ordre est conservé en Firestore.

### Suppression
La suppression d'un compte efface aussi toutes ses transactions, son template et ses données Avantages. Un `deleteField()` Firestore est envoyé explicitement pour les champs imbriqués (templates, benefitData) que `merge:true` ne supprimerait pas.

---

## 6. Transactions

### Structure d'une transaction

```
{
  id:          UUID (généré côté client)
  title:       string
  amount:      number (positif)
  type:        "income" | "expense"
  date:        "YYYY-MM-DD"
  accountId:   string
  pointed:     boolean
  isCard:      boolean (optionnel — dépense carte à débit différé)
  note:        string (optionnel)
  _tpl:        "YYYY-MM" (optionnel — marqueur d'application de template)
  _virementId: UUID (optionnel — relie les deux côtés d'un virement)
  _transferFrom: "YYYY-MM" (optionnel — report de solde du mois précédent)
  _checked:    boolean (optionnel — utilisé par le batch Avantages)
}
```

### Modes de saisie (FormModal)

**Dépense / Revenu**
- Libellé, montant, date, note.
- Sélection du compte parmi les comptes courants.
- Option « Dépense carte » si le compte possède un `cardName`.

**Virement inter-comptes**
- Crée simultanément deux transactions liées par un `_virementId` commun : une dépense sur le compte source et un revenu sur le compte destination.

**Ajustement de montant (édition)**
- Lors de l'édition d'une transaction existante, un champ d'ajustement permet de taper `+X` ou `-X` pour modifier le montant de façon incrémentale sans ressaisir la valeur complète.

### Solde prévisionnel
Lors de la saisie ou de l'édition, l'interface affiche en temps-réel le solde prévisionnel du compte pour le mois sélectionné, en tenant compte de la nouvelle transaction. En mode virement, les soldes prévisionnels des deux comptes sont affichés côte à côte.

### Pointage
Chaque transaction peut être pointée (rapprochée) d'un tap sur l'indicateur coloré à gauche. Une transaction pointée est considérée comme vérifiée vis-à-vis du relevé bancaire. Le pointage est un critère de blocage du nettoyage automatique (voir §12).

### Report de solde
Le bouton "Reporter le solde" dans le Dashboard crée une transaction spéciale `_transferFrom: prevMonthKey` sur le compte courant sélectionné, correspondant au solde net du mois précédent. Cette transaction est visible et deletable comme n'importe quelle autre.

---

## 7. Filtres de la vue Historique

La vue **Historique** (onglet Transactions) offre quatre niveaux de filtrage combinables :

### 7.1 Filtre par type
Chips cliquables mutuellement exclusives :

| Chip | Règle appliquée |
|---|---|
| **Tout** | Toutes les transactions (sauf les comptes Avantages — exclues systématiquement). |
| **Dépenses** | `type === "expense"` **et** `isCard !== true`. |
| **Revenus** | `type === "income"`. |
| **💳 Carte** | `isCard === true`. S'affiche uniquement si au moins un compte possède un `cardName`. |

> Les transactions des comptes de type `benefit` ne sont jamais affichées dans l'historique (elles sont gérées dans leur propre vue dédiée).

### 7.2 Filtre par compte
Affiché uniquement si l'utilisateur possède **plus d'un compte courant**. Un `<select>` permet de choisir :
- "📊 Tous les comptes" (valeur `all`)
- Chaque compte courant individuellement

Quand un compte est sélectionné, la bordure du select passe en doré pour indiquer l'état actif.

### 7.3 Recherche textuelle
La barre de recherche effectue une recherche case-insensitive sur :
- Le **titre** (`title`)
- La **note** (`note`)
- Le **montant** — en valeur brute et en format `fr-FR` (ex. `"17,99 €"`)
- La **date** — en format ISO (`"2025-03-15"`) et en format français (`"15/03/2025"`)

Quand une recherche est active, le filtre "Transactions futures masquées" est contourné (toutes les dates sont affichées).

### 7.4 Masquage des transactions futures
Contrôlable via le toggle **"Transactions futures dans l'historique"** (Réglages → Apparence). Quand il est désactivé, les transactions dont la date est **strictement postérieure à aujourd'hui** sont masquées. Ce filtre est inactif dès qu'une recherche textuelle est en cours.

### 7.5 Groupement et tri
Les transactions filtrées sont regroupées **par date** (une section par jour), triées du plus récent au plus ancien.

---

## 8. Templates (opérations récurrentes)

### Concept
Les templates permettent de pré-remplir automatiquement les opérations récurrentes d'un mois (loyer, salaire, abonnements…) en un seul clic, sans les ressaisir chaque mois.

### Parité pair / impair
Chaque compte courant possède **deux templates** : un pour les mois pairs et un pour les mois impairs. Cela permet de gérer les variations mensuelles (ex. primes trimestrielles, loyer variable…).

La parité d'un mois est déterminée par : `parseInt(monthKey.split("-")[1], 10) % 2 === 0`.

### Structure d'un item de template

```
{
  id:     UUID
  type:   "income" | "expense"
  title:  string (peut contenir des variables)
  amount: number
  day:    number (1 à 28 — jour du mois)
  note:   string (optionnel, peut contenir des variables)
}
```

### Variables de template
Les libellés et notes des items de template peuvent contenir des variables qui sont résolues au moment de l'application :

| Variable | Résultat exemple (février 2025) |
|---|---|
| `$month` | `février` (minuscules) |
| `$Month` | `Février` (première lettre majuscule) |
| `$mm` | `02` |
| `$year` ou `$yyyy` | `2025` |
| `$Year` | `2025` |

Exemple : `"Loyer $Month $year"` → `"Loyer Février 2025"`.

### Application d'un template
Depuis le Dashboard, le bouton "Appliquer le template" génère toutes les transactions du template correspondant à la parité du mois sélectionné. Une protection anti-doublon vérifie si des transactions marquées `_tpl: monthKey` existent déjà pour ce compte et ce mois — si c'est le cas, le template n'est pas réappliqué.

### Gestion dans les Réglages
- Sélection du compte cible et de la parité (pair/impair).
- Affichage du total revenus et dépenses du template courant.
- Ajout, modification et suppression des items via une modale dédiée.
- Le jour du mois est limité à 28 pour éviter les dates invalides en février.

---

## 9. Gestion des cartes à débit différé

### Principe
Certaines cartes bancaires (ex. Visa Premier) débitent les dépenses en **différé** : les achats du mois M sont prélevés en une seule fois le mois M+1. Mon Budget gère cela en permettant d'associer une **date de prélèvement** à un mois donné sur un compte.

### Configuration
Dans les Réglages → section du compte courant, si `cardName` est renseigné, une zone apparaît pour saisir la date de prélèvement mensuel par mois clé (`YYYY-MM`).

La structure de stockage est :
```
cardDebitDates: {
  [accountId]: {
    [monthKey: "YYYY-MM"]: "YYYY-MM-DD"  // date effective du débit
  }
}
```

### Impact sur les transactions
Une transaction avec `isCard: true` appartenant au mois M **n'est pas supprimée** lors d'un nettoyage si sa date de prélèvement (`cardDebitDates[accountId][monthKey]`) tombe dans le mois courant ou un mois futur.

### Affichage dans le Dashboard (CardPanel)
Le Dashboard affiche un panneau dédié aux dépenses carte du mois sélectionné :
- Liste des transactions carte avec leur date d'achat.
- Total des dépenses carte du mois.
- Indication de la date de prélèvement configurée et si elle tombe le mois suivant.
- Possibilité de modifier ou supprimer chaque dépense carte depuis ce panneau.

### SynthCardPanel (vue condensée)
Quand plusieurs mois ont des dépenses carte avec une date de prélèvement commune, le Dashboard peut afficher une vue synthétique regroupant les dépenses par mois source.

---

## 10. Compte Avantages (Ticket Restaurant & assimilés)

### Concept
Un compte de type `benefit` modélise un avantage en nature (Ticket Restaurant, chèques-déjeuner…). Il possède :
- Une **dotation mensuelle** : montant alloué chaque mois par l'employeur.
- Des **dépenses** : achats effectués avec ce moyen de paiement.
- Un **solde reporté** calculé automatiquement depuis l'historique des mois précédents.

### Structure des données

```
benefitData: {
  [accountId]: {
    allotments: {
      [monthKey: "YYYY-MM"]: {
        mode:      "amount" | "units"
        amount?:   number   // mode "amount" : montant direct
        count?:    number   // mode "units"  : nombre de titres
        unitValue?: number  // mode "units"  : valeur unitaire
        _isCarryOver?: boolean  // interne — report de solde après nettoyage
      }
    }
    expenses: [
      {
        id:       UUID
        title:    string
        amount:   number
        date:     "YYYY-MM-DD"
        _checked: boolean  // interne — marqueur batch reconstitution
      }
    ]
  }
}
```

### Modes de dotation
**Mode montant** (`mode: "amount"`) : on saisit directement le montant mensuel alloué.

**Mode unités** (`mode: "units"`) : on saisit le nombre de titres et la valeur unitaire. Le montant est calculé comme `count × unitValue`. Ce mode est utile pour les Tickets Restaurant dont le nombre varie selon les jours travaillés.

### Calcul du solde reporté (`computeCarryOver`)
La fonction `computeCarryOver(benefitAcct, upToMonthKey)` calcule le solde cumulé disponible avant un mois donné :

1. Collecte tous les mois ayant une dotation ou une dépense, **strictement antérieurs** à `upToMonthKey`.
2. Pour chaque mois passé (trié chronologiquement) :
   - `carry = max(0, carry + dotation_du_mois - dépenses_du_mois)`
3. Le résultat est le solde disponible en entrée du mois cible.

Le solde ne peut jamais être négatif (plancher à 0 — on ne peut pas dépenser plus que disponible).

### Interface BenefitPage
La page Avantages (accessible depuis le Dashboard en cliquant sur un compte benefit) propose :
- Navigation par onglets mois (mois courant + mois passés ayant des données).
- Affichage du solde reporté depuis les mois précédents.
- Affichage de la dotation du mois sélectionné (éditable).
- Affichage du solde disponible = report + dotation - dépenses du mois.
- Liste des dépenses avec ajout, modification et suppression.
- Bouton de navigation vers le mois suivant.

---

## 11. Batch de reconstitution des transactions Avantages

### Objectif
Ce batch s'exécute **une seule fois après le chargement initial des données Firestore** (`useEffect` déclenché sur `[ready]`). Son but est de réconcilier des dépenses Avantages importées sans marqueur `_checked` (injectées par une Firebase Function externe ou importées manuellement) avec le solde disponible et le plafond journalier.

### Conditions d'exécution
Le batch s'exécute uniquement si :
1. Les données Firestore sont chargées (`ready === true`).
2. `trustedRef` est `true` (données confirmées comme réelles).
3. Au moins un compte de type `benefit` possède `dailyCap > 0` **et** `overflowAccountId` renseigné.
4. Ce compte possède des dépenses sans marqueur `_checked`.

### Algorithme de reconstitution

Les dépenses non vérifiées (`!e._checked`) sont traitées dans **l'ordre chronologique** (oldest-first) :

**Pour chaque dépense :**

1. **Calcul du solde courant** (`runSolde`) :
   - Reconstruit le solde disponible à la date de la dépense en utilisant uniquement les dépenses déjà marquées `_checked` et les dotations.
   - Calcule le report cumulé depuis tous les mois antérieurs au mois de la dépense.
   - Ajoute la dotation du mois courant.
   - Soustrait les dépenses `_checked` du même mois et de la même date ou antérieures.

2. **Calcul du plafond journalier restant** (`capRestant`) :
   - `capRestant = max(0, dailyCap - somme des dépenses _checked du même jour)`

3. **Répartition TR / carte** :
   - `partTR = min(amount, capRestant, solde)` — portion prise en charge par le TR.
   - `partCarte = amount - partTR` — portion débordant sur le compte courant associé.
   - Arrondi à 2 décimales.

4. **Mise à jour de l'état** :
   - Si `partTR > 0` : la dépense est mise à jour avec `amount = partTR` et `_checked = true`.
   - Si `partTR === 0` : la dépense est supprimée de `benefitData` (100% carte).
   - Si `partCarte > 0` : une **nouvelle transaction** `isCard` est créée sur le compte `overflowAccountId` avec le montant overflow.

5. **Accumulation** : les dépenses nouvellement `_checked` sont ajoutées au tableau de référence pour les calculs des dépenses suivantes de la même journée.

### Résultat
Après le batch :
- Toutes les dépenses Avantages ont un `_checked: true`.
- Les débordements sont visibles comme des transactions ordinaires (ou carte) sur le compte courant associé.
- Le solde TR reste cohérent même sur plusieurs mois avec report.

---

## 12. Nettoyage automatique

### Objectif
Supprimer les anciennes transactions (mois antérieurs au mois courant) pour alléger l'interface et les données Firestore, tout en conservant les données encore utiles.

### Déclenchement automatique
Le nettoyage automatique s'exécute **à chaque chargement de l'application** si :
- `autoCleanup === true` (activé dans les Réglages).
- `trustedRef === true` (données Firestore confirmées).
- Le jour courant `>= cleanupDay` (jour de nettoyage configuré, défaut : le 10 du mois).

### Vérifications de sécurité (`cleanupBlockers`)
Avant d'exécuter le nettoyage, deux conditions bloquantes sont vérifiées :

**Condition 1 — Transactions non pointées**
Si des transactions de mois antérieurs ne sont pas pointées **et** ne sont pas des dépenses carte avec un prélèvement futur, le nettoyage est bloqué. Le message indique le nombre de transactions concernées.

**Condition 2 — Absence de report de solde**
Pour chaque compte courant ayant des transactions dans des mois antérieurs, un report de solde (transaction `_transferFrom`) doit exister. Si ce n'est pas le cas pour au moins un compte, le nettoyage est bloqué et les comptes concernés sont listés.

### Comportement en cas de blocage
- En automatique : un **toast** d'avertissement s'affiche en bas de l'écran pendant 12 secondes. Il n'est montré qu'**une fois par jour** (timestamp enregistré dans `localStorage`). L'utilisateur est invité à corriger les points signalés puis à relancer le nettoyage manuellement.
- En manuel (depuis les Réglages) : une modale de confirmation affiche le message de blocage. Le bouton de confirmation est désactivé.

### Ce qui est supprimé

**Transactions classiques :** toute transaction dont `mKey(date) < currentMonthKey`, sauf :
- Les transactions carte (`isCard: true`) dont la date de prélèvement tombe dans le mois courant ou au-delà.

**Données Avantages :** pour chaque compte benefit :
1. Calcul du solde cumulé sur tous les mois passés (report accumulé).
2. Suppression des dépenses antérieures au mois courant.
3. Suppression des dotations antérieures au mois courant.
4. Si le report est positif : injection d'une dotation synthétique `_isCarryOver: true` sur le mois précédent (`prevMonthKey`) pour que `computeCarryOver` puisse restituer le bon solde disponible après nettoyage.

### Nettoyage manuel
Dans les Réglages, le bouton "Nettoyer le mois" effectue les mêmes vérifications de sécurité. En cas de succès, une modale de confirmation affiche le nombre de transactions et entrées Avantages qui seront supprimées (avec précision sur les transactions carte différées éventuellement conservées).

---

## 13. Snapshots épargne

### Objectif
Photographier périodiquement le solde des comptes épargne pour constituer un historique et afficher l'évolution du patrimoine dans le temps.

### Structure d'un snapshot

```
{
  id:        UUID
  date:      "YYYY-MM-DD"
  available: number   // somme des comptes épargne disponibles
  blocked:   number   // somme des comptes épargne bloqués
  auto:      boolean  // true si créé automatiquement
  note:      string   // note libre
}
```

### Snapshot automatique
Un snapshot automatique est créé **une fois par app-load** si toutes ces conditions sont réunies :
1. `savingsSnapActive === true` (actif dans les Réglages).
2. `trustedRef === true`.
3. Le jour courant (`d = new Date().getDate()`) est dans la liste `savingsSnapDays` (défaut : `[1, 15]`).
4. Aucun snapshot n'existe déjà pour la date du jour.

Le snapshot capture :
- `available` : somme des `balance` des comptes `savings` avec `savingsSubtype !== "blocked"`.
- `blocked` : somme des `balance` des comptes `savings` avec `savingsSubtype === "blocked"`.

### Configuration
Dans les Réglages → section Épargne :
- Toggle pour activer/désactiver les snapshots automatiques.
- Choix des jours de prise de snapshot (multi-sélection de 1 à 31).

### Snapshot manuel
Dans la vue Statistiques épargne, un bouton "+" permet de créer un snapshot manuellement avec date, montants disponible/bloqué et note libres.

### Modification et suppression
Chaque snapshot est éditable et supprimable depuis la vue Statistiques épargne.

---

## 14. Tableau de bord (Dashboard)

### Navigation par mois
Un sélecteur de mois (onglets horizontaux défilants) affiche le mois courant en première position, plus les mois des 12 derniers mois ayant au moins une transaction. La navigation est limitée aux mois ayant des données ou au mois courant.

### Sélecteur de compte
En haut du Dashboard, un sélecteur permet de choisir le compte courant affiché. Il n'est visible que si l'utilisateur possède au moins 2 comptes courants.

### Indicateurs du mois sélectionné
- **Revenus du mois** : somme des transactions `income` du compte sélectionné pour le mois.
- **Dépenses du mois** : somme des transactions `expense` (hors carte si débit différé) du compte sélectionné.
- **Solde net** : revenus - dépenses.

### Transactions non pointées
Un bandeau d'alerte s'affiche si des transactions du mois sélectionné ne sont pas encore pointées, avec le nombre concerné.

### Application de template
Si un template est configuré pour le compte sélectionné et la parité du mois, un bouton "Appliquer le template" est affiché. Il est désactivé si le template a déjà été appliqué ce mois.

### Report de solde
Bouton "Reporter le solde" visible si le mois précédent a des transactions mais pas encore de report. Crée une transaction `_transferFrom`.

### Section Épargne
Affichée si l'utilisateur possède au moins un compte épargne. Chaque compte épargne est listé avec :
- Son solde (cliquable pour modification inline).
- Une mini-sparkline (graphique en ligne condensé) de l'évolution sur les dernières périodes.

Un bouton "→ Statistiques" navigue vers la vue détaillée des snapshots.

### Section Avantages (Ticket Restaurant)
Affichée pour chaque compte benefit. Montre le solde disponible du mois et un lien vers la page détaillée.

### CardPanel
Panneau des dépenses carte du mois sélectionné pour le compte courant actif (affiché si `cardName` est défini sur le compte).

---

## 15. Vue Statistiques épargne

Accessible depuis le Dashboard (bouton "→ Statistiques") ou en cliquant sur l'icône épargne dans le tableau de bord.

### Contenu
- **Graphique principal** (`SavingsChart`) : courbe interactive SVG affichant l'évolution de l'épargne disponible et bloquée dans le temps. Un tap/clic sur la courbe affiche une infobulle avec date, valeurs et variations.
- **Variation** : affichage coloré de la variation absolue et en pourcentage par rapport au snapshot précédent.
- **Mini-sparklines** : deux graphiques condensés (disponible / bloqué) en haut de la vue.
- **Liste des snapshots** : tableau de tous les snapshots, triés du plus récent au plus ancien, avec date formatée, montants, note et indicateur auto/manuel.

### Formatage des dates de snapshot
La fonction `fmtSnapshotDate` produit des formats lisibles : "1er mars", "15 avril", etc.

---

## 16. Paramètres

### Apparence
- Basculer entre thème sombre et clair.
- Activer/désactiver la console de debug.
- Activer/désactiver l'affichage des transactions futures dans l'Historique.

### Comptes bancaires
- Liste de tous les comptes (courants, épargne, avantages) avec possibilité de renommer, configurer, réordonner, supprimer.
- Bouton "Modifier" pour passer en mode édition (affiche les flèches de réordonnancement et les boutons de suppression).
- Bouton "Ajouter un compte" → modale de création.

### Configuration d'un compte courant (AccountRow)
- Renommer le compte.
- Saisir le nom de la carte associée (`cardName`) pour activer le mode débit différé.

### Configuration d'un compte Avantages (AccountRow)
- Renommer le compte.
- Saisir le plafond journalier (`dailyCap`).
- Sélectionner le compte de débordement (`overflowAccountId`).

### Templates
- Sélecteur du compte cible.
- Tabs pair/impair.
- Résumé revenus/dépenses du template.
- Ajout, édition, suppression d'items via `TplAddModal`.

### Nettoyage
- Sélecteur du jour de nettoyage automatique (1 à 28).
- Toggle d'activation du nettoyage automatique.
- Bouton de nettoyage manuel avec confirmation.

### Épargne
- Toggle d'activation des snapshots automatiques.
- Sélection des jours de snapshot (1 à 28).

### Déconnexion
Bouton de déconnexion en bas de page.

### Réinitialisation démo (mode démo uniquement)
Bouton "Réinitialiser les données démo" — recharge les données de démonstration dans Firestore et recharge la page.

---

## 17. Mode démo

### Accès
Sur l'écran de connexion, le bouton "Voir la démo" ouvre une **modale "Mot magique"** demandant un mot de passe secret. Le hash SHA-256 du mot est comparé à `DEMO_MAGIC_HASH` (stocké dans le code — le mot en clair n'y apparaît pas). Si correct, la connexion se fait automatiquement avec le compte `demo@monbudget.fr`.

### Caractéristiques
- Bandeau orange permanent signalant le mode démo et que les modifications sont visibles de tous.
- Session de 15 minutes (inactivité), contre 8h pour un vrai compte.
- Jeu de données de démonstration préchargé :
  - 2 comptes courants (dont un avec carte Visa Premier et dépenses carte).
  - 3 comptes épargne (Livret A, PEL, Assurance vie).
  - 1 compte Ticket Restaurant avec dotations et dépenses.
  - 25 transactions réparties sur le mois courant et le mois précédent.
  - Historique de snapshots épargne sur 2 mois.
  - Dates de prélèvement carte configurées.
- Bouton "Réinitialiser les données démo" dans les Réglages pour remettre le compte dans son état initial.

---

## 18. Console de debug

Activable via un toggle dans les Réglages → Apparence (recharge la page). Quand active, un panneau `DebugOverlay` s'affiche en bas de l'écran, au-dessus de la TabBar.

### Fonctionnalités
- Affiche les 120 derniers logs avec timestamp (heure:minute:seconde:ms).
- Coloration des lignes selon le type d'événement (erreur ❌, succès ✅, avertissement ⚠️, sauvegarde 💾, annulation 🛑…).
- Champ de filtre textuel en temps réel.
- Bouton de copie de tous les logs dans le presse-papiers.
- Bouton d'effacement des logs.
- Bouton de réduction/expansion du panneau.

### Événements tracés
- Initialisation Firestore et réception des snapshots.
- Timers de sauvegarde, échoes ignorés, blocages de persist.
- Nettoyage automatique (déclenchement ou blocage).
- Batch TR : nombre de transactions carte créées.
- Connexion lente.
- Erreurs Firestore.

---

## 19. Thème clair / sombre

L'application supporte deux thèmes complets définis comme variables CSS sur `:root[data-theme]`.

### Variables principales

| Variable | Rôle |
|---|---|
| `--bg` | Fond global |
| `--surface` / `--surface2` / `--surface3` | Surfaces de cartes |
| `--border` / `--border2` | Bordures |
| `--gold` / `--gold-light` / `--gold-dim` | Couleur accent (or) |
| `--green` / `--green-dim` | Revenus |
| `--red` / `--red-dim` | Dépenses |
| `--text` / `--text-muted` / `--text-sub` | Niveaux de texte |
| `--header-bg` | Dégradé d'en-tête |
| `--tab-bg` | Fond de la TabBar |
| `--shadow` | Ombres |

Le thème est persisté dans `localStorage` (`budget_theme`) et l'attribut `data-theme` est appliqué sur `<html>` pour activer les bonnes variables.

---

## 20. Gestion de la connexion lente

Si Firestore ne répond pas dans les **4 secondes** après l'initialisation, l'écran de chargement est remplacé par un écran dédié `SlowConnectionScreen` affichant :
- Une icône de signal réseau.
- Un message explicatif.
- Un bouton "Réessayer" qui incrémente un compteur `retryCount` pour relancer l'abonnement Firestore.
- Une note de rassurance sur la sécurité des données.

Dès qu'un snapshot Firestore arrive, l'écran de connexion lente est remplacé par l'application normalement.

---

## 21. Schéma des données Firestore

Document `budgets/{uid}` :

```
{
  transactions: [
    {
      id, title, amount, type, date, accountId,
      pointed, isCard?, note?, _tpl?, _virementId?, _transferFrom?
    }
  ],

  accounts: [
    {
      id, name, type, balance,
      savingsSubtype?,   // "available" | "blocked"
      cardName?,         // active débit différé
      dailyCap?,         // avantages : plafond journalier
      overflowAccountId? // avantages : compte de débordement
    }
  ],

  templates: {
    [accountId]: {
      even: { items: [{ id, type, title, amount, day, note? }] },
      odd:  { items: [{ id, type, title, amount, day, note? }] }
    }
  },

  cleanupDay:   number,   // défaut: 10
  autoCleanup:  boolean,  // défaut: true
  showFutureTx: boolean,  // défaut: true

  cardDebitDates: {
    [accountId]: {
      [monthKey: "YYYY-MM"]: "YYYY-MM-DD"
    }
  },

  benefitData: {
    [accountId]: {
      allotments: {
        [monthKey]: { mode, amount?, count?, unitValue?, _isCarryOver? }
      },
      expenses: [
        { id, title, amount, date, _checked? }
      ]
    }
  },

  savingsHistory: [
    { id, date, available, blocked, auto, note }
  ],

  savingsSnapDays:   number[],  // défaut: [1, 15]
  savingsSnapActive: boolean    // défaut: true
}
```

---

*Documentation générée depuis le code source de `index.html` — à maintenir à jour à chaque évolution significative du code.*
