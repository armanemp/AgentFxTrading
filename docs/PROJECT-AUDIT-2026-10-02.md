# AgentFxTrading — Deep Baseline Audit

Date: 2026-10-02
Repository: armanemp/AgentFxTrading
Upstream: kienphan/AgentFxTrading
Baseline commit: 95ce4e0d6f06411321a72f25ec02703914525fb7
Baseline commit date: 2026-09-27T04:37:55Z
Audit branch: phase/00-baseline-audit-roadmap

## 1. Baseline status

The fork currently points to exactly the same commit as upstream. The fork was created on 2026-10-02, while the code head remains the upstream merge commit from 2026-09-27.

Observed baseline:
- Default branch: main
- Fork: yes
- Public repository
- 128 tracked files
- 95 Python files
- 3 C# cBots
- 72 Python test files
- No GitHub Actions workflow directory
- GitHub API did not expose a repository license object; README contains an MIT badge. Licensing must be verified before redistribution of modified code.

Latest upstream/fork commit:
95ce4e0d6f06411321a72f25ec02703914525fb7
Merge pull request #64 from kienphan/feat/backtest-paging-modal
feat: page the backtest runs, open run detail in a modal, fold the form

## 2. Repository shape

Major areas:
- app/ — FastAPI server, LLM abstraction, portfolio/risk, news, dashboard, cBot management, backtesting
- cBot/ — three large cTrader Robot implementations
- tests/ — broad regression and integration coverage
- scripts/ — installation, migration, backup and recalibration
- templates/ and static/ — dashboard UI
- docs/ — deployment and screenshots
- .claude/ and .omp/ — project automation and instructions

Largest implementation files observed:
- cBot/AsianRangeJudasSweepBot.cs — about 226 KB
- templates/dashboard.html — about 205 KB
- cBot/FlowRsiBot.cs — about 164 KB
- app/server.py — about 152 KB
- cBot/AiAgentBot.cs — about 140 KB
- app/dashboard.py — about 67 KB
- app/portfolio.py — about 46 KB
- tests/test_integration.py — about 41 KB
- static/js/backtest.js — about 41 KB
- app/news_service.py — about 31 KB

These sizes are not automatically defects, but they are strong indicators of concentrated responsibility and high coupling.

## 3. Current architecture

Current runtime intent:

cTrader cBot
 -> HTTP market snapshot
 -> FastAPI
 -> deterministic gates / portfolio / news
 -> LLM provider
 -> JSON decision
 -> cBot guardrails
 -> cTrader order and position management

The server contains strategy-specific branches for FlowRSI, Asian Range Judas Sweep, and TMS + ORB.

The cBots also contain substantial strategy logic, execution, exits, telemetry and reporting. This creates split ownership: important trading behavior is represented in both C# and Python.

## 4. AI dependency audit

app/llm_client.py provides an LLM abstraction supporting:
- OpenAI-compatible endpoints
- Qwen / DashScope
- DeepSeek
- Anthropic
- Gemini
- custom OpenAI-compatible endpoints

The current environment example defaults to Qwen and expects a provider API key.

There is no true NullLLMClient / RulesOnly provider at baseline.

This means:
- deterministic gates already exist and can skip many LLM calls;
- FlowRSI already has a technical direct-execution branch when AI Gate Mode is disabled;
- TMS + ORB still relies on the AI decision path for eligible entries;
- Judas has deterministic sweep detection plus an AI path and legacy direct execution code.

Conclusion: the current project is not a symmetric, complete no-AI system across all engines.

Target mode model:
- RULES_ONLY
- AI_VALIDATED
- AI_MANAGED

AI remains optional. Risk and execution authority stay in deterministic code.

## 5. Engine audit

### 5.1 TMS + ORB

Strengths:
- Heikin Ashi
- TDI / RSI
- Stochastic
- macro TMS and chart TMS
- EMA and bounce logic
- ORB
- Kaufman Efficiency Ratio regime
- OR flip / chop handling
- post-TP re-entry gate
- MFE and giveback
- break-even
- tiered trailing
- session-close handling
- portfolio reporting and telemetry

Architecture problem:
AiAgentBot.cs is roughly 3,000 lines and owns indicator calculations, state, HTTP orchestration, execution, exits and telemetry.

Important existing design:
LLM proposes, code disposes.

Roadmap:
- extract TMS, ORB and regime logic
- create structured signal candidates
- make deterministic entry decisions first-class
- centralize risk and execution
- reduce cBot to a thin market-data/execution adapter.

### 5.2 Asian Range Judas Sweep

Strengths:
- Asian range
- London and New York killzones
- liquidity sweep detection
- rejection wick validation
- structural side locking
- OB/FVG context
- news filter
- risk sizing
- equity circuit breaker
- trailing and break-even
- Telegram
- dashboard telemetry

Critical problem:
AsianRangeJudasSweepBot.cs is roughly 4,700 lines and contains both AI-gated behavior and legacy direct order execution.

It also exposes broad legacy DCA and averaging-down options.

Roadmap:
- one canonical entry path
- quarantine unvalidated DCA / averaging-down behavior
- explicit Rules-Only and AI-Validated modes
- convert sweep/OB/FVG conditions into a structured candidate object.

### 5.3 FlowRSI / Nested RSI SMC

Strengths:
- fast/slow RSI
- FVG
- premium/discount
- liquidity sweeps
- swings
- current/H1/H4 context
- technical SL/TP
- dynamic volume
- true break-even
- gated trailing
- partial close
- request ID validation
- symbol/timeframe/bot identity validation
- AI direction guard
- execution pre-flight
- restart recovery of initial SL distance
- portfolio reporting

Important finding:
FlowRSI already has an explicit AI gate switch and can execute a technical candidate directly when AI mode is disabled for a fresh position.

