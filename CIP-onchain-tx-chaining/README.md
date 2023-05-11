---
CIP: _???
Title: onchain tx chaining
Category: Ledger | Plutus
Status: Draft
Authors:
    - Michele Nuzzi <michele.nuzzi.2014@gmail.com>
Implementors: []
Discussions:
    - https://github.com/cardano-foundation/cips/pulls/?
Created: 2023-04-13
License: CC-BY-4.0
---

<!-- Existing categories:

- Meta                   | For meta-CIPs which typically serves another category or group of categories.
- Reward-Sharing Schemes | For CIPs discussing the reward & incentive mechanisms of the protocol.
- Wallets                | For standardisation across wallets (hardware, full-node or light).
- Tokens                 | About tokens (fungible or non-fungible) and minting policies in general.
- Metadata               | For proposals around metadata (on-chain or off-chain).
- Tools                  | A broad category for ecosystem tools not falling into any other category.
- Plutus                 | Changes or additions to Plutus
- Ledger                 | For proposals regarding the Cardano ledger
- Catalyst               | For proposals affecting Project Catalyst / the Jörmungandr project

-->

## Abstract
<!-- A short (\~200 word) description of the proposed solution and the technical issue being addressed. -->

Thanks to the transaction determinism in Cardano, it is possible to chain transactions so that a new transaction can be created using transaction outputs of transactions that have not been included in the blockchain yet.

Despite being extremly powerful this process as currently some limitations:
- it gives no guarantees that all of the multiple transactions are valid
- the process happens entrierly offchain; without the possibility for a plutus script to be aware of it

## Motivation: why is this CIP necessary?
<!-- A clear explanation that introduces the reason for a proposal, its use cases and stakeholders. If the CIP changes an established design then it must outline design issues that motivate a rework. For complex proposals, authors must write a Cardano Problem Statement (CPS) as defined in CIP-9999 and link to it as the `Motivation`. -->

The changes proposed in this CIP, if implemented, would allow for composability between transactions while preserving their deterministic behaviour.

The main stakeholders would be dApp developers; who would have access to the transaction chaining functionality also on-chain in plutus scripts, on top of the off-chain guarantee of the transactions validity.

## Specification
<!-- The technical specification should describe the proposed improvement in sufficient technical detail. In particular, it should provide enough information that an implementation can be performed solely on the basis of the design in the CIP. This is necessary to facilitate multiple, interoperable implementations. -->

