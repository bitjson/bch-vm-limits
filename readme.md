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

<details>
<summary>Technical Overview of Changes to C++ Clients</summary>

The following overview summarizes all changes proposed by this document to C++ implementations following the general structure of the original Satoshi client, e.g. [Bitcoin Cash Node](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/185f2d64143e807352c0a18f92d8f3ac14bf6840/src/script/interpreter.cpp):

1. `MAX_SCRIPT_ELEMENT_SIZE` increases from `520` to `10000`.
2. `nOpCount` becomes `nOpCost` (used to measure stack-pushed bytes and operation costs)
3. A new `static inline pushstack` is added to match `popstack`; `pushstack` increments `nOpCost` by the item length.
4. `if (opcode > OP_16 && ++nOpCount > MAX_OPS_PER_SCRIPT) { ... }` becomes `nOpCost += 100;` (not conditional, so also executed for push operations).
5. `case OP_AND/OP_OR/OP_XOR:` adds a `nOpCost += vch1.size();`
6. `case OP_MUL/OP_DIV/OP_MOD:` adds a `nOpCost += a.size() * b.size();`
7. Hashing operations add `1 + ((message_length + 8) / 64)` to `nHashDigestIterations`, and `nOpCost += 192 * iterations;`.
8. Same for signing operations (count iterations only for the top-level preimage, not `hashPrevouts`/`hashUtxos`/`hashSequence`/`hashOutputs`), plus `nOpCost += 26000 * sigchecks;` (and `nOpCount += nKeysCount;` removed)
9. `SigChecks` limits remain unchanged; similar density checks apply to `nHashDigestIterations` and `nOpCost`.
10. Adds `if (vfExec.size() > 100) { return set_error(...`

</details>

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

