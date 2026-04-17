# Escrow using the KaspaFungibleTokenProtocol

## Abstract

This paper introduces a covenant-based mechanism for selling a token built on top of the [KaspaFungibleTokenProtocol](https://github.com/La-Gauffre/KaspaFungibleTokenProtocol/blob/main/KaspaFungibleTokenProtocol.md), with all states stored within **the same UTXO**. The proposed design combines two distinct liquidity phases: an initial **Bonding Curve** distribution phase, followed by a seamless transition to an **Automated Market Maker (AMM)**.


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

`| K_out – K_in | = 1/2(V2^2 - V1^2)`

However, to mitigate the risk of integer overflows, the covenant design should avoid squared computations and other high-growth arithmetic operations. For a linear pricing function, the integral between two points can be simplified into a closed-form expression. The area under the curve is equivalent to the product of the change in volume and the average of the initial and final prices, eliminating the need for higher-order computations.

Thus, we verify:

`| K_out – K_in | = 1/2 * (P(V2) - P(V1)) * (V2 - V1)`

Because exact equality is impractical to enforce on-chain due to rounding, the covenant must verify **inequality constraints** based on the transaction direction:

### Token Purchase (Interaction with Bonding Curve)

As `K_out >= K_in`, we need to verify the following inequality:

`K_out – K_in >= 1/2 * (P(V2) - P(V1)) * (V2 - V1)`

This ensures that the user pays at least the cumulative price.

To simplify implementation in the script:

`P(V2) - P(V1) = a * V2 - a * V1 = a * (S - T_out) - a * (S - T_in)`

### Token Sale (Interaction with Bonding Curve)

As `K_out <= K_in`, we need to verify the following inequality:

`K_in – K_out <= 1/2 * (P(V1) - P(V2)) * (V1 - V2)`

This ensures that the user receives at most the cumulative price.

To simplify implementation in the script:

`P(V1) - P(V2) = a * V1 - a * V2 = a * (S - T_in) - a * (S - T_out)`

---

## Automated Market Maker (AMM)

Once the price exceeds the target defined by the bonding curve, the system transitions into AMM mode. For each transaction, we must verify that the AMM invariant remains consistent between input and output states.

We enforce:

`K_out * T_out >= K_in * T_in`

---

## Switch from the Bonding Curve to the AMM

A binary flag is embedded at the beginning of the covenant script to determine which pricing mechanism is active.

- If the value is `0`, the system operates under the bonding curve model.  
- Once the bonding curve reaches its completion threshold, the flag is permanently switched to `1`, activating AMM mode.

The covenant enforces that this transition is **irreversible**, ensuring that once the AMM phase is reached, the system cannot revert to the bonding curve.

Additionally, the covenant guarantees that the transition can only occur when the bonding curve conditions are fully satisfied, preventing any premature or invalid state change.

The binary flag is switched once the sold supply exceeds a constant `S1` and remains below a constant `S2`. Both constants are 64-byte values.

#### Script Structure

The protocol enforces that the Bonding Curve / AMM UTXO is always located at Input 0 and Output 0.

The `scriptPubKey` is constructed as follows:

`[ ...KaspaFungibleTokenProtocol Logic Opcodes... ] [...Binary Flag Logic Opcode + Security...] [...Bonding Curve Logic Opcode...] [...AMM Logic Opcode...]`

Where:

- `<LEN_LOGIC>` is defined below and has the same length as in the KaspaFungibleTokenProtocol  
- `<LEN_KFTP>` is defined below and represents the total length of the KaspaFungibleTokenProtocol  
- `<LEN_TOTAL>` is defined below and represents the total length of the full script  
- `a` and `S` are the bonding curve constants and the total token supply (64-byte numbers)  
- `S1` and `S2` are constants defining the boundary between the bonding curve and the AMM phase  

#### Assembly Implementation


```

// -------------------------------------------------------------------------
// 1. KASPA FUNGIBLE TOKEN PROTOCOL
// -------------------------------------------------------------------------

//Just copy & paste the KASPA FUNGIBLE TOKEN PROTOCOL logic here

// -------------------------------------------------------------------------
// 2. BINARY FLAG + SECURITY CHECK 
// -------------------------------------------------------------------------

// Binary Flag

0 or 1 (1 byte)


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

    // Check the type of interaction with the bounding curve (Token purchase or sold) 
    0 OpTxOutputAmount 0 OpTxInputAmount OpGreaterThanOrEqual OpIf
        //we have to verify K_out – K_in >= 1/2*(P(V2)-P(V1))*(V2-V1)
        0 OpTxOutputAmount                                         // Extract Number of Kaspa in Output
        0 OpTxInputAmount                                          // Extract Number of Kaspa in Intput
        OpSub Op2 OpMul                                            //Substract amount K_out – K_in, multiply it by 2 and push the number on the stack

        //reminder `P(V2)-P(V1)` = a * V2 - a * V1 = a * (S - T_out) - a * (S - T_in)`

        <A> <S>                                                    //Push A & S on the stack
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxOutputSpkSubstr OpBin2Num // Extract Number of Token in Output
        OpSub OpMul                                                //Push a * (S - T_out)  on the stack 
        <A> <S>                                                    //Push A & S on the stack
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxInputSpkSubstr OpBin2Num  // Extract Number of Token in Intput
        OpSub OpMul                                                //Push a * (S - T_in)  on the stack
        OpSub

        //Next we push (V2-V1) on the stack
        <S>
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxOutputSpkSubstr OpBin2Num
        OpSub
        <S>
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxInputSpkSubstr OpBin2Num
        OpSub

        //Next we multiply (P(V2)-P(V1))*(V2-V1) and compare it to (K_out – K_in)*2
        OpMul
        OpGreatherThanOrEqual OpVerify

    OpElse
        //we have to verify K_in – K_out <= 1/2*(P(V1)-P(V2))*(V1-V2)
        0 OpTxInputAmount                                          // Extract Number of Kaspa in Intput
        0 OpTxOutputAmount                                         // Extract Number of Kaspa in Output
        OpSub Op2 OpMul                                            //Substract amount K_out – K_in, multiply it by 2 and push the number on the stack

        //reminder `P(V1)-P(V2) = a * V1 - a * V2 = a * (S - T_in) - a * (S - T_out)`

        <A> <S>                                                    //Push A & S on the stack
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxInputSpkSubstr OpBin2Num  // Extract Number of Token in Intput
        OpSub OpMul                                                //Push a * (S - T_in)  on the stack
        <A> <S>                                                    //Push A & S on the stack
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxOutputSpkSubstr OpBin2Num // Extract Number of Token in Output
        OpSub OpMul                                                //Push a * (S - T_out)  on the stack 
                                                                   
        OpSub

        //Next we push (V1-V2) on the stack

        <S>
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxInputSpkSubstr OpBin2Num
        OpSub

        <S>
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxOutputSpkSubstr OpBin2Num
        OpSub
        

        //Next we multiply (P(V1)-P(V2))*(V1-V2) and compare it to (K_in – K_out)*2
        OpMul
        OpLessThanOrEqual OpVerify

    OpEndif

    // Next we check the binary flag in Output
    <S1> <S> 
    0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxOutputSpkSubstr OpBin2Num                           // Extract Number of Token in Output
    OpSub                                                                                // Calculate total supply sold
    OpGreaterThan OpIf                                                                   // Compare total supply sold vs S1
        0 <LEN_KFTP> <LEN_KFTP+1> OpTxOutputSpkSubstr OpBin2Num 0 OpEqual OpVerify       // Check Output Binary Flag is equal to 0
    OpElse
        <S2> <S>
        0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxOutputSpkSubstr OpBin2Num                       // Extract Number of Token in Output
        OpGreaterThan OpVerify
        0 <LEN_KFTP> <LEN_KFTP+1> OpTxOutputSpkSubstr OpBin2Num 1 OpEqual OpVerify       // Check Output Binary Flag is equal to 1
OpElse

    // =========================================================================
    // 4. AMM LOGIC OPCODE
    // =========================================================================

    //we have to verify K_out * T_out >= K_in * T_in

    0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxOutputSpkSubstr OpBin2Num // Extract Number of Token in Output
    0 OpTxOutputAmount                                         // Extract Number of Kaspa in Output
    OpMul
    0 <LEN_LOGIC> <LEN_LOGIC+64> OpTxInputSpkSubstr OpBin2Num  // Extract Number of Token in Intput
    0 OpTxInputAmount                                          // Extract Number of Kaspa in Intput
    OpMul
    OpGreatherThanOrEqual OpVerify

    //Check that Binary Flag in Output is equal to 1
    0 <LEN_KFTP> <LEN_KFTP+1> OpTxOutputSpkSubstr OpBin2Num 1 OpEqual OpVerify

OpEndif
