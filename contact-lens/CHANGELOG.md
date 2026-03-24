# Changelog - Contact Lens

## [1.1] - 2026-03-24

### Nouvelles fonctionnalités
- **Export enrichi** : collecte des 10 derniers sujets de messages pour chaque contact
- **Body snippets** : extraction des 300 premiers caractères des 3 derniers messages (contacts bidirectionnels uniquement)
- **Overlay de progression** pour l'export enrichi avec bouton d'annulation
- **Sélecteur de dossiers en arborescence** avec icônes, badges par type, dépliable/repliable, cascade tri-state
- **Page d'options** avec bouton de réinitialisation des données et infos de compatibilité
- **Récap du scan** : détail des comptes et dossiers scannés après chaque scan
- **Détection améliorée des dossiers Envoyés** : fallback par nom en plus du type API
- **Suppression d'emails** : checkbox par contact, sélection multiple, bouton supprimer avec double confirmation (dialog + saisie "SUPPRIMER")
- **Liste d'exclusion** : les contacts supprimés sont automatiquement exclus des prochains scans
- **Import/export des exclusions** : sauvegarde et restauration de la liste d'exclusion (persiste après réinstallation)
- **Export automatique des exclusions** proposé après chaque suppression
- **Skill `/contact-analyze`** : pipeline d'analyse en 5 passes (nettoyage, analyse Claude, consolidation, validation filtres, fusion master)
- **Viewer HTML** : visualiseur interactif avec cards cliquables, filtres, panneau détail, bouton copier email
- **Fichier `contacts-master.json`** : source de vérité cumulative entre les runs d'analyse

### Améliorations
- Format de nom d'export : `contact-lens-YYYY-MM-DD.json` (suppression du préfixe `export-`)
- Colonne "Sujets" dans l'export CSV (sujets séparés par ` | `)
- Exploration récursive des sous-dossiers via `browser.folders.getSubFolders(folder.id)`
- Compatibilité étendue : Thunderbird 128.0 — 148.*
- Alignement des colonnes dans le dashboard (table-layout fixed)

## [1.0] - 2026-03-24

### Fonctionnalités
- Scan de tous les comptes et dossiers email (hors brouillons, corbeille, spam, boîte d'envoi, modèles)
- Extraction et dédoublonnage des contacts par adresse email normalisée
- Classification de la direction de relation (bidirectionnel, reçu seulement, envoyé seulement)
- Scan incrémental avec horodatage par dossier (rapide après le premier passage)
- Bouton "Forcer un rescan complet" pour repartir de zéro
- Dashboard avec arborescence par direction et par compte
- Tri par colonne (email, nom, reçus, envoyés, dates)
- Recherche en temps réel avec debounce
- Barre de progression pendant le scan
- Export CSV (UTF-8 BOM, séparateur point-virgule pour Excel FR)
- Export JSON avec métadonnées
- Localisation français et anglais
- Surveillance de la taille du stockage avec avertissement à 8 Mo
