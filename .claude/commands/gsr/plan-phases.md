---
name: plan-phases
description: >
  Niveau 3 de la planification progressive. Génère les phases atomiques
  d'une story déjà détaillée. Chaque phase est reviewable et produit
  un incrément fonctionnel.
  Args: [epic-slug/story-slug] [--granularity=fine|standard|flexible]
human_ai_ratio: 40/60
---

# /gsr:plan-phases $ARGUMENTS

## 0. Charger la configuration

1. Déterminer le mode config : exécuter `.claude/gsr/bin/gsr-config.sh config-mode`
   - Si `jq` → exécuter `.claude/gsr/bin/gsr-config.sh dump plan` pour obtenir les valeurs
   - Si `claude` → lire `.claude/gsr/config.json` avec Read et extraire `workflow.plan.*`, `workflow.research.*` et `workflow.granularity`
   - Si `.claude/gsr/config.json` n'existe pas → utiliser les valeurs par défaut documentées dans les garde-fous ci-dessous

2. Valeurs chargées :

| Variable | Clé config | Défaut |
|----------|------------|--------|
| `$granularity` | `workflow.granularity` | `flexible` |
| `$max_phases_per_story` | `workflow.plan.max_phases_per_story` | `8` |
| `$max_review_cycles` | `workflow.plan.max_review_cycles` | `3` |
| `$timeout_minutes` | `workflow.plan.timeout_minutes` | `30` |
| `$research_enabled` | `workflow.research.enabled` | `true` |
| `$max_deep_research` | `workflow.research.max_deep` | `3` |
| `$max_quick_research` | `workflow.research.max_quick` | `5` |

Note : si `--granularity=` est passé en argument, il surcharge `$granularity` de la config.

## Parse arguments

1. Extraire depuis `$ARGUMENTS` :
   - `path` : format `[epic-slug]/[story-slug]` — ex: `01-auth/01-login`
   - `--granularity=` : `fine` | `standard` | `flexible` — défaut : `$granularity` (depuis config, ou `flexible`)
   - `--resume` : si présent, reprendre après une Deep Research

## Pré-checks

1. **STORY.md existe et est complété** :
   - Vérifier `docs/plan/epics/[epic-slug]/stories/[story-slug]/STORY.md` existe
   - Si absent → "Story pas encore détaillée. Lance `/gsr:plan-story [epic]/[story]` d'abord."
   - Vérifier qu'il contient des acceptance criteria
   - Si vide → "STORY.md existe mais est incomplet. Relance `/gsr:plan-story [epic]/[story]`."

2. **Phases pas déjà générées** :
   - Vérifier si le dossier `phases/` existe dans le dossier de la story
   - Si oui et contient des fichiers → "Des phases existent déjà pour cette story. Re-planifier ? [Oui] [Non]"
   - Si "Non" → stop

3. **Mode resume** :
   - Si `--resume` → vérifier `.claude/plan-session.md` section `## Pending Research`
   - Intégrer les résultats si disponibles

## Étape 1 — Préparer le contexte

1. Lire ou mettre à jour `.claude/plan-session.md` :
   ```
   Level: 3-phases
   Current: Phases pour [epic-slug]/[story-slug]
   Updated: [timestamp]
   ```

## Étape 2 — Décomposition en phases (Agent)

1. Spawner l'agent `gsr-planner` avec le prompt :
   ```
   Mode : phases
   Session : .claude/plan-session.md
   Story : docs/plan/epics/[epic-slug]/stories/[story-slug]/STORY.md
   Epic : docs/plan/epics/[epic-slug]/EPIC.md
   Roadmap : docs/plan/ROADMAP.md
   Granularité : $granularity
   Répertoire projet : [cwd]

   <config>
   max_phases_per_story=$max_phases_per_story
   max_review_cycles=$max_review_cycles
   research_enabled=$research_enabled
   </config>

   Découpe cette story en phases atomiques.
   Lire l'état actuel du projet pour s'adapter.
   Écrire le résultat dans ## Phases en cours de la session.
   ```

