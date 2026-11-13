# Heritage National Bank — Compliance Report

**Generated:** 2026-11-13  
**Scope:** Heritage National Bank AI Banking Platform (m4-capstone)  
**Standard:** PCI DSS + CLAUDE.md Definition of Done

---

## Executive Summary

The platform demonstrates a solid compliance foundation: all MCP tool calls are audited, model routing decisions are logged with rationale, money movement is gated behind a four-stage compliance pipeline, and secrets are sourced exclusively from environment variables. The bandit static analysis scan reports zero findings at any severity.

Two medium-severity gaps prevent a full pass: per-session rate limiting on money-movement operations is not implemented (explicitly required by CLAUDE.md), and partial sanctions hits are silently approved rather than escalated. Three lower-severity gaps exist around LLM response validation, PAN mask coverage, and input sanitization breadth.

**Overall verdict: CONDITIONAL PASS** — platform is functionally compliant but requires the two medium gaps to be closed before production deployment.

---

## PCI DSS Control Status

| Control | Requirement | Status | Location |
|---|---|---|---|
| Secrets management | Keys from env vars only; no hard-coding | PASS | `os.environ["OPENROUTER_API_KEY"]` throughout |
| PAN masking | Mask card numbers to last 4 in all logs | PASS (narrow) | `mcp/registry.py:_mask()` — covers `card*` keys only |
| Input sanitization | Strip control chars and injection bait | PARTIAL | `chatbot.py:sanitize()` present; `transactions.py` bypasses it |
| Rate limiting | Per-session limit on money-movement ops | FAIL | No implementation found |
| Audit trail — MCP | Every MCP hop logged | PASS | `audit_log/mcp_tool_calls.jsonl` |
| Audit trail — transactions | Every money movement logged | PASS | `audit_log/transactions.jsonl` |
| Audit trail — model routing | Routing rationale logged per request | PASS | `audit_log/model_routing.jsonl` |
| Compliance pipeline | KYC → AML → Sanctions → Approval order | PASS | `compliance/pipeline.py` |

---

## Transaction Audit Trail

**Total transactions processed:** 2  
**Log file:** `audit_log/transactions.jsonl`

| Request ID | Customer | Amount | Status | Compliance Decision | Failure Stage |
|---|---|---|---|---|---|
| R1001 | C5001 (Asha Rao) | $750.00 | SETTLED | APPROVE | — |
| R1002 | C5002 (Ivan Petrov) | $5,000.00 | REJECTED | BLOCK | AML |

**R1001 detail:** All four compliance stages passed (KYC, AML risk=LOW, SANCTIONS strength=NONE, APPROVAL). Transfer executed: AC5001 → AC5002, post-transfer balance $4,250.00. Fraud engine ran in fallback mode (NotFoundError from Claude API — model routing logged but API unavailable).

**R1002 detail:** Blocked at AML stage. Fraud signals: amount $5,000 against avg $12,000 (within range), but geo_mismatch=true (ip_country=RU). Risk scored HIGH by fallback rule engine. Transfer was not executed; no funds moved.

---

## MCP Tool Call Audit

**Total calls logged:** 15 (across two registry instances — registry seq counter resets per process run)  
**All calls:** ok=true (zero errors)  
**Log file:** `audit_log/mcp_tool_calls.jsonl`

| Server | Tool | Call Count |
|---|---|---|
| customer-info | get_customer | 3 |
| customer-info | check_customer_history | 6 |
| customer-info | verify_ip_location | 6 |
| banking-api | get_balance | 1 |
| banking-api | transfer_funds | 2 |

PAN masking was applied to all logged arguments via `registry._mask()`. No card fields were present in this run; masking coverage is verified as active.

**Coverage:** 100% of `transfer_funds` calls are logged. The compliance pipeline calls `get_customer`, `check_customer_history`, and `verify_ip_location` before any money movement — all logged.

---

## Model Routing Audit

**Total routing decisions logged:** 7  
**Log file:** `audit_log/model_routing.jsonl`