To retain the existing limit of `100` on control stack depth, a direct limit is placed on operations which push to the control stack (A.K.A. `vfExec` or `ConditionStack`). See [Rationale: Retention of Control Stack Limit](rationale.md#retention-of-control-stack-limit).

Currently, this limit impacts only the `OP_IF` and `OP_NOTIF` operations. For the purpose of enforcing this limit, `OP_IFDUP` is not considered to internally push to the control stack, i.e. an `OP_IFDUP` executed at a depth of `100` does not violate the `Control Stack Limit`.

### Hashing Limit

A new limit is placed on `OP_RIPEMD160` (`0xa6`), `OP_SHA1` (`0xa7`), `OP_SHA256` (`0xa8`), `OP_HASH160` (`0xa9`), `OP_HASH256` (`0xaa`), `OP_CHECKSIG` (`0xac`), `OP_CHECKSIGVERIFY` (`0xad`), `OP_CHECKMULTISIG` (`0xae`), `OP_CHECKMULTISIGVERIFY` (`0xaf`), `OP_CHECKDATASIG` (`0xba`), and `OP_CHECKDATASIGVERIFY` (`0xbb`) to prevent excessive hashing function usage.

Before a hashing function is performed, its expected cost – in terms of digest iterations – is added to a cumulative total for the transaction. If the cumulative digest iterations required to validate the spending transaction exceed the maximum allowed density, the operation produces an error. See [Rationale: Hashing Limit by Digest Iterations](rationale.md#hashing-limit-by-digest-iterations).

#### Maximum Hashing Density

For standard transactions, the maximum density is `0.5` hash digest iterations per spending transaction byte; for block validation, the maximum density is `3.5` hash digest iterations per spending transaction byte. See [Rationale: Selection of Hashing Limit](rationale.md#selection-of-hashing-limit).

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

Note that hash digest iterations required to produce components of the signing serialization (i.e. `hashPrevouts`, `hashUtxos`, `hashSequence`, and `hashOutputs`) are excluded from the hashing limit, as implementations should cache these components across signature checks. See [Rationale: Exclusion of Signing Serialization Components from Hashing Limit](rationale.md#exclusion-of-signing-serialization-components-from-hashing-limit).

#### Data Signature Checking Operations

The `OP_CHECKDATASIG` (`0xba`) and `OP_CHECKDATASIGVERIFY` (`0xbb`) operations must compute the expected digest iterations for the length of the message to be hashed, adding the result to the spending transaction's cumulative count. If the new total exceeds the limit, validation fails.

### Replacement of Operation Limit

The existing 201-operation limit per evaluation (A.K.A. `MAX_OPS_PER_SCRIPT`) is removed and replaced by the `Operation Cost Limit`.

<details>
<summary>Note on Zero-Signature, Multisignature Operations</summary>

Note that prior to this proposal, the `OP_CHECKMULTISIG` (`0xae`) and `OP_CHECKMULTISIGVERIFY` (`0xaf`) operations incremented the operation count by the count of public keys, even if zero signatures are checked; because the total operation count of any contract evaluation was limited to 201 operations, the overall density of `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` was very limited.

Following the removal of the operation limit, both `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` are limited only by the `SigChecks` and hashing limits; implementations should ensure that evaluations of `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` requiring zero signature checks are sufficiently performant. See [Rationale: Increased Usability of Multisig Stack Clearing](rationale.md#increased-usability-of-multisig-stack-clearing).

</details>

### Operation Cost Limit

An `Operation Cost Limit` is introduced, limiting transactions to a cumulative operation cost of `800` per spending transaction byte. See [Rationale: Selection of Operation Cost Limit](rationale.md#selection-of-operation-cost-limit).

For each evaluated instruction (including push operations), operation cost is incremented by `100`. See [Rationale: Selection of Base Instruction Cost](rationale.md#selection-of-base-instruction-cost).

#### Measurement of Stack-Pushed Bytes

For all operations which push to the primary stack, operation cost is additionally incremented by the byte length of the pushed stack item. See [Rationale: Limitation of Pushed Bytes](rationale.md#limitation-of-pushed-bytes).

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

All operations which increase the cumulative total of hash digest iterations must simultaneously increase the cumulative operation cost by the product of the additional iterations and the `Hash Digest Iteration Cost`. See [Rationale: Unification of Limits into Operation Cost](rationale.md#unification-of-limits-into-operation-cost).

The `Hash Digest Iteration Cost` is set to `64` for block validation (by consensus) and `192` for transaction relay ("standard") validation. See [Rationale: Selection of Hash Digest Iteration Cost](rationale.md#selection-of-hash-digest-iteration-cost).

#### Signature Checking Operation Cost

All operations which increase the cumulative total of `SigChecks` (as defined by the [2020-05 SigChecks specification](https://gitlab.com/bitcoin-cash-node/bchn-sw/bitcoincash-upgrade-specifications/-/blob/master/spec/2020-05-15-sigchecks.md)) must simultaneously increase the cumulative operation cost by the product of the additional sigchecks and `26000`. See [Rationale: Selection of Signature Verification Operation Cost](rationale.md#selection-of-signature-verification-operation-cost).

### Notice of Possible Future Expansion

While unusual, it is possible to design pre-signed transactions, contract systems, and protocols which rely on the rejection of otherwise-valid transactions that exceed current VM limits. Contract authors are advised that future upgrades may further expand VM limits by increasing allowable operation cost density, reducing the accounted cost of particular operations, or otherwise.

**This proposal interprets such failure-reliant constructions as intentional** – they are designed to fail unless/until a possible future network upgrade in which such limits are increased, e.g. upgrade-activation futures contracts. See [Rationale: Inclusion of "Notice of Possible Future Expansion"](rationale.md#inclusion-of-notice-of-possible-future-expansion).

<details>

<summary>Notes</summary>

As always, the security of a contract is the responsibility of the entity locking funds in that contract; funds can always be locked in insecure contracts (e.g. `OP_DROP OP_1`). This notice is provided to warn contract authors and explicitly codify a network policy: the possible existence of poorly-designed contracts will not preclude future upgrades from further expanding VM limits.

To ensure an otherwise-valid transaction will always fail when some limit is exceeded (in the rare case that such a behavior is desirable), that behavior must be either 1) explicitly validated or 2) introduced to the protocol or contract system in question prior to the activation of any future upgrade which expands the limit.

A similar notice also appeared in [CHIP-2021-03: Bigger Script Integers](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md#notice-of-possible-future-expansion).

</details>

### Rationale

- [Appendix: Rationale &rarr;](rationale.md#rationale)
  - [Retention of Control Stack Limit](rationale.md#retention-of-control-stack-limit)
  - [Hashing Limit by Digest Iterations](rationale.md#hashing-limit-by-digest-iterations)
  - [Selection of Hashing Limit](rationale.md#selection-of-hashing-limit)
  - [Exclusion of Signing Serialization Components from Hashing Limit](rationale.md#exclusion-of-signing-serialization-components-from-hashing-limit)
    - [Ongoing Value of `OP_CODESEPARATOR` Operation](rationale.md#ongoing-value-of-op_codeseparator-operation)
  - [Increased Usability of Multisig Stack Clearing](rationale.md#increased-usability-of-multisig-stack-clearing)
  - [Limitation of Pushed Bytes](rationale.md#limitation-of-pushed-bytes)
  - [Unification of Limits into Operation Cost](rationale.md#unification-of-limits-into-operation-cost)
  - [Selection of Operation Cost Limit](rationale.md#selection-of-operation-cost-limit)
  - [Selection of Base Instruction Cost](rationale.md#selection-of-base-instruction-cost)
  - [Selection of Signature Verification Operation Cost](rationale.md#selection-of-signature-verification-operation-cost)
  - [Continued Availability of Deferred Signature Validation](rationale.md#continued-availability-of-deferred-signature-validation)
  - [Selection of Hash Digest Iteration Cost](rationale.md#selection-of-hash-digest-iteration-cost)
  - [Inclusion of "Notice of Possible Future Expansion"](rationale.md#inclusion-of-notice-of-possible-future-expansion)

## Tests & Benchmarks

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