The core idea of the CIP is to add a new field in the transaction body representing the hash of the chained transaction (the follow-up transaction whose inputs are the current transaction's outputs).

Despite being a simple idea it introduces a circular dependency of hashes.

In fact, the current transaction hash is used to describe the input used in the chained transaction;
but the input of the chained transaction will determine the hash of the chained transaction;
and the hash of the chained transaction would be included in the previous transaction body;
and this would modify the hash of the initial transaction.

(`current_tx_id` -> `utxo_input` -> `next_tx_id` -> `current_tx_id` )

To break the circular dependecy we can modify the definition of a valid transaction input so that it doesn't require a transaciton hash if it is an output of a previus transaction.

The only information needed for a chained input is the which previous transaction generated it and the output index.

In particular the current `transacion_input` definition is

```cddl
transaction_input = [ transaction_id : $hash32
                    , index : uint
                    ]
```

We can define a new type of transaction input that here we call `chained_tx_input`

```cddl
chained_tx_input = [ prev_tx_pointer: uint
                   , index : uint
                   ]
```

so that we can introduce a new `chainable_transaction_input` cddl definition

```cddl
chainable_transaction_input = chained_tx_input / transaction_input
```

We can distinguish the two types of inputs based on the type of the first element of the array;

if it is an unsigned integer it should be interpreted as a pointer to a previous transaction (`chained_tx_input` first element)
(`0` being the transaction immediately before the current chained one)

if otherwhise the first element of the array are bytes then it should be interpreted as the `transaciton_input` first element

the final `transaction_body` cddl (taking in consideration the conaway modifications) would then become:

```cddl
; up to babbage only transaction_input
transaction_input = [ transaction_id : $hash32
                    , index : uint
                    ]

; new type of transaction input
chained_tx_input = [ prev_tx_pointer: uint
                   , index : uint
                   ]

; all possible transaction input types
chainable_transaction_input = chained_tx_input / transaction_input

transaction_body =
  { 0 : set<chainable_transaction_input>      ; This CIP; modified inputs field
  , 1 : [* transaction_output]
  , 2 : coin                        ; fee
  , ? 3 : uint                      ; time to live
  , ? 4 : [* certificate]
  , ? 5 : withdrawals
  , ? 7 : auxiliary_data_hash
  , ? 8 : uint                      ; validity interval start
  , ? 9 : mint
  , ? 11 : script_data_hash
  , ? 13 : set<transaction_input>   ; collateral inputs
  , ? 14 : required_signers
  , ? 15 : network_id
  , ? 16 : transaction_output       ; collateral return
  , ? 17 : coin                     ; total collateral
  , ? 18 : set<transaction_input>   ; reference inputs
  , ? 19 : [* voting_procedure]     ; New; Voting procedures
  , ? 20 : [* proposal_procedure]   ; New; Proposal procedures
  , ? 21 : $hash32                  ; This CIP; Chained transaction_id
  }
```

And also a modification to the [`ScriptContext`](https://github.com/input-output-hk/plutus/blob/c3918d6027a9a34b6f72a6e4c7bf2e5350e6467e/plutus-ledger-api/src/PlutusLedgerApi/V2/Contexts.hs#L72)  would be needed, introducing the `txInfoChainedTxId` field of type `Maybe TxId`.

```hs
data TxInfo = TxInfo
    { txInfoInputs          :: [TxInInfo] -- ^ Transaction inputs; cannot be an empty list
    , txInfoReferenceInputs :: [TxInInfo] -- ^ /Added in V2:/ Transaction reference inputs
    , txInfoOutputs         :: [TxOut] -- ^ Transaction outputs
    , txInfoFee             :: Value -- ^ The fee paid by this transaction.
    , txInfoMint            :: Value -- ^ The 'Value' minted by this transaction.
    , txInfoDCert           :: [DCert] -- ^ Digests of certificates included in this transaction
    , txInfoWdrl            :: Map StakingCredential Integer -- ^ Withdrawals
                                                      -- /V1->V2/: changed from assoc list to a 'PlutusTx.AssocMap'
    , txInfoValidRange      :: POSIXTimeRange -- ^ The valid range for the transaction.
    , txInfoSignatories     :: [PubKeyHash] -- ^ Signatures provided with the transaction, attested that they all signed the tx
    , txInfoRedeemers       :: Map ScriptPurpose Redeemer -- ^ /Added in V2:/ a table of redeemers attached to the transaction
    , txInfoData            :: Map DatumHash Datum -- ^ The lookup table of datums attached to the transaction
                                                  -- /V1->V2/: changed from assoc list to a 'PlutusTx.AssocMap'
    , txInfoId              :: TxId  -- ^ Hash of the pending transaction body (i.e. transaction excluding witnesses)
    , txInfoChainedTxId     :: Maybe TxId  -- This CIP; hash of the chained transaction based on this transaction outputs
    }
```

It's important to note that unlike the cddl specification, the `ScriptContext` `txInfoInputs` field doesn't need to change since the creation of the script context is only needed at phase 2 validation and the transaction hash of the prevous transaction is fully defined.

## Rationale: how does this CIP achieve its goals?
<!-- The rationale fleshes out the specification by describing what motivated the design and what led to particular design decisions. It should describe alternate designs considered and related work. The rationale should provide evidence of consensus within the community and discuss significant objections or concerns raised during the discussion.

It must also explain how the proposal affects the backward compatibility of existing solutions when applicable. If the proposal responds to a CPS, the 'Rationale' section should explain how it addresses the CPS, and answer any questions that the CPS poses for potential solutions.
-->

The proposed changes allow to enforce the transaction validation process to fail in the event one or more of the chained transaction fails too.

In particular the Phase-1 validation can check for the existence of another transaction with the specified hash in the local mempool (or wait if none is present)

Phase-1 validation can also fail if some transaction input is defined using only an unsigned integer but no other transaction specifies the current transaction as chained.

DApp developers can check on-chain for eventual chained transaction by exposing the `txInfoChainedTxId` `ScriptContext` field in the output datum going to be used in the chained transaction.

e.g.:
```ts
const MyDatum = pstruct({
    MyDatum: {
        chainedTx: PTxId.type
    }
});
```

and then check the `chainedTx` in the example above to equal the `txInfoId` field of the `ScriptContext` of the chained transaction.

## Path to Active

The suggested changes would have to be implemented in a new version of the `cardano-node`

### Acceptance Criteria
<!-- Describes what are the acceptance criteria whereby a proposal becomes 'Active' -->

A Hard Fork enables the suggested changes.

### Implementation Plan
<!-- A plan to meet those criteria. Or `N/A` if not applicable. -->

N/A

## Copyright
<!-- The CIP must be explicitly licensed under acceptable copyright terms. -->

[CC-BY-4.0]: https://creativecommons.org/licenses/by/4.0/legalcode
[Apache-2.0]: http://www.apache.org/licenses/LICENSE-2.0