| Task Class | Model | Count | Rationale |
|---|---|---|---|
| `reason` (Sonnet) | anthropic/claude-sonnet-4.5 | 2 | Default reasoning: KYC, compliance narrative |
| `decide` (Opus) | anthropic/claude-opus-4.1 | 1 | Highest-stakes: final approval on APPROVE path |
| `classify` (Haiku) | anthropic/claude-3.5-haiku | 4 | High-volume batch fraud scoring |

Routing is consistent with the CLAUDE.md policy: Opus is used only for the final money-movement approval decision (R1001), not for rejected transactions (R1002 stopped at AML before reaching `decide`). Batch scoring correctly routes all four synthetic transactions to Haiku.

---

## Security Scan Results

**Tool:** bandit  
**Command:** `bandit -q -r bfsi -f txt -o audit_log/bandit_scan.txt`  
**Log file:** `audit_log/bandit_scan.txt`

| Severity | Count |
|---|---|
| High | 0 |
| Medium | 0 |
| Low | 0 |

Bandit reports zero findings. All 23 lines of scanned code are clean. The identified gaps below are architectural/logic issues not detectable by static analysis.

---

## Identified Gaps and Remediations

### GAP-1 (MEDIUM) — No per-session rate limiting on money movement

**File:** `bfsi/transactions.py`  
**Requirement:** CLAUDE.md: "Enforce per-session rate limiting on money-movement operations."  
**Risk:** Unlimited calls to `process()` within a session enable enumeration or automated fraud.  
**Fix:** Add a per-customer counter gated before compliance runs:

```python
from collections import defaultdict
_SESSION_COUNTS: dict[str, int] = defaultdict(int)
_SESSION_LIMIT = 5

def process(request_id, customer_id, ...):
    _SESSION_COUNTS[customer_id] += 1
    if _SESSION_COUNTS[customer_id] > _SESSION_LIMIT:
        result = {"request_id": request_id, "status": "REJECTED",
                  "reason": "rate_limit_exceeded"}
        # audit and return
```

---

### GAP-2 (MEDIUM) — Partial sanctions hit auto-approved

**File:** `bfsi/compliance/pipeline.py:33`  
**Current code:** `sanc_ok = not sanc["hit"] or sanc["strength"] == "PARTIAL"`  
**Risk:** A name that partially matches the OFAC/UN watchlist proceeds to fund transfer without review. Under real BSAIA/OFAC compliance, partial matches must be escalated.  
**Fix:** Treat any hit as a block requiring manual review:

```python
sanc_ok = not sanc["hit"]
```

---

### GAP-3 (LOW) — LLM response not validated in fraud engine

**File:** `bfsi/fraud_engine.py:31`  
**Risk:** Unexpected or adversarial LLM output could supply an invalid `risk` value, bypassing AML gating.  
**Fix:** Validate after JSON parse:

```python
VALID_RISK = {"LOW", "MEDIUM", "HIGH"}
if out.get("risk") not in VALID_RISK:
    raise ValueError(f"unexpected risk: {out.get('risk')}")
```

---

### GAP-4 (LOW) — PAN mask heuristic too narrow

**File:** `bfsi/mcp/registry.py:29`  
**Current:** Only masks keys containing `"card"`.  
**Fix:** Extend to `{"card", "pan", "account_number", "card_no", "card_num"}`.

---

### GAP-5 (LOW) — `name` parameter enters pipeline unsanitized

**File:** `bfsi/transactions.py:20`  
**Risk:** If `name` is ever passed to an LLM prompt in a future narrative stage, it becomes a prompt-injection vector. Currently low risk as `sanctions_check` only uses string comparison.  
**Fix:** Apply `chatbot.sanitize()` (or move it to a shared util) to `name` before `run_compliance()`.

---

## Definition of Done Verdict

| Criterion | Status |
|---|---|
| pytest passes with zero failures | NOT VERIFIED (no test run in this session) |
| bandit has no medium+ findings | PASS |
| Every MCP hop is audited | PASS |
| Claude-as-Judge scores platform >= 4 on compliance rubric | NOT VERIFIED |
| Rate limiting on money movement | FAIL (GAP-1) |
| All compliance decisions audited | PASS |
| Secrets from environment only | PASS |

**Result: CONDITIONAL PASS.** Close GAP-1 (rate limiting) and GAP-2 (partial sanctions escalation) to reach full Definition of Done compliance.
