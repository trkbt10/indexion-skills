# indexion-skills

Claude Code plugin providing skills powered by [indexion](https://github.com/trkbt10/indexion) — source code exploration, similarity analysis, and documentation tools.

## Installation

```bash
# Add marketplace and install
claude marketplace add trkbt10/indexion-skills
claude plugin install indexion-skills
```

## Prerequisites

indexion must be installed and available in your PATH:

```bash
curl -fsSL https://raw.githubusercontent.com/trkbt10/indexion/main/install.sh | bash
```

## Available Skills

### Exploration & Analysis

| Skill | Command | Description |
|-------|---------|-------------|
| `indexion-explore` | `indexion explore` | Find similar files and detect duplicates |
| `indexion-segment` | `indexion segment` | Split text into contextual segments |
| `indexion-kgf` | `indexion kgf` | Inspect and debug KGF language specs |

### Documentation & README

| Skill | Command | Description |
|-------|---------|-------------|
| `indexion-documentation` | `indexion plan documentation/reconcile`, `indexion doc graph` | Documentation analysis — coverage, drift detection, dependency graphs |
| `indexion-readme` | `indexion doc init/readme`, `indexion plan readme` | README construction — initialize, generate, plan, assemble |

### Spec-Driven Development

| Skill | Command | Description |
|-------|---------|-------------|
| `indexion-sdd` | `indexion spec *` | SDD validation loop: draft, verify, align, validate with cc-sdd/codex |

### Planning

| Skill | Command | Description |
|-------|---------|-------------|
| `indexion-plan-refactor` | `indexion plan refactor` | Generate refactoring plans from similarity analysis |
| `indexion-plan-solid` | `indexion plan solid` | Plan common code extraction across directories |

## Plugin Structure

```
indexion-skills/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Marketplace registry
├── skills/
│   ├── indexion-explore/     # File similarity analysis
│   ├── indexion-segment/     # Text segmentation
│   ├── indexion-kgf/         # KGF spec inspection
│   ├── indexion-documentation/ # Documentation analysis (coverage, reconcile, graph)
│   ├── indexion-readme/       # README construction (init, generate, assemble)
│   ├── indexion-sdd/         # Spec-Driven Development loop
│   ├── indexion-plan-refactor/
│   ├── indexion-plan-solid/
└── LICENSE
```

## License

Apache-2.0
