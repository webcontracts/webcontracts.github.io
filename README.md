# Web Contracts

**JSON state machines anchored to Bitcoin.**

A web contract is three files working together:

| File | Spec | Role |
|------|------|------|
| `state.json` | this spec | Contract rules, current state, valid transitions |
| `ledger.json` | [webledgers.org](https://webledgers.org) | Who owns what — balances, multi-currency |
| `trail` | [blocktrails.org](https://blocktrails.org) | Proof — state transitions anchored to Bitcoin via BIP-341 key chaining |

No VM. No bytecode. No blockchain. Just JSON + HTTP + Bitcoin.

---

## state.json

A contract is a state machine. It defines what states exist, what transitions are valid, and what effects each transition has on the ledger.

```json
{
  "@context": "https://webcontracts.org/v1",
  "type": "WebContract",
  "profile": "amm.v1",
  "id": "pool-tbtc3-tbtc4",
  "created": 1741737600,
  "updated": 1741737600,
  "seq": 0,
  "prev": null,

  "state": {
    "pair": ["tbtc3", "tbtc4"],
    "reserves": { "tbtc3": 0, "tbtc4": 0 },
    "k": 0,
    "fee": 0.003,
    "totalShares": 0,
    "lpShares": {}
  },

  "transitions": {
    "swap": {
      "requires": ["sell", "amount"],
      "conditions": ["reserves[sell] > 0", "k > 0"],
      "effects": ["debit(sender, sell, amount)", "credit(sender, buy, amountOut)"]
    },
    "add-liquidity": {
      "requires": ["amountA", "amountB"],
      "conditions": ["balance(sender, pairA) >= amountA", "balance(sender, pairB) >= amountB"],
      "effects": ["debit(sender, pairA, amountA)", "debit(sender, pairB, amountB)", "mintShares(sender)"]
    },
    "remove-liquidity": {
      "requires": ["shares"],
      "conditions": ["lpShares[sender] >= shares"],
      "effects": ["burnShares(sender, shares)", "credit(sender, pairA, withdrawA)", "credit(sender, pairB, withdrawB)"]
    }
  }
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `@context` | URI | Always `https://webcontracts.org/v1` |
| `type` | string | Always `WebContract` |
| `profile` | string | Contract type identifier (e.g. `amm.v1`, `escrow.v1`, `sale.v1`) |
| `id` | string | Unique contract identifier |
| `created` | number | Unix timestamp |
| `updated` | number | Unix timestamp of last transition |
| `seq` | number | Transition sequence number, starts at 0 |
| `prev` | string\|null | SHA-256 of previous JCS-canonicalized state (null for genesis) |
| `state` | object | Current contract state (profile-specific) |
| `transitions` | object | Valid transitions with requirements, conditions, and effects |

### Sequencing

Each transition increments `seq` and sets `prev` to the SHA-256 hash of the previous state (JCS-canonicalized). This creates a hash chain that can be verified independently and anchored to Bitcoin via [blocktrails](https://blocktrails.org).

```
genesis (seq:0) --> transition --> state (seq:1) --> transition --> state (seq:2) --> ...
   |                                 |                                 |
   +-- blocktrail -------------------+-- blocktrail ------------------+-- blocktrail
```

---

## Profiles

Profiles define the shape of `state` and the semantics of `transitions`. A contract declares its profile and implementations validate against it.

### amm.v1

Automated market maker with constant product formula (`x * y = k`).

- **State**: pair, reserves, k, fee, totalShares, lpShares
- **Transitions**: swap, add-liquidity, remove-liquidity

### escrow.v1

Two-party escrow with optional arbiter.

- **State**: buyer, seller, arbiter, amount, currency, status (pending|funded|released|disputed|refunded)
- **Transitions**: fund, release, dispute, resolve, refund

### sale.v1

Fixed-price token sale.

- **State**: ticker, rate, supply, sold
- **Transitions**: buy, close

### subscription.v1

Recurring payment agreement.

- **State**: subscriber, provider, amount, currency, interval, nextDue, status (active|paused|cancelled)
- **Transitions**: pay, pause, resume, cancel

---

## Ledger integration

Each transition that moves value updates the [webledger](https://webledgers.org). The contract references the ledger but does not contain balances -- the ledger is the source of truth for ownership.

```
state.json          ledger.json              trail
+----------+       +--------------+       +--------------+
| rules    |------>| who owns what|------>| Bitcoin proof |
| state    |       | balances     |       | key chaining  |
| seq/prev |       | multi-currency|      | BIP-341       |
+----------+       +--------------+       +--------------+
     ^                                          |
     +---------- prev = sha256(state) ---------+
```

Transitions specify effects using ledger operations:

- `credit(who, currency, amount)` -- add to balance
- `debit(who, currency, amount)` -- subtract from balance
- `transfer(from, to, currency, amount)` -- atomic move

---

## Identity

Parties are identified by `did:nostr:<pubkey>`. Authentication uses [NIP-98](https://github.com/nostr-protocol/nips/blob/master/98.md) HTTP auth -- a signed Nostr event in the `Authorization` header.

No accounts. No passwords. No OAuth. Just a keypair.

This makes contracts accessible to both humans (via NIP-07 browser extensions) and AI agents (via stored keys).

---

## Transport

Contracts live on [Solid pods](https://solidproject.org/). All operations are HTTP:

```
GET  /contracts/{id}/state.json     -- read current state
POST /contracts/{id}/               -- submit a transition
GET  /contracts/{id}/ledger.json    -- read balances
GET  /contracts/{id}/trail.json     -- read audit trail
```

Transitions are submitted as JSON:

```json
{
  "transition": "swap",
  "params": { "sell": "tbtc3", "amount": 1000 }
}
```

The pod validates the transition against the contract rules, updates state, updates the ledger, and anchors to Bitcoin.

---

## Agent-friendly

Web contracts are designed for autonomous agents:

1. Agent gets a nostr keypair (identity)
2. Agent reads `state.json` to understand the rules
3. Agent submits transitions via HTTP + NIP-98
4. Agent verifies trail against Bitcoin

No browser required. No human approval. Any HTTP client with a signing key can participate.

---

## Implementations

- [JavaScript Solid Server](https://github.com/JavaScriptSolidServer/JavaScriptSolidServer) -- JSS implements the payment and AMM profiles
- [Web Ledgers](https://webledgers.org) -- balance tracking spec
- [Blocktrails](https://blocktrails.org) -- Bitcoin anchoring spec

---

## License

MIT
