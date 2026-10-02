# AgentFxTrading — Master Roadmap

Repository: armanemp/AgentFxTrading
Upstream: kienphan/AgentFxTrading
Baseline: 95ce4e0d6f06411321a72f25ec02703914525fb7
Baseline date: 2026-09-27
Roadmap created: 2026-10-02

## Product goal

Turn this fork into a clean, testable and personally usable cTrader trading platform with:
- three complementary deterministic strategy engines
- one canonical decision contract
- one authoritative risk/execution policy
- a complete Rules-Only mode
- optional AI validation/management
- reliable news handling
- portfolio-level risk control
- reproducible backtesting and walk-forward validation
- persistent decision and trade history
- fast dashboard/telemetry
- safe cTrader Robot execution
- a future-proof API boundary so analysis can later move to a cloud server while the execution Robot remains local.

## Core architecture rule

One behavior has one authoritative owner.

Target flow:

cTrader / Broker
 -> Market Data Adapter
 -> Canonical Snapshot
 -> Strategy Engine
 -> Decision Policy
 -> Optional AI Validator/Manager
 -> Risk Plan
 -> Execution Gateway
 -> Position Manager
 -> Telemetry / Audit

AI can propose or validate within bounds, but risk and broker execution remain deterministic.

## Modes

RULES_ONLY
- no LLM credential required
- deterministic strategy decision
- deterministic risk and execution
- full backtest support

AI_VALIDATED
- deterministic candidate first
- AI can accept/reject/classify
- AI cannot widen risk or bypass hard safety

AI_MANAGED
- deterministic candidate and risk plan remain authoritative
- AI may manage an open position inside strict policy limits.

## Phase 00 — Baseline, audit and roadmap

Status: DONE

Completed:
- fork baseline captured
- architecture audited
- AI dependency identified
- monolithic modules identified
- safety/risk/execution ownership problems identified
- roadmap written
- docs/PROJECT-AUDIT-2026-10-02.md added
- docs/ROADMAP.md added

Exit:
- no trading behavior changed by documentation work.

## Phase 01 — Authority Map & Full-System Safety Inventory

Objective:
Prove exactly which path owns every trading action before refactoring.

Work:
1. Inventory all entry paths.
2. Inventory all exit paths.
3. Inventory every SL/TP modification path.
4. Inventory volume/risk calculations.
5. Inventory news/session/circuit-breaker gates.
6. Inventory dashboard commands.
7. Inventory portfolio/reporting paths.
8. Detect duplicate logic between Python and C#.
9. Detect legacy and dead flags.
10. Quarantine unvalidated DCA/averaging-down behavior.

Deliverables:
- docs/AUTHORITY-MAP.md
- docs/SAFETY-INVENTORY.md
- ownership table suitable for regression tests.

Exit:
- every behavior has one named owner
- every legacy path is keep/refactor/quarantine/remove
- no undocumented order path remains.

## Phase 02 — Canonical Contracts

Objective:
Stop signal/decision/execution drift.

Create:
- MarketSnapshot
- StrategyContext
- SignalCandidate
- Decision
- RiskPlan
- ExecutionRequest
- ExecutionResult
- PositionState
- NewsState
- ProtectionState

Every message should carry:
- schema_version
- request_id
- bot_id
- strategy_id
- strategy_version
- symbol
- timeframe
- decision_bar_time
- side
- candidate state
- entry reference
- stop
- target
- initial risk
- reward
- RR
- decision expiry
- risk/target/stop authority.

Exit:
- stale/wrong-symbol/wrong-timeframe/wrong-strategy responses are rejected.

## Phase 03 — Rules-Only Foundation

Objective:
Make a complete non-AI trading system.

Work:
- add an explicit disabled-LLM/NullLLM path
- deterministic TMS+ORB entry decision
- deterministic Judas entry decision
- preserve and clean FlowRSI technical entry
- deterministic open-position management
- deterministic safe fallback
- no fake "AI disabled because API failed" behavior.

Exit:
- server starts with no LLM credentials
- all three engines can generate BUY/SELL/HOLD and management decisions where their strategy rules permit
- backtests run without AI
- no LLM HTTP call is made in RULES_ONLY.

## Phase 04 — Thin cBot / Execution Gateway

Objective:
Reduce the cBots to thin market-data/execution adapters.

Work:
- shared execution helper
- shared request validation
- shared broker pre-flight
- shared position lifecycle
- shared report/retry/idempotency
- shared telemetry
- shared protection engine

Refactor target:
cBots collect broker data, exchange contracts, execute approved requests and report results. Strategy policy must not be duplicated inside multiple cBots.

Exit:
- one authoritative execution path per cBot.

## Phase 05 — TMS + ORB Engine Hardening

