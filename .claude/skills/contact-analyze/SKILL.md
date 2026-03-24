---
name: contact-analyze
description: Analyse, nettoie et enrichit les contacts exportes par Contact Lens en 5 passes avec fichier master cumulatif
---

# /contact-analyze — Analyse des contacts Contact Lens

## Vue d'ensemble

Ce skill analyse un export Contact Lens (JSON enrichi avec sujets et body snippets) en 5 passes :
1. **Nettoyage local** (zero tokens Claude) : exclusions + scoring
2. **Analyse Claude par batch** : categorisation intelligente des contacts pertinents (SKIP les contacts deja valides dans le master)
3. **Consolidation** : generation des fichiers de sortie dates
4. **Validation** : suggestions de filtres avec validation humaine
5. **Fusion** : merge avec `contacts-master.json` (source de verite cumulative)

## Fichiers

```
exports/
  scans/                          <- Exports bruts Contact Lens (input)
    contact-lens-YYYY-MM-DD.json
  filters/                        <- Filtres editables
    exclusions.json
    keywords.json
  results/                        <- Sortie du skill
    contacts-master.json          <- SOURCE DE VERITE cumulative, persistante
    enriched-YYYY-MM-DD.json      <- Snapshot du run courant
    enriched-YYYY-MM-DD.csv       <- Version Excel du run courant
    filter-suggestions-YYYY-MM-DD.json
    viewer.html                   <- Visualiseur interactif
```

## contacts-master.json — Source de verite

Ce fichier est **cumulatif et persistant**. Il est cree au premier run et fusionne a chaque run suivant.

Structure :
```json
{
  "metadata": {
    "lastUpdate": "2026-03-24",
    "totalRuns": 2,
    "runHistory": ["2026-03-24", "2026-03-31"],
    "totalContacts": 1780,
    "validated": 45,
    "excluded": 1157
  },
  "excluded": [...],
  "contacts": [
    {
      "email": "jean@brasserie.fr",
      "displayName": "Jean Dupont",
      "category": "client",
      "sector": "Brasserie artisanale",
      "context": "Fournisseur biere, commandes regulieres",
      "tags": ["devis", "commande"],
      "suggestedAction": "relancer",
      "confidence": "high",
      "validated": false,
      "validatedAt": null,
      "score": 85,
      "direction": "bidirectional",
      "receivedCount": 45,
      "sentCount": 32,
      "firstSeen": "2023-01-15T10:30:00.000Z",
      "lastSeen": "2026-03-20T14:22:00.000Z",
      "lastSeenInScan": "2026-03-24",
      "firstAnalyzed": "2026-03-24",
      "subjects": [...],
      "bodySnippets": [...]
    }
  ]
}
```

### Regles de fusion (Passe 5)

| Donnee | Regle |
|--------|-------|
| Nouveau contact (email inconnu dans le master) | Ajoute tel quel avec `firstAnalyzed = today`, `validated = false` |
| Contact existant — compteurs (receivedCount, sentCount) | Mis a jour depuis le scan le plus recent |
| Contact existant — dates (firstSeen, lastSeen) | firstSeen = min(master, scan), lastSeen = max(master, scan) |
| Contact existant — score | Recalcule depuis le scan le plus recent |
| Contact existant — sujets/bodySnippets | Remplaces par ceux du scan le plus recent |
| Contact existant — category/sector/context/tags si `validated = true` | **CONSERVES** (jamais ecrases) |
| Contact existant — category/sector/context/tags si `validated = false` | Mis a jour depuis l'analyse Claude du run courant |
| Contact existant — direction | Mis a jour si change (ex: receivedOnly -> bidirectional) |
| Contact absent du scan courant | Conserve dans le master avec `lastSeenInScan` inchange |
| Contact exclu | Mis a jour dans la liste excluded |

### Contacts valides

Un contact avec `validated: true` :
- N'est **jamais re-categorise** par Claude (economie de tokens)
- Ses compteurs et dates sont quand meme mis a jour
- Seul moyen de le re-categoriser : mettre `validated: false` manuellement ou via le viewer

## Prerequis

- Un export enrichi recent dans `exports/scans/` (genere par Contact Lens v1.1+)
- Les fichiers de filtres dans `exports/filters/` (exclusions.json, keywords.json)
- Le dossier `exports/results/` doit exister
- `contacts-master.json` est cree automatiquement au premier run s'il n'existe pas

## Execution

