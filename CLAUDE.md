# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A containerized Python **Remote MCP Server** that exposes a `send_message` tool for outbound WhatsApp messaging. MCP clients connect over HTTP(S); the server delegates delivery to a swappable provider abstraction.

**Requirements live in `docs/requirements/`.** Each phase has one file named `REQ-NNN-<slug>.md`. Before implementing anything, read the relevant requirement file — new requirements may have been added since this file was last updated.

## Commands

```bash
# Build and start
docker compose up --build

# Run tests
docker compose run --rm app pytest

# Or directly if venv active
python3 -m pytest
```

## Rules

- **Source of truth**: `docs/requirements/` for scope; `docs/design/` for decisions. Read before coding.
- **Provider isolation**: Meta-specific logic must not leak into the MCP tool layer.
- **Input validation**: Reject empty/whitespace-only `recipient` and `message` before reaching the provider.
- **Controlled errors**: Provider failures return structured errors; must not crash the MCP server.
- **Secret hygiene**: Credentials never logged, never hardcoded. `.env` is gitignored; `.env.example` has placeholders only. Local TLS certs/keys never committed.
- **Integration tests opt-in only**: Tests requiring real external config (Meta API, live certs) must be tagged and excluded from the default `pytest` run.