Work:
- macro/chart TMS
- cross freshness
- stochastic confirmation
- Heikin Ashi
- ORB formation
- decisive breakout
- retest/bounce model
- breakout aging
- anti-chase
- anti-overextension
- market regime
- chop override
- post-TP gate
- session state.

Required tests:
- pre-session cross
- broker reconnect/re-published bars
- closed-bar indexing
- failed breakout
- stale breakout
- fresh breakout
- choppy market
- post-TP re-entry
- macro/chart disagreement.

## Phase 06 — Judas SMC Engine Hardening

Work:
- Asian range
- killzones
- sweep direction
- rejection wick
- structural side lock
- stale sweep
- OB confirmation
- FVG confirmation
- liquidity target
- news state
- session state

Legacy cleanup:
- isolate DCA
- disable averaging-down in production profiles
- remove competing direct order path
- consolidate EMA-close behavior under canonical position policy.

Exit:
- one canonical Judas entry decision per bar
- side inversion impossible downstream.

## Phase 07 — FlowRSI / Nested RSI Hardening

Work:
- fast/slow RSI
- cross freshness
- TMS cross-lock
- FVG
- premium/discount
- liquidity sweep
- swing structure
- technical SL/TP
- candidate RR
- AI optional validation.

Exit:
- Rules-Only and AI-Validated use the same candidate
- AI cannot inject unvalidated entry risk.

## Phase 08 — Market Structure, OB and FVG Core

Build reusable components:
- swing detector
- HH/HL/LH/LL classifier
- BOS
- CHOCH
- liquidity pools
- Order Block detector
- OB freshness/mitigation/quality
- FVG detector
- FVG age/mitigation/quality
- premium/discount
- displacement quality
- confluence graph.

Quality principle:
Confluence must be explainable and normalized. Correlated indicators must not be double-counted blindly.

Important data note:
Current candle-close signed-volume logic is only a volume-direction proxy, not true bid/ask delta.

## Phase 09 — Canonical Entry / SL / TP / RR Engine

Objective:
One source of truth for price protection and reward.

Work:
- structural stop
- ATR bounds
- liquidity-aware targets
- minimum reward
- target RR
- broker distance
- spread/slippage buffers
- dynamic volume
- post-clamp volume re-sizing
- exact price normalization.

Invariants:
- wrong-side SL => hard reject
- wrong-side TP => hard reject
- SL only ratchets toward less risk
- TP never silently regresses
- final position size is based on final SL distance.

## Phase 10 — Position Protection Engine

Create a single lifecycle:
- initial risk capture
- break-even
- commission/swap-aware buffer
- staged protection
- gated trailing
- structural trailing
- MFE
- giveback
- TP1/partial close
- runner
- restart recovery.

Exit:
- no TP regression
- no SL widening
- no repeated modification loop
- protection state is deterministic and testable.

## Phase 11 — News & Macro Service

Work:
- provider interface
- cache
- UTC canonical time
- timezone normalization
- currency/symbol mapping
- impact normalization
- pre/post blackout
- stale provider policy
- provider outage policy
- auditable block reason.

Exit:
- news failure cannot accidentally enable new entries.

## Phase 12 — Portfolio & Global Risk Orchestrator

Controls:
- max positions
- symbol cap
- daily loss
- equity drawdown
- margin usage
- currency exposure
- correlation
- strategy exposure
- directional concentration
- session restrictions
- emergency kill switch.

Exit:
- all three engines use one portfolio risk policy.

## Phase 13 — Backtest & Research Lab

Work:
- Rules-Only backtest
- AI comparison backtest
- parameter sweeps
- walk-forward validation
- out-of-sample holdout
- sensitivity analysis
- Monte Carlo trade-order perturbation
- realistic spread/commission/slippage
- regime/session/symbol segmentation
- MAE/MFE
- expectancy
- drawdown duration
- exposure metrics
- strategy-version stamping.

Research rule:
Never optimize directly against the final evaluation period.

Exit:
Every research report contains dataset identity, configuration, strategy version and reproducible parameters.

## Phase 14 — Optional AI Integration

Objective:
Use AI as a bounded validator/co-pilot rather than a hard dependency.

Work:
- provider interface
- structured response schema
- bounded action set
- confidence as advisory metadata
- no risk widening
- no raw volume authority
- stale response rejection
- cost/token budget
- retry/deadline policy
- deterministic failure behavior
- optional local OpenAI-compatible model endpoint.

Exit:
AI outage leaves deterministic trading fully functional.

## Phase 15 — Persistence, History & Calibration

Work:
- append-only trade journal
- decision journal
- candidate/outcome journal
- daily archive
- 90-day rolling dataset plus long-term archive
- strategy-version tagging
- symbol profiles
- regime statistics
- confidence calibration
- import/export bundles
- backup/restore tests.

Exit:
history survives restart/upgrade and no trade silently disappears because telemetry failed.

## Phase 16 — Dashboard & Observability

