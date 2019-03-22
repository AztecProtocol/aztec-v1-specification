# AZTEC Protocol 1.0.0 specification  

## Table of contents  

1.  [Architecture](#architecture)  
    1.  [Cryptography Engine](#the-aztec-cryptography-engine)
    1.  [AZTEC ABI Encodings](#abi-encoding-and-aztec-data-types) 
        1.  [Note ABI](#AZTEC-note-ABI)  
        1.  [UXTO note ABI](#type-1-uxto-notes)  
        1.  [Treasury note ABI](#type-2-El-Gamal-treasury-notes)
        1.  [Note metadata ABI](#note-metadata-recovering-viewing-keys-and-shared-secrets)
1.  [Note Registry](#the-note-registry)  
1.  [ACE](#ace-the-aztec-cryptography-engine)
    1.  [Proof identification](#validating-aztec-proofs-defining-the-proofs-identifier)  
    1.  [proofOutputs ABI](#enacting-confidential-transfer-instructions-defining-the-abi-encoding-of-proofoutputs)
    1.  [proofOutput ABI](#abi-encoding-for-bytes-proofoutput)  
    1.  [Cataloguing-valid-proofs](#cataloguing-valid-proofs-inside-ace)  
    1.  [Responsibilities of ACE](#the-key-responsibilities-of-ace)  
1.  [Contract interactions](#contract-interactions)
    1.  [Bilateral swap interactions](#zero-knowledge-dapp-contract-interaction-an-example-flow-with-bilateral-swaps)
1.  [Validating an AZTEC proof](#validating-an-aztec-proof)  
1.  [Creating a note registry](#creating-a-note-registry)  
1.  [Processing transfer instructions](#processing-a-transfer-instruction)  
1.  [Minting AZTEC notes](#minting-aztec-notes)  
1.  [Burning AZTEC notes](#burning-aztec-notes)  
1.  [Zero-knowledge assets](#interacting-with-ace-zkasset)
    1. [Creating a confidential asset](#creating-a-confidential-asset)  
    1. [Issuing confidential transactions](#issuing-a-confidential-transaction-confidentialtransfer)  
    1. [Issuing delegated confidential transactions](#issuing-delegated-confidential-transactions-confidentialtransferfrom)
    1. [Approving confidential notes](#confidentialapprove)
1.  [AZTEC Verifiers - JoinSplit](#aztec-verifiers-joinsplitsol)
1.  [Appendix](#appendix)
    1.  [Preventing collisions and front-running](#a-preventing-collisions-and-front-running)
    1.  [Interest Streaming](#b-interest-streaming-via-AZTEC-notes)  
1.  [Glossary](#glossary)

# Architecture  

The AZTEC protocol enables efficient confidential transactions through the construction of AZTEC-compatible zero-knowledge proofs. Specifically, the protocol focuses on optimizing confidential *settlement* and other forms of value-transfer

The protocol is architected to optimize for the following factors:  

1. customizability - AZTEC assets must have confidential transaction semantics that can be modified to suit the ends of the user  
2. interoperability - different AZTEC assets must conform to a standard interface that dApps can use to settle confidential transactions  
3. efficiency - no redundant computation should be performed when verifying confidential transactions  
4. qualified upgradability - as improvements are made to the underlying cryptographic protocols, and additional proof systems are added into AZTEC, existing confidential assets should be able to enjoy the benefits of these improvements. At the same time, users of AZTEC must be able to have confidence that they can opt out of these upgrades - that the verification algorithms used to validate existing zero-knowledge proofs are immutable  

## The AZTEC Cryptography Engine  

The focus of our protocol is this cryptography engine (ACE.sol). ACE is the ultimate arbiter of the correctness of an AZTEC zero-knowledge proof. AZTEC assets subscribe to ACE and call on it to validate proofs.  

ACE converts zero-knowledge proof data into *instructions* - directions on the following:  

1. AZTEC notes to be created  
2. AZTEC notes to be destroyed  
3. Public tokens that need to be transferred  

Internally, ACE will create a unique representation of each proof instruction and store it  

## ABI encoding and AZTEC data 'types'  

The nature of zero-knowledge cryptography means that a significant volume of data is processed on-chain in the form of zero-knowlege proof *inputs* and zero-knowledge proof *outputs*.  

Because using structs in external functions is still an experimental feature, the AZTEC protocol defines its own ABI encoding for struct-like data types. These objects are represented by the `bytes` type, where the contents of the bytes array contains data that is formatted according to the AZTEC protocol's ABI specification.  

### AZTEC note ABI

One key feature of ACE is the ability to support multiple note 'types'. Different note types enable the engine to support zero-knowledge proofs that use different techniques to represent encrypted value.  

For example, the basic AZTEC note is the most efficient way to represent encrypted value, however it's UXTO-like form may be unsuitable for some applications. On the other hand, ElGamal 'treasury' notes can be used to emulate a more traditional account-balance model, where the balance is encrypted.  

All notes use the same basic structure, but with different `publicKey` values. Every AZTEC zero-knowlege proof explicitly defines the type of note that it utilizes. Under no circumstances should it be possible to use a note of the wrong 'type' in a zero-knowledge proof.

The ABI encoding of a note is as follows:  

| Offset | Length | Name | Type | Description|
| ------ | ------ | -------- | --- | --- |
| 0x00   | 0x20   | id | uint256 | The 'type' identifier of the note |
| 0x20   | 0x20   | owner | address | Ethereum address of note owner |
| 0x40   | L_pub   | publicKey | bytes | The public key of the note, that is used to encrypt value |  
| 0x40 + L_pub | L_met | noteData | bytes | Note-specific data |

`metadata` is not used by the logic of AZTEC zero-knowlege proof validators, but is instead used when broadcasting events involving a note. The metadata of a note can be used by the note's owner to recover the note's viewing key.

### Type 1: UXTO notes  

This is the default note type and used by the majority of our protocols. The ABI formatting of this note's `publicKey` is as follows:

| Offset | Length | Name | Type | Description|
| ------ | ------ | -------- | --- | --- |
| 0x00   | 0x20   | gamma | bytes32 | (compressed) bn128 group element |
| 0x20   | 0x20   | sigma | bytes32 | (compressed) bn128 group element |
| 0x40   | -      | metadata | bytes | custom metadata associated with note |  

### Type 2: El-Gamal treasury notes  

Treasury notes enable a single 'account' to have their balance represented by a single treasury note (instead of a multitude of AZTEC UXTO-type notes). They are slightly more gas-expensive to use than AZTEC notes and are only used in a small subset of AZTEC zero-kowledge proofs.

| Offset | Length | Name | Type | Description|
| ------ | ------ | -------- | --- | --- |
| 0x00   | 0x20   | ownerPubKey | bytes32 | (compressed) bn128 group element that maps to the public key of the note's owner |
| 0x20   | 0x20   | noteEphemeralKey | bytes32 | (compressed) bn128 group element, a public key component of the note's ephemeral key |
| 0x40 | 0x20 | noteCommitment | bytes32 | (compressed) bn128 group element, the core El-Gamal commitment |
| 0x60   | -      | metadata | bytes | custom metadata associated with note |  

### Note metadata, recovering viewing keys and shared secrets  

The `metadata` field contains information that can be used by a note owner to decrypt a note. Every note viewing key should be distinct, however users should not have to manage a multitude of unique viewing keys. In addition, if user A wishes to send user B a note, they should be able to derive a viewing key that A can recover. This process should be non-interactive.

The solution is to use a shared secret protocol, between an 'ephemeral' public/private key pair and the public key of the note owner. An extension of this protocol can be used to derive 'stealth' addresses, if the recipient has a stealth wallet. Currently, our V1 APIs use the basic shared secret protocol for ease of use (traditional Ethereum wallets can own these kinds of AZTEC notes). At the smart contract level, the protocol is forward-compatible with stealth addresses.  
  
The `metadata` field stores the data that a user requires to recover their note viewing key - the 'ephemeral' public key. Additionally, if an ECIES encrypted message is attatched to the note, it will be located in the `metadata` field.  

# The Note Registry  

The AZTEC note registry contract is a subset of the AZTEC Cryptography Engine, but we describe it explicitly given its importance to the protocol.  

The note registry contains the storage variables that define the set of valid AZTEC notes for a given address. It is expected this address maps to a smart contract, but this is not enforced.  

The note registry enacts the instructions generated by valid AZTEC proofs - creating and destroying the required notes, as well as transferring any required tokens.  

The note registry's `owner` is the only entity that can issue instructions to update the registry. `NoteRegistry` will only enact instructions that have been generated by a valid AZTEC proofs as it is of critical importance that notes are not created/destroyed unless a **balancing relationship** has been satisfied.  

Because every confidential asset that uses an ACE note registry can have 100% confidence in the integrity of the state of every *other* ACE note registry, it makes it possible to express AZTEC notes from one registry as a percentage of notes in a second registry, which in turn is useful for dividend-paying confidential assets and confidential assets that utilize income streaming.

# ACE, the AZTEC Cryptography Engine

The `ACE.sol` contract is responsible for validating the set of AZTEC zero-knowledge proofs and performing any transfer instructions involving AZTEC notes. ACE is the controller of all AZTEC note registries and acts as the custodian of both AZTEC notes and any tokens that have been converted into AZTEC notes.  

While it is possible to define note registries that are external to ACE, the state of these contract's note registries cannot be gauranteed and only a subset of proofs will be usable (i.e. if an asset uses an ACE note registry, transfer instructions from AZTEC proofs that involve multiple note registries are only enacted if every note registry is controlled by ACE).

## Validating AZTEC proofs - defining the proof's identifier  

ACE supports multiple types of zero-knowlege proof and this family of proofs will grow over time. It is important to categorise these proofs in a systematic manner.  

The ACE proof identification and versioning sytem has the following characteristics:

* Extendibility. AZTEC's modular proof system enables composable confidential transaction semantics - adding more proofs enables these semantics to be more expressive. Additionally, it allows the AZTEC protocol to support fundamentally new types of zero-knowledge proving technology as Ethereum scales (e.g. bulletproofs, zk-snarks)
* Opt-out functionality. If an asset controller only wants to listen to a subset of proofs (e.g. whether to listen to newly added proofs is on their terms. This is important for assets that have an internal review process for zero-knowledge proofs)  
* Qualified immutability. The validator code for a given proof id should never change. AZTEC must be able to de-activate a proof if it is later found to contain a bug, but any upgrades or improvement to a proof are expressed by instantiating a new validator contract, with a new proof id.  

A proof is uniquely defined by its `uint24 proofId`. ACE stores a mapping that maps each `proofId` to the address of a smart contract that validates the zero-knowledge proof in question.  

Instead of having a 'universal' validation smart contract, it was chosen to make these contracts discrete for maximum flexibility. Validator contracts should not be upgradable, to gaurantee that users of AZTEC proofs can have confidence that the proofs they are using are not subject to change. Upgrades and changes are implemented by adding new validator contracts and new proofs.  

The `uint24 proofId` variable contains the concatenation of three `uint8` variables (the rationale for this compression is to both reduce `calldata` size and to simplify the interface. Our javascript APIs automatically compose proofs with the correct `proofId`, minimizing the amount of variables that a builder on AZTEC has to keep track of.

The formatting as follows (from most significant byte to least significant byte)

| name | type | description |
| --- | --- | --- |
| epoch | uint8 | the broad family that this proof belongs to |
| category | uint8 | the general function of this proof |
| id | uint8 | the proof's identifier, for the specified category and epoch |  

A semantic-style version system was not used because proof `epoch` defines functionality as well as a form of version control. Proofs with the same `uint8 id` but with different `uint8 epoch` do not neccesarily perform the same function and proofs from a later `epoch` are not strictly 'better' than proofs from an earlier `epoch`.  

For example, if the basic family of AZTEC proofs was adapted for confidential transactions that do not use a trusted setup, these proofs would be categorized by a new `epoch`. However these would not be a strict upgrade over the earlier epoch because the gas costs to verify these proofs would be almost an order of magnitude greater.  

Similarly, when confidential voting mechanics are implemented into `ACE`, these will be defined by a separate `epoch` to emphasise their different functionality vs confidential transactions.

The `uint8 category` variable represents an enum with the four following values:  

| value | name | description |
| --- | --- | --- |  
| 0x01 | BALANCED | proofs that satisfy a balancing relationship |  
| 0x02 | MINT | proofs that can be used to mint AZTEC notes |  
| 0x03 | BURN | proofs that can be used to burn AZTEC notes |  
| 0x04 | UTILITY | utility proofs that can not, in isolation, be used to issue note transfers |  

The `ACE` contract has separate logic to handle `BALANCED`, `MINT` and `BURN` proofs, as the latter two expressly violate the balancing relationship used to prevent double spending. The `MINT` and `BURN` proofs are designed for fully private AZTEC assets, ones with no ERC20 token equivalent, where AZTEC notes are the primary expression of value. Additional restrictions are placed on their use because of this. 

For more information regarding minting and burning, see (REF HERE).  

The `UTILITY` proofs are designed for assets that require additional validation logic before a transaction can be performed. For example, an asset might require a trader to prove that they own less than 10% of the total supply of the asset before a trade is processed. This is supported by our `dividend` utility proof.  

For a complete specification of our proofs, see (BLAH)  

When combined together, `uint8 epoch, uint8 category, uint8 id` create 65025 unique proof identifies for each category. Given the complexity of zero-knowledge cryptographic protocols and the validation that must be performed before integration into `ACE`, it is infeasible for there to ever be demand for more than `65025` types of zero-knowledge proof inside `ACE`.

## Enacting confidential transfer instructions - defining the ABI encoding of proofOutputs

There is a great deal of variation between the zero-knowledge proofs that AZTEC utilizes. Because of this, and the desire to create a simple interface to validate proofs, the interface for proof *inputs* is kept generic. (REF HERE).  

However, the **output** of a zero-knowledge proof is a list of instructions to be performed. It is important that these `proofOutput` variables conform to a common standard so that existing confidential assets can benefit from the addition of future proofs.  

An instruction must contain the following:  

* List the notes to be destroyed, the 'input notes'  
* List the notes to be created, the 'output notes'  
* If public tokens are being transferred, how many tokens are involved, who is the beneficiary and what is the direction of the transfer? (into ACE or out of ACE?)  

In addition to this, ACE must support one zero-knowledge proof producing *multiple* instructions (e.g. the `bilateral-swap` proof provides transfer instructions for two distinct assets).  

Proofs in the `UTILITY` category also conform to this specification, although in this context 'input' and 'output' notes are not created or destroyed.  

To summarise, the output of any AZTEC validator smart contract is a `bytes proofOutputs` variable, that encodes a dynamic array of `bytes proofOutput` objects. The ABI encoding is as follows:  

| Offset | Length | Name | Type | Description |  
| --- | --- | --- | --- | --- |  
| 0x00 | 0x20 | length | uint256 | the number of `proofOutput` objects |  
| 0x20 | (0x20 * length) | offsets | uint256[length] | array of 0x20-sized variables that contain the relative offset to each respective `proofOutput` object |  
| 0x20 + (0x20 * length) + (\sum_{j=0}^{i-1}L[j]) | L[i] | proofOutput[i] | bytes | the `bytes proofOutput` object |  

## ABI encoding for `bytes proofOutput`  

| Offset | Length | Name | Type | Description |  
| --- | --- | --- | --- | --- |  
| 0x00 | 0x20 | inputsOffset | uint256 | the relative offset to `bytes inputNotes` |  
| 0x20 | 0x20 | outputsOffset | uint256 | the relative offset to `bytes outputNotes` |  
| 0x40 | 0x20 | publicOwner | address | the Ethereum address of the owner of tokens being transferred |  
| 0x60 | 0x20 | publicValue | int256 | the amount of public 'value' being transferred |  
| 0x80 | 0x20 | challenge | uint256 | the 'challenge' variable used in the zero-knowledge proof that produced this output |  
| 0xa0 | L_1 | inputNotes | bytes | the `bytes inputNotes` variable |  
| 0xa0 + L_1 | L_2 | outputNotes | bytes | the `bytes outputNotes` variable |  

Both `bytes inputNotes` and `bytes outputNotes` are dynamic arrays of AZTEC notes, encoded according to the AZTEC note ABI spec.  

The `int256 publicValue` variable is a *signed* integer, because negative values are interpreted as tokens being transferred *from* `address publicOwner` and into `ACE`. Similarly, positive values are interpreted as tokens being transferred *to* `address publicOwner`.  

It should be noted that `int256 publicValue` does not represent an absolute number of tokens. Each registry inside `NoteRegistry` has an associated `uint256 scalingFactor`, that defines how many ERC20 tokens are represented by 1 unit of AZTEC note 'value'. This mapping is neccessary because AZTEC note values are approximately 30-bit integers (CAVEAT HERE) and a scaling factor is required to map 256-bit ERC20 token volumes to 30-bit AZTEC values.  

The `uint256 challenge` variable is used to ensure that each `bytes proofOutput` produces a unique hash. The challenge variable is required for every AZTEC zero-knowledge proof, and is itself a unique pseudorandom identifier for the proof (two satisfying zero-knowledge proofs cannot produce matching challenge variables without a hash collision).  For a proof that produces multiple `bytes proofOutput` entries inside `bytes proofOutputs`, it is the responsibility of the verifier smart contract to ensure each challenge variable is unique (i.e. each `bytes proofOutput` contains a challenge variable that is a hash of the challenge variable for the previous entry).

Consequently, a hash of `bytes proofOutput` creates a unique identifier for a proof instruction because of the uniqueness of the challenge variable.  

## Cataloguing valid proofs inside ACE  

Once a `BALANCED`, `MINT` or `BURN` proof has been validated, ACE records this fact so that future transactions can query the proof in question. This is done by creating a keccak256 hash of the following variables (encoded in an unpacked form)  

| Offset | Length | Name | Type | Description |  
| --- | --- | --- | --- | --- |  
| 0x00 | 0x20 | proofHash | bytes32 | a keccak256 hash of `bytes proofOutput` |  
| 0x20 | 0x20 | proofId | uint24 | the proofId of the proof |  
| 0x40 | 0x20 | msg.sender | address | the address of the entity calling `ACE` |  

This creates a unique key, that is mapped to `true` if the proof is valid (invalid proofs are not stored).  

Contracts can query `ACE` with a `bytes proofOutput`, combined with a `uint24 proofId` and the `address` of the entity that issued the instruction. `ACE` can validate whether this instruction came from a valid proof.  

This mechanism enables smart contracts to issue transfer instructions on behalf of both users and other smart contracts, enabling zero-knowledge confidential dApps. 


# The key responsibilities of `ACE`  

The `ACE` engine has two critical responsibilities:  

1. Determine the correctness of valid AZTEC zero-knowledge proofs and permanently record the existence of validated `BALANCED` proofs  
2. Update the state of its note registries when presented with valid transfer instructions

When processing a transfer instruction, the following criteria must be met:  

* Did the transfer instruction originate from the note registry's owner?  
* Is the transfer instruction sourced from a *mathematically legitimate* AZTEC proof?  

Because of these dual responsibilities, valid AZTEC proofs are *not* catalogued against specific note registries. The outputs of any valid proof can, theoretically, be issued to any note registry. After all, the existence of a valid proof indicates the resulting transfer instructions are balanced. This is the critical property that `ACE` *must* ensure, that all of its note registries are balanced and that there is no double spending.  
  
Restricting note registry updates to the creator of a given note registry provides a natural separation of concerns - `ACE` determines whether a transfer instruction *can* happen and the note registry owner determines whether the instruction *should* happen.  

### Separating proof validation and note registry interactions  

Because of these dual responsibilities, it might seem intuitive to roll proof validation and note registry updates into a single function. However this would undermine one of the key strengths of the AZTEC protocol - that third party dApps can validate zero-knowledge proofs and send the resulting transfer instructions to AZTEC-compatible confidential assets. (LINK HERE) demonstrates this type of interaction and, consequently, the importance of separating proof validation from note registry updates.

# Contract Interactions  

![user-to-validator](https://github.com/CreditMint/wiki/blob/master/userToValidator.png?raw=true)  

Transaction #1  

1. `ACE.validateProof(uint24 proofId, address sender, bytes proofData)`  
2. `Validator.validate(bytes proofData, address sender, uint[6] commonReferenceString)` (revert on failure, return `bytes proofOutputs`)  
3. return `bytes proofOutputs` to `ACE`, revert on failure  
4. return `bytes proofOutputs` to caller, log valid proof if category != `UTILITY`, revert on failure  

![user-to-registry](https://github.com/CreditMint/wiki/blob/master/userToRegistry2.png?raw=true)

Transaction #1  

1. `ACE.updateNoteRegistry(uint24 proofId, bytes proofOutput, address sender)`  
2. `NoteRegistry.validateProofByHash(uint24 proofId, bytes proofOutput, address sender)` (revert on failure)
3a. (if `proofOutput.publicValue > 0`) `ERC20.transfer(proofOutput.publicOwner, uint256(proofOutput.publicValue))` (revert on failure)
3b. (if `proofOutput.publicValue < 0`) `ERC20.transferFrom(proofOutput.publicOwner, this, uint256(-proofOutput.publicValue))` (revert on failure)  
4. NoteRegistry: (revert on failure)  
5. ACE: (revert on failure)  

## Zero-knowledge dApp contract interaction, an example flow with bilateral swaps  

The following image depicts the flow of a zero-knowledge dApp that utilizes the `bilateral-swap` proof to issue transfer instructions to two zkERC20 confidential digital assets. This example aims to illustrate the kind of confidential cross-asset interactions that are possible with AZTEC. Later iterations of the protocol will include proofs that enable similar multilateral flows.  

The dApp-to-zkERC20 interactions are identical for both `zkERC20 A` and `zkERC20 B`. To simplify the description we only describe the interactions for one of these two assets.

![zk-dapp-flow](https://github.com/CreditMint/wiki/blob/master/zkDappFlow2.png?raw=true)  

### (1-5) : Validating the proof  

1. `zk dApp` receives a `bilateral-swap` zero-knowledge proof from `caller` (with a defined `uint24 proofId` and `bytes proofData`.  

2. The `zk-dApp` contract queries `ACE` to validate the received proof,  via `ACE.validateProof(proofId, msg.sender, proofData)`. If `proofId` is not supported by `zk-dApp` the transaction will `revert`.

3. On receipt of a valid proof, `ACE` will identify the `validator` smart contract associated with `proofId` (in this case, `BilateralSwap.sol`). `ACE` will then call `validator.validateProof(proofData, sender, commonReferenceString)`. If the `proofId` provided does not map to a valid `validator` smart contract, the transaction will `revert`.

4. If the proof is valid, the `validator` contract will return a `bytes proofOutputs` object to `ACE`. If the proof is invalid, the transaction will `revert`.  

5. On receipt of a valid `bytes proofOutputs`, `ACE` will examine `proofId` to determine if the proof is of the `BALANCED` category. If this is the case, `ACE` will iterate over each `bytes proofOutput` in `bytes proofOutputs`. For each `proofOutput`, the `bytes32 proofHash` is computed. A unique proof identifier, `bytes32 proofIdentifier = keccak256(abi.encode(proofId, msg.sender, proofHash))`, is then computed. This is used as a key to log the existence of a valid proof - `validProofs[proofIdentifier] = true`.  

Once this has been completed, `ACE` will return `bytes proofOutputs` to `zk-dApp`.

### (6-8): Issuing a transfer instruction to `zkERC20 A`  

At this stage, `zk-dApp` is in posession of transfer instructions that result from a valid `bilateral-swap` proof, in the form of a `bytes proofOutputs` object received from `ACE`.  

For the `bilateral-swap` proof, there will be `2` entries inside `proofOutputs`, with each entry mapping to one the two confidential assets - `zkERC20 A` and `zkERC20 B`.  

6. The `zk-dApp` contract issues a transfer instruction to `zkERC20 A` via `zkERC20.confidentialTransferFrom(proofId, proofOutput)`.  

7. On receipt of `uint24 proofId, bytes proofOutput`. The `zkERC20 A` contract validates that `proofId` is on the contract's proof whitlelist. If this is not the case, the transaction will `revert`.  

`zkERC20 A` computes `bytes32 proofHash` and query `ACE` as to the legitimacy of the received instructions, via `ACE.validateProofByHash(proofId, proofHash, msg.sender)`.  

8. `ACE` queries its `validProofs` mapping to determine if a proof that produced `bytes proofOutput` was previously validated and return a boolean indicating whether this is the case.  

If no matching proof was previously validated by `ACE`, `zkERC20 A` will `revert` the transaction.  

### (9-16): Processing the transfer instruction  

Having been provided with a valid `proofOutput` that satisfies a balancing relationship, `zkERC20 A` will validate the following:  

* For every input `note`, is `approved[note.noteHash][msg.sender] == true`?  

If this is not the case, the transaction will `revert`.  

9. If all input notes have been `approved`, `zkERC20 A` will instruct `ACE` to update its note registry according to the instructions in `proofOutput`, via `ACE.updateNoteRegistry(proofId, proofOutput, msg.sender)`.

10. On receipt of `bytes proofOutput`, `ACE` will also validate that the `proofOutput` instruction came from a valid zero-knowledge proof (and `revert` if this is not the case). Having been satisfied of the proof's correctness, `ACE` will instruct the note registry owned by `msg.sender` (`zkERC20 A`) to process the transfer instruction.  

11. `NoteRegistry A` will validate the following is correct:  

* For every input `note`, is `note.noteHash` present inside the `registry`?  
* For every output `note`, is `note.noteHash` *not* present inside the `registry`?  

If `proofOutput.publicValue > 0`, the registry will call `erc20.transfer(proofOutput.publicOwner, uint256(proofOutput.publicValue))`.  

If `proofOutput.publicValue < 0`, the registry will call `erc20.transferFrom(proofOutput.publicOwner, this, uint256(-proofOutput.publicValue))`.  

12. If the resulting transfer instruction fails, the transaction is `reverted`, otherwise control is returned to `Note Registry A`  

13-15. If the transaction is successful, control is returned to `ACE`, followed by `zkERC20 A` and `zk-dApp`.  

16. Following the successful completion of the confidential transfer (from both `zkERC20 A` and `zkERC20 B`), control is returned to `caller`. It is assumed that `zk-dApp` will emit relevant transfer events, according to the ERC-1724 confidential token standard.

### The rationale behind multilateral confidential transactions  

The above instruction demonstrates a practical confidential cross-asset settlement mechanism. Without `ACE`, a confidential digital asset could only process a transfer instruction after validating the instruction conforms to its own internal confidential transaction semantics, a process that would require validating a zero-knowledge proof.  

This would result in 3 distinct zero-knowledge proofs being validated (one each by `zk-dApp`, `zkERC20 A`, `zkERC20 B`). Because zero-knowledge proof validation is the overwhelming contributor to the cost of confidential transactions, this creates a severe obstacle to practical cross-asset confidential interactions.

However, by subscribing to `ACE` as the arbiter of valid proofs, these three smart contracts can work in concert to process a multilateral confidential transfer having validated only a single zero-knowledge proof (this is because the `bilateral-swap` proof produces transfer instructions that lead to two balancing relationships. Whilst `zkERC20 A` and `zkERC20 B` do not know this (the proof in question could have been added to `ACE` *after* the creation of these contracts), `ACE` does, and can act as the ultimate arbiter of whether a transfer instruction is valid or not.  

Whilst it may apear that this situation requires AZTEC-compatible assets to 'trust' that ACE will correctly validate proofs, it should be emphasized that `ACE` is a completely deterministic smart-contract whose code is fully available to be examined. No real-world trust (e.g. oracles or staking mechanisms) is required. The source of the guarantees around the correctness of AZTEC's confidential transactions come from its zero-knowledge proofs, all of which have the properties of completeness, soundness and honest-verifier zero-knowledge.

# Validating an AZTEC proof  

AZTEC zero-knowledge proofs can be validated via `ACE.validateProof(uint24 proofId, address sender, bytes calldata proofData) external returns (bytes memory proofOutputs)`.  

The `bytes proofData` uses a custom ABI encoding that is unique to each proof that AZTEC supports. It is intended that, if a contract requires data from a proof, that data is extracted from `bytes proofOutputs` and not the input data.  

If the `uint8 category` inside `proofId` is not of type `UTILITY`, `ACE` will record the validity of the proof as a state variable inside `mapping(bytes32 => bool) validatedProofs`.  

If the proof is not valid, an error will be thrown. If the proof is valid, a `bytes proofOutputs` variable will be returned, describing the instructions to be performed to enact the proof. For `BALANCED` proofs, each individual `bytes proofOutput` variable inside `bytes proofOutputs` will satisfy a balancing relationship.  


# Creating a note registry  

An instance of a note registry is created inside ACE, via `createNoteRegistry(address _linkedTokenAddress, uint256 _scalingFactor, bool _mintable, bool _burnable, bool _convertable)`.  

The `_mintable`, `_burnable` and `_convertable` flags define whether the note registry owner can directly mint/burn AZTEC notes, as well as whether ERC20 tokens from `_linkedTokenAddress` can be converted into AZTEC notes. If `_convertable` is `false`, then `_linkedTokenAddress = address(0)` and the asset is a fully private asset.  

For a given note registry, only the owner can call `ACE.updateNoteRegistry`. Traditionally this is imagined to be a `zkAsset` smart contract. This allows the `zkAsset` contract to have absolute control over what types of proof can be used to update the note registry, as well as the conditions under which updates can occur (if extra validation logic is required, for example).  

# Processing a transfer instruction  

Once a proof instruction has been received (either through `ACE` or via a third party that validated a proof through `ACE`, for example a confidential decentralized exchange dApp), it can be processed by calling `ACE.updateNoteRegistry(uint24 proofId, bytes proofOutput, address sender)`.  

* If `msg.sender` has not registered a note registry inside `ACE`, the transaction will throw  
* If the the proof instruction was **not** sourced from a proof that `ACE` validated, the transaction will throw  
* If `validatedProofs[keccak256(abi.encode(proofId, sender, keccak256(proofOutput)))] == false`, the transaction will throw  

If the above criteria are satisfied, the instruction is passed to `NoteRegistry`, where the following checks are validated against:  

* If any note in `proofOutput.inputNotes` does not hash to a key that does not exist inside `noteRegistry`, the transaction will throw  
* If any note in `proofOutput.outputNotes` hashes to a key that *already* exists inside `noteRegistry`, the transaction will throw  
* If `proofOutput.publicValue != 0` and the asset is not `mixed`, the transaction will throw  

Once these conditions have been satisfied, every note in `proofOutput.inputNotes` is destroyed, and every note in `proofOutput.outputNotes` is created.  

Additionally, if `proofOutput.publicValue < 0`, `linkedToken.transferFrom(proofOutput.publicOwner, this, uint256(-proofOutput.publicValue))` is called. If this call fails, the transaction will throw.  
If `proofOutput.publicValue > 0`, `linkedToken.transfer(proofOutput.publicOwner, uint256(proofOutput.publicValue))` will be called. If this call fails, the transaction will throw.  

### A note on ERC20 token transfers  

For `mixed` assets, if tokens are withdrawn from AZTEC then, from the balancing relationships checked by AZTEC's zero-knowledge proofs, `ACE` will always have a sufficient balance, as the only way to create AZTEC notes is by depositing tokens in the first place.  

For `mintable` assets that are also `mixed`, there are additional steps that a digital asset builder must implement. If an AZTEC note is directly minted, and then converted into tokens, `ACE` will not have a sufficient token balance to initiate the transfer. 

# Minting AZTEC notes  

Under certain circumstances, a digital asset owner may wish to directly mint AZTEC notes. One example is a confidential digital loan, where the loan originators create the initial loan register directly in the form of AZTEC notes.  

At the creation of a note registry, the registry owner can choose whether their registry is 'mintable' by setting `bool _canMint` to `true` in `ACE.createNoteRegistry(address _linkedTokenAddress, bool _canMint, bool _canBurn, bool _canConvert)`.  

A 'mintable' note registry has access to the `ACE.mint(uint24 _proofId, bytes _proofData)` function. This function will validate the proof defined by `_proofId, _proofData` (and assert that this is a `MINTABLE` proof) and then immediately enact the produced `bytes proofOutput` at the note registry controlled by `msg.sender`.  

A `MINTABLE` proof follows a defined standard. The note registry contains a `bytes32 totalMinted` variable that is the hash of an AZTEC UXTO note that contains the total value of AZTEC notes that been minted by the registry owner.  

A `MINTABLE` proof will produce a `proofOutputs` object with two entries.  

* The first entry contains the old `totalMinted` value and the new `totalMinted` value  
* The second entry contains a list of notes that are to be minted  

If the `totalMinted` value does not match the old `totalMinted` value in `proofOutputs`, the transaction will revert.  

If all checks pass, the relevant AZTEC notes will be added to the note registry.  

## Minting and tokens  

Care should be taken if AZTEC notes are directly minted into an asset that can be converted into ERC20 tokens. It is possible that a conversion is attempted on a note and the token balance of the note registry in question is insufficient. Under these circumstances the transaction will revert. It is the responsibility of the note registry owner to provide `ACE` with sufficient tokens to enable such a transfer, as it falls far outside the remit of the Cryptography Engine to request minting priviledges for any given ERC20 token.  

This can be performed via `ACE.supplementTokens(uint _value)`, which will cause `ACE` to call `transferFrom` on the relevant ERC20 token, using `msg.sender` both as the transferee and the note registry owner. It is assumed that the private digital asset in question has ERC20 minting priviledges, if the note registry is also mintable.  

# Burning AZTEC notes

Burning is enacted in an identical fashion to note minting. The total amount of burned AZTEC notes is tracked by a `bytes32 totalBurned` variable.  

Burn proofs follow a similar pattern - updating the `totalBurned` variable and destroying the specified AZTEC notes.  

It should be stressed that only a note registry owner, who has set the relevant permissions on their note registry, can call `ACE.mint` and `ACE.burn`.

# Interacting with ACE: zkAsset

The `zkAsset.sol` contract is an implementation of a confidential token, that follows the [EIP-1724 standard](https://github.com/ethereum/EIPs/issues/1724). It is designed as a template that confidential digital asset builders can follow, to create an AZTEC-compatible asset.  

## Creating a confidential asset  

A `zkAsset` contract must instantiate a note registry inside `ACE` via `ACE.createNoteRegistry`. If the asset is a mixed, the contract address of the linked `ERC20` token must be supplied.

## Issuing a confidential transaction: confidentialTransfer

The primary method of unilateral value transfer occurs via `zkAsset.confidentialTransfer(bytes proofData)`. In this method, the AZTEC proof defined by the contract's `DEFAULT_PROOF_ID` is used to enact a value transfer. The beneficiaries of the transaction are defined entirely by the contents of `bytes proofData`.  

Both `ACE.validateProof(proofData)` and `ACE.updateNoteRegistry(proofOutput)` must be called, with `proofOutput` being extracted from `ACE.validateProof`'s return data.  

## Issuing delegated confidential transactions: confidentialTransferFrom  

The `confidentialTransferFrom(uint24 _proofId, bytes proofData)` method is used to perform a delegated transfer. As opposed to `confidentialTransfer`, `confidentialTransferFrom` can use any proof supported by `ACE` (assuming the `zkAsset` contract accepts this type of proof).  

A consequence of this is that the `zkAsset` contract must validate permissioning. The default `joinSplit` proof validates that ECDSA signatures have been signed over every input note. However this is not suitable for a delegated transfer, where note 'owners' may be smart contracts (and therefore not capable of creating digital signatures).  

For `confidentialTransferFrom` to be used, `confidentialApprove` must be called on every input note that is consumed  

## confidentialApprove  

The `confidentialApprove(bytes32 _noteHash, address _spender, bool _status, bytes memory _signature)` method gives the `_spender` address permission to use an AZTEC note, whose hash is defined by `_noteHash`, to be used in a zero-knowledge proof.  

The `_status` boolean defines whether permission is being given or revoked.  

The `_signature` variable defines an ECDSA signature over an EIP712 message. This signature is signed by the `address owner` of the AZTEC note being approved.  

If `_signature = bytes(0x00)`, then `msg.sender` is expected to be the `address owner` of the AZTEC note being approved.  

This interface is designed to facilitate stealth addresses. For a stealth address, it is unlikely that the address will have any Ethereum funds to pay for gas costs, and a meta-transaction style transaction is required. In this situation, `msg.sender` will not map to the owner of the note and so an ECDSA signatue is used.  

For other uses, such as a smart contract or a non-stealth address, a direct transaction sent by the correct `msg.sender` is possible by sending a null signature.


# AZTEC Verifiers: JoinSplit.sol  

The `JoinSplit` contract validates the AZTEC join-split proof, and performs ECDSA signature validation logic for signatures signed by each note owner.  

The ABI of `byes proofData` is the following:  

| offset | length | name | type | description |  
| --- | --- | --- | --- | --- |  
| 0x00 | 0x20 | m | uint256 | number of input notes |  
| 0x20 | 0x20 | challenge | uint256 | zero-knowledge proof challenge |  
| 0x40 | 0x20 | publicOwner | address | beneficiary of public tokens being used in proof |  
| 0x60 | 0x20 | notesOffset | uint256 | relative offset to `uint[6][] notes` |  
| 0x80 | 0x20 | inputSignaturesOffset | uint256 | relative offset to `bytes32[3][] inputSignatures` |  
| 0xa0 | 0x20 | outputOwnersOffset | uint256 | relative offset to `address[] outputOwners` |  
| 0xe0 | 0x20 | noteMetadataOffset | uint256 | relative offset to `bytes[] noteMetadata` |  
| 0x100 | L_notes | notes | uint[6][] | zero-knowledge proof data for notes |  
| 0x100 + L_notes | L_sigs | inputSignatures | bytes32[3][] | ECDSA signature data for input notes |  
| 0x100 + L_notes + L_sigs | L_owners | outputOwners | address[] | address of output note owners |  
| 0x100 + L_notes + L_sigs + L_owners | L_metadata | noteMetadata | bytes[] | note metadata, used for event broadcasts |  

**`uint[6][] notes`** contains the zero-knowledge proof data required for the set of input and output UXTO notes used inside `JoinSplit`. The ABI encoding is as follows:  

| offset | length | name | type | description |  
| --- | --- | --- | --- | --- |  
| 0x00 | 0x20 | kBar | uint256 | blinded form of the note's value |  
| 0x20 | 0x20 | aBar | uint256 | blinded form of the note's viewing key |  
| 0x40 | 0x20 | gammaX | uint256 | x-coordinate of UXTO note point 'gamma' |  
| 0x60 | 0x20 | gammaY | uint256 | y-coordinate of UXTO note point 'gamma' |  
| 0x80 | 0x20 | sigmaX | uint256 | x-coordinate of UXTO note point 'sigma' |  
| 0xa0 | 0x20 | sigmaY | uint256 | y-coordinate of UXTO note point 'sigma' |  

The amount of public 'value' being used in the join-split proof, `kPublic`, is defined as the `kBar` value of the last entry in the `uint[6][] notes` array. This value is traditionally empty (the last note does not have a `kBar` parameter) and the space is re-used to house `kPublic`.
  
# AZTEC Verifiers: BilateralSwap.sol  

# AZTEC Verifiers: DividendComputation.sol  

# AZTEC Verifiers: PublicRange.sol  

# AZTEC Verifiers: PrivateRange.sol  

# AZTEC Verifiers: Mint.sol  

# AZTEC Verifiers: Burn.sol  

# ERC1724Mintable.sol  

# ERC1724Burnable.sol  

# Specification of Utility libraries  

# Appendix 
## A: Preventing collisions and front-running  

TODO: mention that we assume 'honeset verifier' because we decide which validators are added to ACE?

For any AZTEC verification smart contract, the underlying zero-knowledge protocol must have a formal proof describing the protocol's completeness, soundness and honest-verifier zero-knowledge properties.  

In addition to this, and faithfully implementing the logic of the protocol inside a smart contract, steps must be undertaken to prevent 'proof collision', where a `bytes proofOutput` instruction from a proof has an identical structure to a `bytes proofOutput` instruction from a different smart contract verifier. This is done by integrating the `uint24 proofId` variable associated with that specific verification smart contract into the `uint256 challenge` variable contained in each `bytes proofOutput` entry.

Secondly, the front-running of proofs must be prevented. This is the act of taking a valid zero-knowledge proof that is inside the transaction pool but not yet mined, and integrating the proof into a malicious transaction for some purpose that is different to that of the transaction sender. This is achieved by integrating the message sender into challenge variable - it will not be possible for a malicious actor to modify such a proof to create a valid proof of their own construction, unless they know the secret witnesses used in the proof.  

Getting `msg.sender` into the verification contract is done by passing through this variable as an input argument from the contract that is calling `ACE.sol`. If this is not done correctly, the asset in question is susceptible to front-running. This does not expose any security risk for the protocol, as assets that correctly use ACE are not affected by assets that incorrectly implement the protocol.

## B - Interest streaming via AZTEC notes  

**concrete example** interest payments. Consider a contract that accepts a DAI note (let's call it the origination note), and issues confidential Loan notes in exchange, where the sum of the values of the loan notes is equal to the sum of the values of the origination note (this is enforced).

When a deposit of confidential DAI is supplied to the contract in the form of an interest payment (call it an interest note), a ratio is defined between the value of the interest note and the origination note.  

The AZTEC Cryptography Engine supports a zero-knowledge proof that allows loan 'note' holders to stream value out of the interest note. Effectively printing zkDAI notes whose value is defined by the above ratio and the absolute value of their loan note. In exchange, the interest note is destroyed.  

What is important to highlight in this exchange, is that the zk-DAI contract is not having to make any assumptions about the zk-Loan contract, or trust in the correctness of the zk-Loan contract's logic.  

The zero-knowledge proofs in ACE enable the above exchange to occur with a gaurantee that there is no double spending. The above mechanism cannot be used to 'print' zk-DAI notes whose sum is greater than the interest note. `NoteRegistry` and `ACE` only validate the mathematical correctness of the transaction - whether the loan notes (and resulting interest payments) are correctly distrubted according to the semantics of the loan's protocol is not relevant to ensure that there is no double spending.

# Glossary  

| term | description |
| --- | --- |
| balancing relationship | a instance of AZTEC note creation / destruction, where the sum of the values of the created notes is equal to the sum of the values of the destroyed notes |  
| mixed asset | a zero-knowledge AZTEC asset that both has a private representation (via AZTEC notes) and a public representation (via ERC20-like tokens) |  
| private asset | a zero-knowledge AZTEC asset where ownership is defined entirely through AZTEC notes and there is no linked ERC20 token. Such assets must directly create AZTEC notes via `confidentialMint` instructions |
