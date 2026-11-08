# Agent Pool: Preliminary Implementation Plan

Status: Proposed design. This document is an implementation roadmap, not a description of completed features.

Suggested repository location: `docs/AGENT_POOL_IMPLEMENTATION_PLAN.md`.

## Goal and starting point

Build a configurable population of Python trading agents that interacts with the C++ LOB through a shared command gateway. Start with simple, testable strategies and reliable accounting before introducing reinforcement learning.

- **Existing agent code:** `backend/app/agents/base.py` contains a placeholder Gymnasium environment; its observation space, action space, `reset()`, and `step()` are not implemented ([agent scaffold](https://raw.githubusercontent.com/RyanDyson/Agentic-Trading-Sandbox/master/backend/app/agents/base.py)).
- **Existing transport:** `LobEngineClient` sends commands through Redis Streams and synchronously polls for correlated replies ([engine client](https://raw.githubusercontent.com/RyanDyson/Agentic-Trading-Sandbox/master/backend/app/storage/engine_client.py)).
- **Existing engine interface:** The service accepts submit, cancel, modify, and price commands and publishes order-lifecycle and trade events to per-symbol streams ([engine service](https://raw.githubusercontent.com/RyanDyson/Agentic-Trading-Sandbox/master/lob_engine/src/main.cpp)).

For the first version, use one symbol, configurable initial cash and holdings, no short selling, and fixed exchange rules. The exchange planner remains separate from the trading population.

## Design principles

- **Strategies propose actions:** They do not write to Redis, modify the LOB, or update cash and holdings.
- **The engine determines execution:** An attempted order is not a confirmed order, and an accepted order is not necessarily a completed trade.
- **One ledger owns accounting:** All strategies share the same validation, reservation, and settlement rules.
- **Simulation time controls decisions:** Wall-clock scheduling should not accidentally determine which strategy wins a race.
- **Start with one process:** Do not create a container or operating-system process for every agent.
- **Add complexity in stages:** Establish a working rule-based baseline before adding learning or concurrent decision workers.

## Proposed code layout

```text
backend/app/agents/
├── __init__.py
├── types.py                 # Observations, actions, identifiers
├── config.py                # Validated run and strategy configuration
├── base.py                  # Strategy interface
├── pool.py                  # Create, reset, and stop agent instances
├── runner.py                # Coordinates a simulation run
├── scheduler.py             # Simulated clock and decision schedule
├── gateway.py               # Engine command adapter and correlation
├── events.py                # Journal reader and event normalisation
├── ledger.py                # Cash, inventory, reservations, open orders
├── observations.py          # Builds consistent strategy observations
├── strategies/
│   ├── market_maker.py
│   ├── momentum.py
│   ├── spoofing.py
│   └── rl_trader.py
└── env.py                   # Gymnasium wrapper, introduced later

backend/tests/agents/
├── test_strategies.py
├── test_ledger.py
├── test_scheduler.py
├── test_gateway.py
└── test_integration.py
```

Pooling means managing independent strategy instances with separate state and settings. It does not mean treating traders as interchangeable jobs in a generic worker pool.

## Strategy interface

Use immutable observations and typed actions. The following is a minimal interface sketch, not the complete production schema:

```python
from dataclasses import dataclass
from typing import Literal, Protocol


@dataclass(frozen=True)
class Observation:
    time_ns: int
    symbol: str
    best_bid_ticks: int | None
    best_ask_ticks: int | None
    position: int
    available_cash_units: int
    available_inventory: int


@dataclass(frozen=True)
class Submit:
    side: Literal["buy", "sell"]
    quantity: int
    order_type: Literal["market", "limit", "market-limit"]
    limit_ticks: int | None = None


@dataclass(frozen=True)
class Cancel:
    order_id: int


Action = Submit | Cancel


class Strategy(Protocol):
    def reset(self, seed: int) -> None: ...

    def decide(self, observation: Observation) -> list[Action]: ...
```

An empty action list means “wait.” Add modification and stop-order actions when a strategy needs them, rather than complicating the first implementation.

Extend observations with immutable views of the agent's confirmed and pending orders, price history, market depth, and current fee policy. Keep private strategy state, such as a momentum window or spoofing phase, inside the strategy instance. Give each agent its own seeded random-number generator.

Use integer price ticks and integer cash units internally, with an explicit conversion between them. The gateway must translate these values to the engine protocol at one boundary; do not assume a cash unit equals one price tick. Specify rounding rules before introducing fees.

## Agent behaviour

### Market-making agent

Begin with configurable quote distance, order size, maximum inventory, and decision interval. Quote on both sides of a valid market reference, reduce buying when inventory is high, and reduce selling when inventory is low.

Track existing quotes instead of placing two additional orders at every decision. Keep unchanged quotes in place; cancel or modify only when required. If the book has no usable reference price, either wait or use an explicitly configured seeding price. Keep book seeding separate from the strategy's normal operation.

Later, add configurable quote widening and liquidity withdrawal under stress. Label these as stylised behaviours, not a full reproduction of commercial HFT.

### Momentum agent

Maintain a rolling price window. Propose a buy or sell when the change over that window exceeds a configured threshold; otherwise wait.

Configure window length, threshold, order size, cooldown, and position limit. Wait until sufficient history exists. Test increasing, decreasing, flat, and missing-price sequences.

### Spoofing-like agent

Implement a state machine rather than unrelated timers:

```text
IDLE → SUBMIT_PENDING → ORDER_LIVE → CANCEL_PENDING → IDLE
```

Track the engine-assigned order ID, remaining quantity, cancellation deadline, and any fills. A fill can occur while cancellation is pending, so execution must still update the portfolio. The agent must carry the same financial exposure as any other trader; do not grant it cost-free or non-executable orders.

Treat quote-stuffing-like traffic as a separate configurable workload. Keep message rate, cancellation delay, and host load measurable so it does not become indistinguishable from an overloaded simulation.

### Reinforcement-learning trader

Add this only after the rule-based loop passes integration tests. Convert the same observations into model inputs and translate model outputs into the same action types; do not give the learned agent a separate execution path.

Define each environment step as a fixed simulation interval or documented event boundary. Other agents must act during that interval before the next observation and reward are returned. A proposed reward is the change in marked-to-market equity after recorded trading costs, with an optional inventory-risk penalty; avoid subtracting fees twice if they are already reflected in cash.

## Runner, scheduling, and hosting

Start with a deterministic, centrally scheduled runner:

```text
Select next scheduled decision
    ↓
Advance simulated time and apply preceding market events
    ↓
Build the agent's observation
    ↓
Ask the strategy for actions
    ↓
Validate actions and reserve resources
    ↓
Submit commands through the gateway
    ↓
Reconcile confirmed events with orders and portfolios
    ↓
Schedule the agent's next decision
```

Use a priority queue ordered by simulated timestamp and a recorded tie-break value. For the first version, process decisions sequentially and define that later decisions see the confirmed results of earlier ones. Use a seeded ordering for simultaneous decisions to avoid always favouring the same agent ID.

The scheduler must also process price-feed events and delayed actions under the same clock. Historical or synthetic input is preferable for repeatable experiments; live data should not be described as inherently repeatable.

Use `asyncio` for waiting on transport and event streams when needed. CPU-heavy learning work should run outside the shared event-loop thread because blocking work delays other tasks ([Python asyncio guidance](https://docs.python.org/3/library/asyncio-dev.html)). Adapt the existing synchronous engine client through a worker thread initially, or add an asynchronous equivalent; do not call its blocking polling loop directly from an asynchronous agent task ([current engine client](https://raw.githubusercontent.com/RyanDyson/Agentic-Trading-Sandbox/master/backend/app/storage/engine_client.py)).

Transport concurrency does not replace scheduling. Introduce parallel strategy evaluation only after defining whether agents share a snapshot and how their actions are ordered. Keep training workers separate from the runner, and use a fixed model checkpoint during baseline evaluation runs.

## Gateway and event handling

Each submitted action should carry its run ID, agent ID, symbol, a stable client action reference, and a correlation ID. Store the mapping to the engine-assigned order ID when acceptance is confirmed.

The engine's command types are `market`, `limit`, `market-limit`, `stop`, and `stop-limit`; journaled trades include buyer, seller, quantity, price, and affected order states ([engine command and event contract](https://raw.githubusercontent.com/RyanDyson/Agentic-Trading-Sandbox/master/lob_engine/src/main.cpp)). Maintain an agent-side event projection that consumes the symbol journal without competing with the persistence pipeline for messages intended for another consumer.

Use the journal as the authoritative input for accepted-order and execution accounting, including later fills of resting orders. Use replies for command resolution and rejection handling; current rejections are reply-only rather than journaled ([engine event handling](https://raw.githubusercontent.com/RyanDyson/Agentic-Trading-Sandbox/master/lob_engine/src/main.cpp)). If a reply also contains a trade, do not settle it again through a second accounting path.

Deduplicate event application using a run-scoped stream identity, such as `(run_id, symbol, stream_entry_id)`. Persist consumer progress consistently with ledger state before claiming resumable accounting.

A reply timeout means the result is unknown, not that the order failed. Retain reservations, reconcile the command, and pause affected trading if its outcome cannot be established. Do not blindly retry using a new correlation ID.

Restart recovery is a separate prerequisite for resumable runs: the current engine holds its books, order records, and correlation-ID deduplication set in memory, so command reclaiming alone does not guarantee safe execution after a restart ([current engine recovery implementation](https://raw.githubusercontent.com/RyanDyson/Agentic-Trading-Sandbox/master/lob_engine/src/main.cpp)). Initially, abort an interrupted experiment and start a new run rather than claim transparent recovery.

## Ledger and risk controls

Keep confirmed cash, confirmed holdings, reserved cash, reserved inventory, pending actions, and live orders separate. The ledger should be updated by one coordinator to avoid independent tasks applying conflicting changes.

- **Buy submission:** Reserve the maximum permitted spend, including the configured fee allowance.
- **Sell submission:** Reserve the quantity offered for sale when short selling is disabled.
- **Partial execution:** Settle the executed quantity and retain only the reservation required for the remaining order.
- **Confirmed cancellation or rejection:** Release the appropriate unused reservation.
- **Pending cancellation:** Keep the reservation until the engine confirms the outcome.
- **Modification:** Reserve any additional exposure before sending an increase; retain old protection until confirmation.
- **Market order:** Require an explicit notional or price-protection rule. For the first version, prefer a bounded `market-limit` order where a strict cash limit must be enforced.

Validate batches of actions sequentially against updated reservations. Otherwise, multiple actions from one decision could each pass a balance check while collectively overspending.

Put these checks on every agent-submission path. If manual API submissions remain enabled, route them through the same controls or exclude them from controlled experiments. Agent-service checks alone should not be presented as exchange-wide risk enforcement.

Introduce fee and rebate settlement through a shared policy interface when those engine/accounting extensions exist. Record the policy version applied to each charge, and keep cancellation penalties separate from trade settlement.

## Implementation phases and acceptance checks

- [ ] **Foundation:** Add types, configuration, strategy interface, and a fake gateway. Strategies can be tested without Redis.
- [ ] **First end-to-end run:** Seed one symbol and run a market maker against a simple scripted counterparty. Confirm acceptance, partial execution, and cancellation reach the ledger correctly.
- [ ] **Accounting:** Implement reservations, order-state reconciliation, event deduplication, and timeout handling. Assert no overspending or unauthorised short selling.
- [ ] **Pool and scheduler:** Create agents from configuration, maintain simulated time, and record seeds and decision order. Repeated runs with the same inputs produce equivalent ordered market outcomes.
- [ ] **Heterogeneous strategies:** Add momentum and spoofing-like behaviour, including execution-before-cancellation tests.
- [ ] **Scale and hosting:** Measure command throughput, queue lag, event-processing lag, and decision latency before adding workers.
- [ ] **RL environment:** Add `reset()` and `step()` around defined simulation boundaries, then validate observation/action spaces and a random-policy rollout before training.
- [ ] **Adaptive policy:** Integrate confirmed fee-policy updates and compare against fixed-policy baselines.

For determinism tests, compare normalised action and execution sequences, not wall-clock timestamps or randomly generated transport identifiers. For accounting tests, verify cash and inventory conservation across both trade participants, allowing for explicitly recorded fees, rebates, and initial endowments.

The first useful milestone is a small, repeatable market with correct accounting, not the maximum possible number of concurrent agents.
