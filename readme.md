# CHIP-2021-05-vm-limits: Targeted Virtual Machine Limits

        Title: Targeted Virtual Machine Limits
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2021-05-12
        Latest Revision Date: 2024-08-06
        Version: 3.0.0

## Summary

This proposal replaces several poorly-targeted virtual machine (VM) limits with alternatives that protect against the same malicious cases, while significantly increasing the power of the Bitcoin Cash contract system:

- The 201 operation limit is removed.
- The 520-byte stack element length limit is raised to 10,000 bytes, a constant equal to the consensus-maximum VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE`) prior to this proposal.
- A new operation cost limit is introduced, limiting contracts to approximately 700 bytes pushed to the stack per spending transaction byte, with increased costs for hashing, signature checking, and expensive arithmetic operations; constants are derived from the effective limit(s) prior to this proposal.
- A new hashing limit is introduced, limiting contracts to 3.5 hash digest iterations per spending transaction byte, a constant rounded up from the effective standard limit prior to this proposal, and further limiting standard contracts to 0.5 hash digest iterations per spending transaction byte, the maximum-known, theoretically-useful hashing density.
- A new control stack limit is introduced, limiting control flow operations (`OP_IF` and `OP_NOTIF`) to a depth of 100, the effective limit prior to this proposal.

This proposal **intentionally avoids modifying other existing properties of the VM**:

- The limits on signature operation count (A.K.A. `SigChecks`) density, maximum standard input bytecode length (A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE` – 1,650 bytes), consensus-maximum VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE` – 10,000 bytes), maximum standard transaction byte-length (A.K.A. `MAX_STANDARD_TX_SIZE` – 100,000 bytes) and consensus-maximum transaction byte-length (A.K.A. `MAX_TX_SIZE` – 1,000,000 bytes) are not modified.
- The cost and incentives around blockchain “data storage” are not measurably affected.
- The worst-case transaction validation processing and memory requirements of the VM are not measurably affected.

## Deployment

Deployment of this specification is proposed for the May 2025 upgrade.

- Activation is proposed for `1731672000` MTP, (`2024-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1747310400` MTP, (`2025-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Motivation

Bitcoin Cash contracts are strictly limited to prevent maliciously-designed transactions from requiring excessive resources during transaction validation. Two of these limits are poorly-targeted, unintentionally preventing valuable use cases.

- The **520-byte stack element length limit** (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) currently prevents items longer than 520 bytes from being pushed, `OP_NUM2BIN`ed, or `OP_CAT`ed on to the stack. This also limits the length of Pay-To-Script-Hash (P2SH) redeem bytecode to 520 bytes.
- The **201 operation limit** prevents contracts with more than 201 non-push operations from being relayed on the network or mined in blocks.

Collectively, existing limits:

- Cap the maximum density of both hashing and elliptic curve math required for validation of signature checking operations. (See the [2020-05 SigChecks specification](https://gitlab.com/bitcoin-cash-node/bchn-sw/bitcoincash-upgrade-specifications/-/blob/master/spec/2020-05-15-sigchecks.md).)
- Cap the maximum density of hashing required in transaction validation to approximately `3.44` hash digest iterations per standard spending transaction byte<sup>1</sup>.
- Cap the maximum stack usage of transaction validation to approximately `628` bytes per spending transaction byte<sup>2</sup>.
- Cap the depth of the control stack to `100`<sup>3</sup>.

While these limits have been sufficient to prevent Denial of Service (DOS) attacks by capping the maximum cost of transaction validation, their current design prevents valuable contract use cases and wastefully increases the size of certain transactions.

<details>
<summary>Notes</summary>

1. Benchmark `lcennk`.
2. Benchmark `d0kxwp`.
3. The existing `201` operation limit prevents any currently-valid contract from requiring a control stack of depth greater than `100`, e.g. `<1> OP_IF <1> OP_IF <1> OP_IF ... OP_ENDIF OP_ENDIF OP_ENDIF`.

</details>

## Benefits

By replacing these limits with better-tuned alternatives, the Bitcoin Cash contract system can be made more powerful and efficient, without sacrificing node validation performance.

### More Advanced Contracts

The 520-byte stack element length limit prevents additional use cases by limiting the length of Pay-To-Script-Hash (P2SH) contracts, as P2SH redeem bytecode is pushed to the stack prior to evaluation. Additionally, the element length limit also prevents the use of larger hash preimages, e.g. in contracts which inspect parent transactions or utilize `OP_CHECKDATASIG`.

By raising this limit, more advanced contracts can be supported.

### More Efficient Contracts

The 520-byte stack element length limit sometimes requires contract authors to design less byte-efficient contracts in order to fit contract code into 520 bytes. For example, rather than embedding data elements directly in P2SH redeem bytecode, authors may be required to pick and validate data from the unlocking bytecode, wasting transaction space.

Likewise, both the stack element length limit and the 201 operation limit sometimes require contract authors to offload validation to additional outputs, increasing transaction sizes with otherwise-unnecessary overhead.

With better-targeted VM limits, many contracts can be made more efficient.

## Technical Specification

The existing **`Stack Element Size Limit`** is raised; two new limits are introduced: a **`Control Stack Limit`** and a **`Hashing Limit`**; and the **`201 Operation Limit`** is replaced by an **`Operation Cost Limit`**.

### Increased Stack Element Length Limit

The existing 520-byte stack element size limit (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) is raised to 10,000 bytes.

<details>
<summary>Note on Maximum Standard Input Bytecode Length</summary>

This increases the maximum size of stack elements to be equal to the maximum allowed VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE` – 10,000 bytes). To narrow the scope of this proposal, the maximum standard input bytecode length (A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE` – 1,650 bytes) is unchanged.

</details>

### Control Stack Limit

To retain the existing limit of `100` on control stack depth, a direct limit is placed on operations which push to the control stack (A.K.A. `vfExec` or `ConditionStack`). See [Rationale: Retention of Control Stack Limit](#retention-of-control-stack-limit).

Currently, this limit impacts only the `OP_IF` and `OP_NOTIF` operations. For the purpose of enforcing this limit, `OP_IFDUP` is not considered to internally push to the control stack, i.e. an `OP_IFDUP` executed at a depth of `100` does not violate the `Control Stack Limit`.

### Hashing Limit

A new limit is placed on `OP_RIPEMD160` (`0xa6`), `OP_SHA1` (`0xa7`), `OP_SHA256` (`0xa8`), `OP_HASH160` (`0xa9`), `OP_HASH256` (`0xaa`), `OP_CHECKSIG` (`0xac`), `OP_CHECKSIGVERIFY` (`0xad`), `OP_CHECKMULTISIG` (`0xae`), `OP_CHECKMULTISIGVERIFY` (`0xaf`), `OP_CHECKDATASIG` (`0xba`), and `OP_CHECKDATASIGVERIFY` (`0xbb`) to prevent excessive hashing function usage.

Before a hashing function is performed, its expected cost – in terms of digest iterations – is added to a cumulative total for the transaction. If the cumulative digest iterations required to validate the spending transaction exceed the maximum allowed density, the operation produces an error. See [Rationale: Hashing Limit by Digest Iterations](#hashing-limit-by-digest-iterations).

#### Maximum Hashing Density

For standard transactions, the maximum density is `0.5` hash digest iterations per spending transaction byte; for block validation, the maximum density is `3.5` hash digest iterations per spending transaction byte. See [Rationale: Selection of Hashing Limit](#selection-of-hashing-limit).

Given a spending transaction byte length, hash digest iteration limits may be calculated with the following C function:

```c
int max_standard_iterations (int transaction_byte_length) {
    return transaction_byte_length / 2;
}
int max_consensus_iterations (int transaction_byte_length) {
    return (transaction_byte_length * 7) / 2;
}
```

<details>
<summary>Calculate Maximum Digest Iterations in JavaScript</summary>

```js
const maxStandardIterations = (transactionByteLength) =>
  Math.floor(transactionByteLength / 2);
const maxConsensusIterations = (transactionByteLength) =>
  Math.floor((transactionByteLength * 7) / 2);
```

</details>

#### Digest Iteration Count

The hashing limit caps the number of iterations required by all hashing functions over the course of verifying a transaction. This places an upper limit on the sum of bytes hashed, including padding.

Given a message length, digest iterations may be calculated with the following C function:

```c
int digest_iterations (int message_length) {
  return 1 + ((message_length + 8) / 64);
}
```

<details>
<summary>Calculate Digest Iterations in JavaScript</summary>

```js
const digestIterations = (messageLength) =>
  1 + Math.floor((messageLength + 8) / 64);
```

</details>

<details>
<summary>Digest Iteration Count Test Vectors</summary>

| Message Length (Bytes) | Digest Iterations |
| ---------------------- | ----------------- |
| 0                      | 1                 |
| 1                      | 1                 |
| 55                     | 1                 |
| 56                     | 2                 |
| 64                     | 2                 |
| 119                    | 2                 |
| 120                    | 3                 |
| 183                    | 3                 |
| 184                    | 4                 |
| 247                    | 5                 |
| 248                    | 5                 |
| 488                    | 8                 |
| 503                    | 8                 |
| 504                    | 9                 |
| 520                    | 9                 |
| 1015                   | 16                |
| 1016                   | 17                |
| 63928                  | 1000              |
| 63991                  | 1000              |
| 63992                  | 1001              |

</details>

<details>
<summary>Note on 64-Byte Message Block Size</summary>

Each VM-supported hashing algorithm – RIPEMD-160, SHA-1, and SHA-256 – uses a [Merkle–Damgård construction](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) with a `block size` of 512 bits (64 bytes), so the number of message blocks/digest iterations required for every message size is equal among all VM-supported hashing functions.

</details>

#### Hashing Operations

The `OP_RIPEMD160` (`0xa6`), `OP_SHA1` (`0xa7`), `OP_SHA256` (`0xa8`), `OP_HASH160` (`0xa9`), and `OP_HASH256` (`0xaa`) operations must compute the expected digest iterations for the length of the message to be hashed, adding the result to the spending transaction's cumulative count. If the new total exceeds the limit, validation fails.

Note that evaluations triggering `P2SH20` and `P2SH32` evaluations must also account for the `3` hash digest iterations required to test validity of the redeem bytecode (`OP_HASH160 <20_bytes> OP_EQUAL` or `OP_HASH256 <32_bytes> OP_EQUAL`, respectively).

<details>
<summary>Note on Two-Round Hashing Operations</summary>

The two-round hashing operations – `OP_HASH160` (`0xa9`) and `OP_HASH256` (`0xaa`) – pass the 32-byte result of their initial SHA-256 hashing round into their second round, each requiring one additional digest iteration beyond the single-round `OP_SHA256`.

</details>

#### Transaction Signature Checking Operations

Following the assembly of the signing serialization, the `OP_CHECKSIG` (`0xac`), `OP_CHECKSIGVERIFY` (`0xad`), `OP_CHECKMULTISIG` (`0xae`), and `OP_CHECKMULTISIGVERIFY` (`0xaf`) operations must compute the expected digest iterations for the length of the signing serialization (including the iteration in which the final result is double-hashed), adding the result to the spending transaction's cumulative count. If the new total exceeds the limit, validation fails.

Note that hash digest iterations required to produce components of the signing serialization (i.e. `hashPrevouts`, `hashUtxos`, `hashSequence`, and `hashOutputs`) are excluded from the hashing limit, as implementations should cache these components across signature checks. See [Rationale: Exclusion of Signing Serialization Components from Hashing Limit](#exclusion-of-signing-serialization-components-from-hashing-limit).

#### Data Signature Checking Operations

The `OP_CHECKDATASIG` (`0xba`) and `OP_CHECKDATASIGVERIFY` (`0xbb`) operations must compute the expected digest iterations for the length of the message to be hashed, adding the result to the spending transaction's cumulative count. If the new total exceeds the limit, validation fails.

### Replacement of Operation Limit

The existing 201-operation limit per evaluation (A.K.A. `MAX_OPS_PER_SCRIPT`) is removed and replaced by the `Operation Cost Limit`.

<details>
<summary>Note on Zero-Signature, Multisignature Operations</summary>

Note that prior to this proposal, the `OP_CHECKMULTISIG` (`0xae`) and `OP_CHECKMULTISIGVERIFY` (`0xaf`) operations incremented the operation count by the count of public keys, even if zero signatures are checked; because the total operation count of any contract evaluation was limited to 201 operations, the overall density of `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` was very limited.

Following the removal of the operation limit, both `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` are limited only by the `SigChecks` and hashing limits; implementations should ensure that evaluations of `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` requiring zero signature checks are sufficiently performant. See [Rationale: Increased Usability of Multisig Stack Clearing](#increased-usability-of-multisig-stack-clearing).

</details>

### Operation Cost Limit

An `Operation Cost Limit` is introduced, limiting transactions to a cumulative operation cost of `800` per spending transaction byte. See [Rationale: Selection of Operation Cost Limit](#selection-of-operation-cost-limit).

For each evaluated instruction (including push operations), operation cost is incremented by `100`. See [Rationale: Selection of Base Instruction Cost](#selection-of-base-instruction-cost).

#### Measurement of Stack-Pushed Bytes

For all operations which push to the primary stack, operation cost is additionally incremented by the byte length of the pushed stack item. See [Rationale: Limitation of Pushed Bytes](#limitation-of-pushed-bytes).

This specification codifies the pushing behavior of each operation based on [`v27.0.0` of Bitcoin Cash Node](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/49ad6a9a95bda543926ec90d07e7bd266581c4d0/src/script/interpreter.cpp), an implementation written in 2008 by Satoshi Nakamoto and continuously improved by various contributors. Counting of pushed bytes are consistent with this implementation with the exception of the bitwise operations (`OP_AND`, `OP_OR`, and `OP_XOR`), which are considered to push their result, even if an implementation performs bitwise operations in-place (e.g. for performance reasons).

<details>
<summary>Operation Cost by Operation</summary>

#### Operation Cost by Operation

> [!NOTE]
>
> **This table is non-normative; it is provided for clarity, but it is not necessary for correct implementation of the proposal.** The provided `Operation Cost` formulas are derived from the proposal's technical specification. The [test vectors](#tests--benchmarks) are also designed to ensure that implementations correctly account for the cost of each operation.

The full, standard operation cost for each operation is provided below, accounting for [Measurement of Stack-Pushed Bytes](#measurement-of-stack-pushed-bytes), [Arithmetic Operation Cost](#arithmetic-operation-cost), [Hash Digest Iteration Cost](#hash-digest-iteration-cost), and [Signature Checking Operation Cost](#signature-checking-operation-cost).

| Opcode(s)                  | Codepoint(s)   | Notes                                 | Operation Cost                                        |
| -------------------------- | -------------- | ------------------------------------- | ----------------------------------------------------- |
| `OP_0`                     | `0x00`         |                                       | `100`                                                 |
| `OP_PUSHBYTES_N` (1-75)    | `0x01`-`0x4b`  |                                       | `100 + N`                                             |
| `OP_PUSHDATA_N`            | `0x4c`-`0x4e`  |                                       | `100 + length`                                        |
| `OP_1NEGATE`               | `0x4f`         |                                       | `100 + 1`                                             |
| `OP_1`-`OP_16`             | `0x51`-`OP_16` |                                       | `100 + 1`                                             |
| `OP_NOP`                   | `0x61`         |                                       | `100`                                                 |
| `OP_IF`                    | `0x63`         |                                       | `100`                                                 |
| `OP_NOTIF`                 | `0x64`         |                                       | `100`                                                 |
| `OP_ELSE`                  | `0x67`         |                                       | `100`                                                 |
| `OP_ENDIF`                 | `0x68`         |                                       | `100`                                                 |
| `OP_VERIFY`                | `0x69`         |                                       | `100`                                                 |
| `OP_RETURN`                | `0x6a`         |                                       | `100`                                                 |
| `OP_TOALTSTACK`            | `0x6b`         |                                       | `100`                                                 |
| `OP_FROMALTSTACK`          | `0x6c`         |                                       | `100 + length`                                        |
| `OP_2DROP`                 | `0x6d`         |                                       | `100`                                                 |
| `OP_2DUP`                  | `0x6e`         |                                       | `100 + 2 * length`                                    |
| `OP_3DUP`                  | `0x6f`         |                                       | `100 + 3 * length`                                    |
| `OP_2OVER`                 | `0x70`         | `a b c d -> a b c d a b`              | `100 + a.length + b.length`                           |
| `OP_2ROT`                  | `0x71`         | `a b c d e f -> c d e f a b`          | `100 + a.length + b.length`                           |
| `OP_2SWAP`                 | `0x72`         |                                       | `100`                                                 |
| `OP_IFDUP`                 | `0x73`         |                                       | `100 + length` OR `100`                               |
| `OP_DEPTH`                 | `0x74`         |                                       | `100 + length`                                        |
| `OP_DROP`                  | `0x75`         |                                       | `100`                                                 |
| `OP_DUP`                   | `0x76`         |                                       | `100 + length`                                        |
| `OP_NIP`                   | `0x77`         |                                       | `100`                                                 |
| `OP_OVER`                  | `0x78`         | `a b -> a b a`                        | `100 + a.length`                                      |
| `OP_PICK`                  | `0x79`         |                                       | `100 + length`                                        |
| `OP_ROLL`                  | `0x7a`         |                                       | `100 + length`                                        |
| `OP_ROT`                   | `0x7b`         |                                       | `100`                                                 |
| `OP_SWAP`                  | `0x7c`         |                                       | `100`                                                 |
| `OP_TUCK`                  | `0x7d`         | `a b -> b a b`                        | `100 + b.length`                                      |
| `OP_CAT`                   | `0x7e`         |                                       | `100`                                                 |
| `OP_SPLIT`                 | `0x7f`         |                                       | `100`                                                 |
| `OP_NUM2BIN`               | `0x80`         |                                       | `100 + length`                                        |
| `OP_BIN2NUM`               | `0x81`         |                                       | `100 + length`                                        |
| `OP_SIZE`                  | `0x82`         |                                       | `100 + length`                                        |
| `OP_AND`                   | `0x84`         |                                       | `100 + length`                                        |
| `OP_OR`                    | `0x85`         |                                       | `100 + length`                                        |
| `OP_XOR`                   | `0x86`         |                                       | `100 + length`                                        |
| `OP_EQUAL`                 | `0x87`         |                                       | `100` OR `100 + 1`                                    |
| `OP_EQUALVERIFY`           | `0x88`         |                                       | `100`                                                 |
| `OP_1ADD`                  | `0x8b`         |                                       | `100 + length`                                        |
| `OP_1SUB`                  | `0x8c`         |                                       | `100 + length`                                        |
| `OP_NEGATE`                | `0x8f`         |                                       | `100 + length`                                        |
| `OP_ABS`                   | `0x90`         |                                       | `100 + length`                                        |
| `OP_NOT`                   | `0x91`         |                                       | `100` OR `100 + 1`                                    |
| `OP_0NOTEQUAL`             | `0x92`         |                                       | `100` OR `100 + 1`                                    |
| `OP_ADD`                   | `0x93`         |                                       | `100 + length`                                        |
| `OP_SUB`                   | `0x94`         |                                       | `100 + length`                                        |
| `OP_MUL`                   | `0x95`         | `a b -> c`                            | `100 + (a.length * b.length) + c.length`              |
| `OP_DIV`                   | `0x96`         | `a b -> c`                            | `100 + (a.length * b.length) + c.length`              |
| `OP_MOD`                   | `0x97`         | `a b -> c`                            | `100 + (a.length * b.length) + c.length`              |
| `OP_BOOLAND`               | `0x9a`         |                                       | `100` OR `100 + 1`                                    |
| `OP_BOOLOR`                | `0x9b`         |                                       | `100` OR `100 + 1`                                    |
| `OP_NUMEQUAL`              | `0x9c`         |                                       | `100` OR `100 + 1`                                    |
| `OP_NUMEQUALVERIFY`        | `0x9d`         |                                       | `100` OR `100 + 1`                                    |
| `OP_NUMNOTEQUAL`           | `0x9e`         |                                       | `100` OR `100 + 1`                                    |
| `OP_LESSTHAN`              | `0x9f`         |                                       | `100` OR `100 + 1`                                    |
| `OP_GREATERTHAN`           | `0xa0`         |                                       | `100` OR `100 + 1`                                    |
| `OP_LESSTHANOREQUAL`       | `0xa1`         |                                       | `100` OR `100 + 1`                                    |
| `OP_GREATERTHANOREQUAL`    | `0xa2`         |                                       | `100` OR `100 + 1`                                    |
| `OP_MIN`                   | `0xa3`         |                                       | `100 + length`                                        |
| `OP_MAX`                   | `0xa4`         |                                       | `100 + length`                                        |
| `OP_WITHIN`                | `0xa5`         |                                       | `100` OR `100 + 1`                                    |
| `OP_RIPEMD160`             | `0xa6`         | Standard: `192`; Consensus: `64`      | `100 + (iterations * 192)`                            |
| `OP_SHA1`                  | `0xa7`         | "                                     | `100 + (iterations * 192)`                            |
| `OP_SHA256`                | `0xa8`         | "                                     | `100 + (iterations * 192)`                            |
| `OP_HASH160`               | `0xa9`         | "                                     | `100 + (iterations * 192) + 1`                        |
| `OP_HASH256`               | `0xaa`         | "                                     | `100 + (iterations * 192) + 1`                        |
| `OP_CODESEPARATOR`         | `0xab`         |                                       | `100`                                                 |
| `OP_CHECKSIG`              | `0xac`         | (Null signature check is `100`)       | `100` OR `100 + 26000 + (iterations * 192) + 1`       |
| `OP_CHECKSIGVERIFY`        | `0xad`         | "                                     | `100` OR `100 + 26000 + (iterations * 192)`           |
| `OP_CHECKMULTISIG`         | `0xae`         | M-of-N; K=M for Schnorr, N for legacy | `100` OR `100 + (26000 * K) + (iterations * 192) + 1` |
| `OP_CHECKMULTISIGVERIFY`   | `0xaf`         | "                                     | `100` OR `100 + (26000 * K) + (iterations * 192)`     |
| `OP_NOP1`                  | `0xb0`         |                                       | `100`                                                 |
| `OP_CHECKLOCKTIMEVERIFY`   | `0xb1`         |                                       | `100`                                                 |
| `OP_CHECKSEQUENCEVERIFY`   | `0xb2`         |                                       | `100`                                                 |
| `OP_NOP4`                  | `0xb3`         |                                       | `100`                                                 |
| `OP_NOP5`                  | `0xb4`         |                                       | `100`                                                 |
| `OP_NOP6`                  | `0xb5`         |                                       | `100`                                                 |
| `OP_NOP7`                  | `0xb6`         |                                       | `100`                                                 |
| `OP_NOP8`                  | `0xb7`         |                                       | `100`                                                 |
| `OP_NOP9`                  | `0xb8`         |                                       | `100`                                                 |
| `OP_NOP10`                 | `0xb9`         |                                       | `100`                                                 |
| `OP_CHECKDATASIG`          | `0xba`         | (Cost `100` for null signature check) | `100` OR `100 + 26000 + (iterations * 192) + 1`       |
| `OP_CHECKDATASIGVERIFY`    | `0xbb`         | "                                     | `100` OR `100 + 26000 + (iterations * 192)`           |
| `OP_REVERSEBYTES`          | `0xbc`         |                                       | `100 + length`                                        |
| `OP_INPUTINDEX`            | `0xc0`         |                                       | `100 + length`                                        |
| `OP_ACTIVEBYTECODE`        | `0xc1`         |                                       | `100 + length`                                        |
| `OP_TXVERSION`             | `0xc2`         |                                       | `100 + length`                                        |
| `OP_TXINPUTCOUNT`          | `0xc3`         |                                       | `100 + length`                                        |
| `OP_TXOUTPUTCOUNT`         | `0xc4`         |                                       | `100 + length`                                        |
| `OP_TXLOCKTIME`            | `0xc5`         |                                       | `100 + length`                                        |
| `OP_UTXOVALUE`             | `0xc6`         |                                       | `100 + length`                                        |
| `OP_UTXOBYTECODE`          | `0xc7`         |                                       | `100 + length`                                        |
| `OP_OUTPOINTTXHASH`        | `0xc8`         |                                       | `100 + length`                                        |
| `OP_OUTPOINTINDEX`         | `0xc9`         |                                       | `100 + length`                                        |
| `OP_INPUTBYTECODE`         | `0xca`         |                                       | `100 + length`                                        |
| `OP_INPUTSEQUENCENUMBER`   | `0xcb`         |                                       | `100 + length`                                        |
| `OP_OUTPUTVALUE`           | `0xcc`         |                                       | `100 + length`                                        |
| `OP_OUTPUTBYTECODE`        | `0xcd`         |                                       | `100 + length`                                        |
| `OP_UTXOTOKENCATEGORY`     | `0xce`         |                                       | `100 + length`                                        |
| `OP_UTXOTOKENCOMMITMENT`   | `0xcf`         |                                       | `100 + length`                                        |
| `OP_UTXOTOKENAMOUNT`       | `0xd0`         |                                       | `100 + length`                                        |
| `OP_OUTPUTTOKENCATEGORY`   | `0xd1`         |                                       | `100 + length`                                        |
| `OP_OUTPUTTOKENCOMMITMENT` | `0xd2`         |                                       | `100 + length`                                        |
| `OP_OUTPUTTOKENAMOUNT`     | `0xd3`         |                                       | `100 + length`                                        |

</details>

#### Arithmetic Operation Cost

To account for <code>O(n<sup>2</sup>)</code> worst-case performance, the operation cost of `OP_MUL` (`0x95`), `OP_DIV` (`0x96`), and `OP_MOD` (`0x97`) are increased by the product of their input lengths in addition to the length of their pushed result, i.e. for `a b -> c`, the operation cost is `100 + (a.length * b.length) + c.length`.

#### Hash Digest Iteration Cost

All operations which increase the cumulative total of hash digest iterations must simultaneously increase the cumulative operation cost by the product of the additional iterations and the `Hash Digest Iteration Cost`. See [Rationale: Unification of Limits into Operation Cost](#unification-of-limits-into-operation-cost).

The `Hash Digest Iteration Cost` is set to `64` for block validation (by consensus) and `192` for transaction relay ("standard") validation. See [Rationale: Selection of Hash Digest Iteration Cost](#selection-of-hash-digest-iteration-cost).

#### Signature Checking Operation Cost

All operations which increase the cumulative total of `SigChecks` (as defined by the [2020-05 SigChecks specification](https://gitlab.com/bitcoin-cash-node/bchn-sw/bitcoincash-upgrade-specifications/-/blob/master/spec/2020-05-15-sigchecks.md)) must simultaneously increase the cumulative operation cost by the product of the additional sigchecks and `26000`. See [Rationale: Selection of Signature Verification Operation Cost](#selection-of-signature-verification-operation-cost).

### Notice of Possible Future Expansion

While unusual, it is possible to design pre-signed transactions, contract systems, and protocols which rely on the rejection of otherwise-valid transactions that exceed current VM limits. Contract authors are advised that future upgrades may further expand VM limits by increasing allowable operation cost density, reducing the accounted cost of particular operations, or otherwise.

**This proposal interprets such failure-reliant constructions as intentional** – they are designed to fail unless/until a possible future network upgrade in which such limits are increased, e.g. upgrade-activation futures contracts. See [Rationale: Inclusion of "Notice of Possible Future Expansion"](#inclusion-of-notice-of-possible-future-expansion).

<details>

<summary>Notes</summary>

As always, the security of a contract is the responsibility of the entity locking funds in that contract; funds can always be locked in insecure contracts (e.g. `OP_DROP OP_1`). This notice is provided to warn contract authors and explicitly codify a network policy: the possible existence of poorly-designed contracts will not preclude future upgrades from further expanding VM limits.

To ensure an otherwise-valid transaction will always fail when some limit is exceeded (in the rare case that such a behavior is desirable), that behavior must be either 1) explicitly validated or 2) introduced to the protocol or contract system in question prior to the activation of any future upgrade which expands the limit.

A similar notice also appeared in [CHIP-2021-03: Bigger Script Integers](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md#notice-of-possible-future-expansion).

</details>

## Rationale

This section documents design decisions made in this specification.

### Retention of Control Stack Limit

This proposal avoids modifying the existing practical limit on control stack depth by introducing a specific `Control Stack Limit` in place of the current effective limit<sup>1</sup>.

While it is possible to implement an [O(1) Control Stack](https://github.com/bitcoin/bitcoin/pull/16902) such that deeply nested conditionals have no impact on transaction validation performance, requiring this optimization in all performance-critical VM implementations may have significant consequences for future protocol complexity, particular if future upgrades extend the VM's control flow capabilities. (E.g. [bounded loops](https://github.com/bitjson/bch-loops), switch or match expressions, word/function definition, etc.)

The existing control stack depth of `100` is already far in excess of all known usage. Additionally, given both the availability of the alternate stack and this proposal's replacement of the operation limit, it is possible to emulate practically any algorithm requiring excessive control stack depth with an equivalent, less-deeply nested implementation. As such, an increase in the existing limit on control stack depth is considered out of this proposal's scope.

<details>

<summary>Notes</summary>

1. The existing `201` operation limit prevents any currently-valid contract from requiring a control stack of depth greater than `100`, e.g. `<1> OP_IF <1> OP_IF <1> OP_IF ... OP_ENDIF OP_ENDIF OP_ENDIF`.

</details>

### Hashing Limit by Digest Iterations

One possible alternative to the proposed [Hashing Limit](#hashing-limit) design is to limit evaluations to a fixed count of "bytes hashed" (rather than digest iterations). While this seems simpler, it performs poorly in an adversarial environment: an attacker can choose message sizes to minimize bytes hashed while maximizing digest iterations required (e.g. 1-byte messages).

By directly measuring digest iterations, non-malicious contracts can be allowed to evaluate a higher density of hashing operations without increasing the worst-case validation requirements of the VM as it existed prior to this proposal.

### Selection of Hashing Limit

This proposal sets the standard (transaction relay policy) hashing density limit at `0.5` digest iterations per spending transaction byte. This value is the asymptotic maximum density of hashing operations in plausibly non-malicious, standard transactions within the current VM limits and instruction set: assuming no transaction encoding overhead and that all hashed material is maximally-padded (1-byte segments), the most efficient methods for reducing hashed material into a single result (without any further processing) requires 1 additional operation (e.g. `OP_CAT`) per hashing operation<sup>1</sup>.

Note that this standard limit is also well in excess of the highest density found in any standard transaction maximizing hashing within signature checking operations: `0.30` digest iterations per byte<sup>2</sup>.

For block validation, this proposal sets the consensus limit (A.K.A. "nonstandard") hashing density at `3.5` digest iterations per spending transaction byte. This limit exceeds the highest-known density of any currently standard transaction of `3.44` digest iterations per byte<sup>3</sup>. Ideally, a separate nonstandard limit could be avoided, but because currently-standard transactions can be designed to exceed the `0.5` limit, a higher nonstandard limit is necessary to avoid immediately invaliding any currently-standard transactions<sup>4</sup>. Future proposals could schedule an upgrade to consolidate these limits.

Alternatively, this proposal could set a consensus hashing limit without a lower standardness limit. However, hashing cannot be [reliably deferred like signature validation](#continued-availability-of-deferred-signature-validation); because the current practical limit is nearly an order of magnitude higher then theoretically useful, limiting the density allowed in standard validation significantly improves worst-case validation performance.

<details>

<summary>Notes</summary>

1. The two-round hashing operations (`OP_HASH160` and `OP_HASH256`) would be wasteful in this idealized scenario, as 1-byte inputs already require padding in the first round of hashing, so the idealized maximum density is 1 digest iteration per 2 opcodes. Note that while a future upgrade could increase this density by compressing the evaluating code (e.g. with [bounded loops](https://github.com/bitjson/bch-loops)) or allowing significant lengths of hashed material to be provided via UTXO bytecode (e.g. by relaxing output standardness), this example still 1) excludes a minimum of 60 bytes of transaction overhead (allowing up to `1911` bytes hashed in `30` hash digest iterations), 2) significantly overestimates hashing density by assuming exclusively 1-byte input lengths rather than more common lengths (e.g. `20` or `32`), and 3) excludes overhead from the non-hashing operations performed in realistic contracts.
2. See benchmark `xfszef`.
3. See benchmark `lcennk`.
4. This proposal follows the [long-established](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/commit/a206a23980c15cacf39d267c509bd70c23c94bfa) process of restraining abusive usage by invalidating currently-standard malicious behavior via relay policy (A.K.A. "standardness"), and then only restricting the behavior from block validation (A.K.A. "consensus") after it has remained restricted for a significant period of time (e.g. [`NULLDUMMY`](https://github.com/bitcoin/bips/blob/master/bip-0147.mediawiki), [`NULLFAIL`](https://upgradespecs.bitcoincashnode.org/nov-13-hardfork-spec/), [`MINIMALDATA`](https://upgradespecs.bitcoincashnode.org/2019-11-15-minimaldata/), etc.).

</details>

### Exclusion of Signing Serialization Components from Hashing Limit

This proposal includes [the cost of hashing all signing serializations](#transaction-signature-checking-operations) in the proposed `Hashing Limit`, but it excludes the cost of any hashing required to produce the internal components of signing serializations (i.e. `hashPrevouts`, `hashUtxos`, `hashSequence`, and `hashOutputs`). This configuration reduces protocol complexity while correctly accounting for the real-world hashing cost of `coveredBytecode` (A.K.A. `scriptCode`), particularly in [preventing abuse of `OP_CODESEPARATOR`](https://gist.github.com/markblundeberg/c2c88d25d5f34213830e48d459cbfb44), future increases to maximum standard input bytecode length (A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE` – 1,650 bytes), and/or increases to consensus-maximum VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE` – 10,000 bytes).

Alternatively, this proposal could also require internally accounting for the cost of hashing within signing serialization components. However, because components can be easily cached across signature checks – and for `hashPrevouts`, `hashUtxos`, `hashSequence`, across the entire transaction validation – such costs would need to be amortized across all of a transaction's signature checks to approximate their fixed, real-world cost. As signature checking is already sufficiently limited by `SigChecks` and operation cost, omitting hash digest iteration counting of signing serialization components reduces unnecessary protocol complexity.

Notably, this proposal's accounting strategy would also remain reasonable if a future upgrade were to [relax output standardness](https://bitcoincashresearch.org/t/p2sh32-a-long-term-solution-for-80-bit-p2sh-collision-attacks/750/13#relaxing-output-standardness-1); though the corresponding output setting of `hashOutputs` (A.K.A. `SIGHASH_SINGLE`) can only be cached for the current input, any increase in the length of an input's corresponding output would necessarily increase the length of the spending transaction, lifting the transaction's cumulative hashing "budget" at a faster rate than the increase in hashing cost.

#### Ongoing Value of OP_CODESEPARATOR Operation

The `OP_CODESEPARATOR` operation modifies the signing serialization (A.K.A. "`SIGHASH`" preimage) used in later-executed signature operations by truncating the `coveredBytecode` component at the index of the last executed `OP_CODESEPARATOR` operation.

The `OP_CODESEPARATOR` behavior is useful for designing contracts in which a single key may provide signatures for use in multiple contract code paths. Without `OP_CODESEPARATOR`, a valid signature by the key could be re-used by an attacker in a manipulated version of the input's unlocking bytecode to unlock the contract following a different code path; if `OP_CODESEPARATOR` is executed at some point before any later signature checking operation, a valid signature for the intended code path will fail validation if the unlocking bytecode is manipulated to follow another code path.

Notably, the behavior of `OP_CODESEPARATOR` can be achieved more directly with `OP_CHECKDATASIG`: for each code path, the locking bytecode must simply require a different message to be signed by the data signature, preventing signatures from being reused across code paths. However, `OP_CODESEPARATOR` can be more byte-efficient in that it requires only one additional byte (the `OP_CODESEPARATOR` instruction) to differentiate the two signature preimages, while `OP_CHECKDATASIG` may require additional instructions or encoded data to differentiate each possible message.

While past proposals (for both Bitcoin Cash and similar networks) have called for deprecating or disabling `OP_CODESEPARATOR`, this proposal retains the feature while preventing excessive resource consumption, as `OP_CODESEPARATOR` is already available in the Bitcoin Cash instruction set, may have active users, and remains useful in some scenarios.

### Increased Usability of Multisig Stack Clearing

Prior to this proposal, the multisig operations (`OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY`) were limited in part via incrementing the operation count by the number of stack items provided as public keys. However, because public keys are only validated prior to their use in a signature check, it is also possible to use these operations to more efficiently drop items from the stack than via repeated calls to `OP_DROP`, `OP_2DROP`, or `OP_NIP`.

While this eccentric behavior was likely unintentional, it has been available to contract authors since Bitcoin Cash's 2009 launch, and it is generally the most byte-efficient method of dropping between 9 and 20 stack items (reducing transaction sizes for some contracts). Because this existing stack-clearing behavior is a useful feature and does not meaningfully impact transaction validation costs, this proposal treats zero-signature multisig operations equivalently to all other constant and linear time operations.

### Limitation of Pushed Bytes

This proposal limits the density of both memory usage and computation by limiting bytes pushed to the stack to approximately `700` per spending transaction byte (the per-byte budget of `800` minus the base instruction cost of `100`).

Because stack-pushed bytes become inputs to other operations, limiting the overall density of pushed bytes is the most comprehensive method of limiting all linear-time operations (bitwise operations, `OP_DUP`, `OP_EQUAL`, `OP_REVERSEBYTES`, etc.); this approach reduces protocol complexity by avoiding special accounting for all but the most expensive operations (arithmetic, hashing, and signature checking).

Additionally, this proposal’s density-based limit caps the maximum memory and memory bandwidth requirements of validating a large stream of transactions, regardless of the number of parallel validations being performed<sup>1</sup>.

Alternatively, this proposal could limit memory usage by continuously tracking total stack usage and enforcing some maximum limit. In addition to increasing implementation complexity (e.g. performant implementations would require a usage-tracking stack), this approach would 1) only implicitly limit memory bandwidth usage and 2) require additional limitations on linear-time operations.

<details>

<summary>Notes</summary>

1. While the 10KB stack item length limit and 1000 item stack depth limit effectively limit maximum memory to 10MB plus overhead per validating thread, without additional limits on writing, memory bandwidth can become a bottleneck. This is particularly critical if future upgrades allow for increased contract or stack item lengths, [bounded loops](https://github.com/bitjson/bch-loops), word/function definition, and/or other features that reduce transaction sizes by increasing their operation density.

</details>

### Unification of Limits into Operation Cost

While this proposal maintains independent limits for both signature checking and hashing (via `SigChecks` and hash digest iterations, respectively), the measurements used to enforce each limit also contribute to the unified `Operation Cost Limit`. This configuration prevents maliciously designed transactions from maximizing validation cost across multiple disparate limits, reducing the potential disparity in worst case performance between honest and malicious usage.

### Selection of Operation Cost Limit

This proposal sets the operation cost density at `800` per spending transaction byte. Given a base instruction cost of `100`, this density ensures a net budget of `700` per spending transaction byte, a constant rounded up from the maximum effective limit prior to this proposal<sup>1</sup>.

<details>

<summary>Notes</summary>

1. Benchmark `d0kxwp` (`99966` bytes) pushes `62757175` bytes, approximately `627.79` bytes per spending transaction byte.

</details>

### Selection of Base Instruction Cost

To retain a conservative limit on contract operation density, this proposal sets a base operation cost of `100` per evaluated instruction.

Given the [operation cost density limit of `800`](#operation-cost-limit), a base instruction cost of `100` ensures that the maximum operation density within spending transactions (8 per byte) remains within an order of magnitude of the current effective limit (approximately 1 per byte) resulting from the 201 operation limit.

Because base instruction cost can be safely reduced - but not increased - in future upgrades without invalidating contracts<sup>1</sup>, a base cost of `100` is more conservative than lower values (like `10`, `1`, or `0`). Additionally, a relatively-high base instruction cost leaves room for future upgrades to differentiate the costs of existing low and fixed-cost instructions, if necessary.

Alternatively, this proposal could omit any base instruction cost, limiting the cost of all instructions to their impact on the stack. However, instruction evaluation is not costless – a nonzero base cost properly accounts for the real world overhead of evaluating an instruction and verifying non-violation of applicable limits.

Finally, setting an explicit base instruction cost reduces the VM’s implicit reliance on bytecode length for limiting worst case validation performance. By explicitly limiting maximum operation density, future upgrades which increase the maximum allowable contract length or reduce transaction sizes by compressing contract code (e.g. [bounded loops](https://github.com/bitjson/bch-loops) or word/function definition) can be supported without incurring technical debt and/or producing de facto limit increases.

<details>

<summary>Notes</summary>

1. Note that pre-signed transactions, contract systems, and protocols can be specifically designed to operate on any observable characteristic of the VM, including limits. See [Notice of Possible Future Expansion](#notice-of-possible-future-expansion).

</details>

### Selection of Signature Verification Operation Cost

In addition to the operation cost of hash digest iterations performed within signature checking operations, this proposal sets the operation cost of signature verification to `26000` per check. This constant is rounded down from the per-check budget available within current limits<sup>1</sup>.

By deriving this limit to be practically equivalent - but slightly more generous - than the existing SigChecks limit, this proposal ensures that all currently-standard transactions remain valid, while avoiding any significant increase in worst case validation performance vs. current limits.

Note that because Schnorr signatures can be batch validated, this proposal could reasonably have set another, lower cost for such signatures. However, as signature validation is generally the bottleneck in worst case validation performance of currently-standard transactions, it may be prudent to avoid extending higher limits for Schnorr signatures and to instead allow any such performance gains to simply improve average validation performance.

Additionally, for contracts to take advantage of any such operation cost discount for Schnorr signatures, the separate SigChecks limits would likely also need to be loosened for Schnorr signatures, and in both cases, a differentiated limit would essentially required all VM implementations to implement the batch verification optimization to avoid negatively impacting worst case performance.

Given these trade-offs, this proposal takes the more conservative approach of applying a fixed operation cost per signature check based only on the existing, worst case scenario.

<details>

<summary>Notes</summary>

1. Given the `SigChecks` standardness limit of 1 check per `33.5` bytes and a per-byte operation budget of `800`, the current implied cost of a signature check is `26800` (`33.5 * 800 = 26800`). Benchmark `l0fhm3` (`Within BCH_2023_05 standard limits, packed 1-of-3 bare multisig inputs, 1 output (all ECDSA signatures, first slot)`) demonstrates transaction validation performance similar to a worst-case scenario for this metric – it contains `2619` signature checks in `99988` bytes, with a standard cost of `3867390` before accounting for signature checks and a remaining budget of approximately `29065` per signature check (`((99988 * 800) - 3867390) / 2619 ~= 29,065.68`).

</details>

### Continued Availability of Deferred Signature Validation

By using the existing `SigCheck` metric in computing operation cost, this proposal retains the ability to defer signature validation (as described in the [2020-05 SigChecks specification](https://gitlab.com/bitcoin-cash-node/bchn-sw/bitcoincash-upgrade-specifications/-/blob/master/spec/2020-05-15-sigchecks.md)) until the rest of a contract has been evaluated. Notably, enforcement of the proposed hashing limit within signature checking operations can also be deferred until immediately prior to signature validation, though because other hashing operations must be evaluated during program execution, deferring either hashing or digest iteration estimation will not improve worst-case validation performance.

### Selection of Hash Digest Iteration Cost

For block validation (consensus), this proposal sets the operation cost of hash digest iterations to `64` (the hashing algorithm block size); for transaction relay (standardness policy) this proposal sets the operation cost of hash digest iterations to `192` (`64*3=192`). These values ensure that all possible standard transactions remain valid by consensus while correctly accounting for the cost of hashing during standard validation.

Each VM-supported hashing algorithm – RIPEMD-160, SHA-1, and SHA-256 – uses a [Merkle–Damgård construction](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) with a `block size` of 512 bits (64 bytes); operation cost is essentially a count of bytes processed by any VM operation, so the hashing algorithm block size (`64`) is the most appropriate multiple upon which to base hashing operation cost calculations.

Under the existing limits, the highest possible hashing density of a standard transaction is `3.44` hash digest iterations per spending transaction byte<sup>1</sup>. Given a budget of `800` per byte, it is not possible to set a higher multiple of `64` for the consensus operation cost of hash digest iterations (e.g. `128`) without invalidating currently-standard transactions<sup>2</sup>. As such, this proposal sets the consensus operation cost of hash digest iterations to `64`.

To determine the standard operation cost of hash digest iterations, the results of multiple representative benchmarks can be compared across multiple independent VM implementations; in a variety of cases, given a [signature checking cost of `26000`](#selection-of-signature-verification-operation-cost), the closest multiple of `64` to real-world performance cost is `192`<sup>3</sup>.

<details>
<summary>Notes</summary>

1. Benchmark `lcennk`.
2. For example, benchmark `lcennk` (`99966` bytes) with a hash digest iteration cost of `128` results in a cumulative operation cost of `87910420`, or `879.40` per byte.
3. On one representative system, [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/), performs the baseline benchmark (`trxhzt`) at `11759.3` Hz (`2` sigchecks and `12` iterations) and a hashing-heavy benchmark (`lcennk`) at `10.6` Hz (`0` sigchecks and `344162` iterations). Given `11759.3 (2s + 12h) = 10.6 (344162h)`, each signature check costs about 149 hash digest iterations; calibrating for a signature check cost of `26000`, `26000 / 149.12 ~= 174`, the measured cost per digest iteration is approximately `174`. Likewise, [Libauth](https://github.com/bitauth/libauth), an unrelated JavaScript implementation using WebAssembly-based, software-only implementations of the relevant cryptographic algorithms, performs `trxhzt` at `2966.08` Hz and `lcennk` at `2.56` Hz. Given `2966.08 (2s + 12h) = 2.56 (344162h)`, each signature check costs about 142 hash digest iterations; calibrating for a signature check cost of `26000`, `26000 / 142.52 ~= 182`, the measured cost per digest iteration is approximately `182`.

</details>

### Inclusion of "Notice of Possible Future Expansion"

This proposal includes a [Notice of Possible Future Expansion](#notice-of-possible-future-expansion) to clarify established precedent in Bitcoin (Cash) protocol development: pre-signed transactions, contract systems, and protocols which rely on the non-existence of VM features must be interpreted as intentional. This precedent was established by Satoshi Nakamoto in [`reverted makefile.unix wx-config -- version 0.3.6`](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/commit/757f0769d8360ea043f469f3a35f6ec204740446) (July 29, 2010) with the addition of new opcodes `OP_NOP1` to `OP_NOP10`, and the precedent was reaffirmed by numerous upgrades to Bitcoin Cash, e.g. [`OP_CHECKLOCKTIMEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki), [Relative locktime](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki), [`OP_CHECKSEQUENCEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki), the [2018 opcode restoration](https://upgradespecs.bitcoincashnode.org/may-2018-reenabled-opcodes/), [CHIP-2021-03: Bigger Script Integers](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md#notice-of-possible-future-expansion), and later upgrades.

### Tests & Benchmarks

This proposal includes a suite of functional tests and benchmarks to verify the performance of all operations within virtual machine implementations. See [Tests & Benchmarks](./tests-and-benchmarks.md) for details.

## Implementations

Please see the following reference implementations for additional examples and test vectors:

- C++:
  - [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/) – A professional, miner-friendly node that solves practical problems for Bitcoin Cash. [Merge Request !1827](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1827).
- JavaScript/TypeScript
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Pull Request #139](https://github.com/bitauth/libauth/pull/139).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Pull Request #101](https://github.com/bitauth/bitauth-ide/pull/101).

## Stakeholders & Statements

(Pending reviews.)

## Feedback & Reviews

- [`Raising the 520 byte push limit & 201 operation limit` – Feb 8, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/raising-the-520-byte-push-limit-201-operation-limit/282)
- [`CHIP: Targeted Virtual Machine Limits` – May 12, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/chip-targeted-virtual-machine-limits/437)

## Changelog

This section summarizes the evolution of this document.

- **v3.0.0**
  - Revise limits to be density based ([#8](https://github.com/bitjson/bch-vm-limits/issues/8))
  - Limit bytes pushed to the stack ([#10](https://github.com/bitjson/bch-vm-limits/issues/10))
  - Limit depth of control stack ([#11](https://github.com/bitjson/bch-vm-limits/issues/11))
  - Note accounting of P2SH redeem bytecode digest iterations ([#14](https://github.com/bitjson/bch-vm-limits/issues/14))
- **v2.0.0** ([`e686b981`](https://github.com/bitjson/bch-vm-limits/commit/e686b981e14f9f69433682dda9df8b204c66b709))
  - Simplify stack memory limit calculation ([#6](https://github.com/bitjson/bch-vm-limits/pull/6))
  - Correct hashing benchmarks, update hashing limit ([#6](https://github.com/bitjson/bch-vm-limits/pull/6))
  - Propose for May 2025 Upgrade
- **v1.0.0 – 2021-05-12** ([`5b24b0ec`](https://github.com/bitjson/bch-vm-limits/commit/ba2785b1f38bdecd8d72d5236c31c6846165c141))
  - Initial publication

## Copyright

This document is placed in the public domain.