2. Vérifier le résultat dans `plan-session.md` section `## Phases en cours`

3. Vérifier les Research Gates → même logique que les autres niveaux

## Étape 3 — Review interactive

1. Afficher le résumé des phases :
   ```
   Phases pour : [story titre]
   Granularité : [niveau]
   Total : [N] phases

   | # | Phase | Objectif | Tasks | Estimé | Dépend de |
   |---|-------|----------|-------|--------|-----------|
   [table résumé]
   ```

2. Pour chaque phase, afficher le détail :
   ```
   Phase [NN] : [Nom]
   Objectif : [1 phrase]
   Estimation : [Nh]
   Dépend de : [phases précédentes]

   Tasks :
   1. [type] [nom]
      Fichiers : [liste]
      Critères : Given... when... then...
      Vérification : [commande]

   Review checklist :
   - [ ] [critère 1]
   - [ ] [critère 2]

   [OK] [Ajuster] [Fusionner avec précédente] [Découper]
   ```

3. Actions :
   - **OK** → phase validée
   - **Ajuster** → modifier (texte libre) → max 3 cycles
   - **Fusionner** → combiner avec la phase précédente
   - **Découper** → diviser en 2 phases plus petites

4. Après toutes les phases — validation :
   - [ ] Chaque phase a ≥ 1 task
   - [ ] Chaque task a des acceptance criteria
   - [ ] Chaque phase a une commande de vérification
   - [ ] Graphe acyclique
   - [ ] Phase 1 sans dépendance intra-story
   - Si échec → signaler et proposer d'ajuster

## Étape 4 — Génération (Agent)

1. Spawner l'agent `gsr-generator` avec le prompt :
   ```
   Mode : phases
   Session : .claude/plan-session.md
   Dossier story : docs/plan/epics/[epic-slug]/stories/[story-slug]/
   Répertoire projet : [cwd]

   Génère PLAN.md + CONTEXT.md par phase.
   Mets à jour STORY.md, EPIC.md et ROADMAP.md.
   Charger les templates depuis .claude/gsr/plan-output.md.
   Lire CLAUDE.md, architecture.md, SPEC.md pour les CONTEXT.md ciblés.
   ```

## Étape 5 — Résumé

```
Phases générées pour : [story titre]
Total : [N] phases
Estimation totale : [N]h

Fichiers créés :
  docs/plan/epics/[epic]/stories/[story]/phases/
  ├── [NN]-[slug]/PLAN.md + CONTEXT.md (× [N])

Ordre d'exécution :
  1. Phase [NN]: [Nom] ([Nh])
  2. Phase [NN]: [Nom] ([Nh])
  ...

Prochaine étape :
  Exécuter la première phase : /gsr:execute [epic]/[story]/[phase]
  Ou planifier une autre story : /gsr:plan-story [epic]/[autre-story]
```

## Mise a jour du suivi

Mettre a jour `docs/GSR-STATUS.md` selon `.claude/gsr/status-output.md` section `<update-plan>` :
- Plan : incrementer Phases generees, Niveau → `3-phases`
- Detail par Epic : incrementer Phases pour l'epic, recalculer Statut (selon `<update-plan-epic-statut>`)
- Si toutes stories de tous epics ont des phases → Pipeline Plan → `OK`
- Historique : "Phases generees pour [epic]/[story] ([N] phases)"

## Garde-fous

| Limite | Valeur (depuis config) | Comportement |
|--------|------------------------|-------------|
| Phases par story | `$max_phases_per_story` (défaut: 8) | "Story trop grosse. Découper en 2 stories ?" |
| Tasks par phase | 5 max | "Phase trop chargée. Découper ?" |
| Cycles review / phase | `$max_review_cycles` (défaut: 3) | "On valide et ajuste en exécution." |
| Durée | `$timeout_minutes` min (défaut: 30) | "On approche de $timeout_minutes min. On finalise ?" |
