---
name: doc-drift-monitor
description: Compare code and config files against their documentation to find drift — places where the docs say one thing and the code does another. Produce a structured drift report and append an evidence record.
---

# Documentation Drift Monitor Skill

## Purpose

This skill teaches Claude Code to act as a documentation drift detector: given a scope config pointing at source code, config files, and documentation, extract key entities (API endpoints, feature flags, configuration keys) from each, compare them, and report every mismatch. Based on `NathanMaine/living-docforce-agent`.

The goal is to help individual engineers catch documentation debt before it becomes a support or onboarding problem.

---

## When to Use This Skill

- After a sprint: "Did we forget to update the docs for the endpoints we changed?"
- Pre-release: "Is our API documentation accurate before we ship?"
- Onboarding: "Are the setup instructions in README still correct for the current config?"
- Continuous quality: run in CI to detect drift as code evolves.
- When a developer reports that the docs are wrong: quickly confirm and locate the specific mismatch.

---

## Prerequisites

1. Source code files (`.py`, `.go`, `.ts`, etc.) or config files (`.yaml`, `.json`, `.toml`, `.env.example`) to check.
2. Documentation files (`.md`, `.rst`, `.txt`, or plain text excerpts) to compare against.
3. A scope config YAML (optional but recommended) that names which files to compare and what to look for.

---

## Scope Config Format

```yaml
scope_id: "notifications-service-endpoints"
code_paths:
  - "src/notifications.py"
  - "src/routes.py"
config_paths:
  - "config/notifications.yaml"
doc_paths:
  - "docs/notifications-api.md"
  - "README.md"
focus: "endpoints"   # "endpoints" | "flags" | "config-keys" | "all"
```

If no scope config is provided, accept code/doc paths directly from the user.

---

## Step-by-Step Methodology

### Step 1 — Load and validate scope

- Load the scope config YAML if provided.
- Resolve all paths relative to `--base-dir` (default: current working directory).
- Verify each file exists. Report missing files before proceeding — do not silently skip them.

### Step 2 — Extract entities from code and config

Based on the `focus` field, extract the relevant entities:

**focus: endpoints**
- Scan Python for `@app.route(...)`, `@router.get(...)`, `@router.post(...)`, FastAPI path decorators, Flask route decorators.
- Scan Go for `http.HandleFunc(...)`, Gin/Echo route registrations.
- Scan TypeScript/JS for Express `.get(`, `.post(`, `.put(`, `.delete(` patterns.
- Also scan OpenAPI/Swagger YAML/JSON for `paths:` entries.

**focus: flags**
- Scan for feature flag patterns: `if feature_enabled(...)`, `if flags.get(...)`, `FeatureFlags.FLAG_NAME`, environment variable reads of flags.

**focus: config-keys**
- Scan config YAML/JSON/TOML for top-level and nested keys.
- Scan `.env.example` for variable names.

**focus: all**
- Run all three extractors.

Output shape:

```json
{
  "source": {
    "endpoints": ["/api/v1/notifications", "/api/v1/notifications/{id}"],
    "flags": ["ENABLE_PUSH_NOTIFICATIONS", "USE_LEGACY_GATEWAY"],
    "config_keys": ["smtp_host", "smtp_port", "queue_url"]
  }
}
```

### Step 3 — Extract entities from documentation

Apply the same extraction patterns to documentation files — look for:
- Endpoint paths mentioned in backticks, tables, or code blocks.
- Flag names in configuration sections.
- Config key names in setup instructions.

Output shape:

```json
{
  "docs": {
    "endpoints": ["/api/v1/notifications", "/api/v1/notifications/{id}", "/api/v1/notifications/bulk"],
    "flags": ["ENABLE_PUSH_NOTIFICATIONS"],
    "config_keys": ["smtp_host", "smtp_port"]
  }
}
```

### Step 4 — Compare and classify drift

Compute three categories of drift items:

| Category | Definition | Severity |
|----------|-----------|----------|
| `in_code_not_docs` | Entity exists in code but is absent from docs | `high` |
| `in_docs_not_code` | Entity exists in docs but is absent from code | `medium` |
| `name_mismatch` | Entity appears in both but with different spelling/casing | `high` |

