# Heritage National Bank — AI Banking Platform Architecture

**Version:** m4-capstone  
**Last updated:** 2026-11-13

---

## System Overview

The Heritage National Bank AI Banking Platform is a production-oriented Python application that integrates Claude-powered fraud detection, compliance automation, and customer service with a Model Context Protocol (MCP) data layer. The platform enforces PCI DSS controls throughout and maintains a complete audit trail across all decision points.

---

## Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Entry Point                        │
│                    bfsi/main.py                         │
└──────────────────┬──────────────┬───────────────────────┘
                   │              │
         ┌─────────▼──────┐  ┌───▼──────────────┐
         │  Transaction   │  │   Batch Fraud    │
         │ transactions.py│  │ batch_fraud.py   │
         └─────────┬──────┘  └───┬──────────────┘
                   │             │
         ┌─────────▼─────────────▼──────────────┐
         │         Model Router                 │
         │        model_router.py               │
         │  Haiku (classify) / Sonnet (reason)  │
         │  / Opus (decide)  → model_routing.jsonl│
         └─────────────────┬────────────────────┘
                           │
         ┌─────────────────▼────────────────────┐
         │       Compliance Pipeline            │
         │      compliance/pipeline.py          │
         │  KYC → AML → Sanctions → Approval    │
         └──┬──────────┬──────────┬─────────────┘
            │          │          │
     ┌──────▼──┐  ┌────▼────┐ ┌──▼──────────────┐
     │  KYC    │  │  AML    │ │   Sanctions      │
     │get_cust.│  │fraud_   │ │ sanctions_check  │
     │(MCP)    │  │engine.py│ │ .py              │
     └─────────┘  └────┬────┘ └──────────────────┘
                       │
         ┌─────────────▼────────────────────────┐
         │          MCP Registry                │
         │         mcp/registry.py              │
         │  Audits every call → mcp_tool_calls  │
         │  .jsonl; masks PAN fields in logs    │
         └──────────────┬───────────────────────┘
                        │
          ┌─────────────┴────────────────┐
          │                              │
  ┌───────▼────────┐           ┌────────▼────────┐
  │ customer-info  │           │  banking-api    │
  │    server      │           │    server       │
  │get_customer    │           │get_balance      │
  │check_history   │           │transfer_funds   │
  │verify_ip_loc.  │           │apply_loan       │
  └────────────────┘           └─────────────────┘
