# Mon Budget

> Application web & mobile de gestion budgétaire personnelle — v1.0 • Février 2026

---

## Fonctionnalités

| | Fonctionnalité | Description |
|---|---|---|
| 💳 | **Multi-comptes** | Gérez jusqu'à 15 comptes (courants et épargne) au même endroit |
| 📊 | **Tableau de bord** | Vue synthétique par mois avec revenus, dépenses et soldes par compte |
| ≡ | **Historique** | Consultez toutes vos transactions avec recherche et filtres avancés |
| 📅 | **Modèles récurrents** | Configurez des opérations mensuelles automatiques par mois pair/impair |
| 💰 | **Suivi de l'épargne** | Enregistrez et visualisez graphiquement l'évolution de votre épargne |
| 🗑️ | **Nettoyage auto.** | Supprimez automatiquement les anciennes transactions à une date configurable |
| 🎨 | **Thèmes** | Basculez entre un thème sombre et un thème clair |
| ☁️ | **Synchronisation** | Données sauvegardées en temps réel sur Firebase |

---

## Connexion

### Compte utilisateur
Saisissez votre adresse e-mail et votre mot de passe sur l'écran de connexion. Les données sont synchronisées automatiquement depuis le cloud dès l'authentification.

### Mode démo
Cliquez sur **« Voir la démo »** pour explorer l'application sans créer de compte. Un jeu de données pré-rempli est chargé automatiquement.

> ⚠️ **Attention** — Les modifications effectuées en mode démo sont visibles par tous les visiteurs. Ne saisissez aucune donnée personnelle.

### Déconnexion automatique
- Compte normal : après **8 heures** d'inactivité
- Mode démo : après **15 minutes** d'inactivité

---

## Navigation

L'application s'articule autour de trois onglets dans la barre de navigation fixe en bas de l'écran :

| Icône | Onglet | Description |
|---|---|---|
| ◈ | **Tableau de bord** | Vue d'ensemble du mois sélectionné |
| ≡ | **Historique** | Liste complète de toutes les transactions |
| ⚙ | **Réglages** | Configuration des comptes, modèles, apparence... |

---

## Tableau de bord

- **Sélection du mois** : naviguez entre le mois précédent, le mois courant et les deux mois suivants.
- **Filtrage par compte** : basculez entre « Tous les comptes » et un compte courant individuel.
- **Solde du mois** : affiché en vert (positif) ou en rouge (négatif).
- **Transactions récentes** : les 5 dernières du mois, avec actions de pointage (✓), modification (✎) et suppression (✕).
- **Modèles** : un bouton **« ↓ Appliquer le modèle »** permet d'insérer automatiquement les opérations récurrentes du mois.

---

## Historique

Affiche l'ensemble des transactions, tous mois et comptes confondus, regroupées par date.

- **Recherche** par titre (insensible à la casse)
- **Filtres** : Tout / Dépenses / Revenus
- **Ajout** via le bouton **« + Ajouter »** en haut à droite

---

## Formulaire de transaction

Champs disponibles : titre, montant (€), type (dépense / revenu), date (JJ/MM/AAAA), compte.

---

## Réglages

### Apparence
Bascule entre le thème sombre (par défaut) et le thème clair. Préférence sauvegardée localement.

### Comptes bancaires
Gestion de jusqu'à **15 comptes**. Pour chaque compte :
- Type : **Courant** (💳) ou **Épargne** (💰)
- Pour l'épargne : sous-type **Disponible** (livret A…) ou **Bloquée** (PEL, assurance vie…)
- Réorganisation (↑ ↓) et suppression (🗑️ — irréversible, supprime aussi les transactions associées)

### Modèles récurrents
Pré-configurez des opérations mensuelles répétitives, différenciées par parité du mois (pairs : 2, 4, 6… / impairs : 1, 3, 5…).

> ℹ️ Un modèle ne peut être appliqué qu'une seule fois par mois et par compte.

### Nettoyage automatique
Supprime automatiquement les transactions des mois antérieurs à partir d'un jour configurable (1–31).

> ⚠️ Choisir le 29, 30 ou 31 peut empêcher le déclenchement en février ou dans certains mois courts.

Un bouton **« 🗑️ Nettoyer les mois précédents maintenant »** permet un nettoyage manuel immédiat.

### Instantanés d'épargne automatiques
Configurez des jours du mois (ex. : 1 et 15) pour enregistrer automatiquement un snapshot du total de votre épargne.

---

## Statistiques d'épargne

Accessible via le bouton **« 📈 Stats »** dans la section Comptes épargne du tableau de bord.

Le graphique affiche deux séries chronologiques :
- 🟢 **Épargne disponible** (Livret A, etc.)
- 🔴 **Épargne bloquée** (PEL, assurance vie, etc.)

Ajout, modification (✎) et suppression (✕) d'instantanés possibles manuellement.

---

## Données & synchronisation

- Synchronisation en temps réel via **Firebase** (délai max. 1 seconde).
- Données stockées : transactions, comptes, modèles récurrents, historique d'épargne, préférences.
- Chaque utilisateur accède **uniquement à ses propres données**.

---

## FAQ

**L'application fonctionne-t-elle hors ligne ?**
Non, une connexion internet est requise. Firebase peut mettre en cache certaines données, mais l'usage hors ligne n'est pas garanti.

**Y a-t-il une limite de transactions ?**
Pas de limite fixe, mais les performances peuvent être affectées avec un très grand nombre d'entrées. Le nettoyage automatique est conçu pour maintenir la base légère.

**Que se passe-t-il si je supprime un compte ?**
La suppression est irréversible et entraîne la suppression de toutes les transactions et du modèle récurrent associés.

**Comment réinitialiser les données démo ?**
Dans les réglages (mode démo uniquement), le bouton **« Réinitialiser les données démo »** restaure les données d'exemple d'origine.

**À quoi sert le pointage d'une transaction ?**
Le pointage (✓ dorée) permet de marquer une transaction comme vérifiée sur votre relevé bancaire. C'est une aide au rapprochement bancaire, sans effet sur les calculs de solde.

---

*Mon Budget v1.0 — Février 2026*
