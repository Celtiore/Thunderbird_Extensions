# Thunderbird Extensions

Extensions Thunderbird maintenues et corrigees pour assurer la compatibilite avec les versions recentes (128 — 148).

## Extensions

### [Email Cleaner](email-cleaner/README.md)

Nettoyage automatique des emails selon des criteres personnalisables (expediteurs, destinataires, domaines, objets, taille). Fork ameliore de l'extension originale.

- **Auteur original** : [DomoGeek.ovh](https://www.domogeek.ovh) (2024)
- **Contact** : pluginthunderbird@gmail.com
- **Licence** : Licence originale DomoGeek.ovh
- **Copyright** : © 2024 DomoGeek.ovh

> [Fonctionnalites detaillees, configuration et changelog](email-cleaner/README.md)

### [Xpunge](xpunge/README.md)

Vide la corbeille et les dossiers spam, compacte les dossiers pour plusieurs comptes en un clic. Fork de compatibilite de l'extension originale.

- **Auteur original** : [Theodore Tegos](mailto:xpungetb@gmail.com) (2005)
- **Rewrite TB 128** : [John Bieling](https://github.com/jobisoft)
- **Site officiel** : [theodoretegos.net](http://www.theodoretegos.net/mozilla/tb/)
- **GitHub original** : [theodore-tegos/xpunge-tb](https://github.com/theodore-tegos/xpunge-tb)
- **Licence** : MPL 2.0 / GPL 2.0 / LGPL 2.1
- **Contributeurs** : Micz (IT), Dagobert_78 (FR), Joergen (DA), kiki (JA), Marky Mark DE (DE), Piryatinskiy Vitaliy (RU)

> [Fonctionnalites detaillees et changelog](xpunge/README.md)

### [Contact Lens](contact-lens/README.md)

Analyse et structure les contacts depuis vos emails. Scanne tous les comptes et dossiers pour extraire, dedupliquer et classer les contacts par type de relation. Suppression d'emails avec double confirmation et liste d'exclusion persistante.

> [Fonctionnalites detaillees, pipeline d'analyse et changelog](contact-lens/README.md)

### [Inventaire & Désabonnement](tb-unsub/README.md)

Inventorie les expéditeurs de newsletters sur les comptes sélectionnés, se désabonne via RFC 8058 (POST one-click) quand c'est possible et garde une fiche par expéditeur avec un journal de preuves horodaté. Prototype v0.1 (MailExtension Manifest V3), cadrage dans `tb-unsub/RAPPORT.md`.

- **Auteur** : Patrick (projet original, 2026)
- **ID** : `inventaire-desabonnement@fteventspro.app`

> [Installation, briques et feuille de route](tb-unsub/README.md)

## Compatibilite

| Extension | Version | Thunderbird min | Thunderbird max |
|---|---|---|---|
| [Email Cleaner](email-cleaner/README.md) | 2.2+ | 128.0 | 148.* |
| [Xpunge](xpunge/README.md) | 5.0.2+ | 128.2.0 | 148.* |
| [Contact Lens](contact-lens/README.md) | 1.1 | 128.0 | 148.* |
| [Inventaire & Désabonnement](tb-unsub/README.md) | 0.1.0 | 128.0 | — (pas de max déclaré) |

## Installation

1. Ouvrir Thunderbird
2. Menu > Modules complementaires et themes
3. Roue dentee > Installer un module depuis un fichier
4. Selectionner le fichier `.xpi` souhaite

## Licence

Chaque extension conserve la licence de son auteur original. Voir les README individuels pour les details.