This is the best starting point for project-wide Rules-Only parity.

## 6. Server audit

app/server.py is about 150 KB and currently combines:
- API transport
- strategy routing
- deterministic gates
- prompt creation
- LLM orchestration
- response post-processing
- fallback decisions
- parts of portfolio/risk orchestration

Strengths:
- structured JSON parsing and repair
- timeout/retry/deadline controls
- deterministic pre-LLM gates
- TMS cross-lock
- Judas structural gate
- FlowRSI direction guard
- TMS close validation
- Judas adjust validation
- news blackout
- portfolio risk checks
- deterministic emergency fallback

Risks:
- too much responsibility in one module
- business rules duplicated between Python and C#
- prompt text carries strategy rules
- AI post-processing contains policy that should be first-class code.

Target decomposition:
app/api
app/contracts
app/engines
app/policy
app/risk
app/execution
app/llm
app/news
app/portfolio
app/backtest
app/observability

## 7. Risk and execution audit

Existing protections include:
- max loss streak
- max positions
- currency exposure
- correlation limits
- max daily loss
- margin usage
- equity circuit breaker
- news blackout
- spread/session checks
- wrong-side stop protection
- request identity checks
- one-way stop ratchets
- report retries and idempotency logic
- restart recovery
- post-TP re-entry gating
- MFE/giveback controls

The problem is distribution across multiple generations of code.

Target authority hierarchy:
1. Snapshot integrity
2. Strategy candidate
3. Decision policy
4. Risk policy
5. Execution pre-flight
6. Broker execution
7. Position lifecycle
8. Telemetry

No layer may silently redefine another layer's decision.

## 8. SL/TP audit

FlowRSI contains the strongest current pattern:
- code owns entry risk sizing
- AI entry SL/TP are discarded
- final stop boundary changes trigger volume re-sizing
- wrong-side stops are rejected
- broker safety buffers are enforced

Make this project-wide.

Canonical risk contract must carry:
- entry reference
- stop price
- target price
- initial risk
- reward
- initial RR
- authority of stop and target
- protection state
- last protected level

Invariants:
- wrong-side SL is a hard rejection
- wrong-side TP is a hard rejection
- SL only moves toward reduced risk
- TP cannot regress after a protected milestone without explicit structural reset
- final volume is sized from the final stop distance.

## 9. Market structure / OB / FVG

Existing concepts:
- swing highs/lows
- HH/HL/LH/LL
- structural breaks
- Order Blocks
- FVG
- liquidity sweeps
- premium/discount

They are currently spread between cBot code, server helpers and prompt text.

Known limitation:
The signed-volume calculation built from candle close location is an order-flow proxy, not true bid/ask delta. Do not label it as true delta until real tick or order-book data supports it.

Target reusable structure layer:
- swing detector
- BOS / CHOCH
- market structure state
- liquidity levels
- OB detector
- OB freshness / mitigation / quality
- FVG detector
- FVG age / mitigation / quality
- confluence graph.

## 10. News audit

Existing ForexFactory/news integration is useful but must be abstracted.

Required:
- provider interface
- cached event state
- UTC canonical time
- timezone normalization
- currency/symbol mapping
- impact normalization
- stale-provider policy
- provider-failure policy
- auditable news-block reason.

## 11. Backtest audit

Existing:
- cTrader backtest jobs
- Docker execution
- .algo metadata extraction
- SHA-256 metadata cache
- parameter override validation
- report parsing
- partial-close grouping
- backtest dashboard

Upgrade path:
- Rules-Only backtests
- AI-on vs AI-off comparison
- walk-forward
- out-of-sample holdout
- sensitivity analysis
- Monte Carlo
- realistic spread/commission/slippage
- MAE/MFE and expectancy
- regime/session/symbol segmentation
- strategy-version stamping.

Any historical performance claims in README remain self-reported until independently reproduced.

## 12. Observability audit

Existing:
- daily logs
- cBot event telemetry
- tick stream
- dashboard
- portfolio reporting
- watchdog
- account heartbeat

Need one end-to-end decision trace containing:
- request ID
- strategy/version
- deterministic gate sequence
- candidate
- confluence
- risk plan
- AI interaction, if any
- final decision
- execution request/result
- latency by stage
- rejection reason.

## 13. Security audit

Priority:
- no cloud secret embedded in cBots
- local/VPS secret storage
- authenticated cBot-to-server transport
- live/demo separation
- explicit shadow mode
- stale-response rejection
- exact identity checks
- idempotent commands and execution
- audit all dashboard controls
- emergency kill switch.

## 14. Engineering quality

Strengths:
- 72 test modules
- many regression tests around historical production failures
- install/backup tooling
- dashboard and integration coverage

Gaps:
- no visible GitHub Actions workflow
- large monoliths
- no single canonical contract package
- duplicated business logic across C# and Python
- transitional/legacy behavior
- no explicit project-wide release/version discipline.

## 15. Target architecture

cTrader / Broker
 -> Market Data Adapter
 -> Canonical Snapshot
 -> Strategy Engines
 -> Decision Policy
 -> Optional AI Validator / Manager
 -> Risk Plan
 -> Execution Gateway
 -> Position Manager
 -> Telemetry / Audit

AI is a bounded participant. It never becomes the sole authority for capital protection or broker execution.

## 16. Baseline acceptance

No trading behavior is intentionally changed by the documentation phase.

The baseline is captured when:
- fork and upstream SHA are recorded
- audit is committed
- roadmap is committed
- next implementation phase is explicitly defined.

## 17. Next implementation phase

Phase 01 is Authority Map & Full-System Safety Inventory.

Before changing thresholds or claiming strategy improvements, map every path that can enter, exit, modify SL/TP, size volume, block trading, apply news policy, execute dashboard commands or report telemetry.
