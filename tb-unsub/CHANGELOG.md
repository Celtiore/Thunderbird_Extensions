# Changelog - Inventaire & Désabonnement

## [0.1.0] - 2026-08-19

Squelette fonctionnel issu du cadrage du 19 août 2026 (voir `RAPPORT.md`) :

- Sélecteur de comptes et profondeur de scan (popup), choix mémorisé dans `storage.local`
- Inventaire des expéditeurs de newsletters (un `getFull()` par expéditeur, jamais par message)
- Désabonnement RFC 8058 (POST one-click), repli `mailto:` en brouillon, lien externe
- Fiche persistante par expéditeur et journal de preuves horodaté (append-only)

Regroupé dans ce monorepo le 2026-09-04 (ex-dossier `files/` du workspace, livrable claude.ai).
