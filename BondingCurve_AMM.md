# Bonding Curve + AMM using the KaspaFungibleTokenProtocol

## Abstract

This paper introduces a covenant-based mechanism for launching and distributing tokens built on top of the KaspaFungibleTokenProtocol. The proposed design combines two distinct liquidity phases: an initial **Bonding Curve** distribution phase, followed by a seamless transition to an **Automated Market Maker (AMM)**.

During the first phase, tokens are issued deterministically along a predefined pricing curve, ensuring predictable and transparent price discovery from genesis. Once the bonding curve reaches its completion threshold, liquidity is migrated into an AMM, enabling continuous and decentralized trading.

This hybrid approach aims to provide fair initial distribution, eliminate reliance on off-chain mechanisms, and maintain fully on-chain price integrity enforced through covenant logic.

---

## Mathematical Foundation

The bonding curve defines a deterministic relationship between the token price and the cumulative volume of tokens sold since inception.

In this protocol, we adopt a simple quadratic pricing function:

P(T) = a + b * V(T)^2

Where:

- P(T) denotes the price of token T expressed in Kaspa  
- a, b are constant parameters defining the curve shape  
- V(T) represents the cumulative volume of tokens sold since genesis  

This formulation ensures that the token price increases non-linearly as supply is distributed, incentivizing early participation while preserving mathematical simplicity for on-chain verification.

---

## Covenant Validation Logic

Within the covenant framework, it is not practical to verify the absolute position on the bonding curve due to computational and precision constraints inherent to on-chain execution. Instead, the protocol validates **value differentials** between transaction inputs and outputs.

Let:

- K_in be the Kaspa amount provided as input  
- K_out be the Kaspa amount returned as output  
- T_in, T_out be the token balances before and after the transaction  

The ideal equality condition would be:

K_out - K_in = P(T_out) - P(T_in)

However, strict equality is impractical to enforce due to integer arithmetic and rounding limitations in covenant execution. Therefore, the protocol enforces **inequality constraints**, depending on the transaction direction:

### Token Purchase (Interaction with Bonding Curve)

When a user buys tokens from the bonding curve:

K_out - K_in >= P(T_out) - P(T_in)

This ensures that the user provides sufficient Kaspa relative to the increase in token valuation.

### Token Sale (Interaction with Bonding Curve)

When a user sells tokens back to the bonding curve:

K_out - K_in <= P(T_out) - P(T_in)

This guarantees that the Kaspa returned does not exceed the theoretical value defined by the curve.

---

## Design Rationale

By enforcing inequalities rather than strict equalities, the covenant remains:

- **Robust to rounding errors** inherent in integer-based computation  
- **Efficient to verify on-chain**, minimizing script complexity  
- **Secure**, preventing value extraction through precision exploits  

This design ensures that all interactions remain bounded by the intended pricing function, while preserving the feasibility of implementation within the Kaspa covenant model.
