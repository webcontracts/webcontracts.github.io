# Web Contracts

**State machines anchored to Bitcoin.**

A web contract separates **logic** from **data**. The contract defines rules in any language. The state, ledger, and trail are always JSON — universal, language-agnostic, verifiable.

| Layer | File | Mutable? | Role |
|-------|------|----------|------|
| **Contract** | `contract.json` + source | No | Rules & transition logic (immutable once deployed) |
| **State** | `state.json` | Yes | Current values, updated each transition |
| **Ledger** | `ledger.json` | Yes | Balance tracking ([webledgers](https://webledgers.org)) |
| **Trail** | blocktrail | Append | Bitcoin-anchored audit log ([blocktrails](https://blocktrails.org)) |

---

## contract.json

The contract declares its profile, language, and source. The logic lives in a separate file.

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

The source file exports transition functions. Each takes `(state, params, ledger)` and returns `{ state, effects }`:

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
```

### Contract languages

The contract logic is pluggable. The pod needs an executor for the language.

| Language | Runtime |
|----------|---------|
| JavaScript | Native on Node.js pods |
| Solidity | Via solc / EVM in WASM |
| Python | Via sandboxed runtime |

The interface is always the same: `(state, params, ledger) → { state, effects }`.

---

## state.json

Pure data — no logic. Current values plus a hash chain for verification.

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

Each transition increments `seq` and sets `prev` to SHA-256 of the previous state (JCS-canonicalized). This hash chain is anchored to Bitcoin via [blocktrails](https://blocktrails.org).

---

## Ledger effects

Transition functions return an `effects` array. The executor applies these against the [webledger](https://webledgers.org) atomically.

- `{ op: "credit", who, currency, amount }` — add to balance
- `{ op: "debit", who, currency, amount }` — subtract from balance
- `{ op: "transfer", from, to, currency, amount }` — atomic move

The contract never touches the ledger directly. It declares intent; the executor enforces it.

---

## Validation

Web contracts use a **trust but verify** model:

1. **Executor** — runs the contract source, validates transitions, updates state + ledger + trail atomically. Any server or local process that can run the contract language.
2. **Client** — submits transitions. Can optionally run the contract source locally to preview results before submitting.
3. **Verifier** — anyone. Download the contract source, replay the state chain from genesis, check each `prev` hash, verify the blocktrail against Bitcoin. If the executor cheated, the chain breaks.

No consensus mechanism. No global blockchain. The executor is efficient (single process), and the hash chain + Bitcoin trail provides cryptographic tamper evidence.

---

## Profiles

| Profile | Description | Transitions |
|---------|-------------|-------------|
| `amm.v1` | Automated market maker (x * y = k) | swap, add-liquidity, remove-liquidity |
| `escrow.v1` | Two-party escrow with arbiter | fund, release, dispute, resolve, refund |
| `sale.v1` | Fixed-price token sale | buy, close |
| `subscription.v1` | Recurring payment agreement | pay, pause, resume, cancel |

---

## Identity

Parties are identified by `did:nostr:<pubkey>`. Authentication uses [NIP-98](https://nip98.com) HTTP auth.

Works for humans (NIP-07 browser extensions) and AI agents (stored keys) alike.

---

## Transport

Web contracts are transport-agnostic. The four files can live anywhere:

| Host | Example |
|------|---------|
| [Solid pod](https://solidproject.org/) | Full integration — auth, ledger, and trail built in |
| Web server | Any HTTP server with a contract executor |
| Git repo | State as commits, trail as tags |
| Local directory | Agent runs executor locally, syncs state |

When served over HTTP:

```
PUT  /contracts/{id}/contract.json  — deploy contract
GET  /contracts/{id}/state.json     — read current state
POST /contracts/{id}/               — submit a transition
GET  /contracts/{id}/trail.json     — read audit trail
```

---

## Implementations

- [JavaScript Solid Server](https://github.com/JavaScriptSolidServer/JavaScriptSolidServer) — implements payment and AMM profiles
- [Web Ledgers](https://webledgers.org) — balance tracking spec
- [Blocktrails](https://blocktrails.org) — Bitcoin anchoring spec
- [NIP-98](https://nip98.com) — HTTP authentication

## License

MIT
