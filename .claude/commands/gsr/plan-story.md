---
name: plan-story
description: >
  Niveau 2 de la planification progressive. Détaille une story spécifique
  avec acceptance criteria, composants, risques et estimation. Prend en
  compte l'état actuel du projet pour s'adapter aux changements.
  Args: [epic-slug/story-slug]
human_ai_ratio: 40/60
---

# /gsr:plan-story $ARGUMENTS

## 0. Charger la configuration

1. Déterminer le mode config : exécuter `.claude/gsr/bin/gsr-config.sh config-mode`
   - Si `jq` → exécuter `.claude/gsr/bin/gsr-config.sh dump plan` pour obtenir les valeurs
   - Si `claude` → lire `.claude/gsr/config.json` avec Read et extraire `workflow.plan.*` et `workflow.research.*`
   - Si `.claude/gsr/config.json` n'existe pas → utiliser les valeurs par défaut

2. Valeurs chargées :

| Variable | Clé config | Défaut |
|----------|------------|--------|
| `$max_review_cycles` | `workflow.plan.max_review_cycles` | `3` |
| `$timeout_minutes` | `workflow.plan.timeout_minutes` | `30` |
| `$research_enabled` | `workflow.research.enabled` | `true` |
| `$max_deep_research` | `workflow.research.max_deep` | `3` |
| `$max_quick_research` | `workflow.research.max_quick` | `5` |

## Parse arguments

1. Extraire depuis `$ARGUMENTS` :
   - `path` : format `[epic-slug]/[story-slug]` — ex: `01-auth/01-login`
   - `--resume` : si présent, reprendre après une Deep Research

## Pré-checks

1. **ROADMAP.md existe** :
   - Vérifier `docs/plan/ROADMAP.md` existe
   - Si absent → "Aucun plan trouvé. Lance `/gsr:plan` d'abord."

2. **Epic identifié** :
   - Vérifier `docs/plan/epics/[epic-slug]/EPIC.md` existe
   - Si absent → lister les epics disponibles et demander de choisir

3. **Story identifiée** :
   - Vérifier que le dossier `docs/plan/epics/[epic-slug]/stories/[story-slug]/` existe
   - Si absent → lister les stories de l'epic et demander de choisir

4. **Story pas déjà détaillée** :
   - Vérifier si `STORY.md` existe déjà dans le dossier de la story
   - Si oui → "Cette story est déjà détaillée. Veux-tu la re-planifier ? [Oui] [Non]"
   - Si "Non" → stop

5. **Mode resume** :
   - Si `--resume` → vérifier `.claude/plan-session.md` section `## Pending Research`
   - Si pas de recherche en attente → "Aucune recherche en attente. Lancement normal."
   - Si recherche complétée → demander les résultats, les intégrer dans la session

## Étape 1 — Préparer le contexte

1. Lire ou créer `.claude/plan-session.md` :
   - Si existe → vérifier qu'il contient `## Analyse` (sinon, c'est une session corrompue)
   - Si n'existe pas → "Session manquante. Relance `/gsr:plan` pour initialiser."

2. Mettre à jour la session :
   ```
   Level: 2-story
   Current: Détail story [epic-slug]/[story-slug]
   Updated: [timestamp]
   ```

## Étape 2 — Détail de la story (Agent)

1. Spawner l'agent `gsr-planner` avec le prompt :
   ```
   Mode : story
   Session : .claude/plan-session.md
   Epic : docs/plan/epics/[epic-slug]/EPIC.md
   Story slug : [story-slug]
   Répertoire projet : [cwd]

   <config>
   max_review_cycles=$max_review_cycles
   research_enabled=$research_enabled
   </config>

   Détaille cette story : acceptance criteria, composants, risques,
   estimation, pré-requis, notes d'implémentation.
   Lire l'état actuel du projet pour s'adapter.
   Écrire le résultat dans ## Story en cours de la session.
   ```

2. Vérifier le résultat dans `plan-session.md` section `## Story en cours`

3. Vérifier les Research Gates dans `## Research Needed` :
   - Si présent → même logique que `/gsr:plan` étape 2 :
     - Proposer [A] Quick / [B] Deep / [C] Continuer sans
     - Si Deep → sauvegarder + instructions pour reprendre via `/gsr:plan-story [path] --resume`

## Étape 3 — Review interactive

1. Afficher le draft complet de la story :
   ```
   Story : [titre complet]
   Epic : [N] — [Nom]
   Estimation : [fourchette]

   Acceptance Criteria :
   | # | Given | When | Then |
   |---|-------|------|------|
   [table]

   Composants : [liste]
   Risques : [liste ou "Aucun identifié"]
   Pré-requis : [liste ou "Aucun"]

   Notes d'implémentation :
   [observations]

   [Valider] [Ajuster] [Recherche]
   ```

2. Si **Valider** → continuer vers la génération

3. Si **Ajuster** → demander quoi changer :
   - Appliquer les modifications dans la session
   - Re-présenter le draft modifié
   - Max 3 cycles

4. Si **Recherche** → déclencher Research Gate manuellement :
   - Demander la question à résoudre
   - Proposer [A] Quick / [B] Deep / [C] Annuler

## Étape 4 — Génération (Agent)

1. Spawner l'agent `gsr-generator` avec le prompt :
   ```
   Mode : story
   Session : .claude/plan-session.md
   Dossier story : docs/plan/epics/[epic-slug]/stories/[story-slug]/
   Répertoire projet : [cwd]

   Génère STORY.md et mets à jour EPIC.md et ROADMAP.md.
   Charger les templates depuis .claude/gsr/plan-output.md.
   ```

## Étape 5 — Résumé

```
Story détaillée : [titre]
Epic : [N] — [Nom]
Acceptance criteria : [N]
Estimation : [fourchette]

Fichier créé : docs/plan/epics/[epic]/stories/[story]/STORY.md

Prochaine étape :
  Détailler une autre story : /gsr:plan-story [epic]/[autre-story]
  Ou générer les phases : /gsr:plan-phases [epic]/[story]
```

## Mise a jour du suivi

Mettre a jour `docs/GSR-STATUS.md` selon `.claude/gsr/status-output.md` section `<update-plan>` :
- Plan : incrementer Stories detaillees, Niveau → `2-story`
- Detail par Epic : incrementer Detaillees pour l'epic concerne, recalculer Statut
- Historique : "Story [epic]/[story] detaillee"
