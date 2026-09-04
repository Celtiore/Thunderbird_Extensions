# CLAUDE.md

Contexte de projet pour les sessions Claude Code. Lire aussi `RAPPORT.md` (le cadrage
complet et les décisions déjà prises) et `README.md` (installation et usage).

> Dans le monorepo `Thunderbird_Extensions`, le code de cette extension vit dans `src/`
> (non suivi sur `main` — cf. `.gitignore` racine ; sources sur la branche locale
> `dev/tb-unsub`). Les chemins cités ci-dessous sont relatifs à `src/`. `src/build.sh`
> produit `inventaire-desabonnement@fteventspro.app.xpi` à côté de `src/` (committé).

## Ce qu'on construit

Extension Thunderbird (MailExtension, **Manifest V3**, TB 128 ESR minimum) qui :

1. inventorie les expéditeurs de newsletters sur les comptes sélectionnés
2. se désabonne via RFC 8058 (POST one-click) quand c'est possible
3. garde une fiche persistante par expéditeur + un journal de preuves horodaté

Ce n'est **pas** un outil de suppression de mails. La suppression est un effet de bord
optionnel, pas l'objectif.

## Stack et contraintes

- JavaScript vanille, aucune dépendance, aucun build step. Pas de bundler, pas de framework.
- Scripts classiques (pas de modules ES) : les fichiers `lib/*.js` exposent des objets
  globaux (`Store`, `Parse`, `Scan`, `Unsub`) chargés dans l'ordre déclaré au manifest.
- API accessible via `messenger.*` (équivalent de `browser.*`, plus idiomatique côté TB).
- Tout est asynchrone et renvoie des Promises.
- Tout reste en local dans `storage.local`. **Aucun appel réseau sortant** hors des URL
  de désinscription elles-mêmes. C'est une règle du projet, pas une préférence.

## Architecture

```
manifest.json          permissions, points d'entrée
background.js          menu contextuel sur la sélection de messages
lib/store.js           réglages, fiches expéditeurs, journal (storage.local)
lib/parse.js           parsing des en-têtes List-Unsubscribe, normalisation d'adresses
lib/scan.js            scan des comptes, agrégation par expéditeur
lib/unsub.js           exécution : POST one-click, mailto, lien externe
popup/                 UI : sélecteur de comptes, inventaire, traitement par lot
```

`lib/*.js` est partagé entre le background et la popup — la popup a le même accès aux
API `messenger.*`, donc pas de `runtime.sendMessage` inutile entre les deux.

## Règles de conception à respecter

**Une lecture d'en-têtes par expéditeur, jamais par message.** `messages.query()` lit
l'index local (instantané) ; `getFull()` télécharge le message. Le scan regroupe d'abord,
puis échantillonne le message le plus récent de chaque expéditeur. Ne pas casser ça.

**Rien ne part sans validation humaine, sauf le POST one-click.** Le POST RFC 8058 est un
opt-out standardisé et idempotent : automatisable. Un `mailto:` envoie un vrai mail depuis
l'identité de l'utilisateur → brouillon par défaut (`autoSendMailto: false`).

**Ne jamais déclencher d'action sur un message sans en-tête `List-Unsubscribe`.** C'est
probablement du spam ; cliquer confirmerait que l'adresse est vivante. On fiche, on ne clique pas.

**Chaque tentative est journalisée** (date, expéditeur, méthode, cible, code HTTP, résultat).
Le journal est une pièce juridique potentielle (opposition RGPD, plainte CNIL) : ne jamais
l'écraser, seulement y ajouter.

## Tester

Pas de tests automatisés — l'API n'existe qu'à l'intérieur de Thunderbird.

```
☰ → Modules complémentaires et thèmes → roue dentée
  → Déboguer des modules → Charger un module temporaire → manifest.json
```

« Inspecter » ouvre la console. Après modification d'un fichier : **Recharger** dans le même
écran. Le chargement temporaire disparaît au redémarrage de TB.

`./build.sh` produit le `.xpi` pour une installation permanente.

## Pièges connus de l'API

- Les listes de messages sont **paginées** : toujours dérouler avec
  `messages.continueList(list.id)`. Le helper `Scan.collect()` le fait, l'utiliser partout.
- `getFull()` ne remonte pas toujours les en-têtes exotiques → repli sur `getRaw()` +
  `Parse.rawHeaders()`. Déjà implémenté dans `Scan.headers()`, ne pas contourner.
- `messages.query({fromDate})` ratisse **tous** les comptes et **tous** les dossiers, corbeille
  et spam compris. Le filtrage par compte et par type de dossier se fait côté client
  (`folder.accountId`, `folder.type`).
- Les signatures d'API bougent d'une version de TB à l'autre (`messages.delete`,
  `parseMailboxString`, `folder` vs `folderId`). En cas de doute, vérifier la doc de la
  version cible plutôt que de supposer — `ctx7` sur la doc WebExtension Thunderbird.

## Chantier prioritaire

`Parse.senderKey()` normalise les adresses de routage jetables
(`alerte_at_safti_fr_ntv515bmh0244q_s0f53372@…` → `alerte@…`). Les regex sont empiriques et
n'ont pas encore été confrontées à des données réelles. Premier vrai scan → repérer les
expéditeurs dupliqués dans l'inventaire → affiner. C'est le point le plus susceptible de
fausser tout l'inventaire.

## Feuille de route (v0.2)

- relance J+7 : `alarms` + rescan des expéditeurs marqués « désabonné » qui réapparaissent
- création automatique d'un filtre pour les expéditeurs sans mécanisme
- génération du courrier d'opposition RGPD depuis la fiche expéditeur
- suppression en masse des mails d'un expéditeur (nécessite `messagesDelete`)
