## Dark Alex

An encrypted version of the [Alex Protcol](https://github.com/pmcgoohan/targeting-zero-mev/blob/main/content-layer.md)
(under construction - unresolved issues - hard hats must be worn)

### Overview
- user encrpyts txs before they enter the mempool (mitigates probabalistic attacks)
- user splits tx key and shares with [vaults](https://github.com/pmcgoohan/targeting-zero-mev/blob/main/content-layer.md#shuffler-vaults) (mitigates user withholding)
- [printer](https://github.com/pmcgoohan/targeting-zero-mev/blob/main/content-layer.md#printer) (miner/validator/sequencer) creates a chunk of encrpyted txs (encryption mitigates transaction censorship/re-ordering)
- printer adds chunk to the encrypted chunk queue at ≈network latency (≈1.2 secs - preserving time order [where meaningful](https://github.com/pmcgoohan/alex-latency-width))
- when users/vaults see a new encrypted chunk they release keys to decrypt the chunk
- printer adds chunk to the (decrypted) chunk queue (allowing auditability and visible guarantees of tx order before a block is printed)
- printer writes [contiguous chunks](https://github.com/pmcgoohan/targeting-zero-mev/blob/main/content-layer.md#printer-withholding) to the block from the chunk queue

### DDOS / Gas Price Visibility
There are two problems with users encrpyting txs (in any user encrypted mempool solution)
1) Printers cannot see the gas price in an encrypted tx
2) Users can DDOS the network with invalid txs

To solve this, we trust users to give us accurate information/validity guarantees about the content of an encrypted tx, and then penalize them if it turns out to be inaccurate once decrypted.
A user takes out a small, recoverable bond for each address. The bond only needs to be enough to cover a few invalid txs in order to mitigate DDOS.

##### Filtering
Vaults/Mempool can quickly filter out:
- any txs with zero bond balances
- any originator addresses with less than MinGas in eth

##### User Fraud proofs
- user key does not decrypt tx
- decrypted tx is invalid
- decrypted tx gasCost does not match encryptedTxMessage gasCost
- MinGas is not available to execute and lastBlockHash is only unlock time blocks old

#### ```EncryptedTxContract``` (L1 smart contract)

##### encryptedTxMessage (signed by originator)
```.encryptedTx```

```.gasPrice``` must match the encryptedTx gasPrice once decrypted

```.maxBlockHeight``` set to current blockheight + unlock time

```.lastBlockHash``` 

##### User functions
```.LockBond()``` send bond to sc (enough for a few failed txs)

```.UnlockBond()``` release remaining, cannot withdraw for n blocks

```.WithdrawBond()``` withdraw unlocked remaining bond once the minimum time has elapsed

##### Vault functions
```.PenalizeBadKey(encryptedTxMessage,keys0,1,...)``` if invalid, apply penalty to  user bond

```.InvalidTx(decryptedTx)``` if invalid, apply penalty to  user bond

```.InvalidGasCost(encryptedTxMessage,decryptedTx)``` if invalid, apply penalty to  user bond

```.InvalidBalance(not sure yet)``` is this neeeded if we are filtering out originator addresses with <MinGas above?

Vault/Printer fraud proofs will be similar to [Alex](https://github.com/pmcgoohan/targeting-zero-mev/blob/main/content-layer.md#validation-rules-and-proofs) 

##### TVM
For fraud proofs to work, the ```.InvalidTx``` and ```.InvalidGasCost``` functions must be calculable in the EVM.

This may require something like the OVM in Optimism (so a lot of this work has been done already). It only needs to allow for verification of a user tx which may simplify the problem.

[EIP #726](https://github.com/ethereum/EIPs/issues/726) would be a great solution.

For now let's call this the TVM (transaction verification virtual machine).

### Vaults

Vault set formation TBC.
A user might split their key between for eg: 2 groups of 3 vaults.

Users will choose vaults/vault groups based on how quick they are to respond and how secure in terms of not releasing data early. All this is publicly visible.

This means that these vital client/vault heuristics can be modified over time without any fork risk.

Vaults that misbehave by not releasing key confirmations/keys promptly and when required, or that release keys early will be dropped by users and will not recieve rewards.

Vaults could either be scheduled together in sets, or they could negotiate with each other to form groups, or some combination of both.