### Passe 1 — Nettoyage local (zero tokens)

1. **Trouver le scan le plus recent** :
   - Lister les fichiers `exports/scans/contact-lens-*.json`
   - Prendre le plus recent par nom de fichier (le format YYYY-MM-DD permet un tri alphabetique)
   - Si aucun fichier trouve, afficher une erreur et arreter

2. **Charger les filtres** :
   - Lire `exports/filters/exclusions.json` — si invalide ou manquant, afficher une erreur
   - Lire `exports/filters/keywords.json` — si invalide ou manquant, afficher une erreur

3. **Appliquer les exclusions** sur chaque contact :
   - Test 1 : email exact dans `exclusions.emails`
   - Test 2 : domaine (partie apres @) dans `exclusions.domains`
   - Test 3 : email matche un pattern dans `exclusions.patterns` (compiler chaque pattern avec `new RegExp(pattern, "i")`)
   - Si exclu : stocker dans la liste `excluded` avec `{email, displayName, reason}` ou reason = `"email:exact"`, `"domain:stripe.com"`, `"pattern:^noreply@"`

4. **Calculer le score de qualite** pour les contacts non exclus :

   | Critere | Points | Condition |
   |---------|--------|-----------|
   | Domaine professionnel | +20 | Domaine PAS dans la liste des domaines gratuits ci-dessous |
   | Nom exploitable | +10 | `displayName` contient au moins 2 mots, dont un de 2+ lettres |
   | Bidirectionnel | +25 | `sentCount > 0 AND receivedCount > 0` |
   | Frequence > 5 messages | +10 | `receivedCount + sentCount > 5` |
   | Frequence > 20 messages | +5 | `receivedCount + sentCount > 20` (cumulable avec le precedent) |
   | Contact recent < 6 mois | +15 | `lastSeen` est dans les 6 derniers mois |
   | Contact recent < 1 mois | +5 | `lastSeen` est dans le dernier mois (cumulable avec le precedent) |
   | Pattern no-reply | -30 | Email matche `^(noreply|no-reply|no\.reply|newsletter|donotreply|do-not-reply)@` |
   | Domaine gratuit | -10 | Domaine dans la liste des domaines gratuits ci-dessous |
   | Message unique | -15 | `receivedCount + sentCount == 1` |

   **Liste des domaines d'email gratuits** (pour le critere de scoring, PAS pour l'exclusion) :
   ```
   gmail.com, googlemail.com, yahoo.com, yahoo.fr, yahoo.co.uk, yahoo.co.jp, yahoo.de, yahoo.it, yahoo.es,
   outlook.com, outlook.fr, hotmail.com, hotmail.fr, hotmail.co.uk, hotmail.de, hotmail.it, hotmail.es,
   live.com, live.fr, live.co.uk, msn.com,
   aol.com, aol.fr,
   protonmail.com, protonmail.ch, proton.me, pm.me,
   icloud.com, me.com, mac.com,
   mail.com, email.com, usa.com, post.com, europe.com,
   zoho.com, zohomail.com,
   yandex.com, yandex.ru, ya.ru,
   gmx.com, gmx.fr, gmx.de, gmx.net,
   web.de, freenet.de, t-online.de,
   orange.fr, wanadoo.fr, laposte.net, sfr.fr, free.fr, neuf.fr, alice.fr, voila.fr, bbox.fr,
   numericable.fr, noos.fr, club-internet.fr, cegetel.net, aliceadsl.fr,
   bluewin.ch, hispeed.ch,
   libero.it, virgilio.it, tin.it, alice.it, tiscali.it, fastwebnet.it,
   terra.es, telefonica.net, ono.com,
   btinternet.com, sky.com, virgin.net, talktalk.net, ntlworld.com,
   comcast.net, verizon.net, att.net, sbcglobal.net, cox.net, charter.net, earthlink.net,
   shaw.ca, rogers.com, sympatico.ca, videotron.ca, bell.net,
   bigpond.com, optusnet.com.au,
   rediffmail.com, sify.com,
   163.com, 126.com, qq.com, sina.com, sohu.com, foxmail.com,
   naver.com, daum.net, hanmail.net,
   rambler.ru, mail.ru, bk.ru, inbox.ru, list.ru,
   wp.pl, o2.pl, interia.pl, onet.pl, poczta.fm,
   abv.bg, centrum.cz, seznam.cz,
   sapo.pt, bol.com.br, uol.com.br, terra.com.br, ig.com.br,
   tiscali.co.uk, tiscali.fr
   ```