Dashboard should expose:
- active positions
- current candidate
- final decision
- strategy/version
- MTF context
- OB/FVG state
- risk plan
- SL/TP/RR
- gate reasons
- news state
- execution state
- decision/execution latency
- AI latency/cost
- watchdog state
- recent errors
- full decision timeline.

Performance:
UI rendering must not perform expensive trading calculations.

## Phase 17 — Reliability & Security

Work:
- authenticated local API
- secret management
- live/demo isolation
- shadow mode
- stale market-data checks
- stale decision checks
- idempotent commands
- broker reconnect recovery
- server restart recovery
- cBot restart recovery
- database recovery
- watchdog
- emergency kill switch
- audit logging.

## Phase 18 — CI/CD & Engineering Quality

Add:
- GitHub Actions
- Python unit/integration suite
- lint/format
- type checking
- dependency/security scanning
- C# build validation where runner support permits
- contract tests
- smoke tests
- backtest regression snapshots.

Merge requirement:
No strategy/risk change without tests and documentation.

## Phase 19 — Demo / Shadow Validation

Modes:
- Rules-Only demo
- AI-Validated shadow
- AI-Managed demo
- execution shadow with no broker order.

Compare:
- candidate
- AI verdict
- final policy
- hypothetical execution
- actual demo execution.

Go-live decisions must be based on measured evidence from target symbols, timeframes and broker conditions.

## Phase 20 — Production Packaging

Work:
- local Windows/cTrader deployment
- optional VPS
- Docker
- system service
- profile validation
- environment validation
- backups
- health endpoint
- release artifacts
- upgrade/rollback.

## Phase 21 — Future Cloud Brain / Local Robot

Target:
Local cTrader Robot
 -> secure decision API
 -> remote analysis brain

Remote layer may contain:
- deterministic strategy engines
- optional AI
- news
- portfolio context.

The local Robot must retain enough safety policy to refuse unsafe execution when the remote service is unavailable or stale.

## Phase 22 — Continuous Optimization & Whole-Project Audit

Repeat after every substantial milestone.

Performance:
- server latency
- cBot latency
- dashboard responsiveness
- CPU
- memory
- network calls
- LLM calls/tokens
- backtest throughput.

Code:
- dead code
- duplicate behavior
- oversized modules
- obsolete flags
- duplicated calculations
- noisy logs
- unnecessary polling.

Trading correctness:
- candidate/decision consistency
- entry/SL/TP consistency
- risk/volume consistency
- position/report consistency
- news consistency
- dashboard consistency.

This is a recurring audit, not a one-time phase.

# Definition of done for every phase

A phase is complete only when:
1. implementation is complete
2. no half-migrated path remains
3. old code is removed or explicitly quarantined
4. relevant tests pass
5. trading behavior is backtested when applicable
6. docs are updated
7. whole-project audit is run
8. performance impact is checked
9. exact commit/PR is recorded
10. user is told whether local git pull is required.

# Branching policy

Use one branch per phase:
- phase/01-authority-map
- phase/02-canonical-contracts
- phase/03-rules-only
- ...

Preferred delivery:
implementation -> tests -> audit -> docs -> PR -> merge

Avoid:
temporary workaround -> duplicate path -> later cleanup.

# Upstream synchronization policy

For every upstream sync record:
- upstream SHA
- local/fork SHA before sync
- local/fork SHA after sync
- files changed
- conflicts
- accepted changes
- rejected changes and reasons.

Never merge upstream strategy behavior blindly.

# Current status

| Phase | Status |
|---|---|
| 00 Baseline, audit and roadmap | DONE |
| 01 Authority map | PLANNED |
| 02 Canonical contracts | PLANNED |
| 03 Rules-Only foundation | PLANNED |
| 04 Thin cBot / execution gateway | PLANNED |
| 05 TMS + ORB hardening | PLANNED |
| 06 Judas SMC hardening | PLANNED |
| 07 FlowRSI hardening | PLANNED |
| 08 Market structure / OB / FVG core | PLANNED |
| 09 Canonical SL/TP/RR | PLANNED |
| 10 Position protection | PLANNED |
| 11 News & macro | PLANNED |
| 12 Portfolio risk | PLANNED |
| 13 Backtest/research lab | PLANNED |
| 14 Optional AI | PLANNED |
| 15 Persistence/calibration | PLANNED |
| 16 Dashboard/observability | PLANNED |
| 17 Reliability/security | PLANNED |
| 18 CI/CD | PLANNED |
| 19 Demo/shadow validation | PLANNED |
| 20 Production packaging | PLANNED |
| 21 Cloud brain / local robot | PLANNED |
| 22 Continuous optimization/audit | ONGOING |

# Immediate next phase

Phase 01 — Authority Map & Full-System Safety Inventory.

Do not tune strategy thresholds first. First make every entry, exit, SL, TP, volume, risk, news, dashboard and telemetry path explicit and authoritative.
