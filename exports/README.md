# exports/ — Structure des donnees Contact Lens

## Organisation

```
exports/
  scans/                              <- Exports bruts de Contact Lens (generes par l'extension)
    contact-lens-YYYY-MM-DD.json        Scan enrichi (contacts + sujets + body snippets)
    contact-lens-YYYY-MM-DD.csv         Version CSV

  filters/                            <- Fichiers de filtres (editables)
    exclusions.json                     Domaines, emails, patterns regex a exclure
    keywords.json                       Mots-cles pour categoriser les sujets
    exclusions.example.json             Exemple de fichier d'exclusions
    keywords.example.json               Exemple de fichier de mots-cles

  results/                            <- Sortie du skill /contact-analyze
    contacts-master.json                Source de verite cumulative (persiste entre les runs)
    enriched-YYYY-MM-DD.json            Snapshot du run (contacts categorises)
    enriched-YYYY-MM-DD.csv             Version Excel (UTF-8 BOM, separateur ;)
    filter-suggestions-YYYY-MM-DD.json  Suggestions de nouveaux filtres
    viewer.html                         Visualiseur interactif (ouvrir dans un navigateur)
```

## Utilisation

1. **Installer Contact Lens** dans Thunderbird et lancer un scan
2. **Exporter** depuis le dashboard (bouton "Exporter JSON") → sauvegarder dans `scans/`
3. **Configurer les filtres** : copier `exclusions.example.json` en `exclusions.json` et adapter
4. **Lancer l'analyse** : `/contact-analyze` dans Claude Code
5. **Consulter les resultats** : ouvrir `results/viewer.html` et charger `contacts-master.json`

## Fichiers exemples

Les fichiers `*.example.json` sont des templates. Pour les utiliser :
```bash
cp exports/filters/exclusions.example.json exports/filters/exclusions.json
cp exports/filters/keywords.example.json exports/filters/keywords.json
```

Puis adapter le contenu a vos besoins (ajouter vos domaines, mots-cles, etc.).
