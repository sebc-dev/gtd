---
name: set-profile
description: Switch model profile (quality/balanced/budget)
argument-hint: <profile>
human_ai_ratio: 10/90
---

# /gsr:set-profile $ARGUMENTS

## 1. Valider l'argument

Extraire le profil depuis `$ARGUMENTS`. Valeurs acceptées : `quality`, `balanced`, `budget`.

- Si vide ou invalide → "Profil inconnu. Choisis : `quality`, `balanced`, `budget`"
- Si identique au profil actuel → "Déjà sur le profil **[profil]**. Rien à changer."

## 2. Écrire le profil dans la config

1. Exécuter `.claude/gsr/bin/gsr-config.sh ensure` (crée config.json si absent)
2. Déterminer le mode config : `.claude/gsr/bin/gsr-config.sh config-mode`
3. Écrire :
   - Si mode `jq` → `.claude/gsr/bin/gsr-config.sh set models.active_profile $PROFILE`
   - Si mode `claude` → modifier `.claude/gsr/config.json` avec Edit (changer la valeur de `active_profile`)

## 3. Résoudre les modèles pour chaque agent

Pour chaque agent, déterminer le modèle effectif :

- Si mode `jq` → `.claude/gsr/bin/gsr-config.sh dump models` (retourne le mapping complet)
- Si mode `claude` → appliquer le mapping manuellement :

| Profil | orchestrator | worker | generator |
|--------|-------------|--------|-----------|
| quality | opus | opus | sonnet |
| balanced | opus | sonnet | sonnet |
| budget | sonnet | sonnet | haiku |

| Agent | Rôle |
|-------|------|
| gsr-planner | orchestrator |
| gsr-analyst | worker |
| gsr-synthesizer | worker |
| research-prompt-agent | worker |
| gsr-generator | generator |
| gsr-bootstrapper | generator |

Vérifier aussi les overrides : si `models.overrides.<agent>` existe dans config.json, utiliser cette valeur au lieu du profil.

## 4. Modifier les frontmatters des agents

Pour chaque fichier agent dans `.claude/agents/gsr/` :

1. Lire le fichier
2. Trouver la ligne `model: <valeur>` dans le frontmatter YAML (entre les `---`)
3. Si le modèle actuel est différent du modèle résolu → modifier avec Edit
4. Ne pas toucher au reste du fichier

Fichiers agents :
- `.claude/agents/gsr/gsr-planner.md`
- `.claude/agents/gsr/gsr-analyst.md`
- `.claude/agents/gsr/gsr-synthesizer.md`
- `.claude/agents/gsr/research-prompt-agent.md`
- `.claude/agents/gsr/gsr-generator.md`
- `.claude/agents/gsr/gsr-bootstrapper.md`

## 5. Afficher le résultat

```
Profil : **[profil]**

| Agent              | Rôle          | Modèle  |
|--------------------|---------------|---------|
| gsr-planner        | orchestrator  | [modèle] |
| gsr-analyst        | worker        | [modèle] |
| gsr-synthesizer    | worker        | [modèle] |
| research-prompt    | worker        | [modèle] |
| gsr-generator      | generator     | [modèle] |
| gsr-bootstrapper   | generator     | [modèle] |

[N] agents mis à jour.
```
