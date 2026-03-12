# Web Contracts — JSON State Machines Anchored to Bitcoin

## Purpose

Build smart contracts using plain JSON and any programming language.
No VM, no gas fees, no blockchain — just deterministic state machines
with Bitcoin-anchored proof.

## Architecture

A web contract has four layers:

| Layer | File | Mutable? | Role |
|-------|------|----------|------|
| Contract | `contract.json` + source | No | Rules and transition logic |
| State | `state.json` | Yes | Current values, hash-chained |
| Ledger | `ledger.json` | Yes | Balance tracking ([webledgers](https://webledgers.org)) |
| Trail | blocktrail | Append | Bitcoin proof ([blocktrails](https://blocktrails.org)) |

## contract.json

Declares the profile, language, and source file:

```json
{
  "@context": "https://webcontracts.org/v1",
  "type": "WebContract",
  "profile": "amm.v1",
  "id": "pool-tbtc3-tbtc4",
  "language": "javascript",
  "source": "amm.js",
  "created": 1741737600
}
```

## state.json

Pure data — no logic. Updated each transition:

```json
{
  "contract": "pool-tbtc3-tbtc4",
  "seq": 5,
  "prev": "a1b2c3d4e5f6...",
  "updated": 1741737600,
  "pair": ["tbtc3", "tbtc4"],
  "reserves": { "tbtc3": 5000, "tbtc4": 10000 },
  "k": 50000000,
  "fee": 0.003,
  "totalShares": 1000,
  "lpShares": { "did:nostr:abc123...": 1000 }
}
```

- `seq` increments each transition
- `prev` is SHA-256 of the previous JCS-canonicalized state
- This creates a hash chain verifiable against the Bitcoin trail

## Writing Contract Logic

Transition functions take `(state, params, ledger)` and return `{ state, effects }`:

```javascript
// amm.js
export function swap(state, params, ledger) {
  const { sender, sell, amount } = params
  const buy = state.pair.find(u => u !== sell)
  const rIn = state.reserves[sell], rOut = state.reserves[buy]
  const amtAfterFee = amount * (1 - state.fee)
  const out = Math.floor((rOut * amtAfterFee) / (rIn + amtAfterFee))

  return {
    state: {
      ...state,
      reserves: { [sell]: rIn + amount, [buy]: rOut - out },
      k: (rIn + amount) * (rOut - out),
    },
    effects: [
      { op: 'debit', who: sender, currency: sell, amount },
      { op: 'credit', who: sender, currency: buy, amount: out },
    ]
  }
}

export function addLiquidity(state, params, ledger) {
  const { sender } = params
  const [a, b] = state.pair
  const amtA = params[a], amtB = params[b]

  let shares
  if (state.totalShares === 0) {
    shares = Math.floor(Math.sqrt(amtA * amtB))
  } else {
    shares = Math.floor(
      Math.min(amtA / state.reserves[a], amtB / state.reserves[b]) * state.totalShares
    )
  }

  const newReserves = {
    [a]: state.reserves[a] + amtA,
    [b]: state.reserves[b] + amtB,
  }

  return {
    state: {
      ...state,
      reserves: newReserves,
      k: newReserves[a] * newReserves[b],
      totalShares: state.totalShares + shares,
      lpShares: {
        ...state.lpShares,
        [sender]: (state.lpShares[sender] || 0) + shares,
      },
    },
    effects: [
      { op: 'debit', who: sender, currency: a, amount: amtA },
      { op: 'debit', who: sender, currency: b, amount: amtB },
    ]
  }
}

export function removeLiquidity(state, params, ledger) {
  const { sender, shares } = params
  const [a, b] = state.pair
  const ratio = shares / state.totalShares
  const withdrawA = Math.floor(state.reserves[a] * ratio)
  const withdrawB = Math.floor(state.reserves[b] * ratio)

  const newReserves = {
    [a]: state.reserves[a] - withdrawA,
    [b]: state.reserves[b] - withdrawB,
  }

  return {
    state: {
      ...state,
      reserves: newReserves,
      k: newReserves[a] * newReserves[b],
      totalShares: state.totalShares - shares,
      lpShares: {
        ...state.lpShares,
        [sender]: state.lpShares[sender] - shares,
      },
    },
    effects: [
      { op: 'credit', who: sender, currency: a, amount: withdrawA },
      { op: 'credit', who: sender, currency: b, amount: withdrawB },
    ]
  }
}
```

## Ledger Effects

Transition functions return effects — they never modify the ledger directly:

- `{ op: "credit", who, currency, amount }` — add to balance
- `{ op: "debit", who, currency, amount }` — subtract from balance
- `{ op: "transfer", from, to, currency, amount }` — atomic move

The executor applies effects against the [webledger](https://webledgers.org) atomically.

## Submitting Transitions

Submit a transition as a JSON POST:

```json
{
  "transition": "swap",
  "params": { "sell": "tbtc3", "amount": 1000 }
}
```

Authenticate with [NIP-98](https://nip98.com) — a signed Nostr event in the `Authorization` header. The sender's `did:nostr:<pubkey>` is derived from the signature.

## Validation Model

Three roles:

1. **Executor** — loads contract source, runs transition function, updates state + ledger + trail atomically
2. **Client** — submits transitions; can run contract source locally to simulate/preview
3. **Verifier** — downloads contract source, replays state chain from genesis, checks `prev` hashes, verifies blocktrail against Bitcoin

## Contract Languages

The contract interface is always `(state, params, ledger) → { state, effects }`. The language is pluggable:

| Language | Runtime |
|----------|---------|
| JavaScript | Native on Node.js |
| Solidity | Via solc / EVM in WASM |
| Python | Via sandboxed runtime |

## Profiles

| Profile | Description | Transitions |
|---------|-------------|-------------|
| `amm.v1` | Constant product AMM (x * y = k) | swap, add-liquidity, remove-liquidity |
| `escrow.v1` | Two-party escrow with arbiter | fund, release, dispute, resolve, refund |
| `sale.v1` | Fixed-price token sale | buy, close |
| `subscription.v1` | Recurring payment | pay, pause, resume, cancel |

## Transport

Web contracts are transport-agnostic. When served over HTTP:

```
PUT  /contracts/{id}/contract.json  — deploy contract
GET  /contracts/{id}/state.json     — read current state
POST /contracts/{id}/               — submit a transition
GET  /contracts/{id}/trail.json     — read audit trail
```

## Identity

Parties use `did:nostr:<pubkey>`. Auth via [NIP-98](https://nip98.com).
Works for humans (NIP-07 browser extensions) and AI agents (stored keys).

## References

- [Web Contracts spec](https://webcontracts.org) — this spec
- [Web Ledgers](https://webledgers.org) — balance tracking
- [Blocktrails](https://blocktrails.org) — Bitcoin anchoring via BIP-341 key chaining
- [NIP-98](https://nip98.com) — HTTP authentication
- [JavaScript Solid Server](https://github.com/JavaScriptSolidServer/JavaScriptSolidServer) — reference implementation
