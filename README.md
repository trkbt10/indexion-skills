# indexion-skills

Claude Code skills powered by [indexion](https://github.com/trkbt10/indexion).

indexion is a language-agnostic code analysis tool built on KGF (Knowledge Graph
Framework). It understands your code's token structure, dependency graph, and
semantic relationships — not just raw text. These skills teach Claude Code how
to use indexion for searching, refactoring, documenting, and maintaining codebases.

## What Can Claude Do With This?

### Find code by intent, not just by name

"Where is the function that parses configuration?" — Claude uses `indexion grep`
for token-level pattern search, `indexion search` for semantic full-text search,
and `indexion digest query` for purpose-indexed function lookup. Each tool fits
a different search intent:

- **By name** (`grep "Ident:parse_config"`) — exact token match across the codebase
- **By pattern** (`grep "for ... for"`) — find structural patterns like nested loops
- **By description** (`digest query "parse configuration"`) — find functions by what they do
- **By structure** (`grep --semantic=long:50`) — find long functions, proxies, undocumented APIs
- **By similarity** (`explore --format=list`) — find files that overlap with each other

### Detect duplication before it becomes a problem

After writing code, Claude runs `indexion plan refactor` to find copy-pasted blocks,
`plan solid` to find cross-package extraction candidates, and `plan unwrap` to find
trivial wrapper functions. Beyond textual duplication, it detects **concept-level**
duplication — where multiple modules independently implement the same logic — using
`indexion explore` and SoT (Single Source of Truth) analysis.

### Keep documentation in sync with code

Claude assesses documentation coverage with `indexion plan documentation`, generates
package READMEs from doc comments with `indexion doc readme`, and detects drift
between code and documentation with `indexion plan reconcile`. When reconcile reports
vocabulary divergence, Claude knows which docs are stale and what terms are missing.

### Implement specifications with quantitative verification

Given an RFC or spec document, Claude generates SDD requirements with `indexion spec
draft`, implements them, and verifies conformance with `indexion spec align`. The
alignment tool provides a quantitative gate: MATCHED, DRIFTED, or SPEC_ONLY for each
requirement. Claude uses this as a CI-style pass/fail check between implementation tasks.

### Maintain a project wiki that stays current

Claude creates wiki pages via `indexion wiki pages add`, detects when source files
change with `wiki pages ingest`, and updates stale pages accordingly. Structural
integrity checks (`wiki lint`) catch broken links, orphan pages, and missing
cross-references. The wiki is a synchronized knowledge base, not a static document dump.

## Installation

```bash
claude marketplace add trkbt10/indexion-skills
claude plugin install indexion-skills
```

### Prerequisites

indexion must be installed and available in your PATH:

```bash
curl -fsSL https://raw.githubusercontent.com/trkbt10/indexion/main/install.sh | bash
```

## Skills Reference

| Skill | What it's for |
|-------|---------------|
| **indexion-search** | Find code by name, pattern, description, structure, or file similarity. Manage cached search indexes (digest build/status/rebuild). |
| **indexion-refactor** | Detect and eliminate duplication at three levels: textual (plan refactor), structural (plan solid, plan unwrap), and conceptual (explore + SoT analysis). |
| **indexion-documentation** | Assess documentation coverage, generate READMEs, detect code↔doc drift with plan reconcile. |
| **indexion-sdd** | RFC/spec → SDD requirements → implementation with quantitative conformance gates (spec draft/align/verify). |
| **indexion-wiki** | Create, update, lint, and reconcile a project wiki. Track source changes and keep pages current. |
| **indexion-segment** | Split text into contextual chunks for RAG/embedding pipelines. |
| **indexion-kgf** | Debug KGF language specs — inspect tokenization, parse trees, and extracted edges. |

Each skill contains detailed instructions with verified examples. Claude loads them
automatically when the task matches.

## How It Works

indexion analyzes code through KGF (Knowledge Graph Framework) specs — declarative
language definitions that describe tokenization, grammar, and semantic rules. This
means indexion understands your language's structure without hardcoding. It works
for any language with a KGF spec.

The skills in this plugin don't add new tools to Claude. They teach Claude **when
and how** to use `indexion` CLI commands — which command to pick for which intent,
what the output means, and how to act on it. Think of them as expert knowledge that
Claude loads on demand.

## License

Apache-2.0
