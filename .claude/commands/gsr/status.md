---
name: status
description: >
  Affiche l'etat d'avancement global du projet dans le workflow GSR.
  Peut regenerer le fichier de suivi depuis l'etat reel du projet.
  Args: [--rebuild]
human_ai_ratio: 10/90
---

# /gsr:status $ARGUMENTS

## Parse arguments

- `--rebuild` : regenerer `docs/GSR-STATUS.md` depuis l'etat reel du projet (scan fichiers)

## Mode --rebuild

Si `--rebuild` est present, OU si `docs/GSR-STATUS.md` n'existe pas :

1. Charger `.claude/gsr/status-output.md` section `<rebuild-logic>`
2. Scanner le projet :
   - `discovery.md` / `.claude/discovery-session.md`
   - `CLAUDE.md`, `SPEC.md`, `docs/agent_docs/architecture.md`, etc.
   - `docs/plan/ROADMAP.md`, arborescence `docs/plan/epics/`
3. Generer `docs/GSR-STATUS.md` depuis le template `<status-template>`
4. Remplir chaque section selon l'etat detecte

## Mode normal (lecture)

1. Lire `docs/GSR-STATUS.md`
   - Si absent → "Aucun fichier de suivi. Lance `/gsr:status --rebuild` pour le generer, ou demarre un workflow avec `/gsr:discover`."

2. Afficher le contenu formatte :
   - Pipeline avec statuts
   - Phase active mise en evidence
   - Si Plan en cours → inclure le detail par epic (comme plan-status)
   - Prochaine action suggeree

## Suggestions intelligentes

Selon la phase active, proposer la prochaine commande logique :

| Phase active | Suggestion |
|-------------|------------|
| `--` (rien) | `/gsr:discover "description"` |
| Discovery en cours | `/gsr:discover-resume` |
| Discovery OK | `/gsr:bootstrap` |
| Discovery Partiel | `/gsr:discover-resume` ou `/gsr:bootstrap --minimal` |
| Bootstrap OK | `/gsr:plan` |
| Plan en cours | Depend du niveau — meme logique que plan-status |
| Plan OK | `/gsr:execute` (a venir) |
