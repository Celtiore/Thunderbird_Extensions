# Changelog - Email Cleaner

## [2.2-fork] - 2026-03-24

### Améliorations
- **Icône dans la barre d'outils** : ajout d'un bouton dans la barre d'outils Thunderbird (unified toolbar) qui ouvre directement les options de l'extension. Plus besoin de passer par le panneau des modules complémentaires.

### Corrections de bugs
- **searchAllFolders ignoré** : le paramètre "Explorer tous les dossiers" était sauvegardé dans les options mais jamais lu par le script d'arrière-plan. Seuls les dossiers Inbox étaient parcourus, rendant les dossiers locaux inaccessibles.
- **recipients jamais vérifiés** : les destinataires configurés dans les options n'étaient jamais utilisés lors du filtrage des messages à supprimer.
- **includeTrash ignoré** : l'option "Inclure la corbeille" n'était pas lue par le background.
- **batchSize ignoré** : la taille de lot configurée n'était jamais utilisée (toujours la valeur par défaut de 100).
- **Validation trop stricte** : l'extension refusait de s'exécuter si aucun expéditeur n'était configuré, même avec des filtres par destinataire ou par objet.
- **Matching par domaine avec point** : `.alibaba.com` générait un filtre `@.alibaba.com` qui ne matchait jamais. Corrigé pour matcher le domaine et tous ses sous-domaines.

### Nouvelles fonctionnalités

#### Filtrage par domaine
- `@tf1.fr` ou `tf1.fr` : matche toutes les adresses du domaine exact
- `.alibaba.com` : matche le domaine et tous ses sous-domaines (promo.alibaba.com, mail.alibaba.com, etc.)
- Fonctionne quel que soit le mode de correspondance (exacte ou partielle)
- Logique centralisée dans une fonction `matchFilter` unique, utilisée par tous les filtres

#### Filtrage par taille de message
- Nouveau bloc "Filtre par taille" dans les paramètres
- Seuil configurable en Ko (défaut : 5000 Ko = 5 Mo)
- Option "Appliquer le filtre de taille seul" : supprime les gros messages indépendamment des filtres d'expéditeur/objet
- Pris en charge par le mode simulation (Dry Run)

#### Sélecteur de dossiers en arborescence
- Remplacement de la simple liste de comptes à cocher par un arbre complet des dossiers
- Affichage de toute l'arborescence avec icônes par type de dossier et badges colorés (Inbox, Envoyés, Corbeille, Spam, Brouillons)
- Noeuds dépliables/repliables avec flèches
- Boutons globaux : Tout déplier, Tout replier, Tout sélectionner, Tout désélectionner

#### Sélection en cascade (tri-state)
- Cocher un dossier parent coche automatiquement tous ses sous-dossiers
- Décocher un dossier parent décoche tous ses sous-dossiers
- Sélection partielle : le parent affiche un état indéterminé (case semi-cochée)
- Cascade vers le haut et vers le bas sur toute la profondeur
- L'état est recalculé au chargement depuis la sélection sauvegardée
- Cocher un compte cascade sur tous ses dossiers

#### Mode simulation (Dry Run)
- Nouveau bouton "Simuler (Dry Run)" dans la section Actions
- Parcourt tous les dossiers/comptes sélectionnés avec les mêmes filtres
- N'effectue aucune suppression
- Affiche un rapport avec :
  - Nombre total de messages analysés et correspondants
  - Répartition par type de filtre (expéditeur, destinataire, objet, taille)
  - Répartition par dossier
  - Tableau détaillé des 500 premiers messages (date, auteur, objet, dossier, raison)

#### Planification automatique
- Nouveau bloc "Nettoyage automatique" dans les paramètres
- Fréquences configurables : toutes les heures, 6h, 12h, 1 jour, 1 semaine
- Utilise l'API `browser.alarms` de Thunderbird
- Premier déclenchement 5 minutes après le démarrage
- Affiche la date du dernier nettoyage et du prochain prévu

#### Historique des nettoyages
- Nouvelle section "Historique des nettoyages" dans les options
- Tableau avec : date, type (Manuel/Auto), messages analysés, supprimés, erreurs, durée
- Badges colorés pour distinguer les nettoyages manuels et automatiques
- Conservation des 50 dernières entrées
- Bouton "Effacer l'historique"
- Rafraîchissement automatique après chaque nettoyage

#### Compteurs en temps réel dans l'arborescence
- Pendant le nettoyage, un badge apparaît à côté de chaque dossier traité
- Format : `supprimés/analysés`
- Badge rouge si des suppressions ont eu lieu, gris sinon

### Améliorations de l'interface
- Section Paramètres réorganisée en blocs encadrés (fieldsets) avec titres clairs
- Bloc "Quand supprimer ?" avec le champ jours simplifié
- Bloc "Comment comparer les adresses ?" avec un encadré d'exemples concrets
- Bloc "Où chercher ?" avec rappel vers l'arborescence
- Bloc "Performance" avec explication pratique
- Suppression des labels doublés et des descriptions trop techniques

## [2.2] - 2024 (version originale par DomoGeek.ovh)

- Filtrage par expéditeur et par objet
- Sélection des comptes à traiter
- Menu contextuel pour ajouter un expéditeur
- Import/export des filtres
- Support multilingue (FR, EN, DE)
