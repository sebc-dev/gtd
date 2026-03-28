---
name: plan
description: >
  Niveau 1 de la planification progressive. Analyse les documents bootstrap
  et génère un ROADMAP avec Epics et Stories. Ne descend pas au niveau des
  phases — celles-ci seront planifiées au fur et à mesure avec /gsr:plan-phases.
  Args: [path/to/SPEC.md] [--granularity=fine|standard|flexible]
human_ai_ratio: 40/60
---

# /gsr:plan $ARGUMENTS

## 0. Charger la configuration

1. Déterminer le mode config : exécuter `.claude/gsr/bin/gsr-config.sh config-mode`
   - Si `jq` → exécuter `.claude/gsr/bin/gsr-config.sh dump plan` pour obtenir toutes les valeurs en un appel
   - Si `claude` → lire `.claude/gsr/config.json` avec Read et extraire `workflow.plan.*`, `workflow.research.*` et `workflow.granularity`
   - Si `.claude/gsr/config.json` n'existe pas → utiliser les valeurs par défaut documentées dans les garde-fous ci-dessous

2. Valeurs chargées :

| Variable | Clé config | Défaut |
|----------|------------|--------|
| `$granularity` | `workflow.granularity` | `flexible` |
| `$max_stories_per_epic` | `workflow.plan.max_stories_per_epic` | `6` |
| `$max_epics` | `workflow.plan.max_epics` | `10` |
| `$max_review_cycles` | `workflow.plan.max_review_cycles` | `3` |
| `$timeout_minutes` | `workflow.plan.timeout_minutes` | `30` |
| `$research_enabled` | `workflow.research.enabled` | `true` |
| `$max_deep_research` | `workflow.research.max_deep` | `3` |
| `$max_quick_research` | `workflow.research.max_quick` | `5` |

Note : si `--granularity=` est passé en argument, il surcharge `$granularity` de la config.

## Parse arguments

1. Extraire depuis `$ARGUMENTS` :
   - `spec_path` : premier argument positionnel (chemin vers SPEC.md) — si absent, chercher `SPEC.md` dans le répertoire courant
   - `--granularity=` : `fine` | `standard` | `flexible` — défaut : `$granularity` (depuis config)
   - `--dry-run` : si présent, afficher ce qui serait créé sans créer

## Pré-checks

1. **SPEC.md existe** :
   - Vérifier que `spec_path` pointe vers un fichier existant et non vide
   - Si absent → "SPEC.md introuvable. Lance `/gsr:bootstrap` d'abord pour générer les documents projet."

2. **Documents bootstrap complets** :
   - Vérifier `docs/agent_docs/architecture.md` existe
   - Vérifier `docs/discovery.md` existe
   - Vérifier `CLAUDE.md` existe
   - Si un document obligatoire manque → lister les manquants et proposer `/gsr:bootstrap`

3. **Pas de session en cours** :
   - Vérifier si `.claude/plan-session.md` existe
   - Si oui → "Une session de planification est en cours (Niveau [N]). Utilise `/gsr:plan-status` pour voir l'état, ou `/gsr:plan-abort` pour recommencer. [Continuer quand même] [Annuler]"
   - Si "Continuer" → archiver l'ancienne session (renommer avec timestamp)

4. **Pas de plan existant** :
   - Vérifier si `docs/plan/ROADMAP.md` existe
   - Si oui → "Un plan existe déjà. Veux-tu le remplacer ? [Oui, re-planifier] [Non, annuler]"
   - Si "Non" → stop

## Étape 1 — Analyse (Agent)

1. Spawner l'agent `gsr-analyst` avec le prompt :
   ```
   Analyse les documents bootstrap du projet.
   SPEC.md : [spec_path]
   Granularité : $granularity
   Répertoire projet : [cwd]

   <config>
   max_epics=$max_epics
   max_stories_per_epic=$max_stories_per_epic
   </config>
   ```

2. Vérifier que `.claude/plan-session.md` a été créé avec la section `## Analyse`

3. Si la section `## Alertes` contient des éléments :
   - Afficher les alertes à l'utilisateur
   - Pour chaque incohérence : "Incohérence détectée : [détail]. Continuer quand même ? [Oui] [Non, je corrige d'abord]"
   - Si "Non" → stop

## Étape 2 — Décomposition (Agent)

1. Spawner l'agent `gsr-planner` avec le prompt :
   ```
   Mode : roadmap
   Session : .claude/plan-session.md
   Granularité : $granularity

   <config>
   max_epics=$max_epics
   max_stories_per_epic=$max_stories_per_epic
   research_enabled=$research_enabled
   </config>

   Décompose le projet en Epics et Stories (SANS phases).
   Lire la section ## Analyse de la session pour les données.
   Écrire le résultat dans la section ## Roadmap de la session.
   ```

