# Email Cleaner - Extension Thunderbird

Nettoyage automatique des emails selon des critères personnalisables.

## Informations

| | |
|---|---|
| **Auteur original** | DomoGeek.ovh ([https://www.domogeek.ovh](https://www.domogeek.ovh)) |
| **Contact** | pluginthunderbird@gmail.com |
| **Version originale** | 2.2 |
| **Version modifiée** | 2.2-fork |
| **ID** | `email-cleaner@example.com` |
| **Compatibilité** | Thunderbird 128.0 - 140.* |
| **Licence** | Licence originale de DomoGeek.ovh |

## Fonctionnalités

### Originales
- Suppression automatique des emails par expéditeur
- Filtrage par objet de message (commence par, contient, se termine par)
- Sélection des comptes à traiter
- Menu contextuel pour ajouter un expéditeur depuis un message
- Import/export des filtres
- Support multilingue (FR, EN, DE)

### Ajoutées dans ce fork
- **Accès depuis la barre d'outils** : icône cliquable dans la toolbar Thunderbird pour ouvrir les options directement
- **Filtrage par domaine** : `@tf1.fr`, `tf1.fr` ou `.alibaba.com` (avec sous-domaines)
- **Filtrage par destinataire** : supprimer les messages envoyés à certaines adresses
- **Filtrage par taille** : supprimer les messages dépassant une taille configurable
- **Sélecteur de dossiers en arborescence** : choisir précisément quels dossiers scanner
- **Sélection en cascade** : cocher un dossier parent coche/décoche ses sous-dossiers, état indéterminé pour sélection partielle
- **Support des dossiers locaux** : l'option "Explorer tous les dossiers" fonctionne réellement
- **Mode simulation (Dry Run)** : visualiser ce qui serait supprimé sans rien supprimer
- **Planification automatique** : nettoyage périodique configurable (1h à 1 semaine)
- **Historique des nettoyages** : journal des 50 dernières exécutions (manuelles et automatiques)
- **Compteurs en temps réel** : badges de progression par dossier pendant le nettoyage
- **Interface repensée** : paramètres organisés en blocs clairs avec exemples

### Corrections de bugs
- `searchAllFolders` était sauvegardé mais jamais lu par le background
- `recipients` configurés mais jamais vérifiés lors de la suppression
- `includeTrash` ignoré dans le traitement
- `batchSize` configurable mais jamais utilisé (toujours 100 par défaut)
- Validation trop stricte : refusait de s'exécuter sans expéditeurs même avec d'autres filtres
- `.domaine.com` provoquait un matching sur `@.domaine.com` (invalide)

## Formats de filtrage par adresse

| Format | Signification | Exemple de match |
|---|---|---|
| `user@example.com` | Adresse exacte | `user@example.com` uniquement |
| `@tf1.fr` | Tout le domaine | `info@tf1.fr`, `news@tf1.fr` |
| `tf1.fr` | Idem (domaine implicite) | `info@tf1.fr`, `news@tf1.fr` |
| `.alibaba.com` | Domaine + sous-domaines | `user@alibaba.com`, `user@promo.alibaba.com` |

## Installation

Installer le fichier `email-cleaner@example.com.xpi` via le gestionnaire de modules de Thunderbird.
