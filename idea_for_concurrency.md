# Enabling Concurrent Swaps on a Kaspa DEX Using KCC20 Covenants

## Overview

Traditional DeFi AMMs are generally designed around a single mutable state representing a liquidity pool.

For example, on account-based blockchains, an AMM contract maintains a pool containing two assets:
- A token/a coin
- A token

Every swap modifies this shared state by updating the reserves of both assets.

On Kaspa, using the UTXO model and KCC20 covenants, one current approach is conceptually similar:
- A covenant ID represents the AMM logic.
- A single UTXO owned by this covenant contains both:
  - the KAS liquidity
  - the KCC20 token liquidity

The UTXO itself represents the state of the liquidity pool.

When a swap occurs, this UTXO is consumed and a new UTXO is created with updated reserves.

The limitation of this model is that every swap competes for the same UTXO. This creates a serialization point: only one transaction can consume this specific pool UTXO at a time.

---

## A Different Approach: Multiple AMM Pool UTXOs Sharing the Same Covenant

Instead of creating a single AMM UTXO, the initial AMM covenant creation transaction could generate multiple UTXOs, all with the same covenant ID.

**Example:**

```text
AMM Covenant ID
├── AMM #1
├── AMM #2
.
.
└── AMM #N
```

Each UTXO contains the complete AMM state and follows the exact same covenant rules.
The covenant logic remains unique, but the executable state is distributed across multiple UTXOs.
We also create several Pool UTXOs that all belong to the same covenant ID, which is the covenant ID of the AMM contract.
---

## Achieving Concurrent Swaps

# Achieving Concurrency Through UTXO Allocation

With multiple AMM pool UTXOs sharing the same covenant ID, concurrency can be achieved by allocating different ranges of pool UTXOs to different swap transactions.

Instead of having every user compete for the same pool state, the frontend can distribute available pool UTXOs across users.

**Example of Concurrent Swaps:**

| 🟢 User A Transaction | 🔵 User B Transaction |
| :--- | :--- |
| **Input:** Pool UTXO #3 | **Input:** Pool UTXO #7 |
| **Output:** Updated Pool UTXO #3 | **Output:** Updated Pool UTXO #7 |

Because the two transactions consume different UTXOs, they do not conflict.

The result is natural concurrency derived from Kaspa's UTXO architecture.

---

## Frontend UTXO Selection Strategy

The frontend can implement a simple optimistic selection mechanism:

1. Scan available AMM pool UTXOs.
2. Select one or several available UTXOs depending on the swap size.
3. Build the transaction.
4. If the selected UTXO has already been consumed:
   - retry with another previously scanned pool UTXO.

This allows the frontend to quickly route users toward available liquidity without requiring a global lock.

---

## The Liquidity Is Not Fragmented

A key point is that this design does **not** divide liquidity into isolated pools.

All pool UTXOs share the same:
- covenant ID
- AMM logic
- trading pair
- pricing mechanism

Furthermore, a single transaction can consume multiple pool UTXOs simultaneously.

For example, a large swap could use:

```text
Transaction:

Inputs:
Pool UTXO #1
Pool UTXO #2
Pool UTXO #3

Outputs:
Updated Pool UTXOs
User swap output
```

This means liquidity remains unified at the protocol level.

The multiple UTXOs are only a way to distribute the state and remove unnecessary contention, not a way to create separate liquidity silos.

---

## Handling Contention and Slippage Protection

A possible edge case is that one user transaction consumes all available pool UTXOs.

For example:

**Before:**
```text
Pool UTXO #1
Pool UTXO #2
Pool UTXO #3
...
Pool UTXO #10
```

**Large transaction consumes:**
```text
Pool UTXO #1 → #10
```

In this case, another user may need to wait until new pool UTXOs are created.

However, this behavior can actually protect users.

A user building a swap transaction against an outdated pool state risks receiving an unexpected execution price and suffering excessive slippage.

By requiring transactions to consume currently available pool UTXOs, the system naturally prevents execution against stale liquidity information.

---

## Conclusion

A Kaspa DEX using KCC20 covenants does not need to rely on a single AMM state UTXO.

By creating multiple pool UTXOs sharing the same covenant ID, the AMM can achieve parallel execution while preserving a unified liquidity model.

This approach provides:
- concurrent swaps,
- reduced UTXO contention,
- no liquidity fragmentation,
- compatibility with large swaps using multiple pool UTXOs,
- and a design that fully leverages Kaspa's native UTXO architecture.

Instead of fighting the UTXO model by recreating account-based state management, this design embraces UTXOs as parallelizable units of execution.
