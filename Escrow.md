# Escrow using the KaspaFungibleTokenProtocol (Draft)

## Abstract

This paper introduces a covenant-based mechanism for buying or selling a token built on top of the [KaspaFungibleTokenProtocol](https://github.com/La-Gauffre/KaspaFungibleTokenProtocol/blob/main/KaspaFungibleTokenProtocol.md), with all states stored within **the same UTXO**. 

---

## Mathematical Foundation


---


### Token Purchase



### Token Sale 



---

#### Script Structure


The `scriptPubKey` is constructed as follows:

`[ ...KaspaFungibleTokenProtocol Logic Opcodes... ] [...Escrow Logic Opcode...] 

Where:

- `<LEN_LOGIC>` is defined below and has the same length as in the KaspaFungibleTokenProtocol  


#### Assembly Implementation


```

// -------------------------------------------------------------------------
// 1. KASPA FUNGIBLE TOKEN PROTOCOL
// -------------------------------------------------------------------------

//Just copy & paste the KASPA FUNGIBLE TOKEN PROTOCOL logic here

// -------------------------------------------------------------------------
// 2. BINARY FLAG + SECURITY CHECK 
// -------------------------------------------------------------------------


