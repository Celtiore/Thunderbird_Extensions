# Xpunge - Extension Thunderbird

Vide la corbeille et les dossiers spam, compacte les dossiers pour plusieurs comptes Thunderbird en un seul clic.

## Informations

| | |
|---|---|
| **Auteurs originaux** | Theodore Tegos, John Bieling |
| **Site web** | [http://www.theodoretegos.net/mozilla/tb/](http://www.theodoretegos.net/mozilla/tb/) |
| **GitHub original** | [theodore-tegos/xpunge-tb](https://github.com/theodore-tegos/xpunge-tb) |
| **Version originale** | 5.0.2 |
| **Version modifiée** | 5.0.2-fork |
| **ID** | `{786abda0-fd14-d247-bf69-38b2fc18491b}` |
| **Compatibilité originale** | Thunderbird 128.2.0 - 131.* |
| **Compatibilité modifiée** | Thunderbird 128.2.0 - 148.* |
| **Licence** | Licence originale des auteurs |

## Fonctionnalités

- Vider la corbeille de tous les comptes ou de comptes spécifiques
- Vider les dossiers spam/indésirables
- Compacter les dossiers pour récupérer l'espace disque
- Bouton dans la barre d'outils pour exécution rapide
- Entrée dans le menu Outils
- Menu contextuel sur les dossiers
- Planification automatique
- Support multilingue (FR, EN, DE, DA, EL, IT, JA, RU)

## Modification apportée

La seule modification dans ce fork est le relèvement de `strict_max_version` de `131.*` à `148.*` dans le `manifest.json` pour permettre l'installation sur Thunderbird 140 ESR et 148 Release.

Le code de l'extension utilise des APIs modernes (`ChromeUtils.importESModule`, WebExtension standard) et est compatible sans autre changement.

## Note

Un bug connu dans Thunderbird 140 (bugzilla #1977840) peut affecter le compactage des dossiers spam non rafraîchis. C'est un bug côté Thunderbird, pas de l'extension.

## Installation

Installer le fichier `{786abda0-fd14-d247-bf69-38b2fc18491b}.xpi` via le gestionnaire de modules de Thunderbird.
