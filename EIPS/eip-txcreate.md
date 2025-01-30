---
title: EOF - `TXCREATE` instruction and `InitcodeTransaction` type
description: Adds a `TXCREATE` instruction to the EVM Object Format (EOF) and an accompanying transaction type allowing to create EOF contracts from transaction data
author: Piotr Dobaczewski (@pdobacz), Andrei Maiboroda (@gumb0), Paweł Bylica (@chfast), Alex Beregszaszi (@axic), Danno Ferrin (@shemnon)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2025-01-28
requires: 3540, 3670, 3860, 7620
---

## Abstract

EVM Object Format (EOF) removes the possibility to create contracts using creation transactions (with an empty `to` field), `CREATE` or `CREATE2` instructions. We introduce a new instruction: `TXCREATE`, to complement `EOFCREATE` and `RETURNCONTRACT` from [EIP-7620](./eip-7620.md), as well as a new transaction type (`InitcodeTransaction`) to provide a way to create contracts using EOF containers.

## Motivation

This EIP uses terminology from the [EIP-3540](./eip-3540.md) which introduces the EOF format.

Creation transaction and creation instructions `CREATE` and `CREATE2` are means provided by legacy EVM to deploy new code. Given that legacy creation instructions (`CREATE` and `CREATE2`) are not allowed to deploy EOF code, as per requirement of removing code observability in EOF, there must be a way to create EOF contracts using bytecode delivered in transaction data.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Parameters

