# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.0.x   | Yes       |

## Scope

`memoriant-docforce-skill` is a Claude Code plugin that reads source code, config, and documentation files to detect documentation drift. It writes drift reports and JSONL evidence records to the local filesystem. It contains no executable server code, no authentication logic, and no credential handling beyond an optional LLM API key.

## What This Plugin Does

- Reads `.py`, `.go`, `.ts`, `.yaml`, `.json`, `.md`, and related files from paths specified by the user.
- Extracts entity names (API endpoints, feature flags, config keys) using heuristic pattern matching.
- Writes a drift report (JSON or Markdown) to a user-specified path.
- Appends evidence records to a local JSONL file.
- Optionally calls an OpenAI-compatible LLM API for enhanced entity extraction.

It does **not**:
- Execute code from the scanned files.
- Modify source code or documentation.
- Store or transmit file contents to any third party beyond the user-configured LLM endpoint.
- Require credentials of any kind beyond an optional LLM API key.

## Sensitive File Handling

The plugin reads file contents to extract entity names. It does not index, cache, or retain file contents beyond the current session. Do not configure it to scan files containing secrets (e.g., `.env` files with real credentials, private keys). Use `.env.example` or similar placeholder files instead.

## Reporting a Vulnerability

If you discover a security concern in this plugin — for example, a path traversal issue in scope config loading, prompt injection in skill instruction files, or improper handling of the LLM API key — please report it responsibly.

**Contact:** Open a [GitHub Security Advisory](https://github.com/NathanMaine/memoriant-docforce-skill/security/advisories/new) on this repository.

Please include:
- A clear description of the issue.
- Steps to reproduce.
- The potential impact.

Do not open a public issue for security vulnerabilities.

## Response Timeline

- Acknowledgement within 5 business days.
- Assessment and patch (if warranted) within 30 days.

## Audit Notes

All skill and agent definitions in this plugin are plain Markdown files. They contain no executable code, no shell scripts, no binary blobs, and no network-facing logic. They are safe to audit as plain text.