Assign overall severity:
- `high` if any `high` item exists.
- `medium` if only `medium` items exist.
- `low` if all differences are minor casing variants.
- `none` if no drift detected.

### Step 5 — Produce the drift report

**JSON format:**

```json
{
  "scope_id": "notifications-service-endpoints",
  "checked_at": "2026-03-25T10:00:00Z",
  "focus": "endpoints",
  "overall_severity": "high",
  "total_items": 3,
  "items": [
    {
      "category": "in_code_not_docs",
      "entity_type": "endpoint",
      "value": "/api/v1/notifications/{id}/read",
      "severity": "high",
      "detail": "Found in src/notifications.py line 42. Not mentioned in docs/notifications-api.md."
    },
    {
      "category": "in_docs_not_code",
      "entity_type": "endpoint",
      "value": "/api/v1/notifications/bulk",
      "severity": "medium",
      "detail": "Documented in docs/notifications-api.md but not found in any code file."
    }
  ]
}
```

**Markdown format:**

```markdown
## Documentation Drift Report: notifications-service-endpoints

**Checked:** 2026-03-25 | **Focus:** endpoints | **Severity:** HIGH

### Drift Items (3)

| # | Category | Type | Value | Severity | Detail |
|---|---------|------|-------|----------|--------|
| 1 | in_code_not_docs | endpoint | `/api/v1/notifications/{id}/read` | HIGH | Found in `src/notifications.py:42`. Not in docs. |
| 2 | in_docs_not_code | endpoint | `/api/v1/notifications/bulk` | MEDIUM | In `docs/notifications-api.md`. Not in code. |

### Recommendation
Update `docs/notifications-api.md` to add the `/{id}/read` endpoint and remove or implement the `/bulk` endpoint.
```

### Step 6 — Append evidence record

```json
{"timestamp":"2026-03-25T10:00:00Z","scope_id":"notifications-service-endpoints","total_items":3,"overall_severity":"high"}
```

---

## Input Format

### Scope file

```bash
/doc-drift-monitor --scope fixtures/scope.yaml
/doc-drift-monitor --scope fixtures/scope.yaml --format markdown --log evidence.jsonl
```

### Natural language

```
Check if our API docs are up to date with the code in src/
```

```
Find documentation drift between README.md and the current config keys in config.yaml
```

```
Are the feature flags documented in ARCHITECTURE.md still accurate?
```

---

## Output Format

1. **JSON drift report** — to `--out` path or stdout (default).
2. **Markdown drift report** — printed to the conversation when `--format markdown` or when invoked naturally.
3. **Evidence record** — appended to `--log` path if specified.

---

## Error Handling

| Situation | Response |
|-----------|----------|
| Scope config file not found | Report the path. Ask for the correct location. |
| A code or doc file listed in scope does not exist | Report each missing file. Ask before continuing. |
| No entities found in code files | Warn that the extractor found nothing. Ask user to confirm the focus type is correct. |
| No entities found in docs | Warn and offer to scan the docs more broadly. |
| No drift found | Report "No documentation drift detected." Do not write an empty report. |
| Evidence log directory does not exist | Create it automatically. Log a note. |

---

## Examples

### Example 1 — Endpoint drift check

User: "Check if my API docs match the code"

Claude Code actions:
1. Locate `src/` and `docs/` in working directory.
2. Extract endpoint patterns from Python source files.
3. Extract endpoint paths mentioned in Markdown docs.
4. Compare. Find 2 endpoints in code not documented.
5. Report with Markdown drift table.

### Example 2 — Feature flag audit

User: "Are the feature flags in ARCHITECTURE.md still in the code?"

Claude Code actions:
1. Extract flag names from ARCHITECTURE.md.
2. Scan source files for flag reads.
3. Report flags mentioned in docs but removed from code.

### Example 3 — Full scope config run with evidence

```bash
/doc-drift-monitor --scope fixtures/scope.yaml --format markdown --log evidence.jsonl --out report.json
```

---

## Version History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2026-03-25 | Initial release — heuristic extraction, drift comparison, evidence logging |
