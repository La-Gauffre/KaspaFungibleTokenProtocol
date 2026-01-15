# KaspaFungibleTokenProtocol
## Abstract

This paper presents a preliminary framework for implementing covenants leveraging the opcodes introduced in KIP-10 and KIP-17. The primary objective is to facilitate the issuance of fungible tokens and the establishment of escrow mechanisms, thereby enabling secure, atomic asset transfers directly on the Kaspa Layer 1 (L1).

We begin by defining the parameters of the proposed native tokens, which can be configured for either a fixed or dynamic total supply. While the entirety of the initial circulating supply is minted during the genesis transaction, the protocol allows for future issuance through a dual-output genesis structure. Alongside the primary token output, a secondary UTXO is generated populated with the unique **Covenant ID** but devoid of the restrictive covenant logic. This unconstrained UTXO functions as a "minting baton," granting the owner the discretion to introduce new tokens into the ecosystem or simply maintain the UTXO to preserve future minting rights.

Conversely, the architecture supports voluntary deflation. To reduce the supply, assets can simply be transferred to the established burn address (`kaspa:qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqkx9awp4e`), effectively removing them from circulation. Furthermore, the protocol architecture deliberately stores all state and logic within the scriptPubKey rather than utilizing the transaction payload.

## Motivation

Current methods for tokenization often rely on Layer 2 indexers or heavy state machines. This proposal leverages the UTXO model and new introspection capabilities to create "Colored UTXO" that are enforced mathematically by the Kaspa network consensus itself.

The introduction of this framework will:
1.  Enable atomic swaps and decentralized exchanges purely on L1.
2.  Eliminate the need for centralized indexers to validate token legitimacy.
3.  Provide a standard for "Stateless Covenants" using the `CovenantID` mechanism.

## Specification

### 1. Protocol Logic & State
The state of the token (the amount) is stored directly in the `scriptPubKey` following the logic block. The validation logic ensures that the sum of inputs matches the sum of outputs for any transaction involving the specific Covenant ID.

### 2. The Covenant Script
Below is the optimized opcode implementation for the token standard. This script enforces the conservation of mass for the asset.

#### Script Structure
The `scriptPubKey` is constructed as follows:
`[ ...Logic Opcodes... ] [ OpPushData1 ] [ 0x08 ] [ 8-byte Amount ] [ OpDrop ]`

Where `<LEN_LOGIC>` defined below includes the `OpPushData` instruction bytes.

#### Assembly Implementation

```assembly
// =========================================================================
// STATELESS COVENANT (Integrated PushData)
// =========================================================================

// Note: <LEN_LOGIC> now includes the OpPushData opcode at the very end.
// Structure: [ ...Logic... OpPushData1 0x08 ] [ Amount ] [ OpDrop ]

// -------------------------------------------------------------------------
// 1. SECURITY & INITIALIZATION
// -------------------------------------------------------------------------

// Check Input Count < 6
OpTxInputCount
Op6
OpLessThan
OpVerify

// Check Output Count < 6
OpTxOutputCount
Op6
OpLessThan
OpVerify

// Initialize accumulator to 0 on the stack
// Stack: [0]
OpFalse


// =========================================================================
// 2. INPUTS LOOP (ADDITION) - INDEX 0 to 4
// =========================================================================

// --- INPUT 0 ---
OpTxInputCount OpFalse OpGreaterThan OpIf
    OpFalse OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Script Integrity (Includes the Push instruction)
        OpFalse OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Add (Starts exactly after logic)
        OpFalse OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxInputSpkSubstr
        OpBin2Num OpAdd
    OpEndIf
OpEndIf

// --- INPUT 1 ---
OpTxInputCount OpTrue OpGreaterThan OpIf
    OpTrue OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Script Integrity
        OpTrue OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Add
        OpTrue OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxInputSpkSubstr
        OpBin2Num OpAdd
    OpEndIf
OpEndIf

// --- INPUT 2 ---
OpTxInputCount Op2 OpGreaterThan OpIf
    Op2 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Script Integrity
        Op2 OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Add
        Op2 OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxInputSpkSubstr
        OpBin2Num OpAdd
    OpEndIf
OpEndIf

// --- INPUT 3 ---
OpTxInputCount Op3 OpGreaterThan OpIf
    Op3 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Script Integrity
        Op3 OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Add
        Op3 OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxInputSpkSubstr
        OpBin2Num OpAdd
    OpEndIf
OpEndIf

// --- INPUT 4 ---
OpTxInputCount Op4 OpGreaterThan OpIf
    Op4 OpTxInputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Script Integrity
        Op4 OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Add
        Op4 OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxInputSpkSubstr
        OpBin2Num OpAdd
    OpEndIf
OpEndIf


// =========================================================================
// 3. OUTPUTS LOOP (SUBTRACTION) - INDEX 0 to 4
// =========================================================================

// --- OUTPUT 0 ---
OpTxOutputCount OpFalse OpGreaterThan OpIf
    OpFalse OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Output Script matches My Script
        OpFalse OpFalse OpPushData1 <LEN_LOGIC> OpTxOutputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Subtract
        OpFalse OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxOutputSpkSubstr
        OpBin2Num OpSub
    OpEndIf
OpEndIf

// --- OUTPUT 1 ---
OpTxOutputCount OpTrue OpGreaterThan OpIf
    OpTrue OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Output Script matches My Script
        OpTrue OpFalse OpPushData1 <LEN_LOGIC> OpTxOutputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Subtract
        OpTrue OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxOutputSpkSubstr
        OpBin2Num OpSub
    OpEndIf
OpEndIf

// --- OUTPUT 2 ---
OpTxOutputCount Op2 OpGreaterThan OpIf
    Op2 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Output Script matches My Script
        Op2 OpFalse OpPushData1 <LEN_LOGIC> OpTxOutputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Subtract
        Op2 OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxOutputSpkSubstr
        OpBin2Num OpSub
    OpEndIf
OpEndIf

// --- OUTPUT 3 ---
OpTxOutputCount Op3 OpGreaterThan OpIf
    Op3 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Output Script matches My Script
        Op3 OpFalse OpPushData1 <LEN_LOGIC> OpTxOutputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Subtract
        Op3 OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxOutputSpkSubstr
        OpBin2Num OpSub
    OpEndIf
OpEndIf

// --- OUTPUT 4 ---
OpTxOutputCount Op4 OpGreaterThan OpIf
    Op4 OpTxOutputCovId OpTxInputIndex OpTxInputCovId OpEqual OpIf
        // Verify Output Script matches My Script
        Op4 OpFalse OpPushData1 <LEN_LOGIC> OpTxOutputSpkSubstr
        OpTxInputIndex OpFalse OpPushData1 <LEN_LOGIC> OpTxInputSpkSubstr
        OpEqualVerify
        // Extract Amount and Subtract
        Op4 OpPushData1 <LEN_LOGIC> OpPushData1 <LEN_LOGIC+8> OpTxOutputSpkSubstr
        OpBin2Num OpSub
    OpEndIf
OpEndIf


// =========================================================================
// 4. FINAL VERIFICATION
// =========================================================================

// Stack state: [Accumulator]
// Must be equal to 0 (Total Inputs == Total Outputs)
OpFalse
OpEqual

// -------------------------------------------------------------------------
// PADDING (THIS IS NOW PART OF <LEN_LOGIC>)
// -------------------------------------------------------------------------
// OpPushData1  (The OpCode)
// 0x08         (The Length of data to push)

// -------------------------------------------------------------------------
// DATA (Starts at <LEN_LOGIC>)
// -------------------------------------------------------------------------
// [8 bytes AMOUNT]
// OpDrop
