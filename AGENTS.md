# AGENTS.md — Codex CLI Agent Definitions

This file follows the [Codex CLI AGENTS.md convention](https://github.com/openai/codex).

## memoriant-docforce-skill

**Purpose:** Monitor documentation drift by comparing code and config files against documentation. Surface every mismatch with severity, location, and a recommendation. Append evidence records to a JSONL audit log.

**Activation:** `codex run docforce-agent`

### Agent Behavior

The agent loads a scope config (or infers scope from the user request), extracts entities (endpoints, feature flags, or config keys) from both code and documentation files, compares them, and produces a structured drift report. It classifies every mismatch as high or medium severity with a locator pointing to the source file. It appends a JSONL evidence record on every run.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `scope` | string (path) | Yes | — | Path to the scope config YAML file |
| `out` | string (path) | No | stdout | Path to write the drift report |
| `log` | string (path) | No | `evidence.jsonl` | Path to the evidence log file |
| `format` | string | No | `json` | Report format: `json` or `markdown` |
| `base_dir` | string (path) | No | `.` | Base directory for resolving relative file paths |

### Outputs

| Name | Description |
|------|-------------|
| Drift report (JSON or Markdown) | Structured list of all drift items with severity and location |
| `evidence.jsonl` | Appended JSONL record with timestamp, scope ID, item count, and severity |

### Agent File

See [`agents/docforce-agent.md`](agents/docforce-agent.md) for the full system prompt.

### Skill File

See [`skills/doc-drift-monitor/SKILL.md`](skills/doc-drift-monitor/SKILL.md) for the full methodology.

---

## Usage with Codex CLI

```bash
# JSON drift report to stdout
codex run docforce-agent --var scope=fixtures/scope.yaml

# Markdown report written to file
codex run docforce-agent --var scope=fixtures/scope.yaml --var format=markdown --var out=report.md

# With evidence log
codex run docforce-agent --var scope=fixtures/scope.yaml --var log=evidence.jsonl

# Custom base directory
codex run docforce-agent --var scope=scope.yaml --var base_dir=/home/dev/my_project
```

---

## Scope Config Format

```yaml
scope_id: "api-docs-check"
code_paths:
  - "src/api.py"
config_paths:
  - "config/app.yaml"
doc_paths:
  - "docs/api-reference.md"
focus: "endpoints"   # "endpoints" | "flags" | "config-keys" | "all"
```

---

## Integration Notes

- Compatible with Claude Code, Codex CLI, and any agent runtime that supports Markdown-based agent definitions.
- No LLM API key required for pure heuristic extraction mode. An LLM key enhances entity recognition for ambiguous patterns.
- Evidence log is append-only JSONL — safe for CI pipelines and audit trails.
- Can be run as a pre-release gate: fail the pipeline if `overall_severity` is `high`.