2. Vérifier que `plan-session.md` section `## Roadmap` est remplie

3. Vérifier les Research Gates dans `## Research Needed` :
   - Si présent → pour chaque Research Gate :
     a. Lire `.claude/gsr/plan-research.md` section `<integration-flow>`
     b. Proposer à l'utilisateur :
        ```
        Research Gate — [TYPE]
        [Description du point à résoudre]

        Options :
        [A] Recherche rapide (~30s)
        [B] Deep Research (~15-30min)
        [C] Continuer sans recherche
        ```
     c. Si [A] → spawner `research-prompt-agent` mode=quick → exécuter web searches → intégrer → re-spawner planner
     d. Si [B] → spawner `research-prompt-agent` mode=deep → afficher instructions → sauvegarder session → stop (reprendre via /gsr:plan-resume ou relancer /gsr:plan)
     e. Si [C] → continuer avec l'approche best-effort

## Étape 3 — Review interactive

Présenter le plan draft epic par epic :

Pour chaque epic dans `## Roadmap` :

1. Afficher :
   ```
   Epic [N] : [Nom]
   Feature MVP : [référence]
   Dépend de : [dépendances]
   Stories : [N]

   | # | Story | Priorité | Complexité |
   |---|-------|----------|------------|
   [table des stories]

   [Valider] [Ajuster] [Passer]
   ```

2. Si **Valider** → marquer l'epic comme validé dans la session

3. Si **Ajuster** → demander quoi changer :
   - "Que veux-tu ajuster ?" (texte libre)
   - Appliquer les modifications dans la session
   - Re-présenter l'epic modifié
   - Max 3 cycles par epic → "3ème itération. On valide et on ajuste en exécution si besoin."

4. Si **Passer** → marquer comme draft, continuer

Après tous les epics :

5. Validation finale — vérifier :
   - [ ] ≥ 1 epic validé
   - [ ] Chaque epic a ≥ 1 story
   - [ ] Graphe de dépendances acyclique
   - [ ] Chaque feature MVP a au moins 1 epic
   - Si échec → signaler et proposer d'ajuster

6. Afficher le résumé :
   ```
   Plan validé : [N] epics, [N] stories

   Graphe :
   [Diagramme ASCII des dépendances]

   Ordonnancement : [liste ordonnée]
   ```

## Étape 4 — Génération (Agent)

Si `--dry-run` :
- Afficher la liste des fichiers qui seraient créés
- Stop

Sinon :

1. Mettre à jour `plan-session.md` : déplacer le contenu validé dans `## Roadmap`

2. Spawner l'agent `gsr-generator` avec le prompt :
   ```
   Mode : roadmap
   Session : .claude/plan-session.md
   Répertoire de sortie : docs/plan/
   Répertoire projet : [cwd]

   Génère ROADMAP.md + EPIC.md par epic + arborescence stories.
   Charger les templates depuis .claude/gsr/plan-output.md.
   ```

3. Vérifier que les fichiers ont été créés

## Étape 5 — Résumé final

```
Planification niveau 1 terminée.

Fichiers créés :
- docs/plan/ROADMAP.md
- docs/plan/epics/[NN]-[slug]/EPIC.md (× [N])
- [N] dossiers stories créés (à détailler)

Prochaine étape :
  /gsr:plan-story [epic-slug]/[story-slug]

  Suggestion : commence par l'epic [premier dans l'ordonnancement]
  → /gsr:plan-story [slug du premier epic]/[slug de la première story]
```

## Mise a jour du suivi

Mettre a jour `docs/GSR-STATUS.md` selon `.claude/gsr/status-output.md` section `<update-plan>` :
- Phase active → `Plan`
- Pipeline Plan → `En cours`
- Plan : Niveau → `1-roadmap`, remplir Epics/Stories counts, Granularite
- Detail par Epic : generer la table complete
- Historique : "Roadmap genere ([N] epics, [N] stories)"
- Ne pas mettre a jour en mode `--dry-run`

## Garde-fous

| Limite | Valeur (depuis config) | Comportement |
|--------|------------------------|-------------|
| Total epics | `$max_epics` (défaut: 10) | "$max_epics epics = projet très ambitieux pour solo dev." |
| Stories par epic | `$max_stories_per_epic` (défaut: 6) | "Beaucoup de stories. Certaines sont-elles post-MVP ?" |
| Cycles review / epic | `$max_review_cycles` (défaut: 3) | "[N]ème itération. On valide et ajuste en exécution." |
| Recherches / session | `$max_deep_research` deep + `$max_quick_research` quick (défaut: 3+5) | "Continuons avec ce qu'on a." |
| Durée totale | `$timeout_minutes` min (défaut: 30) | "On approche de $timeout_minutes min. On finalise ?" |
