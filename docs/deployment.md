# Heritage National Bank — Deployment Guide

**Version:** m4-capstone  
**Last updated:** 2026-11-13

---

## Prerequisites

- Python 3.10 or higher
- pip

Install dependencies:

```bash
pip install -r requirements.txt
```

The core dependencies are:

```
anthropic
python-dotenv
mcp
bandit
pytest
```

---

## Environment Variables

Create a `.env` file in the project root (never commit this file):

```dotenv
# Required
OPENROUTER_API_KEY=sk-or-...        # API key for OpenRouter / Anthropic API

# Optional — override model slugs if routing to a different provider or version
ANTHROPIC_BASE_URL=https://openrouter.ai/api
MODEL_HAIKU=anthropic/claude-3.5-haiku
MODEL_SONNET=anthropic/claude-sonnet-4.5
MODEL_OPUS=anthropic/claude-opus-4.1
```

The platform will raise a `KeyError` at startup if `OPENROUTER_API_KEY` is not set.

---

## Directory Structure

The following directories must exist before the first run (created automatically on first execution, but create them explicitly for CI):

```
audit_log/      # runtime audit logs (gitignored in production)
reports/        # generated compliance reports
docs/           # this documentation
skills/         # SKILL.md skill definitions
```

```bash
mkdir -p audit_log reports docs skills
```

---

## Running the Platform

Execute the end-to-end banking workflow:

```bash
python -m bfsi.main
```

Expected output:

```
=== End-to-end banking workflow ===
  R1001: SETTLED
  R1002: REJECTED -> AML
=== Batch fraud pipeline ===
  B1: risk=LOW (fallback)
  B2: risk=HIGH (fallback)
  B3: risk=HIGH (fallback)
  B4: risk=HIGH (fallback)
```

After running, audit logs are written to:
- `audit_log/transactions.jsonl`
- `audit_log/model_routing.jsonl`
- `audit_log/mcp_tool_calls.jsonl`
- `audit_log/batch_results.json`

---

## MCP Server Startup

The MCP servers are registered in `.claude/mcp.json` for Claude Code sessions. For standalone use, start them directly:

```bash
# Banking API server
python -m bfsi.mcp.banking_api_server

# Customer Info server
python -m bfsi.mcp.customer_info_server
```

In the current in-process registry implementation (`bfsi/mcp/registry.py`), the servers run as direct Python function calls rather than separate processes — no separate server startup is needed for `python -m bfsi.main`.

---

## Running Tests

```bash
pytest
```

All tests must pass with zero failures before merging (Definition of Done).

To run with verbose output:

```bash
pytest -v
```

---

## Running Security Scans

```bash
bandit -q -r bfsi -f txt -o audit_log/bandit_scan.txt || true
```

The `|| true` prevents CI failure on informational output. Inspect the output:

```bash
tail -n 20 audit_log/bandit_scan.txt
```

**Definition of Done:** bandit must report zero medium or high findings.

---

## Generating the Compliance Report

```bash
# Using the Claude Code skill (requires Claude Code session)
/generate-compliance-report

# Or read the existing report
cat reports/compliance_report.md
```

---

## Audit Log Reference

| File | Contents | Written by |
|---|---|---|
| `audit_log/mcp_tool_calls.jsonl` | Every MCP tool invocation with server, tool, masked args, and ok status | `bfsi/mcp/registry.py` |
| `audit_log/model_routing.jsonl` | Model selection per request: task class, model slug, rationale | `bfsi/model_router.py` |
| `audit_log/transactions.jsonl` | Full transaction record: compliance stage results, transfer outcome | `bfsi/transactions.py` |
| `audit_log/batch_results.json` | Batch fraud scores by transaction ID | `bfsi/batch_fraud.py` |
| `audit_log/bandit_scan.txt` | Static analysis output from bandit | CI / manual run |

Audit logs are the compliance system of record. Do not delete them between runs in production.

---

## CI/CD Gate

The CI pipeline (`.github/workflows/ci_gate.yml`) runs three checks in order:

1. `pytest` — zero failures required
2. `bandit -r bfsi` — zero medium+ findings required
3. Claude-as-Judge — compliance rubric score >= 4 required

A merge is blocked if any check fails.

---

## Known Operational Gaps

The following items are tracked as open findings and must be resolved before full production deployment:

1. **No per-session rate limiting on money movement** (`bfsi/transactions.py`): `process()` can be called unlimited times per session. Add a per-customer counter capped at a configurable limit (suggested: 5 per session, backed by Redis in production).

2. **Partial sanctions hits are auto-approved** (`bfsi/compliance/pipeline.py:33`): A `PARTIAL` match against the watchlist currently passes the sanctions stage. Change `sanc_ok = not sanc["hit"]` to block all hits and route them to a manual review queue.

See `reports/compliance_report.md` for the full gap analysis with recommended code fixes.
