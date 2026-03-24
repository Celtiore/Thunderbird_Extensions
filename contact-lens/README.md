# Contact Lens - Extension Thunderbird

Analyse, structure et nettoie les contacts depuis vos emails. Scanne tous les comptes et dossiers pour extraire, dedupliquer, classer et supprimer les contacts par type de relation.

## Informations

| | |
|---|---|
| **Version** | 1.1 |
| **ID** | `contact-lens@example.com` |
| **Compatibilite** | Thunderbird 128.0 — 148.* |
| **Locales** | Francais (defaut), Anglais |

## Fonctionnalites

### Scan & Extraction
- **Scan complet** : parcourt tous les comptes, dossiers et sous-dossiers (hors brouillons, corbeille, spam, boite d'envoi, modeles)
- **Extraction intelligente** : extrait les contacts depuis les headers From/To/Cc/Bcc
- **Dedoublonnage** : une entree unique par adresse email (normalisee en minuscules)
- **Scan incremental** : horodatage par dossier, seuls les nouveaux messages sont analyses au rescan
- **Rescan complet** : bouton pour reinitialiser les timestamps et tout rescanner
- **Detection des dossiers Envoyes** : par type API + fallback par nom (Envoyes, Sent, Sent Messages, etc.)

### Classification
- **Bidirectionnel** : vous avez envoye ET recu des messages (contacts actifs)
- **Recu seulement** : vous recevez mais ne repondez jamais (newsletters, pubs, services)
- **Envoye seulement** : vous ecrivez sans reponse

### Dashboard
- **Arborescence** : contacts groupes par direction (Bidirectionnel / Recu / Envoye) puis par compte
- **Tri par colonne** : email, nom, recus, envoyes, premier contact, dernier contact
- **Recherche en temps reel** avec debounce 300ms
- **Barre de progression** pendant le scan avec detail des dossiers scannes
- **Recap du scan** : detail des comptes et dossiers scannes apres chaque scan (nombre de dossiers, dossiers Envoyes detectes)

### Selecteur de dossiers
- **Arborescence hierarchique** avec exploration recursive de tous les sous-dossiers
- **Icones par type** : Inbox, Envoyes, Spam, Corbeille, Brouillons, Archives
- **Badges colores** par type de dossier
- **Depliable/repliable** avec boutons Tout deplier / Tout replier
- **Selection en cascade** : cocher un compte ou un dossier parent coche tous ses enfants
- **Tri-state** : etat indetermine sur les parents si selection partielle
- **Dossiers exclus** affiches en grise/barre (drafts, trash, junk, outbox, templates)
- **Persistance** : la selection est sauvegardee et restauree au rechargement

### Export
- **CSV** : UTF-8 avec BOM, separateur point-virgule pour Excel FR, colonne Sujets
- **JSON** : export standard avec metadonnees
- **Export enrichi** : collecte des 10 derniers sujets par contact + body snippets (300 premiers chars des 3 derniers messages) pour les contacts bidirectionnels
- **Overlay de progression** pour l'export enrichi avec possibilite d'annulation
- **Nom de fichier date** : `contact-lens-YYYY-MM-DD.json`

### Suppression d'emails
- **Checkbox par contact** dans le tableau avec selection multiple
- **Barre d'actions flottante** : apparait des qu'un contact est coche, avec compteur
- **Tout cocher / Tout decocher** : boutons de selection globale
- **Double confirmation** :
  1. Premier dialog : "Vous allez supprimer TOUS les emails de X contacts. Continuer ?"
  2. Deuxieme dialog : "ACTION IRREVERSIBLE — tapez SUPPRIMER pour confirmer" avec champ texte obligatoire
- **Progression en temps reel** pendant la suppression
- **Annulation** possible en cours de suppression
- **Suppression definitive** : utilise `browser.messages.delete(ids, true)` (emails + envoyes)
- **Exclusion automatique** : les adresses supprimees sont ajoutees a une liste d'exclusion et ignorees lors des prochains scans

### Liste d'exclusion
- **Automatique** : alimentee lors de la suppression d'emails
- **Persistante** : sauvegardee dans `browser.storage.local`
- **Gerable** depuis la page d'options (cle a molette dans les modules) :
  - Liste scrollable triee alphabetiquement
  - Bouton "Retirer" par email pour reautoriser un contact
  - Bouton "Vider la liste" avec confirmation
- **Respect lors du scan** : les emails exclus sont ignores par `upsertContact()`
- **Reinitialisation** : videe lors du reset complet des donnees

### Page d'options
- **Compatibilite** affichee (Thunderbird 128.0 — 148.*)
- **Nombre de contacts** en memoire
- **Reinitialisation** : bouton pour supprimer toutes les donnees (contacts, timestamps, selection, exclusions)
- **Liste d'exclusion** : gestion complete (voir ci-dessus)

### Bouton toolbar
- **Icone dans la barre d'outils** Thunderbird (unified toolbar)
- **Clic** ouvre le dashboard dans un onglet complet

## Pipeline d'analyse (Claude Code)

L'extension est couplee avec un skill Claude Code `/contact-analyze` qui enrichit les exports :

### Structure des fichiers
```
exports/
  scans/                          <- Exports bruts dates (JSON + CSV)
  filters/                        <- Filtres editables
    exclusions.json                 <- ~287 domaines, ~21 patterns regex a exclure
    keywords.json                   <- 8 categories de mots-cles (FR + EN)
  results/                        <- Sortie du skill
    contacts-master.json            <- Source de verite cumulative
    enriched-YYYY-MM-DD.json        <- Snapshot du run
    enriched-YYYY-MM-DD.csv         <- Version Excel
    filter-suggestions-*.json       <- Suggestions de nouveaux filtres
    viewer.html                     <- Visualiseur interactif
```

### Skill `/contact-analyze` — 5 passes
1. **Nettoyage** (0 tokens) : exclusions par domaine/pattern/email + scoring qualite
2. **Analyse Claude** : categorisation par batch (client, prospect, fournisseur, admin, personnel, service, media, association)
3. **Consolidation** : JSON enrichi + CSV + suggestions de filtres
4. **Validation** : suggestions de filtres avec validation humaine interactive
5. **Fusion** : merge avec `contacts-master.json` (contacts valides conserves entre les runs)

### Viewer HTML
- **Chargement** de tout fichier JSON enrichi ou du master
- **Cards cliquables** par categorie avec filtrage
- **Tableau triable** avec toutes les colonnes
- **Filtres** : recherche, categorie, action, direction, confiance
- **Panneau detail** : clic sur un contact ouvre sujets, body snippets, contexte
- **Bouton copier** : copie l'email dans le presse-papier

## Donnees collectees par contact

| Champ | Description |
|---|---|
| Email | Adresse normalisee (minuscules, trim) |
| Nom | Nom d'affichage le plus recent |
| Compte(s) | Comptes email ou le contact apparait |
| Recus | Nombre de messages recus de ce contact |
| Envoyes | Nombre de messages envoyes a ce contact |
| Premier contact | Date du premier message |
| Dernier contact | Date du dernier message |
| Direction | bidirectional / receivedOnly / sentOnly |
| Sujets | 10 derniers sujets de messages |
| Body snippets | 300 premiers chars des 3 derniers messages (bidirectionnels uniquement) |

## Permissions

| Permission | Usage |
|---|---|
| `storage` | Persistance des contacts, timestamps, selection, exclusions |
| `accountsRead` | Enumeration des comptes email |
| `accountsFolders` | Enumeration des dossiers et sous-dossiers |
| `messagesRead` | Lecture des messages (headers, sujets, body) |
| `messagesDelete` | Suppression definitive des emails |

## Installation

1. Desinstaller l'ancienne version si presente
2. Redemarrer Thunderbird
3. Installer `contact-lens@example.com.xpi` via Menu > Modules complementaires > Installer depuis un fichier
4. Le bouton Contact Lens apparait dans la personnalisation de la barre d'outils
