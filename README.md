# 💰 Mon Budget

Application web de gestion de budget personnel, hébergée sur GitHub Pages avec Firebase pour l'authentification et le stockage des données.

🔗 **URL** : `https://votrenom.github.io/mon-budget`

---

## 🏗️ Architecture

```
GitHub Pages (hébergement + HTTPS)
        ↕
Firebase Authentication (connexion email/mot de passe)
        ↕
Firebase Firestore (données synchronisées PC ↔ mobile)
```

---

## ✨ Fonctionnalités

### Tableau de bord
- Vue mensuelle avec navigation pair/impair
- Sélecteur de compte (tous les comptes ou compte individuel)
- Solde, revenus et dépenses du mois
- 5 transactions récentes avec actions rapides
- Application des modèles récurrents en un clic

### Transactions
- Liste complète groupée par date
- Recherche par libellé
- Filtres : tout / dépenses / revenus
- Ajout, édition et suppression avec confirmation
- Pointage des transactions (rapprochement bancaire)

### Comptes bancaires
- Comptes courants (solde calculé depuis les transactions)
- Comptes épargne (solde saisi manuellement)
- Jusqu'à 10 comptes
- Vue consolidée de tous les comptes

### Modèles récurrents
- Un modèle par compte courant
- Séparation mois pairs / mois impairs
- Application en un clic depuis le tableau de bord
- Tag "modèle" sur les transactions importées (disparaît si la transaction est éditée)
- Nettoyage automatique des modèles lors de la suppression d'un compte

### Autres
- Thème clair / sombre
- Nettoyage automatique des transactions du mois précédent (jour configurable)
- Nettoyage manuel disponible dans les réglages
- Déconnexion sécurisée

---

## 🔧 Technologies

| Composant | Technologie |
|-----------|-------------|
| Interface | React 18 (via CDN, sans build) |
| Style | CSS variables, thème clair/sombre |
| Police | DM Sans + DM Serif Display (Google Fonts) |
| Auth | Firebase Authentication (Email/Password) |
| Base de données | Firebase Firestore |
| Hébergement | GitHub Pages (HTTPS automatique) |

---

## 📁 Structure Firestore

```
budgets/
  {userId}/
    transactions[]     → liste des transactions
    accounts[]         → comptes bancaires
    templates{}        → modèles par compte (pair/impair)
    cleanupDay         → jour de nettoyage automatique
```

---

## 🚀 Mise à jour de l'application

1. Télécharger le nouveau fichier `budget.html`
2. Le renommer en `index.html`
3. Dans le dépôt GitHub → **Add file** → **Upload files**
4. Déposer le fichier → **Commit changes**
5. GitHub Pages se met à jour automatiquement en 1-2 minutes

---

## 🔐 Sécurité Firebase

Règles Firestore à configurer dans la console Firebase :

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /budgets/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

---

## 📦 Structure des données

### Transaction
```json
{
  "id": "abc123",
  "title": "Courses",
  "amount": 87.50,
  "type": "expense",
  "accountId": "xyz789",
  "date": "2026-02-17",
  "note": "",
  "pointed": false
}
```

### Compte
```json
{
  "id": "xyz789",
  "name": "Compte courant",
  "type": "checking",
  "balance": 0
}
```

### Modèle (par compte)
```json
{
  "{accountId}": {
    "even": { "items": [{ "id": "...", "title": "Loyer", "amount": 950, "type": "expense" }] },
    "odd":  { "items": [] }
  }
}
```
