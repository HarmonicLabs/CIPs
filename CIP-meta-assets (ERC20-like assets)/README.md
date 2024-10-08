---
CIP: 113
Title: Programmable token-like assets
Category: Tokens
Status: Proposed
Authors:
    - Michele Nuzzi <michele.nuzzi.204@gmail.com>
    - Harmonic Laboratories <harmoniclabs@protonmail.com>
    - Matteo Coppola <m.coppola.mazzetti@gmail.com>
Implementors: []
Discussions:
    - https://github.com/cardano-foundation/CIPs/pull/444
Created: 2023-01-14
License: CC-BY-4.0
---

## Abstract
This CIP proposes a standard that if adopted would allow the same level of programmability of other ecosistems at the price of the token true ownership.

This is achieved imitating the way the account model works laveraging the UTxO structure adopted by Cardano.

## Motivation: why is this CIP necessary?

This CIP proposes a solution at the Cardano Problem Statement 3 ([CPS-0003](https://github.com/cardano-foundation/CIPs/pull/382/files?short_path=5a0dba0#diff-5a0dba075d998658d72169818d839f4c63cf105e4d6c3fe81e46b20d5fd3dc8f)).

If adopted it would allow to introduce the programmability over the transfer of tokens (meta-tokens) and their lifecycle.

The solution proposed includes (answering to the open questions of CPS-0003):

1) very much like account based models, wallets supporting this standard will require to know the address of the smart contract (validator)

2) the standard defines what is expected for normal transfer transaction, so that wallets can independently construct transactions

3) the solution can co-exist with the existing native tokens

4) the implementation is possible without hardfork since Vasil

