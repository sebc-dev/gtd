# GTD — Solo Dev Workflow for Claude Code

Plugin Claude Code qui structure le cycle de vie d'un projet solo : de l'idee au deploiement.

## Phases

| Phase | Status | Description |
|-------|--------|-------------|
| Discovery | Done | Interview guidee en 6 phases generant `discovery.md` + bootstrap projet |
| Plan | Done | Planification progressive : Epics → Stories → Phases atomiques (JIT) |
| Execute | Planned | TDD, commits atomiques, quality gates |
| Ship | Planned | Merge, deploy, smoke tests |

## Installation

```bash
curl -fsSL https://raw.githubusercontent.com/sebc-dev/gtd/main/install.sh | bash
```

### Options

```bash
# Projet specifique
GTD_TARGET=/path/to/project curl -fsSL ... | bash

# Dry-run
GTD_DRY_RUN=1 curl -fsSL ... | bash

# Phase specifique
GTD_PHASES=discovery curl -fsSL ... | bash

# Lister les phases disponibles
GTD_LIST=1 curl -fsSL ... | bash

# Ecraser les fichiers existants
GTD_FORCE=1 curl -fsSL ... | bash
```

## Utilisation

### Discovery

```
/gtd:discover "description du projet"   # Demarrer une discovery
/gtd:discover-resume                     # Reprendre une session
/gtd:discover-save                       # Sauvegarder un partiel
/gtd:discover-abort                      # Abandonner
/gtd:bootstrap [path]                    # Generer la structure projet
```

### Plan (progressif)

```
/gtd:plan [SPEC.md] [--granularity=flexible]   # Niveau 1 : ROADMAP (Epics + Stories)
/gtd:plan-story [epic/story]                    # Niveau 2 : Detailler une story
/gtd:plan-phases [epic/story]                   # Niveau 3 : Phases atomiques
/gtd:plan-status                                # Voir la progression
/gtd:plan-abort                                 # Annuler la session
```

## Architecture

Architecture **Command + Agents + References** (pattern GSD) — pas de skill-orchestrateur.

```
.claude/
├── commands/gtd/          # Slash commands (orchestrateurs legers)
│   ├── discover.md        # Interface conversationnelle discovery
│   ├── discover-resume.md
│   ├── discover-save.md
│   ├── discover-abort.md
│   ├── bootstrap.md       # Thin dispatcher → agent
│   ├── plan.md            # Niveau 1 : ROADMAP
│   ├── plan-story.md      # Niveau 2 : Detail story
│   ├── plan-phases.md     # Niveau 3 : Phases atomiques
│   ├── plan-status.md
│   └── plan-abort.md
├── agents/                # Agents specialises (contexte frais)
│   ├── gtd-synthesizer.md # Phase 6 discovery : synthese + validation
│   ├── gtd-bootstrapper.md# Bootstrap : CLAUDE.md, SPEC.md, etc.
│   ├── gtd-analyst.md     # Plan : analyse docs bootstrap
│   ├── gtd-planner.md     # Plan : decomposition multi-mode
│   ├── gtd-generator.md   # Plan : generation fichiers
│   └── research-prompt-agent.md  # Recherche : prompts optimises
└── gtd/                   # References partagees (chargees par les agents)
    ├── discovery-phases.md
    ├── discovery-output.md
    ├── discovery-research.md
    ├── plan-output.md
    └── plan-research.md
```

## License

MIT
