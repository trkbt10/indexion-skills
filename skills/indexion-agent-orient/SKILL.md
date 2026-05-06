---
name: indexion-agent-orient
description: Generate and use a pre-edit structure brief so coding agents learn likely owners, consumer surfaces, and unsafe edit locations before implementing.
---

# Agent Orientation Workflow

Use this skill before starting implementation in an unfamiliar or ambiguous part
of a codebase.

## Why

Coding agents often anchor on the path or noun in the user prompt and start
editing before they have learned the repository's ownership structure. `indexion
agent orient` creates a short, evidence-backed brief that can be pasted into
AGENTS.md, CLAUDE.md, a Claude slash command, a Codex skill, or a subagent prompt.

## Pipeline

1. Generate the brief:

   ```bash
   indexion agent orient --task-file task.md --output=.indexion/cache/agent/orient.md .
   ```

   For short tasks:

   ```bash
   indexion agent orient --task "add identity audit feature" .
   ```

2. Read these sections before editing:

   - `Likely Owners`: core packages that should own domain behavior.
   - `Consumer Surfaces`: CLI, skills, docs, or adapters likely to call the core.
   - `Do Not Implement Here`: files to avoid as domain implementation targets.
   - `Required Preflight`: files the agent should read before patching.

3. Confirm the owner with focused tools:

   ```bash
   indexion doc graph --format=text <likely-owner>
   indexion grep --semantic=name:<term> .
   indexion search "<task concept>" .
   ```

4. Gate implementation:

   - If the intended edit path appears in `Do Not Implement Here`, stop and
     explain the conflict.
   - If the intended owner is absent from `Likely Owners`, gather more evidence
     with `doc graph`, `grep`, `search`, or `explore`.
   - Keep CLI code thin unless the brief and follow-up evidence show it owns the
     behavior.

5. Use for zero-knowledge delegation:

   Give a subagent only the task and the generated orientation brief, then ask it
   to identify likely owner files and unsafe edit locations. It should be able to
   name the core owner and avoid CLI-first implementation.

## External Agent Mapping

- Claude Code: store stable guidance in `CLAUDE.md`, project commands in
  `.claude/commands/`, and project subagents in `.claude/agents/`.
- Codex: store stable guidance in AGENTS.md or skills, and paste the orientation
  brief into delegated task context.
- Multi-agent workflows: use the brief as the structured handoff payload so each
  isolated agent starts with the same repository-specific assumptions.
