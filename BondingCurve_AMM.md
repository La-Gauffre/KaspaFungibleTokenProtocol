# Bonding Curve + AMM using the KaspaFungibleTokenProtocol

## Abstract

This paper introduces a covenant-based mechanism for launching and distributing tokens built on top of the [KaspaFungibleTokenProtocol](https://github.com/La-Gauffre/KaspaFungibleTokenProtocol/blob/main/KaspaFungibleTokenProtocol.md) with the states in **the same UTXO**. The proposed design combines two distinct liquidity phases: an initial **Bonding Curve** distribution phase, followed by a seamless transition to an **Automated Market Maker (AMM)**.

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

---

## Automated Market Maker (AMM)

Once the price is above the target price of the bounding curve, then, we step up in the AMM mode. we need to check that for each transaction, the constant of the AMM is the same between the input and the output. 

We have to verify: 

`K_out * T_out >= K_in * T_in` 

---

## Switch from the Bonding Curve to the AMM

A binary flag is embedded at the beginning of the covenant script to determine which pricing mechanism is active.

- If the value is `0`, the system operates under the bonding curve model.  
- Once the bonding curve reaches its completion threshold, the flag is permanently switched to `1`, activating the AMM mode.

The covenant enforces that this transition is **irreversible**, ensuring that once the AMM phase is reached, the system cannot revert to the bonding curve.

Additionally, the covenant guarantees that the transition can only occur when the bonding curve conditions are fully satisfied, preventing any premature or invalid state change.

#### Script Structure

The protocol will force to always have the Bonding Curve / AMM UTXO in Input 0 & in Output 0.
The `scriptPubKey` is constructed as follows:

`[ ...KaspFungibleTokenProtocol Logic Opcodes... ] [...Binary flag Logic Opcode + Security...] [...Bonding curve Logic Opcode...] [...AMM Logic Opcode...]`

Where:
    `<LEN_LOGIC>` defined below is the same lenght than the one in the KaspFungibleTokenProtocol.
    `<LEN_KFTP>` defined below is the total lenght of the KaspFungibleTokenProtocol.
    `<LEN_TOTAL>` defined below is the total lenght of the KaspFungibleTokenProtocol.
        

#### Assembly Implementation


```

// -------------------------------------------------------------------------
// 1. KASPAFUNGIBLETOKENPROTOCOL
// -------------------------------------------------------------------------



// -------------------------------------------------------------------------
// 2. BINARY FLAG + SECURITY CHECK 
// -------------------------------------------------------------------------

// Binary Flag

0 or 1 (1 byte)

// Duplication of the Binary flag as we'll need to check the output also

OpDup

// Check Input 0 & Output 0 have the same covenant_ID
0 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpVerify
OpTxInputIndex OpTxInputCovId 0 OpTxOutputCovId OpEqual OpVerify

// Check that logic is the same between Input 0 and output 0
0 <LEN_KFTP+1> <LEN_TOTAL> OpTxInputSpkSubstr  // Extract Input Logic without Token logic and binary flag
0 <LEN_KFTP+1> <LEN_TOTAL> OpTxOutputSpkSubstr // Extract Output Logic without Token logic and binary flag
OpEqual OpVerify

// Check if we calculate via the bonding curve or the AMM
0 OpEqual OpIf

    // -------------------------------------------------------------------------
    // 3. BONDING CURVE LOGIC OPCODE
    // -------------------------------------------------------------------------



OpElse

    // =========================================================================
    // 4. AMM LOGIC OPCODE
    // =========================================================================
    









OpEndif





// --- INPUT 1 ---
OpTxInputCount 1 OpGreaterThan OpIf
    1 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf 
        1 0  <LEN_LOGIC> OpTxInputSpkSubstr OpTxInputIndex 0 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify // Verify Script Integrity 
        1 <LEN_LOGIC> <LEN_LOGIC+8> OpTxInputSpkSubstr  // Extract Amount
        OpBin2Num OpAdd //Add amount on the stack
    OpEndIf
OpEndIf

// --- INPUT 2 ---
OpTxInputCount Op2 OpGreaterThan OpIf
    Op2 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf 
        Op2 0  <LEN_LOGIC> OpTxInputSpkSubstr OpTxInputIndex 0 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify // Verify Script Integrity 
        Op2 <LEN_LOGIC> <LEN_LOGIC+8> OpTxInputSpkSubstr  // Extract Amount
        OpBin2Num OpAdd //Add amount on the stack
    OpEndIf
OpEndIf

// --- INPUT 3 ---
OpTxInputCount Op3 OpGreaterThan OpIf
    Op3 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf 
        Op3 0  <LEN_LOGIC> OpTxInputSpkSubstr OpTxInputIndex 0 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify // Verify Script Integrity 
        Op3 <LEN_LOGIC> <LEN_LOGIC+8> OpTxInputSpkSubstr  // Extract Amount
        OpBin2Num OpAdd //Add amount on the stack
    OpEndIf
OpEndIf

// --- INPUT 4 ---
OpTxInputCount Op4 OpGreaterThan OpIf
    Op4 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf 
        Op4 0  <LEN_LOGIC> OpTxInputSpkSubstr OpTxInputIndex 0 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify // Verify Script Integrity 
        Op4 <LEN_LOGIC> <LEN_LOGIC+8> OpTxInputSpkSubstr  // Extract Amount
        OpBin2Num OpAdd //Add amount on the stack
    OpEndIf
OpEndIf


// =========================================================================
// 3. OUTPUTS LOOP (SUBTRACTION) - INDEX 0 to 4
// =========================================================================

// --- OUTPUT 0 ---
0 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf   
        0 0 <LEN_LOGIC> OpTxOutputSpkSubstr OpTxInputIndex 0 OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify //Verify Script Integrity 
        0 <LEN_LOGIC> <LEN_LOGIC+8> OpTxOutputSpkSubstr  //Extract Amount
        OpBin2Num OpSub //Substract amount on the stack
OpEndIf

// --- OUTPUT 1 ---
OpTxOutputCount 1 OpGreaterThan OpIf
    1 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf   
        1 0 <LEN_LOGIC> OpTxOutputSpkSubstr OpTxInputIndex 0 OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify //Verify Script Integrity 
        1 <LEN_LOGIC> <LEN_LOGIC+8> OpTxOutputSpkSubstr  //Extract Amount
        OpBin2Num OpSub //Substract amount on the stack
    OpEndIf
OpEndIf

// --- OUTPUT 2 ---
OpTxOutputCount Op2 OpGreaterThan OpIf
    Op2 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf   
        Op2 0 <LEN_LOGIC> OpTxOutputSpkSubstr OpTxInputIndex 0 OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify //Verify Script Integrity 
        Op2 <LEN_LOGIC> <LEN_LOGIC+8> OpTxOutputSpkSubstr  //Extract Amount
        OpBin2Num OpSub //Substract amount on the stack
    OpEndIf
OpEndIf

// --- OUTPUT 3 ---
OpTxOutputCount Op3 OpGreaterThan OpIf
    Op3 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf   
        Op3 0 <LEN_LOGIC> OpTxOutputSpkSubstr OpTxInputIndex 0 OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify //Verify Script Integrity 
        Op3 <LEN_LOGIC> <LEN_LOGIC+8> OpTxOutputSpkSubstr  //Extract Amount
        OpBin2Num OpSub //Substract amount on the stack
    OpEndIf
OpEndIf

// --- OUTPUT 4 ---
OpTxOutputCount Op4 OpGreaterThan OpIf
    Op4 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf   
        Op4 0 <LEN_LOGIC> OpTxOutputSpkSubstr OpTxInputIndex 0 OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr OpEqualVerify //Verify Script Integrity 
        Op4 <LEN_LOGIC> <LEN_LOGIC+8> OpTxOutputSpkSubstr  //Extract Amount
        OpBin2Num OpSub //Substract amount on the stack
    OpEndIf
OpEndIf


// =========================================================================
// 4. FINAL VERIFICATION
// =========================================================================

// Stack state: [Accumulator]
// Must be equal to 0 (Total Inputs == Total Outputs)
0
OpEqual

// Amount of tokens
0x08  <Amount>  OpDrop

```

