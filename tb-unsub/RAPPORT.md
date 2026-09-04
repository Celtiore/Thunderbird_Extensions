# Rapport — Nettoyage et désabonnement des boîtes mail

**Date :** 19 août 2026
**Contexte :** conversation de cadrage, aboutissant à un prototype d'extension Thunderbird.

---

## 1. Le besoin

Nettoyer plusieurs comptes email des spams et newsletters — non pas seulement supprimer
les messages, mais **faire cesser les envois** : inventorier les expéditeurs, se désabonner,
et garder trace de qui a été traité.

Contrainte de départ : les comptes sont répartis sur des infrastructures différentes
(Gmail via OAuth, boîte pro `@fumesmaispasque.fr` en IMAP chez LWS), ce qui exclut toute
solution mono-fournisseur.

## 2. Le mécanisme technique sous-jacent

Tout repose sur la **RFC 8058** et deux en-têtes :

```
List-Unsubscribe: <https://exemple.fr/unsub?id=abc123>, <mailto:unsub@exemple.fr>
List-Unsubscribe-Post: List-Unsubscribe=One-Click
```

Quand les deux sont présents, un simple `POST` avec le corps `List-Unsubscribe=One-Click`
vers l'URL HTTPS désabonne, sans navigateur ni page de confirmation. L'endpoint doit
répondre 200 sans redirection.

Point décisif pour la couverture : depuis février 2024, Google et Yahoo **imposent** ce
mécanisme aux expéditeurs de plus de 5 000 messages/jour, avec traitement sous 48 h.
Les vraies newsletters commerciales sont donc très majoritairement automatisables.

Trois cas dégradés subsistent :

| Cas | Traitement |
|---|---|
| `List-Unsubscribe` HTTPS sans `-Post` | ouvrir la page dans le navigateur (semi-auto) |
| `List-Unsubscribe` `mailto:` seul | préparer un mail de désinscription (validation humaine) |
| Aucun en-tête | **ne rien cliquer** — filtre + signalement |

Le dernier cas est une règle de sécurité, pas une limitation technique : sur un vrai spam,
cliquer « se désabonner » confirme que l'adresse est vivante et augmente le volume reçu.

## 3. Les options étudiées

**Niveau 0 — natif Gmail.** La vue « Gérer les abonnements » (raccourci : remplacer `#inbox`
par `#sub` dans l'URL) liste les expéditeurs par volume avec un bouton de désinscription.
Gratuit, immédiat, mais couvre uniquement Gmail. Piège : *bloquer* ≠ *désabonner* — un
expéditeur bloqué continue d'envoyer, ses mails vont juste en spam.

**Niveau 1 — outils tiers.** Cleanfox est gratuit parce qu'il monétise des statistiques
extraites des boîtes (éditeur : NielsenIQ), et des tests récents indiquent qu'il déplace les
mails vers la corbeille sans réellement désinscrire. Clean Email et Leave Me Alone sont
payants, ne revendent pas de données, et supportent l'IMAP générique — donc compatibles LWS.

**Niveau 2 — outil maison.** Scanner Python sur VPS + app SwiftUI + Firestore. Techniquement
solide mais impose de gérer OAuth, les credentials IMAP, un serveur et une base : beaucoup
d'infrastructure pour un besoin personnel.

**Option retenue — extension Thunderbird.** Les comptes sont déjà authentifiés dans le client :
zéro credential à gérer, un seul code pour Gmail et LWS, tout en local (aucune donnée ne
sort de la machine), aucun serveur. Thunderbird n'a toujours pas de désinscription native
(bug Bugzilla 29041, en backlog), mais l'API MailExtension expose tout le nécessaire.

**Réserve identifiée :** une migration vers Mail d'Apple rendrait l'extension caduque.
Arbitrage rendu en séance — on reste sur Thunderbird.

## 4. L'existant à ne pas réécrire

`BetterUnsubscribe` (add-on actif, compatible TB 137+) fait déjà le désabonnement
mail par mail : POST RFC 8058, brouillon `mailto:`, ouverture des liens HTTPS, détection
des liens enfouis dans le corps du message.

Le delta qui justifie de coder quelque chose :

- inventaire **agrégé** par expéditeur, trié par volume et non-lus
- traitement **par lot** validé en une passe
- **fiche persistante** par expéditeur (statut, historique)
- **journal de preuves** exploitable pour le volet juridique
- relance à J+7 sur les expéditeurs qui continuent d'envoyer

## 5. Ce qui a été livré

Extension MailExtension Manifest V3 (Thunderbird 128 ESR+), version 0.1.0 :

- **sélecteur de comptes** dans la popup, avec profondeur de scan configurable
- **scan** avec agrégation par expéditeur et détection de la méthode disponible
- **action sur multi-sélection** : clic droit → regroupement par expéditeur → traitement
- **fiches expéditeurs** persistées dans `storage.local`
- **journal de preuves** exportable en CSV

Décision de conception : le POST one-click est automatique, tout le reste demande une
validation. `autoSendMailto` est à `false` par défaut — aucun mail ne part sans accord.

**Optimisation structurante :** `messages.query()` lit l'index local de Thunderbird
(instantané, aucun téléchargement) ; les en-têtes complets ne sont lus qu'une fois par
expéditeur, sur le message le plus récent. Sur 20 000 messages et 300 expéditeurs
distincts : 300 lectures au lieu de 20 000.

## 6. Volet juridique (non implémenté)

En France, la prospection par email est en opt-in (art. L34-5 CPCE). Le RGPD ouvre un droit
d'opposition (art. 21) et d'effacement (art. 17), avec réponse due sous un mois.

Séquence pour les récalcitrants : mail d'opposition formelle au `dpo@` ou `contact@` du
domaine → si l'envoi persiste, plainte CNIL appuyée sur le journal de preuves. C'est
précisément la raison pour laquelle chaque tentative est horodatée avec son code HTTP.

## 7. Points ouverts

- **Normalisation des adresses de routage.** Certains expéditeurs génèrent une adresse
  unique par envoi (`alerte_at_safti_fr_ntv515bmh0244q_s0f53372@…`). `Parse.senderKey()`
  coupe ces suffixes de façon empirique — à affiner sur données réelles.
- **Mesure à faire :** sur un scan à 90 jours, combien d'expéditeurs distincts, et quelle
  proportion en « Automatique » ? Ce ratio décide si la v0.2 vaut le coup.
- **Signature de l'extension** si installation permanente plutôt que chargement temporaire.

## Sources

- RFC 8058 — One-Click List-Unsubscribe
- Documentation MailExtension : https://webextension-api.thunderbird.net/en/mv3/
- Exemples officiels : https://github.com/thunderbird/webext-examples
- BetterUnsubscribe : https://github.com/LucBennett/BetterUnsubscribe
- Bugzilla 29041 — support natif RFC 2369/8058 dans Thunderbird
- Aide Gmail — Gérer vos abonnements
