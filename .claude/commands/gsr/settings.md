---
name: settings
description: Configure GSR workflow and model profile
human_ai_ratio: 60/40
---

# /gsr:settings

## 0. Initialiser la configuration

1. Exécuter `.claude/gsr/bin/gsr-config.sh ensure` pour créer `config.json` si absent
2. Déterminer le mode config : exécuter `.claude/gsr/bin/gsr-config.sh config-mode`

## 1. Charger toutes les valeurs

Selon le mode :
- Si `jq` → exécuter les commandes dump pour chaque section :
  ```
  .claude/gsr/bin/gsr-config.sh dump environment
  .claude/gsr/bin/gsr-config.sh dump models
  .claude/gsr/bin/gsr-config.sh dump git
  .claude/gsr/bin/gsr-config.sh dump discovery
  .claude/gsr/bin/gsr-config.sh dump plan
  .claude/gsr/bin/gsr-config.sh dump output
  ```
- Si `claude` → lire `.claude/gsr/config.json` avec Read et extraire toutes les valeurs

## 2. Afficher l'état actuel

Présenter les settings groupés par catégorie :

```
## Configuration GSR actuelle

### Environnement (détecté à l'installation)
| Element | Statut | Détail |
|---------|--------|--------|
| jq | [✓ Disponible / ✗ Absent] | Config mode: [jq/claude] |
| Git CLI | [gh/glab/—] | Authentifié: [✓/✗] |
| Git MCP | [github/gitlab/—] | [Détecté/Non détecté] |
| Provider | [github/gitlab/—] | [Auto-détecté/Non détecté] |

### Modèles
| Setting | Valeur | Options |
|---------|--------|---------|
| Profil actif | **[profil]** | quality / balanced / budget |
| Overrides | [liste ou aucun] | Voir ci-dessous |

Mapping actuel :
| Agent | Rôle | Modèle |
|-------|------|--------|
| gsr-planner | orchestrator | [modèle] |
| gsr-analyst | worker | [modèle] |
| gsr-synthesizer | worker | [modèle] |
| research-prompt | worker | [modèle] |
| gsr-generator | generator | [modèle] |
| gsr-bootstrapper | generator | [modèle] |

### Workflow
| Setting | Valeur | Options |
|---------|--------|---------|
| Mode | **[mode]** | interactive / yolo |
| Granularité | **[granularity]** | fine / standard / flexible |
| Research | **[activé/désactivé]** | activé / désactivé |

### Discovery
| Setting | Valeur |
|---------|--------|
| Questions par phase | [N] |
| Retours par phase | [N] |
| Échanges max | [N] |
| Timeout | [N] min |
| Recherches deep max | [N] |
| Recherches quick max | [N] |

### Plan
| Setting | Valeur |
|---------|--------|
| Epics max | [N] |
| Stories par epic | [N] |
| Phases par story | [N] |
| Cycles review | [N] |
| Timeout | [N] min |

### Git
| Setting | Valeur | Options |
|---------|--------|---------|
| Branching | **[strategy]** | none / phase / story |
| Conventional commits | **[oui/non]** | oui / non |

### Output
| Setting | Valeur | Options |
|---------|--------|---------|
| CLAUDE.md max lignes | [N] | nombre |
| Format spec | **[format]** | lean / detailed |
| Format plan | **[format]** | xml / markdown |

Que veux-tu modifier ?
```

## 3. Appliquer les modifications

L'utilisateur indique en langage naturel ce qu'il veut changer. Exemples :
- "Passe en profil budget"
- "Augmente le timeout discovery à 60 min"
- "Désactive la research"
- "Passe le branching à phase"

Pour chaque modification :

1. **Valider la valeur** — vérifier qu'elle est dans les options autorisées
2. **Écrire** :
   - Si mode `jq` → `.claude/gsr/bin/gsr-config.sh set <key> <value>`
   - Si mode `claude` → modifier `.claude/gsr/config.json` avec Edit
3. **Si changement de profil** → exécuter la logique de `/gsr:set-profile` :
   - Résoudre le modèle pour chaque agent via `resolve-model`
   - Modifier le `model:` dans le frontmatter YAML de chaque agent dans `.claude/agents/gsr/`

## 4. Confirmation

Réafficher uniquement les sections modifiées avec les nouvelles valeurs.

## 5. Re-scan environnement (optionnel)

Proposer : "Relancer le scan d'environnement ? [Oui] [Non]"
- Si Oui → exécuter `.claude/gsr/bin/gsr-config.sh scan`
- Si Non → terminer
