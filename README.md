# ⟳ SmartReplayOps
### Intelligent Trade Replay & Recovery Automation

> A portfolio concept by **Gowri BP — Senior Application Support Analyst · Banking Domain**
> demonstrating how banking trade failures caused by MQ and database outages can be
> detected, classified, and recovered intelligently — with idempotency guarantees,
> settlement cut-off controls, instrument-aware governance, and immutable audit trails.

---

**[🚀 View the live interactive demo →](https://gowribp.github.io/smartreplayops)**
&nbsp;&nbsp;&nbsp;
**[📄 Download the case study →](docs/SmartReplayOps-CaseStudy.pdf)**

---

## What this is

This is a **production support concept portfolio** — built to demonstrate the systems
thinking, domain knowledge, and automation mindset that distinguishes a senior application
support analyst from a junior one.

The interactive dashboard simulates a real banking scenario:

> A trade enters the system. Mid-pipeline, the MQ queue manager goes down — or the
> database times out — or the outbound publish fails after the trade is already booked
> internally. What happens next?

Most support teams investigate manually. This platform demonstrates what an intelligent,
automated recovery system looks like — with the governance controls and audit evidence
that regulators and change management processes require.

---

## The scenario

Trades in a bank flow through a distributed pipeline:

```
Front Office → MQ Queue → Trade Processor → Database → Settlement → Risk → Reporting
```

When any component fails mid-pipeline, trades end up in partial states:

| Failure | What breaks | Risk |
|---|---|---|
| MQ outage | Trade never reaches processor | Lost trade |
| DB timeout | Trade validates but cannot book | Partial state |
| Partial publish | Trade books internally, downstream unaware | Reconciliation break |
| Poison message | Malformed data fails every attempt | Infinite retry loop |
| Post-cut-off failure | Failure after settlement window closes | False success — nothing settles |

---

## The four recovery levels

### Level 1 — MQ Auto-Recovery
Detects queue manager disconnect via heartbeat failure. Automatically reconnects
consumers and restarts listeners. Consumer-side deduplication fires on reconnect —
if a trade was partially processed before disconnect, the idempotency key check
prevents re-processing.

### Level 2 — Safe Replay Verification
Before replaying any failed trade, the engine verifies safety using a
`UNIQUE(trade_ref, workflow_stage)` database constraint — not just a state table check.
The replay uses upsert semantics. A constraint violation means a duplicate has been
detected and is rejected at the database level before any processing begins.

### Level 3 — Partial-State Detection
Checks which downstream steps completed per trade. If a trade is booked but not
published, scope is set to `PUBLISH_ONLY`. Before executing, the engine checks the
settlement system's active consumer count — if zero, replay is deferred rather than
firing into a silent queue that would produce a false `REPLAY_SUCCESS`.

### Level 4 — Checkpoint-Based Resumption
Every trade stores its last successful `workflow_stage` as a checkpoint. On replay,
processing resumes from that exact stage. A pre-replay amendment check also verifies
that no newer version of the same trade exists — preventing an old price from
overwriting a price amendment received after the original failure.

---

## Governance

### Instrument-aware approval matrix

| Instrument | Auto-replay limit | Above limit | Rationale |
|---|---|---|---|
| EQUITY | ≤ $500,000 | Manual approval | T+1 DTCC — bilateral risk low |
| FX | ≤ $2,000,000 | Manual approval | Bilateral reversible — CLS T+2 |
| RATES | Always manual | Always manual | Notional ≠ exposure (DV01) |
| CREDIT | Always manual | Always manual | Counterparty risk sensitivity |

### Settlement cut-off windows

| Asset class | Settlement | Cut-off | Action past cut-off |
|---|---|---|---|
| EQUITY | T+1 via DTCC | 16:30 ET | `REQUIRES_AMENDMENT` |
| FX | T+2 via CLS | 17:00 ET | `REQUIRES_AMENDMENT` |
| RATES | T+2 via LCH | 15:00 ET | `REQUIRES_AMENDMENT` |
| CREDIT | T+3 via DTCC | 14:00 ET | `REQUIRES_AMENDMENT` |

---

## Poison message handling

Trades that fail on every attempt are quarantined after `MAX_RETRIES = 3`:

```
Retry 1 → auto-retry (5s backoff)
Retry 2 → auto-retry (15s backoff)
Retry 3 → QUARANTINED → routed to DLQ → manual investigation flag raised
```

---

## The audit trail

Every state transition is logged immutably with a correlation ID:

```
10:42:01  TRADE_RECEIVED      PRODUCER      BUY 600 QQQ @ 438.90                         SUCCESS
10:42:03  TRADE_FAILED        PROCESSOR     MQ connection lost at PUBLISH                 FAILED
10:44:15  IDEMPOTENCY_CHECK   AUTO_ENGINE   UNIQUE(TRD-012:BOOKED) — no prior commit      SUCCESS
10:44:15  GOVERNANCE_CHECK    AUTO_ENGINE   EQUITY $263,340 ≤ $500K — auto approved       SUCCESS
10:44:15  REPLAY_INITIATED    AUTO_ENGINE   Scope: FULL | Resume: ENRICHED | Batch: 1/10  PENDING
10:44:17  STAGE_BOOKED        PROCESSOR     DB upsert — idempotency key accepted           SUCCESS
10:44:18  STAGE_PUBLISHED     PROCESSOR     Consumer count: 3 active — publish OK          SUCCESS
10:44:19  REPLAY_SUCCESS      AUTO_ENGINE   Trade fully recovered                          SUCCESS
```

Decisions **not** to replay are also logged — `CLOSED_NO_ACTION` with mandatory
reason and approver identity. This is the evidence that internal audit and compliance
teams require.

---

## How to use the demo

1. Open the [live demo](https://gowribp.github.io/smartreplayops)
2. Go to **⚡ Break things** — inject an MQ Outage
3. Go to **Trades** → click **+ Produce trade**
4. Watch the trade fail — observe failure type and stage
5. Go back to **Break things** → click **Heal all systems**
6. Click the failed trade → click **Replay trade**
7. Watch the audit timeline update stage by stage with idempotency and governance checks
8. Try **Partial Publish** — observe `PUBLISH_ONLY` scope decision
9. Try **Settlement Cut-off** — observe `REQUIRES_AMENDMENT` blocking auto-replay
10. Try **Poison Message** — watch 3 retries then `QUARANTINED` → DLQ routing
11. Check **TRD-009** (5Y IRS, RATES) — always manual regardless of notional
12. Check **TRD-016** (META) — `CLOSED_NO_ACTION` with audited reason

---

## Additional failure scenarios covered

| Scenario | Handling |
|---|---|
| Network partition — ACK lost | Consumer-side deduplication before every processing attempt |
| Amendment received after failure | Pre-replay supersession check — original marked `SUPERSEDED` |
| Downstream system also degraded | Consumer count check before publish-only replay |
| SSL/TLS certificate expiry | Reconnect failure classified — loop stopped, infra team alerted |
| Trade dependency violation | `depends_on_trade_id` — sequenced by dependency chain then timestamp |
| Mass replay after outage | `REPLAY_BATCH_SIZE` config — prioritised by cut-off proximity |

---

## Project structure

```
smartreplayops/
├── index.html                  # Interactive demo dashboard (no server needed)
├── README.md                   # This file
├── _config.yml                 # GitHub Pages configuration
└── docs/
    └── SmartReplayOps-CaseStudy.pdf   # Full written case study
```

---

## Running locally

No installation required. Simply open `index.html` in any browser.

```bash
# Clone the repository
git clone https://github.com/gowribp/smartreplayops.git

# Open in browser
open smartreplayops/index.html
```

---

*Portfolio — Gowri BP · Senior Application Support Analyst · Banking Domain*
