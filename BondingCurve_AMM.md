# Bonding Curve + AMM using the KaspaFungibleTokenProtocol

## Abstract

This paper introduces a covenant-based mechanism for launching and distributing tokens built on top of the KaspaFungibleTokenProtocol. The proposed design combines two distinct liquidity phases: an initial **Bonding Curve** distribution phase, followed by a seamless transition to an **Automated Market Maker (AMM)**.

During the first phase, tokens are issued deterministically along a predefined pricing curve, ensuring predictable and transparent price discovery from genesis. Once the bonding curve reaches its completion threshold, liquidity is migrated into an AMM, enabling continuous and decentralized trading.

This hybrid approach aims to provide fair initial distribution, eliminate reliance on off-chain mechanisms, and maintain fully on-chain price integrity enforced through covenant logic.

---

## Mathematical Foundation

The bonding curve defines a deterministic relationship between the token price and the cumulative volume of tokens sold since inception.

In this protocol, we adopt a simple linear pricing function:

`P(V) = a * V + b`


Where:

- `P(V)` denotes the price of token T expressed in Kaspa  
- `a`, `b` are constant parameters defining the linear curve  
- `V` represents the cumulative volume of tokens sold since genesis  

This formulation ensures that the token price increases as supply is distributed, incentivizing early participation while preserving mathematical simplicity for on-chain verification.

---

## Transaction Verification along the Curve

Since the price depends on the number of tokens sold since genesis, the **cumulative cost of tokens** must be verified for each transaction to prevent exploitation. This corresponds to calculating the **area under the curve** between the token balances before and after a transaction.

Let:

- `T_in`, `T_out` be the token balances before and after the transaction  
- `S` be the total supply of tokens  

The cumulative number of tokens sold is:

`V1 = S - T_in`

`V2 = S - T_out`


Let:

- `K_in` be the Kaspa amount provided by the user  
- `K_out` be the Kaspa amount returned to the user  

Then, the total Kaspa exchanged should correspond to the area under the bonding curve between `V1` and `V2`:


`| K_out – K_in | = 1/2(V2^2-V1^2)` 

However, To mitigate the risk of integer overflows, the covennat design should avoid squared computation and other high-growth arithmetic operations. For a linear pricing function, the integral between two points can be simplified into a closed-form expression. The area under the curve is equivalent to the product of the change in volume and the average of the initial and final prices, eliminating the need for higher-order computations. Then we need to verify: 

`| K_out – K_in | = 1/2*(P(V2)-P(V1))*(V2-V1)`

Because exact equality is impractical to enforce on-chain due to  rounding, the covenant must verifies **inequality constraints** based on the transaction direction:

### Token Purchase (Interaction with Bonding Curve) 

As 'K_out >= K_in', we need to verifiy the below inequality:

` K_out – K_in >= 1/2*(P(V2)-P(V1))*(V2-V1)`

So, we are sure to pay more or equal than the cumulated price

### Token Sold (Interaction with Bonding Curve) 

As 'K_out <= K_in', we need to verifiy the below inequality:

` K_in – K_out <= 1/2*(P(V1)-P(V2))*(V1-V2)`

So, we are sure to receive less or equal than the cumulated price