```

---

## Component Descriptions

### `bfsi/main.py` — Entry Point

Orchestrates the end-to-end banking workflow. Calls `transactions.process()` for two representative transactions (one approved, one AML-blocked), then runs `batch_fraud.run_batch()` over four synthetic transactions. Loads environment via `python-dotenv`.

### `bfsi/transactions.py` — Transaction Processing

The primary money-movement path. For each request:
1. Routes to Sonnet tier (`route("reason", ...)`)
2. Runs the full compliance pipeline
3. If APPROVE: routes to Opus tier (`route("decide", ...)`) and calls `transfer_funds` via MCP
4. Appends a complete audit record to `audit_log/transactions.jsonl` regardless of outcome

### `bfsi/batch_fraud.py` — Batch Fraud Pipeline

Scores a list of synthetic transactions through `fraud_engine.score_transaction()`, routing each to the Haiku tier for cost-efficient classification. Results are written to `audit_log/batch_results.json`.

### `bfsi/fraud_engine.py` — Fraud Detection Engine

Combines MCP geo/history signals with a Claude (Haiku) model opinion. Fetches `check_customer_history` and `verify_ip_location` from the customer-info MCP server, then sends the signals as JSON to the fraud-triage skill. Falls back to a deterministic rule (high amount or geo mismatch → HIGH) if the Claude API is unavailable.

### `bfsi/chatbot.py` — Customer Service Chatbot

Provides a guardrailed chat interface with:
- **Input sanitization** (`sanitize()`): strips non-printable characters, removes prompt-injection patterns (`ignore previous`, `system prompt`, `full card number`), truncates to 2000 chars
- **Fraud signals**: fetches customer history and geo data via MCP before every reply
- **Streaming**: uses `client.messages.stream()` by default
- **Cached system prompt**: built from the `customer-support` skill via `skills_loader`

### `bfsi/model_router.py` — Model Routing

Routes tasks to the appropriate model tier and logs every decision to `audit_log/model_routing.jsonl`:

| Task key | Env var | Default model | Use case |
|---|---|---|---|
| `classify` | `MODEL_HAIKU` | `anthropic/claude-3.5-haiku` | Batch fraud scoring, intent tagging |
| `reason` | `MODEL_SONNET` | `anthropic/claude-sonnet-4.5` | Chatbot, KYC, compliance narrative |
| `decide` | `MODEL_OPUS` | `anthropic/claude-opus-4.1` | Final approval, judge evaluation |

### `bfsi/skills_loader.py` — Skills System

Reads SKILL.md files from `skills/<name>/SKILL.md`, parses YAML front matter, and assembles a system prompt with Anthropic prompt-caching `cache_control` blocks. Skills with `cache: true` in front matter participate in prompt caching to reduce latency and cost on repeated calls.

### `bfsi/compliance/pipeline.py` — Compliance Pipeline

Implements the mandated KYC → AML → Sanctions → Approval sequence. Each stage must pass before the next runs; any BLOCK short-circuits the pipeline and returns the failed stage name for auditing. The pipeline never calls `transfer_funds` — that responsibility belongs to `transactions.py`.

### `bfsi/compliance/sanctions_check.py` — Sanctions Screening

Screens a name against a demo watchlist (`ivan petrov`, `acme shell holdings`, `redline logistics`). Returns `{"hit": bool, "strength": "NONE"|"PARTIAL"|"STRONG"}`. In production this would screen against OFAC/UN/EU consolidated lists.

### `bfsi/mcp/registry.py` — MCP Registry

In-process adapter that routes `call(server, tool, **args)` to the correct MCP server handler, audits every call to `audit_log/mcp_tool_calls.jsonl`, and applies PAN masking (`_mask()`) to logged arguments.

### `bfsi/mcp/banking_api_server.py` — Banking API MCP Server

Exposes `get_balance`, `transfer_funds`, and `apply_loan` as MCP tools. `transfer_funds` validates sufficient balance before mutating in-memory account state.

### `bfsi/mcp/customer_info_server.py` — Customer Info MCP Server

Exposes `get_customer`, `check_customer_history`, and `verify_ip_location` as MCP tools. Returns KYC status, transaction history averages, and geographic mismatch signals.

---

## Audit Trail Architecture

| Log file | Written by | Contents |
|---|---|---|
| `audit_log/mcp_tool_calls.jsonl` | `mcp/registry.py` | Every MCP call: server, tool, masked args, ok status |
| `audit_log/model_routing.jsonl` | `model_router.py` | Request ID, task class, model chosen, rationale |
| `audit_log/transactions.jsonl` | `transactions.py` | Full transaction record: compliance stages, transfer result |
| `audit_log/batch_results.json` | `batch_fraud.py` | Batch risk scores by transaction ID |
| `audit_log/bandit_scan.txt` | CI / manual `bandit` run | Static analysis findings |

---

## PCI DSS Controls in Architecture

| Control | Implementation |
|---|---|
| Secrets from env only | `os.environ[...]` throughout; `load_dotenv()` in entry point |
| PAN masking | `registry._mask()` on all MCP audit log entries |
| Input sanitization | `chatbot.sanitize()` on all user-facing input |
| Compliance gate before money movement | `run_compliance()` must return APPROVE before `transfer_funds` |
| Full audit trail | Layered logs: MCP hops + routing + transaction outcome |
| Model routing rationale | Every routing decision logged with human-readable rationale |

---

## Known Architectural Gaps

1. **No per-session rate limiting** on `transactions.process()` — required by CLAUDE.md, not yet implemented.
2. **Partial sanctions hits are approved** in `compliance/pipeline.py:33` — should be escalated for manual review.
3. **LLM response from fraud engine is not validated** before use as a risk signal.