5. **Tagger les sujets par keywords** :
   - Pour chaque contact non exclu, chercher les mots-cles de `keywords.json` dans ses `subjects`
   - Match case-insensitive, mot partiel (`subject.toLowerCase().includes(keyword.toLowerCase())`)
   - Stocker les categories matchees comme `subjectTags: ["devis", "commande"]`

6. **Afficher les stats de la passe 1** :
   ```
   === PASSE 1 — Nettoyage ===
   Total contacts dans le scan : XXXX
   Exclus : XXX (XX.X%)
     - par domaine : XXX
     - par pattern : XXX
     - par email exact : XXX
   Restants a analyser : XXXX
   Score >= 30 (analyse Claude) : XXXX
   Score < 30 (low-priority) : XXX
   Distribution des scores :
     -55 a 0  : XXX
      1 a 20  : XXX
     21 a 40  : XXX
     41 a 60  : XXX
     61 a 80  : XXX
     81 a 100 : XXX
   ```

### Passe 2 — Analyse Claude par batch

**Pre-requis** : Avant de commencer, charger `exports/results/contacts-master.json` (s'il existe) pour identifier les contacts avec `validated: true`. Ces contacts sont **exclus de l'analyse Claude** — on conserve leur categorisation existante du master.

**Seuil** : ne traiter que les contacts avec `score >= 30` ET `validated != true` dans le master. Les contacts sous le seuil recoivent `category: "low-priority"`, `confidence: "low"`, `suggestedAction: "archiver"`. Les contacts valides conservent leur categorisation du master.

**Taille des batchs** : 40 contacts par batch.

**Pour chaque batch**, envoyer ce prompt au modele :

```
Tu es un analyste de contacts email professionnel. Tu travailles pour une entreprise dans le secteur de la brasserie artisanale et de l'evenementiel.

Pour chaque contact ci-dessous, determine :
- category : un parmi [client, prospect, fournisseur, admin, personnel, service, media, association, autre]
- sector : secteur d'activite concis (ex: "Brasserie artisanale", "Mairie", "Evenementiel", "Restauration")
- context : une phrase de contexte libre decrivant la relation (ex: "Fournisseur de malt, commandes regulieres depuis 2023")
- tags : tableau de tags pertinents parmi [devis, commande, event, facture, partenariat, admin, candidature, relance, technique, logistique, comptabilite] ou tout autre tag pertinent
- suggestedAction : un parmi [relancer, archiver, ajouter-crm, surveiller, contacter]
- confidence : un parmi [high, medium, low]

Base-toi sur les indices suivants pour chaque contact :
- L'adresse email et le domaine (indice fort sur le type d'organisation)
- Le displayName (indice sur la personne)
- Les sujets recents des emails (indice fort sur la nature de la relation)
- Les extraits de messages (indice tres fort si disponibles)
- Les compteurs (volume = importance)
- La direction (bidirectionnel = relation active)
- Le score de qualite (indicateur pre-calcule)

Reponds UNIQUEMENT avec un tableau JSON, un objet par contact, dans l'ordre recu :
[
  {
    "email": "...",
    "category": "...",
    "sector": "...",
    "context": "...",
    "tags": [...],
    "suggestedAction": "...",
    "confidence": "..."
  }
]
```

**Donnees envoyees par contact dans le batch** :
```json
{
  "email": "jean@brasserie-du-sud.fr",
  "displayName": "Jean Dupont",
  "domain": "brasserie-du-sud.fr",
  "subjects": ["Re: Devis futs 30L mars", "Commande salon Aix"],
  "bodySnippets": ["Bonjour, suite a notre echange au salon..."],
  "receivedCount": 45,
  "sentCount": 32,
  "direction": "bidirectional",
  "score": 85,
  "subjectTags": ["devis", "commande", "event"],
  "firstSeen": "2023-01-15",
  "lastSeen": "2026-03-20"
}
```

**Procedure** :
1. Preparer tous les batchs (ceil(contactsEligibles / 40))
2. Pour chaque batch :
   a. Construire le message avec les 40 contacts
   b. Envoyer au modele
   c. Parser la reponse JSON
   d. Valider que chaque contact a les 6 champs attendus
   e. Si le parsing echoue, re-essayer une fois avec un rappel du format
   f. Afficher la progression : "Batch 5/30 — 200/1218 contacts traites"
3. Pour les contacts sous le seuil (score < 30), assigner directement :
   ```json
   {
     "category": "low-priority",
     "sector": "",
     "context": "Score faible, non analyse par Claude",
     "tags": [],
     "suggestedAction": "archiver",
     "confidence": "low"
   }
   ```

### Passe 3 — Consolidation

Generer 3 fichiers dans `exports/results/` :

#### 3.1 — `enriched-YYYY-MM-DD.json`

```json
{
  "metadata": {
    "date": "2026-03-24T...",
    "scanSource": "contact-lens-2026-03-24.json",
    "stats": {
      "total": 2904,
      "excluded": 874,
      "analyzed": 2030,
      "analyzedByClaude": 1218,
      "lowPriority": 812,
      "byCategory": { "client": 342, "prospect": 156, "fournisseur": 89 },
      "bySector": { "Brasserie": 12, "Restauration": 45 },
      "byAction": { "relancer": 89, "archiver": 456 },
      "scoreDistribution": { "-55-0": 234, "1-20": 578, "21-40": 456, "41-60": 398, "61-80": 218, "81-100": 146 }
    }
  },
  "excluded": [
    { "email": "noreply@stripe.com", "reason": "domain:stripe.com", "displayName": "Stripe" }
  ],
  "contacts": [
    {
      "email": "jean@brasserie-du-sud.fr",
      "displayName": "Jean Dupont",
      "category": "client",
      "sector": "Brasserie artisanale",
      "context": "Fournisseur biere, commandes regulieres depuis 2023",
      "tags": ["devis", "commande", "event"],
      "suggestedAction": "relancer",
      "confidence": "high",
      "score": 85,
      "direction": "bidirectional",
      "receivedCount": 45,
      "sentCount": 32,ok
      "firstSeen": "2023-01-15T10:30:00.000Z",
      "lastSeen": "2026-03-20T14:22:00.000Z",
      "subjects": ["Re: Devis futs 30L mars"],
      "bodySnippets": [{ "subject": "...", "date": "...", "snippet": "..." }]
    }
  ]
}
```

**Ordre des contacts** : tries par score decroissant dans le JSON.

#### 3.2 — `enriched-YYYY-MM-DD.csv`

- Encodage : UTF-8 avec BOM (`\uFEFF`)
- Separateur : `;`
- Retours a la ligne : `\r\n` (Windows/Excel)
- Colonnes :
  ```
  Email;Nom;Categorie;Secteur;Contexte;Tags;Score;Action;Direction;Recus;Envoyes;Premier contact;Dernier contact;Confiance
  ```
- Tags joints par `, `
- Dates au format `DD/MM/YYYY`
- Pas de body snippets dans le CSV
- Pas de contacts exclus dans le CSV (uniquement les non-exclus)
- Trier par score decroissant

#### 3.3 — `filter-suggestions-YYYY-MM-DD.json`

Analyser les contacts et detecter :

**Domaines recurrents a exclure** :
- Parmi les contacts non-exclus avec score < 10, grouper par domaine
- Si un domaine a 3+ contacts avec score < 10 et TOUS en `receivedOnly` : suggerer l'exclusion
- Format : `{ "domain": "...", "count": N, "reason": "..." }`

**Patterns recurrents** :
- Parmi les contacts non-exclus avec score < 10, chercher des prefixes d'email recurrents
- Si un prefixe (partie avant @) apparait 3+ fois identique : suggerer un pattern
- Format : `{ "pattern": "^prefixe@", "count": N, "reason": "..." }`

```json
{
  "date": "2026-03-24",
  "suggestions": {
    "domains": [
      { "domain": "newsletters.example.com", "count": 7, "reason": "7 contacts, tous receivedOnly, score moyen -5" }
    ],
    "patterns": [
      { "pattern": "^info@", "count": 12, "reason": "12 contacts info@, 10 sont receivedOnly, score moyen 2" }
    ]
  }
}
```

### Passe 4 — Validation humaine

Apres la generation des fichiers, afficher un resume interactif dans la conversation :

```
=== PASSE 4 — Suggestions de filtres ===

Nouveaux domaines a exclure (N suggestions) :
  1. [+] newsletters.example.com (7 contacts, tous receivedOnly, score moyen -5)
  2. [+] marketing.autresite.fr (4 contacts, tous receivedOnly, score moyen 3)
  ...

Nouveaux patterns a exclure (N suggestions) :
  1. [+] ^billing@ (5 contacts, 4 receivedOnly, score moyen 1)
  ...

Voulez-vous valider ces suggestions ?
- "oui" / "yes" : accepter toutes les suggestions
- "non" / "no" : rejeter toutes les suggestions
- Numeros separes par virgule pour accepter certaines (ex: "1,3,5") : accepter uniquement celles-ci
- "edit" : modifier manuellement exclusions.json
```

**Si validation (totale ou partielle)** :
- Lire `exports/filters/exclusions.json`
- Ajouter les domaines valides au tableau `domains`
- Ajouter les patterns valides au tableau `patterns`
- Mettre a jour `_updated` avec la date du jour
- Reecrire le fichier
- Afficher un resume des modifications

### Passe 5 — Fusion avec contacts-master.json

Cette passe est **automatique** apres la passe 4.

1. **Charger le master existant** :
   - Lire `exports/results/contacts-master.json`
   - Si le fichier n'existe pas, creer un master vide : `{"metadata": {...}, "excluded": [], "contacts": []}`
   - Indexer les contacts du master par email dans un dictionnaire pour un acces O(1)

2. **Charger les contacts validés du master** :
   - Extraire tous les contacts avec `validated: true`
   - Ces contacts ne seront JAMAIS re-categorises

3. **Fusionner les contacts du run courant** (depuis `enriched-YYYY-MM-DD.json`) :
   Pour chaque contact du run courant :

   **Si le contact existe dans le master ET est `validated: true`** :
   - Conserver : category, sector, context, tags, suggestedAction, confidence, validated, validatedAt
   - Mettre a jour : receivedCount, sentCount, score, direction, subjects, bodySnippets, accounts
   - Mettre a jour : firstSeen = min(master, run), lastSeen = max(master, run)
   - Mettre a jour : lastSeenInScan = date du run courant

   **Si le contact existe dans le master ET est `validated: false`** :
   - Ecraser : category, sector, context, tags, suggestedAction, confidence (valeurs du run courant)
   - Mettre a jour : tout le reste comme ci-dessus
   - Mettre a jour : lastSeenInScan = date du run courant

   **Si le contact est nouveau (absent du master)** :
   - Ajouter tel quel avec : `validated: false`, `validatedAt: null`, `firstAnalyzed: date du run`, `lastSeenInScan: date du run`

4. **Conserver les contacts absents du scan courant** :
   - Les contacts du master qui ne sont PAS dans le run courant sont conserves tels quels
   - Leur `lastSeenInScan` n'est PAS mis a jour (permet de detecter les contacts disparus)

5. **Fusionner les exclus** :
   - Union des listes excluded du master et du run courant (deduplication par email)

6. **Mettre a jour les metadonnees** :
   ```json
   {
     "lastUpdate": "YYYY-MM-DD",
     "totalRuns": master.totalRuns + 1,
     "runHistory": [...master.runHistory, "YYYY-MM-DD"],
     "totalContacts": nombre de contacts dans le master fusionne,
     "validated": nombre de contacts avec validated=true,
     "excluded": nombre d'exclus
   }
   ```

7. **Trier par score decroissant** et ecrire `exports/results/contacts-master.json`

8. **Afficher le resume de fusion** :
   ```
   === PASSE 5 — Fusion avec contacts-master.json ===
   Contacts dans le master precedent : XXXX
   Contacts dans le run courant : XXXX

   Nouveaux contacts ajoutes : XX
   Contacts mis a jour : XXXX
   Contacts valides conserves (non re-categorises) : XX
   Contacts absents du scan (conserves) : XX

   Master mis a jour : XXXX contacts totaux (XX valides)
   Fichier : exports/results/contacts-master.json
   ```

**Fin** :
```
=== Analyse terminee ===
Fichiers generes :
  - exports/results/contacts-master.json (XXXX contacts, source de verite)
  - exports/results/enriched-YYYY-MM-DD.json (XXXX contacts, snapshot)
  - exports/results/enriched-YYYY-MM-DD.csv (XXXX lignes, Excel-compatible)
  - exports/results/filter-suggestions-YYYY-MM-DD.json

Prochaines etapes suggerees :
  - Ouvrir viewer.html et charger contacts-master.json pour review
  - Valider les categorisations importantes (le viewer marquera validated=true)
  - Relancer /contact-analyze apres le prochain scan Contact Lens
```
