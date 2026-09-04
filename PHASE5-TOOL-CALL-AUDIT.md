# Phase 5 (Proposed): Client-side Tool-Call Audit Log

> **Status:** Proposal / RFC — not yet implemented. Filed to align on
> approach per [CONTRIBUTING.md](CONTRIBUTING.md) before a PR is opened.
> Referenced from [ENTERPRISE.md](ENTERPRISE.md)'s "Audit logging"
> section, which explicitly names this gap as a Phase 5 candidate.

## 1) Problem

`ENTERPRISE.md` states audit happens at the **gateway layer**: every
provider call carries the user's SSO identity, so "who talked to which
model, when" is fully covered once `policies.gateway.enabled: true`.

What the gateway **cannot** see is what happens *between* provider
calls on the user's machine:

- which files a `write_file`/`edit_file` tool call touched, and what
  changed
- which shell commands ran via `bash` (and whether `confine.rs`
  allowed, restricted, or passed them through unconfined)
- which MCP tools were invoked, with what arguments, and their result
- which `pre_tool_use` hook/gate decision applied (approve, deny, or
  bot-gated approval via LINE/Telegram — see `permissions.rs`)

For regulated environments (finance, healthcare, government) "the
gateway logged an LLM call" is not sufficient evidence — auditors need
the actual side effects, correlated with the request that caused them.

## 2) Non-goals

- **Not** a replacement for gateway audit. This is a client-side
  supplement, correlated by `run_id`, not a competing source of truth
  for "who called which model."
- **Not** a new approval mechanism. `permissions.rs` (`ApprovalSink`,
  `ApprovalDecision`) and `confine.rs` (OS-level confinement) already
  decide *whether* an action proceeds. This proposal only **records**
  what already happened/was decided.
- **Not** enabled by default for open-source builds — this is a
  `policies.audit` block, same activation model as `branding` /
  `plugins` / `gateway` / `sso`. Absent policy → zero behavior change,
  same as every other EE phase.

## 3) Design

### 3.1 Policy block (extends the existing schema)

```json
"policies": {
  "audit": {
    "enabled": true,
    "sink": "file",
    "path": "/var/log/thclaws/audit.jsonl",
    "include_tool_input": true,
    "include_tool_output": false,
    "redact_patterns": ["(?i)api[_-]?key", "(?i)password", "(?i)secret"]
  }
}
```

- `sink`: `file` (JSONL, this proposal's default) now; `syslog` /
  `http` (push to an internal collector) as follow-up sinks behind the
  same trait, not blocking v1.
- `include_tool_output`: defaults `false` — tool results can contain
  file contents; opt-in only, since it changes the data-sensitivity
  profile of the log itself.
- `redact_patterns`: applied to both input and output before the
  record is written, reusing the same regex-redaction approach
  `gateway`'s `auth_header_template` masking already establishes as a
  precedent in this codebase.

### 3.2 Integration points (no core behavior changes, only observation)

| Existing mechanism | What Phase 5 adds |
|---|---|
| `hooks::fire_pre_tool_use_gate` (`hooks.rs`) | After the gate returns `PreToolDecision`, emit one audit record with the decision (`Approve`/`Deny`/reason) — this already has `tool_name` + truncated input in scope |
| `permissions::ApprovalSink::approve` (`permissions.rs`) | Wrap sinks (`AutoApprover`, `ReplApprover`, bot-gated sinks) so every `ApprovalDecision` is also emitted as an audit record, tagged with which sink decided (Repl/LINE/Telegram/Auto) |
| `confine::ConfinePolicy` (`confine.rs`) | Record which `ConfineMode` (`workspace`/`strict`/`off`) applied to a given `bash` spawn, and whether the platform confiner was actually available (the module already logs "unconfined" once — Phase 5 turns that into a structured, per-call record instead of a one-time log line) |
| MCP tool dispatch (`mcp.rs`) | Emit one record per MCP `call_tool`, including server name + tool name (not full arguments unless `include_tool_input`) |

All four hook points already exist and already have the data Phase 5
needs in scope — this is additive instrumentation, not new plumbing.

### 3.3 Record schema (JSONL, one object per line)

```json
{
  "ts": "2026-09-04T06:00:00Z",
  "run_id": "01M1...",
  "session_id": "...",
  "user": "alice@acme.example",
  "event": "tool_call",
  "tool": "bash",
  "decision": "approve",
  "decided_by": "repl",
  "confine_mode": "workspace",
  "confine_active": true,
  "input_redacted": "cmd=git status",
  "output_redacted": null,
  "duration_ms": 42
}
```

`run_id` is the same identifier the gateway integration already
attaches to provider calls (per `policies.gateway` identity
injection), so client-side and gateway-side logs join on one key
without inventing a second correlation scheme.

### 3.4 Fail-open, not fail-closed

Unlike `gateway.fail_closed`, audit sink failures (disk full, syslog
unreachable) **must not** block tool execution — losing an audit
record is recoverable (alert + backfill from session JSONL, which
already persists under `.thclaws/sessions/`); blocking a user's shell
command because the audit disk is full is not an acceptable trade-off
for most deployments. This mirrors how `confine.rs` already treats an
unavailable confiner as "passthrough, logged once" rather than a hard
failure.

## 4) Acceptance criteria

- With `policies.audit.enabled: false` or block absent: zero overhead,
  identical behavior to today (matches the existing phase activation
  contract).
- With it enabled: every `bash`, `write_file`/`edit_file`, and MCP tool
  call produces exactly one audit record, including denied/blocked
  calls (a denial is itself an auditable event).
- Redaction patterns apply before the record is serialized, not after
  (never write, then redact-in-place).
- Audit sink outage does not block or delay tool execution.
- `thclaws-policy-tool inspect policy.json` shows the `audit` block
  like it already shows the other four.

## 5) Rollout

1. `sink: file` only, `include_tool_output: false` by default — ship
   the narrowest useful version first.
2. Add `syslog` sink once `file` is validated in a real deployment.
3. Add `http` sink (push to internal collector/SIEM) as a follow-up,
   reusing whatever HTTP client the `gateway` policy block already
   uses internally.

## 6) Relationship to this fork

Filed from `kenjiroe/thClaws` (a fork of `thClaws/thClaws`) as a design
doc first, per `CONTRIBUTING.md`'s "for anything non-trivial please
open an issue first to align on approach." Intent is to open the
corresponding GitHub issue against `thClaws/thClaws`, get maintainer
sign-off on the approach above, then implement and submit as a PR
against upstream `main` — not to maintain this as a long-lived
divergent fork feature.