| Constant | Value |
| - | - |
| `INITCODE_TX_TYPE` | `Bytes1(0x05)` |
| `MAX_INITCODE_COUNT` | `256` |
| `CREATOR_CONTRACT_ADDRESS` | tbd |
| `CREATOR_CONTRACT_BYTECODE` | tbd |
| `TX_CREATE_COST` | Defined as `32000` in the [Ethereum Execution Layer Specs](https://github.com/ethereum/execution-specs/blob/0f9e4345b60d36c23fffaa69f70cf9cdb975f4ba/src/ethereum/shanghai/fork_types.py#L42) |
| `STACK_DEPTH_LIMIT` | Defined as `1024` in the [Ethereum Execution Layer Specs](https://github.com/ethereum/execution-specs/blob/0f9e4345b60d36c23fffaa69f70cf9cdb975f4ba/src/ethereum/shanghai/vm/interpreter.py#L60) |
| `GAS_CODE_DEPOSIT` | Defined as `200` in the [Ethereum Execution Layer Specs](https://github.com/ethereum/execution-specs/blob/0f9e4345b60d36c23fffaa69f70cf9cdb975f4ba/src/ethereum/shanghai/vm/gas.py#L44) |
| `TX_DATA_COST_PER_ZERO` | Defined as `4` in the [Ethereum Execution Layer Specs](https://github.com/ethereum/execution-specs/blob/0f9e4345b60d36c23fffaa69f70cf9cdb975f4ba/src/ethereum/shanghai/fork_types.py#L41) |
| `TX_DATA_COST_PER_NON_ZERO` | Defined as `16` in the [Ethereum Execution Layer Specs](https://github.com/ethereum/execution-specs/blob/0f9e4345b60d36c23fffaa69f70cf9cdb975f4ba/src/ethereum/shanghai/fork_types.py#L40) |
| `INITCODE_WORD_COST` | Defined as `2` in [EIP-3860](./eip-3860.md) |
| `MAX_INITCODE_SIZE` | Defined as `2 * MAX_CODE_SIZE` in [EIP-3860](./eip-3860.md) |
| `MAX_CODE_SIZE` | Defined as `24576` in [EIP-170](./eip-170.md) |

### Transaction Types

Introduce new transaction `InitcodeTransaction` (type `INITCODE_TX_TYPE`) which extends [EIP-1559](./eip-1559.md) (type 2) transaction by adding a new field `initcodes: List[ByteList[MAX_INITCODE_SIZE], MAX_INITCODE_COUNT]`.

The `initcodes` can only be accessed via the `TXCREATE` instruction (see below), therefore `InitcodeTransactions` are intended to be sent to contracts including `TXCREATE` in their execution.

We introduce a standardised Creator Contract, which eliminates the need to have create transactions with empty `to`. See [Creator Contract](#creator-contract) for details.

#### Gas schedule

<!-- TODO: sync gas schedule with EIP-7623 -->

Each `initcodes` item data costs the same as calldata (`TX_DATA_COST_PER_NON_ZERO` gas for non-zero bytes, `TX_DATA_COST_PER_ZERO` for zero bytes -- see [EIP-2028](./eip-2028.md)). The intrinsic gas of an `InitcodeTransaction` is extended by the sum of all those items' costs. Using the conventions from [`calculate_intrinsic_cost` in Ethereum Execution Layer Specs](https://github.com/ethereum/execution-specs/blob/0f9e4345b60d36c23fffaa69f70cf9cdb975f4ba/src/ethereum/shanghai/fork.py#L687), the additional cost is calculated as:

```
initcode_cost = 0
    for initcode in tx.initcodes:
        for byte in initcode:
            if byte == 0:
                initcode_cost += TX_DATA_COST_PER_ZERO
            else:
                initcode_cost += TX_DATA_COST_PER_NON_ZERO
```

#### Transaction validation

- `InitcodeTransaction` is invalid if there are zero entries in `initcodes`, or if there are more than `MAX_INITCODE_COUNT` entries.
- `InitcodeTransaction` is invalid if any entry in `initcodes` is zero length, or if any entry exceeds `MAX_INITCODE_SIZE`.
- `InitcodeTransaction` is invalid if the `to` is `nil`.

Under transaction validation rules `initcodes` are not validated for conforming to the EOF specification. They are only validated when accessed via `TXCREATE`. This avoids potential DoS attacks of the mempool. If during the execution of an `InitcodeTransaction` no `TXCREATE` instruction is called, such transaction is still valid.

#### RLP and signature

Given the definitions from [EIP-2718](./eip-2718.md) the `TransactionPayload` for an `InitcodeTransaction` is the RLP serialization of:

```
[chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, to, value, data, access_list, initcodes, y_parity, r, s]
```

`TransactionType` is `INITCODE_TX_TYPE` and the signature values `y_parity`, `r`, and `s` are calculated by constructing a secp256k1 signature over the following digest:

```
keccak256(INITCODE_TX_TYPE || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, to, value, data, access_list, initcodes]))
```

The [EIP-2718](./eip-2718.md) `ReceiptPayload` for this transaction is `rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])`.

### Execution Semantics
    
Introduce a new instruction on the same block number [EIP-3540](./eip-3540.md) is activated on: `TXCREATE` (`0xed`).
    
Legacy creation transactions (any transactions with empty `to`) are invalid in case `data` contains EOF code (starts with `EF00` magic).

#### `TXCREATE`
    
- deduct `TX_CREATE_COST` gas
- halt with exceptional failure if the current frame is in `static-mode`.
- pop `tx_initcode_hash`, `value`, `salt`, `input_offset`, `input_size` from the operand stack
- peform (and charge for) memory expansion using `[input_offset, input_size]`
- load initcode EOF container from the transaction `initcodes` array which hashes to `tx_initcode_hash`
    - fails (returns 0 on the stack) if such initcode does not exist in the transaction, or if called from a transaction of `TransactionType` other than `INITCODE_TX_TYPE`
        - caller's nonce is not updated and gas for initcode execution is not consumed. Only `TXCREATE` constant gas was consumed
    - let `initcontainer` be that EOF container, and `initcontainer_size` its length in bytes
- deduct `INITCODE_WORD_COST * ((initcontainer_size + 31) // 32)` gas ([EIP-3860](./eip-3860.md) charge)
- check that current call depth is below `STACK_DEPTH_LIMIT` and that caller balance is enough to transfer `value`
    - in case of failure return 0 on the stack, caller's nonce is not updated and gas for initcode execution is not consumed.
- **validate the initcode container and all its subcontainers recursively**
    - unlike in general validation, `initcontainer` is additionally required to have `data_size` declared in the header equal to actual `data_section` size.
    - validation includes checking that the `initcontainer` does not contain `RETURN` or `STOP`
- fails (returns 0 on the stack) if container was invalid
    - caller’s nonce is not updated and gas for initcode execution is not consumed. Only `TX_CREATE_COST` constant and [EIP-3860](./eip-3860.md) gas were consumed
- caller's memory slice `[input_offset:input_size]` is used as calldata
- execute the container and deduct gas for execution. The 63/64th rule from [EIP-150](./eip-150.md) applies.
- increment `sender` account's nonce
- calculate `new_address` as `keccak256(0xff || sender || salt)[12:]`
- behavior on `accessed_addresses` and address collision is same as `CREATE2` (rules for `CREATE2` from [EIP-684](./eip-684.md) and [EIP-2929](./eip-2929.md) apply to `TXCREATE`)
- an unsuccessful execution of initcode results in pushing `0` onto the stack
    - can populate returndata if execution `REVERT`ed
- a successful execution ends with initcode executing `RETURNCONTRACT{deploy_container_index}(aux_data_offset, aux_data_size)` instruction (see below). After that:
    - load deploy EOF subcontainer at `deploy_container_index` in the container from which `RETURNCONTRACT` is executed
    - concatenate data section with `(aux_data_offset, aux_data_offset + aux_data_size)` memory segment and update data size in the header
    - if updated deploy container size exceeds `MAX_CODE_SIZE` instruction exceptionally aborts
    - set `state[new_address].code` to the updated deploy container
    - push `new_address` onto the stack
- deduct `GAS_CODE_DEPOSIT * deployed_code_size` gas

### Creator Contract

At the start of the block in which [EIP-3540](./eip-3540.md) activates, set the code of `CREATOR_CONTRACT_ADDRESS` to `CREATOR_CONTRACT_BYTECODE`. `CREATOR_CONTRACT_ADDRESS` is not treated like a precompile. `CREATOR_CONTRACT_BYTECODE` corresponds to the following source code compiled to EOF:

```solidity
{
    /// Takes [tx_initcode_hash][salt][init_data] as input,
    /// creates contract and returns the address or failure otherwise

    // init_data.length can be 0, but the first 2 words are mandatory
    let size := calldatasize()
    if lt(size, 64) { revert(0, 0) }

    let tx_initcode_hash := calldataload(0)
    let salt := calldataload(32)

    mstore8(0, 0xff)  // a magic value to ensure a specific preimage space
    calldatacopy(1, 0, 64)  // copy tx_initcode_hash and salt to memory to hash
    let final_salt := keccak256(0, 65)

    let init_data_size := sub(size, 64)
    calldatacopy(0, 64, init_data_size)

    let ret := txcreate(tx_initcode_hash, callvalue(), final_salt, 0, init_data_size)
    if iszero(ret) { revert(0, 0) }

    mstore(0, ret)
    return(0, 32)

    // Helper to compile this with existing Solidity (with --strict-assembly mode)
    function txcreate(a, b, c, d, e) -> f {
        f := verbatim_5i_1o(hex"ed", a, b, c, d, e)
    }
}
```

## Rationale

### `TXCREATE` failure modes
    
`TXCREATE` has two "light" failure modes in case the initcontainer is not present and in case the EOF validation is unsuccessful. An alternative design where both cases led to a "hard" failure (consuming the entire gas available) was considered. We decided to have the more granular and forgiving failure modes in order to align the gas costs incurred to the actual work the EVM performs.

### Creator Contract

EOF contract creation requires a predeployed Creator Contract be introduced, because neither legacy contracts nor create transactions can deploy EOF code to bootstrap. The alternative approach was to continue using legacy creation mechanisms, by either still relying on fetching the *initcode* from memory and not satisfy the overarching requirement of code non-observability or abuse the legacy creation transactions (ones with empty `to`) mechanisms.

Since previous discussions about `TXCREATE`, which led to its removal from EOF (then in [EIP-7620](./eip-7620.md)) along with the Creator Contract, there is wider consensus to allow predeploy contracts be put into the state, for example as means to reduce the number of supported precompiles. Meanwhile, opinions about the importance of supporting deployments to deterministic, counterfactual, cross-chain addresses have been voiced by the community, making the case for this proposal stronger.

### `keccak256(initcontainer)` in the `new_address` hashing scheme

`new_address = keccak256(0xff || sender || salt || keccak256(initcontainer))[12:]` was originally proposed as the way to calculate the address of newly created contract, similar, but not exactly equal, to what legacy EVM's `CREATE2` does.

This alternative however goes against the "code non-observability" because it locks in the contents of the initcontainer e.g. preventing re-writing it in some future upgrade. It also seems unnecessarily expensive: `EOFCREATE` can only pick up one of its subcontainers, yet the hash value would need to be recalculated on every execution of `EOFCREATE`.

Other ways of removing code observability, yet keeping some form of witness of the code, were considered. However, keeping only `sender` and `salt` allows the implementer of the factory contract (i.e. one containing the `EOFCREATE` instruction) to include the code witness via the `salt` anyway, if that's necessary for the particular use case. Therefore, keeping the `new_address` formula minimal is the most flexible approach with better separation of concerns.

Leaving the `keccak256(initcontainer)` out of the `new_address` hash has also the benefit of making the `new_address` independent of the metadata section (proposed for the EOF in a separate EIP), which is a desired property. Unfortunately, if a factory wants to opt into committing to a particular `initcontainer`, it needs to include it in the `salt`, and that will include the metadata section.

### EOF creation transactions vs deployment patterns

Relying on the alternative solution being, the EOF creation transactions, makes it impossible for smart contract wallets to deploy arbitrary EOF contracts (only Externally Owned Accounts (EOAs) can). At the same time, is a use case current legacy creation rules allow for, thanks to `CREATE` and `CREATE2` instructions. A workaround where those arbitrary EOF contracts are first "uploaded" to a factory contract, and then deployed using an `EXTDELEGATECALL`-`EOFCREATE` sequence, is very expensive, as it requires the deployed contract to be put on-chain twice. Because of this, the approach proposed in this EIP is more compatible with the Account Abstraction (AA) roadmap, where smart contract wallets should have feature parity with EOAs.

On top of this, relying on nonce-based hashing scheme to obtain addresses of newly created contracts, like in the case of the EOF creation transactions, would prevent EOF contracts from being deployed counterfactually to deterministic, cross-chain addresses. Introduction of the `TXCREATE` instruction, combined with the proposed "toe-hold" Creator Contract predeployed to an agreed upon address, supports this out of the box.

## Backwards Compatibility

This change poses no risk to backwards compatibility, as it is introduced at the same time EIP-3540 is. The new instruction are not introduced for legacy bytecode (code which is not EOF formatted), and the contract creation options do not change for legacy bytecode. The transactions bearing the new transaction type were invlid until this change activates.

## Security Considerations

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
