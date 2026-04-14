# indexion-skills

Claude Code skills powered by [indexion](https://github.com/trkbt10/indexion) — code search, similarity analysis, refactoring, documentation, and knowledge management.

## Installation

```bash
claude marketplace add trkbt10/indexion-skills
claude plugin install indexion-skills
```

## Prerequisites

indexion must be installed and available in your PATH:

```bash
curl -fsSL https://raw.githubusercontent.com/trkbt10/indexion/main/install.sh | bash
```

## Available Skills

### Search & Exploration

| Skill | When to use | Commands |
|-------|-------------|----------|
| `indexion-search` | Find code by name, description, structure, or similarity. Manage cached search indexes. | `grep`, `search`, `explore`, `digest`, `sim` |
| `indexion-segment` | Split text into contextual chunks for RAG/embedding pipelines. | `segment` |
| `indexion-kgf` | Debug KGF specs — inspect tokenization, parsing, and edge extraction. | `kgf` |

### Documentation

| Skill | When to use | Commands |
|-------|-------------|----------|
| `indexion-documentation` | Assess coverage, generate READMEs, detect code-to-doc drift. | `doc init/graph/readme`, `plan documentation/readme/reconcile` |

### Refactoring

| Skill | When to use | Commands |
|-------|-------------|----------|
| `indexion-refactor` | Detect duplication (textual, structural, conceptual) and enforce SoT. | `plan refactor/solid/unwrap`, `explore` |

### Specification

| Skill | When to use | Commands |
|-------|-------------|----------|
| `indexion-sdd` | RFC/spec to implementation with quantitative conformance verification. | `spec draft/verify/align` |

### Knowledge Base

| Skill | When to use | Commands |
|-------|-------------|----------|
| `indexion-wiki` | Create, update, lint, and reconcile a project wiki. Manage search indexes. | `wiki pages/index/lint/export/import/log` |

## Plugin Structure

```
indexion-skills/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   ├── indexion-search/         # Code search & exploration
│   ├── indexion-segment/        # Text segmentation
│   ├── indexion-kgf/            # KGF spec inspection
│   ├── indexion-documentation/  # Documentation lifecycle
│   ├── indexion-refactor/       # Duplication detection & SoT
│   ├── indexion-sdd/            # Spec-Driven Development
│   ├── indexion-wiki/           # Project wiki lifecycle
└── LICENSE
```

## License

Apache-2.0
