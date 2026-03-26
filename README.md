# memoriant-docforce-skill

![Version](https://img.shields.io/badge/version-1.0.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-Claude%20Code%20%7C%20Codex%20CLI%20%7C%20Gemini%20CLI-lightgrey)

A Claude Code plugin that monitors documentation drift — the gap between what your code does and what your docs say. Compares source code and config files against documentation files, surfaces every mismatch with severity and file location, and appends evidence records to a JSONL audit log.

Built on the methodology of [`NathanMaine/living-docforce-agent`](https://github.com/NathanMaine/living-docforce-agent).

---

## What It Does

1. Loads a scope config specifying which code, config, and doc files to compare (or infers scope from natural language).
2. Extracts entities from code — API endpoints, feature flags, or config keys — using heuristic pattern matching.
3. Extracts the same entity types from documentation files.
4. Compares the two sets and classifies every mismatch:
   - `in_code_not_docs` — exists in code, missing from docs (severity: high)
   - `in_docs_not_code` — exists in docs, absent from code (severity: medium)
   - `name_mismatch` — exists in both but named differently (severity: high)
5. Produces a structured drift report (JSON or Markdown) with file locators.
6. Appends a JSONL evidence record.

---

## Installation

### Claude Code

```bash
claude mcp add memoriant-docforce-skill
```

Or clone locally:

```bash
git clone https://github.com/NathanMaine/memoriant-docforce-skill ~/.claude/plugins/memoriant-docforce-skill
```

### Codex CLI

```bash
codex install NathanMaine/memoriant-docforce-skill
```

### Gemini CLI

```bash
gemini extension install NathanMaine/memoriant-docforce-skill
```

---

## Usage

### In Claude Code (natural language)

```
Check if my API docs are up to date with the code in src/
```

```
Find documentation drift between README.md and the current config keys in config.yaml
```

```
Are the feature flags documented in ARCHITECTURE.md still in the codebase?
```

### Via skill invocation

```
/doc-drift-monitor --scope fixtures/scope.yaml --format markdown
```

### Via Codex CLI

```bash
codex run docforce-agent --var scope=fixtures/scope.yaml --var format=markdown
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

Paths are resolved relative to the current working directory (or `--base-dir` if specified).

---

## Plugin Structure

```
memoriant-docforce-skill/
├── .claude-plugin/
│   └── plugin.json               # Plugin manifest
├── skills/
│   └── doc-drift-monitor/
│       └── SKILL.md              # Full methodology for Claude Code
├── agents/
│   └── docforce-agent.md        # Autonomous agent definition
├── AGENTS.md                     # Codex CLI agent definitions
├── gemini-extension.json         # Gemini CLI extension manifest
├── SECURITY.md                   # Security policy
├── README.md                     # This file
└── LICENSE                       # MIT
```

---

## Sample Output

```markdown
## Documentation Drift Report: api-docs-check

**Checked:** 2026-03-25 | **Focus:** endpoints | **Severity:** HIGH

### Drift Items (2)

| # | Category | Type | Value | Severity | Detail |
|---|---------|------|-------|----------|--------|
| 1 | in_code_not_docs | endpoint | `/api/v1/users/{id}/activate` | HIGH | Found in `src/api.py:87`. Not in `docs/api-reference.md`. |
| 2 | in_docs_not_code | endpoint | `/api/v1/users/bulk-import` | MEDIUM | In `docs/api-reference.md:42`. Not found in any code file. |

### Recommendations
- Add `/api/v1/users/{id}/activate` to `docs/api-reference.md`.
- Remove or implement `/api/v1/users/bulk-import`.
```

---

## Evidence Log Format

```json
{"timestamp":"2026-03-25T10:00:00Z","scope_id":"api-docs-check","total_items":2,"overall_severity":"high"}
```

---

## Source Reference

Derived from [`NathanMaine/living-docforce-agent`](https://github.com/NathanMaine/living-docforce-agent), a Python CLI prototype for documentation drift monitoring with heuristic entity extraction and JSONL evidence logging.

---

## License

MIT — see [LICENSE](LICENSE).
