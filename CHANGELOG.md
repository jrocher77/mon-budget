# Changelog — Mon Budget

**Date :** 18 février 2026

---

## 🐛 Corrections de bugs

### Modification des transactions non persistée en base de données
Les modifications de transactions s'affichaient correctement dans l'interface mais n'étaient pas sauvegardées dans Firestore. Après rechargement de la page, les changements disparaissaient.

**Cause :** La fonction `update` assignait `_tpl: undefined` sur les transactions modifiées. Or, Firestore rejette silencieusement les objets contenant des valeurs `undefined`, ce qui empêchait la sauvegarde.

**Correction :** Remplacement de `_tpl: undefined` par un `delete` propre de la clé, et ajout d'un nettoyage des valeurs `undefined` dans la fonction de persistance avant chaque envoi à Firestore.

---

### Mois précédent non visible sur le Dashboard (nettoyage désactivé)
Avec le nettoyage automatique désactivé, les transactions saisies sur un mois antérieur (ex : janvier) apparaissaient dans l'historique mais le mois n'était pas accessible dans les onglets du Dashboard.

**Cause :** La condition d'affichage du mois précédent ne tenait pas compte du paramètre `autoCleanup` — elle ne se basait que sur le jour de nettoyage.

**Correction :** Le mois précédent s'affiche désormais dès qu'il contient des transactions si le nettoyage automatique est désactivé, ou si l'on est avant le jour de nettoyage.

---

### Nettoyage automatique non déclenché à l'activation
Activer le nettoyage automatique depuis les réglages ne déclenchait pas le nettoyage. Les transactions du mois précédent restaient visibles dans l'historique.

**Cause :** Le cleanup ne s'exécutait qu'une seule fois au montage du composant (via un verrou `cleanupDone`), sans jamais se re-déclencher lors d'un changement de paramètres.

**Correction :** Suppression du verrou et ajout de `autoCleanup`, `cleanupDay` et `transactions` dans les dépendances du `useEffect`, permettant un re-déclenchement automatique à chaque changement pertinent.

---

### Nettoyage limité au mois précédent uniquement
Le nettoyage automatique et manuel ne supprimait que les transactions du mois M-1. Les transactions plus anciennes (M-2, M-3, etc.) étaient conservées.

**Cause :** Le filtre utilisait une clé de mois fixe (`prevKey`) au lieu de comparer avec le mois courant.

**Correction :** Le nettoyage supprime désormais toutes les transactions dont le mois est strictement antérieur au mois en cours.

---

## ✨ Améliorations

### Tag « modèle » masqué sur les transactions pointées
Les transactions importées depuis un modèle récurrent affichaient en permanence le tag « modèle ». Celui-ci disparaît désormais lorsque la transaction est pointée, pour une meilleure lisibilité.

---

### Réorganisation du menu Réglages
L'ordre des sections a été modifié pour un parcours plus logique :

1. 🎨 Apparence
2. 🏦 Comptes bancaires
3. 📅 Modèles récurrents
4. 🗑️ Nettoyage automatique
5. 💾 Données

---

### Mise à jour des libellés du nettoyage
- Description du nettoyage actif : *« Actif — les transactions des mois passés sont supprimées automatiquement »*
- Bouton de nettoyage manuel : *« Nettoyer les mois précédents maintenant »*
- Message de confirmation mis à jour pour refléter le nombre total de transactions antérieures au mois courant.

---

### Confirmation avant suppression d'un compte avec transactions
La suppression d'un compte bancaire contenant des transactions affiche désormais une modale d'avertissement indiquant le nombre de transactions qui seront également supprimées. Si le compte n'a aucune transaction, la suppression reste immédiate. Les transactions liées au compte sont supprimées en même temps que le compte.