5) optimized implementations SHOULD NOT take significant computation, especially on transfers.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
[RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

In the specification we'll use the haskell data type `Data`:
```hs
data Data
    = Constr Integer [Data]
    | Map [( Data, Data )]
    | List [Data]
    | I Integer
    | B ByteString
```

we'll use the [plu-ts syntax for structs definition](https://pluts.harmoniclabs.tech/onchain/Values/Structs/definition#pstruct) as an abstraction over the `Data` type.

finally, we'll use [`mermaid`](https://mermaid.js.org/) to help with the visualization of transactions and flow of informations.

In order to do so we introduce here a legend

### Legend

```mermaid
flowchart TD

    subgraph relations
        direction LR

        a[ ] -.-> b[ ]
        c[ ] <-.-> d[ ]

        style a fill:#FFFFFF00, stroke:#FFFFFF00;
        style b fill:#FFFFFF00, stroke:#FFFFFF00;
        style c fill:#FFFFFF00, stroke:#FFFFFF00;
        style d fill:#FFFFFF00, stroke:#FFFFFF00;
 
    end
    style relations fill:#FFFFFF00, stroke:#FFFFFF00;

    subgraph entities
        direction LR

        A[speding contract]
        pkh((pkh signer))
    end
    style entities fill:#FFFFFF00, stroke:#FFFFFF00;

    subgraph scripts
        direction LR

        mint[(minting policy)]
        obs([withdraw 0 observer])
    end
    style scripts fill:#FFFFFF00, stroke:#FFFFFF00;


    subgraph utxos
        direction LR

        subgraph input/output
            direction LR
            inStart[ ] --o inStop[ ]
        
            style inStart fill:#FFFFFF00, stroke:#FFFFFF00;
            style inStop  fill:#FFFFFF00, stroke:#FFFFFF00;
        end

        subgraph ref_input
            direction LR
            refInStart[ ]-.-orefInStop[ ]
        
            style refInStart fill:#FFFFFF00, stroke:#FFFFFF00;
            style refInStop  fill:#FFFFFF00, stroke:#FFFFFF00;
        end

        subgraph minted_asset
            direction LR
            mntInStart[ ] -.- mntInStop[ ]
        
            style mntInStart fill:#FFFFFF00, stroke:#FFFFFF00;
            style mntInStop  fill:#FFFFFF00, stroke:#FFFFFF00;
        end

        subgraph burned_asset
            direction LR
            brnInStart[ ] -.-x brnInStop[ ]
        
            style brnInStart fill:#FFFFFF00, stroke:#FFFFFF00;
            style brnInStop  fill:#FFFFFF00, stroke:#FFFFFF00;
        end

        style input/output fill:#FFFFFF00, stroke:#FFFFFF00;
        style ref_input fill:#FFFFFF00, stroke:#FFFFFF00;
        style minted_asset fill:#FFFFFF00, stroke:#FFFFFF00;
        style burned_asset fill:#FFFFFF00, stroke:#FFFFFF00;
    end
    style utxos fill:#FFFFFF00, stroke:#FFFFFF00;


    subgraph transactions
        direction LR

        subgraph Transaction
            .[ ]
            style . fill:#FFFFFF00, stroke:#FFFFFF00;
        end

        subgraph transaction
            direction LR
            start[ ] --> stop[ ]
        
            style start fill:#FFFFFF00, stroke:#FFFFFF00;
            style stop  fill:#FFFFFF00, stroke:#FFFFFF00;
        end
        style transaction fill:#FFFFFF00, stroke:#FFFFFF00;

    end
    style transactions fill:#FFFFFF00, stroke:#FFFFFF00;

    

```

### High level idea

The core idea of the implementation is to emulate the ERC20 standard; where tokens are entries in a map with addresses (or credentials in our case) as key and integers (the balances) as value. ([see the OpenZeppelin implementation for reference](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/9b3710465583284b8c4c5d2245749246bb2e0094/contracts/token/ERC20/ERC20.sol#L16));

Unlike the ERC20 standard; this CIP:

- allows for multiple entries with the same key (same credentials can be used for multiple accounts, though not particularly useful)
- DOES NOT include an equivalent of the `transferFrom` method; if needed it can be added by a specific implementation but it won't be considered part of the standard.
- allows for multiple inputs (from the same sender, scripts and multisigs MAY also be used) and multiple outputs (possibly many distinct reveivers)

> **NOTE: Multiple Inputs**
>
> the UTxO model allows for multiple transfers in one transaction
>
> this would allow for a more powerful model than the account based equivalent but implies higher execution costs
>
> with the goal of keeping the standard simple we allow only a single sender
>
> we MUST however account for multiple inputs from the same sender due to
> the creation of new UTxOs in receiving transactions.

### Design

with the introduction of [CIP-69](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0069) in Plutus V3 the number of contracts required are only 2, covering different purposes.

1) a `stateManager` contract
    - with minting policy used to proof the validity of account's state utxos
    - spending validator defining the rules to update the states
2) a `transferManager` parametrized with the `stateManager` hash
    - having minting policy for the tokens to be locked in the spending validator
    - spending validator for the validation of single utxos (often just forwarding to the withdraw 0)
    - certificate validator to allow registering the stake credentials (always succeed MAY be used) and specify the rules for de-registration (always fail MAY be used, but some logic based on the user state is RECOMMENDED)
    - withdraw validator to validate the spending of multiple utxos.


```mermaid
flowchart LR
    subgraph transferManager
        transferManagerPolicy[(transfer manager policy)]
        transferManagerContract[transfer manager]
        transferManagerObserver([transfer manager observer])

        transferManagerPolicy -. mints CNTs .-> transferManagerContract

        transferManagerContract <-. validates many inputs .-> transferManagerObserver 

        transferManagerContract -- transfer --> transferManagerContract

    end

    subgraph stateManager
        stateManagerPolicy[(state manager policy)]
        stateManagerContract[state manager]

        stateManagerPolicy -. mints validity NFTs .-> stateManagerContract

        stateManagerContract -- mutates state --> stateManagerContract
    end

    stateManager -. hash .-> transferManager
```

an external contract that needs to validate the transfer of a programmable token should be able to get all the necessary informations about the transfer by looking for the redeemer with withdraw purpose and `transferManager` stake credentials. 


### Datums and Redeemers

#### `Account`

The `Account` data type is used as the `stateManager` datum; and is defined as follows:

```ts
const Account = pstruct({
    Account: {
        credentials: PCredential.type,
        state: data
    }
});
```

The only rule when spending an utxo with `Account` datum is that the NFT present on the utxo
stays in the same contract.

The standard does not impose any rules on the redeemer to spend the utxo,
as updating the state is implementation-specific.

#### `TransferManagerDatum`

The `TransferManagerDatum` data type is used as the `transferManager` utxo datum; and is defined as follows:

```ts
const TransferManagerDatum = pstruct({
    TransferManagerDatum: {
        userStateTokenName: PTokenName.type, // alias of bytestring
        state: PMaybe( data ).type // wallet may use `Nothing` constructor
    }
});
```

the purpose of `userStateTokenName` is to prevent the user from creating a new account at the `stateManager` with same credentials and use it for this same utxo.

This makes sure only one `Account` is valid for a given utxo, even if possibly many `Account`s may exsist for the same credentials.

A specific implementation MAY additionaly implement other checks to ensure unique accounts, such as centralized actors, or merkle-patricia trees.

#### `TransferRedeemer`

redeemer to be used in the withdraw 0 contract
when spending (possibly many) utxos from the `transferManager` contract.

> NOTE:
>
> there is no standard datum for the `transferManager` since the utxos might not have a datum at all
>
>

```ts
const TransferOutput = pstruct({
    TransferOutput: {
        credential: PCredential.type,
        amount: int,
        stateIndex: int
    }
});

const TransferRedeemer = pstruct({
    Transfer: {
        totInputAmt: int,
        outputs: list( TransferOutput.type ),
        stateIndex: int
        firstOutputIndex: int,
        // inputIndicies: list( int ), // filter
    }
});
```

#### `SingleTransferRedeemer`

redeemer to be used on a single utxo of the `transferManager`;

```ts
const SingleTransferRedeemer = pstruct({
    Transfer: {},
    Burn: {},
});
```


### Transactions

#### New Account Generation Transaction

```mermaid
flowchart LR

    stateManagerPolicy[(state manager policy)]

    subgraph transaction
        .[ ]
        style . fill:#FFFFFF00, stroke:#FFFFFF00;
    end

    stateManagerContract[state manager]

    stateManagerPolicy -. validity NFTs .-> transaction 
    --o stateManagerContract
```

The purpose of the "New Account generation" transaction is to create a new account for a user and validate the initial state of an account.

The validation is performed in the minting policy.

The minting policy MUST succeed if only one asset is created under the policy respecting the naming convention explained below, and MAY succeed if other assets are created as long as their name is NOT of length 32.

The name of the asset MUST be derived from the first input's utxo reference **spent** by the minting transaction.

in particular MUST be the result of applying the serialised ([`serialiseData CIP`](https://github.com/cardano-foundation/CIPs/blob/125ac054179074d832927ec9f5ff53f46098be4a/CIP-0042/README.md) and [opinionated standard serialization in Haskell](https://github.com/IntersectMBO/plutus/blob/1f31e640e8a258185db01fa899da63f9018c0e85/plutus-core/plutus-core/src/PlutusCore/Data.hs#L108)) utxo reference according to the [standard plutus v3 onchain type definition](https://github.com/IntersectMBO/plutus/blob/ed3799004eaf2040e7f978d6ab83209c2bac8157/plutus-ledger-api/src/PlutusLedgerApi/V3/Tx.hs#L66) (where `txId` is an alias of a bytestring, and NOT a bytestring wrapped in a constr 0 like previous plutus versions) to the builtin `sha3_256`.

In textual uplc:

```uplc
(lam utxoRefData
    [
        (builtin sha3_256)
        [
            (builtin serialiseData)
            utxoRefData
        ]
    ]
)
```

**at all times** only **one (1)** NFT under `stateManager` policy MUST be present on a `stateManager` utxo.

The transaction MUST have one output going to the `stateManager` having correct `Account` **inline** datum.

by correct `Account` datum is implied the data is a constructor with index 0, first field being the credentials of the user, and second field being the state associated.

Additional checks regarding the state are left to the implementation.

#### Account State Update

```mermaid
flowchart LR

    subgraph transaction
        .[ ]
        style . fill:#FFFFFF00, stroke:#FFFFFF00;
    end

    stateManagerContract[state manager]
    stateManager[state manager]

    stateManager --o transaction 
    --o stateManagerContract
```

The purpose of this transaction is to modify the state of an account.

The only two rules regarding this transaction are:

1) The NFT present in the input MUST be preserved in the output going back to the `stateManager`
2) the credentials (first field of the datum) associated to an NFT MUST be preserved for that NFT (ie. the output having the nft has `Account` datum with first field equal to the first field of the input datum). 

#### Minting Tokens

```mermaid
flowchart LR

    subgraph transaction
        .[ ]
        style . fill:#FFFFFF00, stroke:#FFFFFF00;
    end

    stateManager[state manager]
    transferManagerPolicy[(transfer manager policy)]
    transferManagerContract[transfer manager]

    stateManager -. receiver account state .-o transaction

    transferManagerPolicy -. mints CNTs .->  transaction
    -- minted CNTs --o transferManagerContract

```

The "Minting Tokens" transaction allows to create new programmable tokens.

Programmable tokens are normal CNTs that are only present on `transferManager`
contract with same hash as their own policy.

If, by incorrect implementation of the standard, these tokens are found outside the respective transfer manager, they are no longer to be considered as programmable tokens.

The transaction MUST have an output going to the `transferManager` contract with the correct [`TransferManagerDatum`](#TransferManagerDatum) **inline** datum.
 
The first field of the datum MUST be the name of the NFT present on the reference input coming from the state manager.

Additional, implementation-specific, arbitrary logic (eg. capped supply, or one-time mints, etc. ) MAY be added.

#### Burning Tokens

```mermaid
flowchart LR

    subgraph transaction
        .[ ]
        style . fill:#FFFFFF00, stroke:#FFFFFF00;
    end

    stateManager[state manager]
    transferManagerContract[transfer manager]

    stateManager -. sender account state .-o transaction

    transferManagerContract -- Burn Redeemer --o  transaction -..-x a[ ]

    style a fill:#FFFFFF00, stroke:#FFFFFF00;
```

If an utxo in the `transferManager` is spent with `Burn` redeemer, the **entire** amount of the programmable token on that utxo MUST be burnt.

Only **one (1)** utxo from the `transferManager` is allowed to be spent in a burn transaction.

If a user desires to burn a different amount than the one currently present they should break the operation in 2 different transactions:

one `Transfer` transaction to create the utxo with that amount.

one `Burn` transaction spending the desired utxo.

Additional, implementation-specific, arbitrary logic MAY be added.

#### Transfer

```mermaid
flowchart LR
    transferManagerObserver([transfer manager observer])
    stateManagerContract[state manager]
    transferManagerContract[transfer manager]

    sender((sender))

    A[transfer manager]
    B[transfer manager]
    same[transfer manager]

    subgraph transaction
        direction LR
        .[ ]
        _[ ]
        z[ ]
        style . fill:#FFFFFF00, stroke:#FFFFFF00;
        style _ fill:#FFFFFF00, stroke:#FFFFFF00;
        style z fill:#FFFFFF00, stroke:#FFFFFF00;
    end

    transferManagerObserver -. validates inputs .-> transaction

    stateManagerContract -. sender account state .-o transaction
    stateManagerContract -. receiver A account state .-o transaction
    stateManagerContract -. receiver B account state .-o transaction

    transferManagerContract --o transaction
    transferManagerContract -- possibly --o transaction
    transferManagerContract -- many --o transaction
    transferManagerContract -- inputs --o transaction
    transferManagerContract --o transaction

    sender -. signs .-> transaction

    transaction -- stake creds A --o A
    transaction -- stake creds B --o B
    transaction -- change (if needed) --o same
```


The `Transfer` transaction allows the transfer of programmable tokens from one account to another within the `transferManager` contract. This transaction involves the following components:

- **transferManagerObserver**: The contract that validates the inputs and ensures the correctness of the transfer.
- **stateManagerContract**: The contract that manages the state of the accounts involved in the transfer.
- **transferManagerContract**: The contract that handles the actual transfer of tokens.

The purpose of this transaction is to transfer tokens from the sender's account to one or more receiver accounts. The transaction MUST ensure that the transfer is valid according to the states of each account involved if necessary.

The transaction MUST call the `transferManagerObserver` contract with the `TransferRedeemer`. External contracts should look for this redeemer to obtain information regarding the transfer. 

Additionally, any UTxOs spent from the `transferManager` contract MUST be spent with the `SingleTransferRedeemer` with `Transfer` constructor (index `0`).

The transaction MUST include:

1. One or more inputs from the sender's account in the `transferManager` with `SingleTransferRedeemer` for each UTxO spent.
2. Outputs to the receives' accounts in the `transferManager` as specified in the `TransferRedeemer` of the transfer manager observer.
3. The `TransferRedeemer` for the `transferManager` (as observer) contract.
4. one `Account` reference input for each of the parties involved (sender + receivers).

The information in the `TransferRedeemer` MUST be validated by the observer.

That includes that the sum of the programmable tokens coming from the `transferManager` is stirctly equal to `totInputAmt`.

There is one output for each element of the 2nd field (`outputs`) starting at the index specified by the 4th field (`firstOutputIndex`), in the same order.

Each of the outputs going to the `transferManager` MUST have `TransferManagerDatum` **inline** datum, and MUST make sure the first field of the datum matches the token name of the NFT present on the respective reference input `Account`.


## Rationale: how does this CIP achieve its goals?
<!-- The rationale fleshes out the specification by describing what motivated the design and what led to particular design decisions. It SHOULD describe alternate designs considered and related work. The rationale SHOULD provide evidence of consensus within the community and discuss significant objections or concerns raised during the discussion.

It MUST also explain how the proposal affects the backward compatibility of existing solutions when applicable. If the proposal responds to a CPS, the 'Rationale' section SHOULD explain how it addresses the CPS, and answer any questions that the CPS poses for potential solutions.
-->

The [first proposed implementation](https://github.com/cardano-foundation/CIPs/pull/444/commits/525ce39a89bde1ddb62e126e347828e3bf0feb58) (which we could informally refer as v0) was quite different by the one shown in this document

Main differences were in the proposed:
- [use of sorted merkle trees to prove uniqueness](https://github.com/cardano-foundation/CIPs/pull/444/commits/525ce39a89bde1ddb62e126e347828e3bf0feb58#diff-370b6563a47be474523d4f4dbfdf120c567c3c0135752afb61dc16c9a2de8d74R72) of an account during creation;
- account credentials as asset name

this path was abandoned due to the logaritmic cost of creation of accounts, on top of the complexity.

Other crucial difference with the first proposed implementation was in the `accountManager` redeemers;
which included definitions for `TransferFrom`, `Approve` and `RevokeApproval` redeemers, aiming to emulate ERC20's methods of `transferFrom` and `approve`;

after [important feedback by the community](https://github.com/cardano-foundation/CIPs/pull/444#issuecomment-1399356241), it was noted that such methods would not only have been superfluous, but also dangerous, and are hence removed in this specification.

After a first round of community feedback, a [reviewed stadard was proposed](https://github.com/cardano-foundation/CIPs/pull/444/commits/f45867d6651f94ba53503833098d550326913a0f) (which we could informally refer to as v1).
[This first revision even had a PoC implementation](https://github.com/HarmonicLabs/erc20-like/commit/0730362175a27cee7cec18386f1c368d8c29fbb8), but after further feedback from the community it was noted that the need to spend an utxo on the receiving side could cause UTxO contention in the moment two or more parties would have wanted to send a programmable token to the same receiver at the same time.

The specification proposed in this file addresses all the previous concerns.

The proposal does not affect backward compatibilty being the first proposing a standard for programmability over transfers;

exsisting native tokens are not conflicting for the standard, instead, native tokes are used in this specification for various purposes.

## Path to Active

### Acceptance Criteria
<!-- Describes what are the acceptance criteria whereby a proposal becomes 'Active' -->

- having at least one instance of the smart contracts described on:
    - mainnet
    - preview testnet
    - preprod testnet
- having at least 2 different wallets integrating meta asset functionalities, mainly:
    - displayning balance of a specified meta asset if the user provides the address of the respecive account manager contract
    - independent transaction creation with `Transfer` redeemers

### Implementation Plan
<!-- A plan to meet those criteria. Or `N/A` if NOT applicable. -->

- [ ] [PoC implementation](https://github.com/HarmonicLabs/erc20-like)
- [ ] [showcase transactions](https://github.com/HarmonicLabs/erc20-like)
- [ ] wallet implementation 

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
