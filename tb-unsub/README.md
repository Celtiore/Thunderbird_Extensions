# Inventaire & Désabonnement — extension Thunderbird

Squelette fonctionnel (v0.1). MailExtension Manifest V3, compatible Thunderbird 128 ESR et plus.

## Installer en développement

1. Thunderbird → menu ☰ → **Modules complémentaires et thèmes**
2. Roue dentée → **Déboguer des modules** → **Charger un module temporaire**
3. Sélectionner `manifest.json` à la racine du dossier

Le chargement temporaire disparaît au redémarrage de Thunderbird — c'est le mode de travail
pendant le développement. Pour une installation permanente, zipper le dossier
(`manifest.json` doit être **à la racine du zip**), renommer en `.xpi` et l'installer via
« Installer un module depuis un fichier ». Le fichier `.xpi` fourni est déjà prêt.

La console de débogage est dans ce même écran « Déboguer des modules » → *Inspecter*.

## Les trois briques

**1. Sélecteur de comptes** — le bouton dans la barre d'outils ouvre la popup :
liste des comptes détectés (`accounts.list()`), profondeur du scan (30 j → 1 an),
et le choix est mémorisé dans `storage.local`.

**2. Scan et inventaire** — un expéditeur par ligne, trié par volume, avec le
nombre de mails, les non-lus et la méthode de désinscription disponible.

**3. Action sur multi-sélection** — clic droit dans la liste des messages →
« Désabonner et ficher l'expéditeur ». Les messages sélectionnés sont regroupés
par expéditeur : 12 mails de SAFTI = 1 seule opération, pas 12.

## La perf, qui est le vrai sujet

`messages.query()` lit l'index local de Thunderbird : instantané, aucun téléchargement.
Les en-têtes complets (`getFull`, avec repli sur `getRaw`) ne sont lus qu'**une fois par
expéditeur**, sur le message le plus récent. Sur une boîte de 20 000 messages et
300 expéditeurs distincts, ça fait 300 lectures au lieu de 20 000.

Les adresses de routage jetables (`alerte_at_safti_fr_ntv515bmh0244q…@…`) sont normalisées
par `Parse.senderKey()` pour ne pas produire une fiche par envoi. À ajuster selon ce que
tu vois passer — c'est le point le plus susceptible de demander du réglage.

## Ce que fait chaque méthode

| En-têtes détectés | Action | Statut |
|---|---|---|
| `List-Unsubscribe` HTTPS + `List-Unsubscribe-Post` | POST RFC 8058 automatique | Désabonné |
| `List-Unsubscribe` HTTPS seul | ouvre la page dans le navigateur | À finaliser |
| `List-Unsubscribe` mailto | prépare le brouillon (envoi manuel par défaut) | À finaliser |
| aucun | **ne clique sur rien**, fiche l'expéditeur | Échec → filtre à créer |

Le dernier cas est délibéré : un mail sans `List-Unsubscribe` et d'un expéditeur inconnu est
probablement du vrai spam, et cliquer confirmerait que l'adresse est vivante.

`autoSendMailto` est à `false` dans `Store.DEFAULTS` : aucun mail ne part sans validation.
Le passer à `true` fait appel à `compose.sendMessage()`.

## Journal de preuves

Chaque tentative est journalisée (date, expéditeur, méthode, cible, code HTTP, résultat).
Export CSV depuis la popup. C'est cette trace qui sert si un expéditeur continue d'envoyer
après désinscription — pièce jointe d'une mise en demeure RGPD ou d'une plainte CNIL.

## Permissions demandées

`accountsRead` (lister les comptes) · `messagesRead` (lire les en-têtes) · `menus` (menu
contextuel) · `storage` (fiches et journal) · `compose` (brouillons mailto) ·
`notifications` · `downloads` (export CSV) · accès aux URL externes (le POST one-click).

Thunderbird affichera « Accéder à vos données pour tous les sites web » : c'est la contrepartie
inévitable du POST vers des domaines arbitraires.

## Pistes pour la v0.2

- Relance J+7 : `alarms` + rescan des expéditeurs marqués « désabonné » qui réapparaissent
- Création automatique d'un filtre pour les expéditeurs sans mécanisme
- Génération du courrier d'opposition RGPD depuis la fiche expéditeur
- Suppression en masse des mails d'un expéditeur (`messagesDelete`)
