# CHIP-2021-05 VM Limits: Targeted Virtual Machine Limits

        Title: Targeted Virtual Machine Limits
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Final
        Initial Publication Date: 2021-05-12
        Latest Revision Date: 2025-05-08
        Version: 3.1.3

<details>

<summary><strong>Table of Contents</strong></summary>

- [Summary](#summary)
- [Motivation](#motivation)
- [Benefits](#benefits)
- [Deployment](#deployment)
- [Technical Summary](#technical-summary)
- [Technical Specification](#technical-specification)
- [Rationale](#rationale)
- [Tests \& Benchmarks](#tests--benchmarks)
- [Implementations](#implementations)
- [Evaluations of Alternatives](#evaluations-of-alternatives)
- [Risk Assessment](#risk-assessment)
- [Stakeholders \& Statements](#stakeholders--statements)
- [Feedback \& Reviews](#feedback--reviews)
- [Acknowledgements](#acknowledgements)
- [Changelog](#changelog)
- [Copyright](#copyright)

</details>

## Summary

This proposal re-targets virtual machine (VM) limits to enable more advanced Bitcoin Cash contracts, reduce transaction sizes, and reduce full node compute requirements.

## Motivation & Benefits

This proposal enables:

- **More advanced contracts** – The 201 opcode limit and 520-byte, Pay-to-Script-Hash (P2SH) contract length limit each raise the cost of developing Bitcoin Cash products by requiring contract authors to remove important features or otherwise complicate products with harder-to-audit, multi-input systems. Replacing these limits reduces the cost of contract development and security audits.

- **Larger stack items** – Re-targeted limits enable new post-quantum cryptographic applications, stronger escrow and settlement strategies, larger hash preimages, more practical zero-knowledge proofs, homomorphic encryption, and other important developments for the future security and competitiveness of Bitcoin Cash.

Additionally, this proposal renders the number length limit (A.K.A. `nMaxNumSize`) unnecessary. Following [cross-implementation verification work](risk-assessment.md#consideration-of-possible-future-changes), an [additional proposal](https://github.com/bitjson/bch-bigint) was created to enable:

- **Simpler, easier-to-audit, high-precision math** – Lowers the cost of developing, maintaining, and auditing contract systems relying on high-precision math: automated market makers, decentralized exchanges, decentralized stablecoins, collateralized loan protocols, cross-chain and sidechain bridges, and other decentralized financial applications. By allowing Bitcoin Cash to offer contracts maximum-efficiency, native math operations, this proposal would significantly reduce transaction sizes, block space usage, and node CPU utilization vs. existing emulated-math solutions.

## Deployment

Deployment of this specification is proposed for the May 2025 upgrade.

- Activation is proposed for `1731672000` MTP, (`2024-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1747310400` MTP, (`2025-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Technical Summary

- The 201 operation limit is removed.
- The 520-byte stack element length limit is raised to 10,000 bytes, a constant equal to the consensus-maximum VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE`) prior to this proposal.
- A new operation cost limit is introduced, limiting contracts to approximately 700 bytes pushed to the stack per spending input byte, with increased costs for hashing, signature checking, and expensive arithmetic operations; constants are derived from the effective limit(s) prior to this proposal.
- A new hashing limit is introduced, limiting contracts to approximately 3.5 hash digest iterations per spending input byte, a constant rounded up from the effective standard limit prior to this proposal, and further limiting standard contracts to approximately 0.5 hash digest iterations per spending input byte, the maximum-known, theoretically-useful hashing density.
- A new control stack limit is introduced, limiting control flow operations (`OP_IF` and `OP_NOTIF`) to a depth of 100, the effective limit prior to this proposal.

This proposal **intentionally avoids modifying other existing properties of the VM**:

- Other existing limits are not modified:
  - Signature operation count (A.K.A. `SigChecks`) density,
  - Maximum cumulative stack and altstack depth (A.K.A. `MAX_STACK_SIZE`; 1000 items),
  - Maximum standard input bytecode length (A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE`; 1,650 bytes),
  - Consensus-maximum VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE`; 10,000 bytes),
  - Maximum standard transaction byte-length (A.K.A. `MAX_STANDARD_TX_SIZE`; 100,000 bytes), and
  - Consensus-maximum transaction byte-length (A.K.A. `MAX_TX_SIZE`; 1,000,000 bytes).
- The cost and incentives around blockchain “data storage” are [not measurably affected](./rationale.md#non-impact-on-data-storage-costs-and-incentives).
- The worst-case processing and memory requirements of the VM are [not measurably affected](./tests-and-benchmarks.md).

### Technical Overview of Changes to C++ Clients

The following overview summarizes all changes proposed by this document to C++ implementations following the general structure of the original Satoshi client, e.g. [Bitcoin Cash Node](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/185f2d64143e807352c0a18f92d8f3ac14bf6840/src/script/interpreter.cpp):

1. `MAX_SCRIPT_ELEMENT_SIZE` increases from `520` to `10000`.
2. `nOpCount` becomes `nOpCost` (used to measure stack-pushed bytes and operation costs)
3. For every push to the stack, increment `nOpCost` by the item length. (See [Operation Costs](./operation-costs.md).)
4. `if (opcode > OP_16 && ++nOpCount > MAX_OPS_PER_SCRIPT) { ... }` becomes `nOpCost += 100;` (not conditional, so also added for unexecuted and push operations).
5. `case OP_ROLL:` adds `nOpCost += depth;`
6. `case OP_AND/OP_OR/OP_XOR:` adds `nOpCost += result.size();`
7. `case OP_1ADD...OP_ABS/OP_ADD...OP_MOD/OP_MIN/OP_MAX:` adds twice the push cost to account for numeric encoding, i.e. `nOpCost += 2 * result.size();`
8. `case OP_MUL/OP_DIV/OP_MOD:` adds an additional `nOpCost += a.size() * b.size();`
9. Hashing operations add `1 + ((message_length + 8) / 64)` to `nHashDigestIterations`, and `nOpCost += 192 * iterations;`.
10. Same for signing operations (count iterations only for the top-level preimage, not `hashPrevouts`/`hashUtxos`/`hashSequence`/`hashOutputs`), plus `nOpCost += 26000 * sigchecks;` (and `nOpCount += nKeysCount;` is removed)
11. `SigChecks` limits remain unchanged; similar density checks apply to `nHashDigestIterations` and `nOpCost`.
12. Adds `if (vfExec.size() > 100) { return set_error(...`

## Technical Specification

The existing **`Stack Element Length Limit`** is raised; two new limits are introduced: a **`Control Stack Limit`** and a **`Hashing Limit`**; and the **`201 Operation Limit`** is replaced by an **`Operation Cost Limit`**.

### Increased Stack Element Length Limit

The existing 520-byte stack element length limit (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) is raised to 10,000 bytes.

<details>
<summary>Note on Maximum Standard Input Bytecode Length</summary>

This increases the maximum length of stack elements to be equal to the maximum allowed VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE` – 10,000 bytes). To narrow the scope of this proposal, the maximum standard input bytecode length (A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE` – 1,650 bytes) is unchanged.

</details>

### Control Stack Limit

To retain the existing limit of `100` on control stack depth, a direct limit is placed on operations which push to the control stack (A.K.A. `vfExec` or `ConditionStack`). See [Rationale: Retention of Control Stack Limit](rationale.md#retention-of-control-stack-limit).

This limit impacts only the `OP_IF` and `OP_NOTIF` operations. For the purpose of enforcing this limit, `OP_IFDUP` is not considered to internally push to the control stack, i.e. an `OP_IFDUP` executed at a depth of `100` does not violate the `Control Stack Limit`.

### Hashing Limit

To prevent excessive hashing function usage, a direct limit is placed on `OP_RIPEMD160` (`0xa6`), `OP_SHA1` (`0xa7`), `OP_SHA256` (`0xa8`), `OP_HASH160` (`0xa9`), `OP_HASH256` (`0xaa`), `OP_CHECKSIG` (`0xac`), `OP_CHECKSIGVERIFY` (`0xad`), `OP_CHECKMULTISIG` (`0xae`), `OP_CHECKMULTISIGVERIFY` (`0xaf`), `OP_CHECKDATASIG` (`0xba`), and `OP_CHECKDATASIGVERIFY` (`0xbb`).

Before a hashing function is performed, its expected cost – in terms of digest iterations – is added to a cumulative total for the transaction input. If the cumulative total for the transaction input exceeds the maximum allowed density, the operation produces an error. See [Rationale: Hashing Limit by Digest Iterations](rationale.md#hashing-limit-by-digest-iterations).

Note that hash digest iterations are cumulative across all evaluation stages of a transaction input: unlocking bytecode, locking bytecode, and redeem bytecode (of P2SH evaluations). This differs from the behavior of the existing operation limit (A.K.A. `nOpCount`), which resets its count to `0` prior to each evaluation stage.

#### Density Control Length

The `Density Control Length` is computed by adding the unlocking bytecode length to the constant `41` – the minimum possible per-input overhead of version `1` and `2` transactions. See [Rationale: Selection of Input Length Formula](rationale.md#selection-of-input-length-formula).

#### Maximum Hashing Density

For standard transactions, the maximum density is `0.5` hash digest iterations per [Density Control Length](#density-control-length) byte. For block validation, the maximum density is `3.5` hash digest iterations per [Density Control Length](#density-control-length) byte. See [Rationale: Selection of Hashing Limit](rationale.md#selection-of-hashing-limit) and [Rationale: Use of Input-Length Based Densities](rationale.md#use-of-input-length-based-densities).

Given the spending input's `unlocking_bytecode_length` (A.K.A. `scriptSig` length), hash digest iteration limits (`0.5` and `3.5`, respectively) may be calculated with the following C functions:

```c
int max_standard_iterations (int unlocking_bytecode_length) {
    return (41 + unlocking_bytecode_length) / 2;
}
int max_consensus_iterations (int unlocking_bytecode_length) {
    return ((41 + unlocking_bytecode_length) * 7) / 2;
}
```

<details>
<summary>Calculate Maximum Digest Iterations in JavaScript</summary>

```js
const maxStandardIterations = (unlockingBytecodeLength) =>
  (41n + unlockingBytecodeLength) / 2n;
const maxConsensusIterations = (unlockingBytecodeLength) =>
  ((41n + unlockingBytecodeLength) * 7n) / 2n;
```

</details>

Note that this formula relies on the transaction input's [Density Control Length](#density-control-length) rather than the precise encoded length of the input. See [Rationale: Selection of Input Length Formula](rationale.md#selection-of-input-length-formula).

#### Digest Iteration Count

The hashing limit caps the number of iterations required by all hashing functions over the course of verifying an input. This places an upper limit on the sum of bytes hashed, including padding.

Given a message length, digest iterations may be calculated with the following C function:

```c
int digest_iterations(int message_length, bool is_double) {
  return 1 + ((message_length + 8) / 64) + (is_double ? 1 : 0);
}
```

Note that the double-hashing operations (`OP_HASH160` and `OP_HASH256`) and all [Transaction Signature Checking Operations](#transaction-signature-checking-operations) (but not the [Data Signature Checking Operations](#data-signature-checking-operations)) perform a final hash digest iteration on the result produced by the initial round of hashing; for these operations, the `is_double` parameter must be set to `true` to increment the final count by one.

<details>
<summary>Calculate Digest Iterations in JavaScript</summary>

```js
const digestIterations = (messageLength, isDouble) =>
  1n + (messageLength + 8n) / 64n + (isDouble ? 1n : 0n);
```

</details>

<details>
<summary>Digest Iteration Count Test Vectors</summary>

These test vectors reflect the required hash digest iterations for a variety of message lengths, **without accounting for double hashing**. For double-hashed messages, the `Digest Iterations` shown must be further incremented by one.

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
| 247                    | 4                 |
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
<summary>Explanation of Digest Iteration Formula</summary>

Each VM-supported hashing algorithm – RIPEMD-160, SHA-1, and SHA-256 – uses a [Merkle–Damgård construction](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) with a 512-bit (64-byte) message block length, so the number of message blocks/digest iterations required for every message length is equal among all VM-supported hashing functions. The specified formula correctly accounts for padding: hashed messages are padded with a `1` bit, followed by enough `0` bits and a 64-bit (8-byte) message length to produce a padded message with a length that is a multiple of 512 bits. Note that even small messages require at least one hash digest iteration, and an additional hash digest iteration is required for each additional 512-bit message block in the padded message.

</details>

#### Hashing Operations

The `OP_RIPEMD160` (`0xa6`), `OP_SHA1` (`0xa7`), `OP_SHA256` (`0xa8`), `OP_HASH160` (`0xa9`), and `OP_HASH256` (`0xaa`) operations must compute the expected digest iterations for the length of the message to be hashed, adding the result to the spending transaction input's cumulative count. If the new total exceeds the hashing limit, validation fails.

Note that evaluations triggering `P2SH20` and `P2SH32` evaluations must also account for the (`2` or more) hash digest iterations required to test validity of the redeem bytecode (`OP_HASH160 <20_bytes> OP_EQUAL` or `OP_HASH256 <32_bytes> OP_EQUAL`, respectively).

<details>
<summary>Note on Two-Round Hashing Operations</summary>

The two-round hashing operations – `OP_HASH160` (`0xa9`) and `OP_HASH256` (`0xaa`) – pass the 32-byte result of their initial SHA-256 hashing round into their second round, each requiring one additional digest iteration beyond the single-round `OP_SHA256`.

</details>

#### Transaction Signature Checking Operations

Before each signature check, the `OP_CHECKSIG` (`0xac`), `OP_CHECKSIGVERIFY` (`0xad`), `OP_CHECKMULTISIG` (`0xae`), and `OP_CHECKMULTISIGVERIFY` (`0xaf`) operations must add the count of hash digest iterations required for the length of the signing serialization (including the iteration in which the final result is double-hashed) to the spending transaction input's cumulative count. If the new total exceeds the hashing limit, validation fails.

Note that the hash digest iteration count is incremented each time a non-null signature is checked, even if a previous signature check used the same signing serialization. See [Rationale: Stateless Costing of Signing Serialization Hashing](./rationale.md#stateless-costing-of-signing-serialization-hashing).

Note that hash digest iterations required to produce components of the signing serialization (i.e. `hashPrevouts`, `hashUtxos`, `hashSequence`, and `hashOutputs`) are excluded from the hashing limit, as implementations should cache these components across signature checks. See [Rationale: Exclusion of Signing Serialization Components from Hashing Limit](rationale.md#exclusion-of-signing-serialization-components-from-hashing-limit).

#### Data Signature Checking Operations

The `OP_CHECKDATASIG` (`0xba`) and `OP_CHECKDATASIGVERIFY` (`0xbb`) operations must compute the expected digest iterations for the length of the message to be hashed, adding the result to the spending transaction input's cumulative count. If the new total exceeds the limit, validation fails.

In counting digest iterations, note that these operations perform only a single round of hashing.

### Operation Cost Limit

An `Operation Cost Limit` is introduced, limiting transaction inputs to a cumulative operation cost of `800` per spending [Density Control Length](#density-control-length) byte. See [Rationale: Selection of Operation Cost Limit](rationale.md#selection-of-operation-cost-limit) and [Rationale: Use of Input-Length Based Densities](rationale.md#use-of-input-length-based-densities).

Given the spending input's unlocking bytecode length (A.K.A. `scriptSig`), the operation cost limit may be calculated with the following C function:

```c
int max_operation_cost (int unlocking_bytecode_length) {
    return (41 + unlocking_bytecode_length) * 800;
}
```

<details>
<summary>Calculate Maximum Operation Cost in JavaScript</summary>

```js
const maxOperationCost = (unlockingBytecodeLength) =>
  (41n + unlockingBytecodeLength) * 800n;
```

</details>

Note that this formula relies on the [Density Control Length](#density-control-length) rather than the precise encoded length of the input. See [Rationale: Selection of Input Length Formula](rationale.md#selection-of-input-length-formula).

### Replacement of 201-Operation Limit with Operation Cost Limit

The existing 201-operation limit per evaluation (A.K.A. `MAX_OPS_PER_SCRIPT`) is removed and replaced by the [`Operation Cost Limit`](#operation-cost-limit).

<details>
<summary>Note on Zero-Signature, Multisignature Operations</summary>

Note that prior to this proposal, the `OP_CHECKMULTISIG` (`0xae`) and `OP_CHECKMULTISIGVERIFY` (`0xaf`) operations incremented the operation count by the count of public keys, even if zero signatures were checked. Since the total operation count of any contract evaluation was limited to 201 operations, the overall density of `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` was restricted by the remaining operation count in applicable contracts.

Following the removal of the operation limit, both `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` are limited only by the `SigChecks` and hashing limits; implementations should ensure that evaluations of `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` requiring zero signature checks are sufficiently performant. See [Rationale: Increased Usability of Multisig Stack Clearing](rationale.md#increased-usability-of-multisig-stack-clearing).

</details>

#### Base Instruction Cost

For each evaluated instruction (including unexecuted and push operations), operation cost is incremented by `100`. See [Rationale: Selection of Base Instruction Cost](rationale.md#selection-of-base-instruction-cost).

#### Cumulative Cost Across Evaluation Stages

Note that operation costs are cumulative across all evaluation stages: unlocking bytecode, locking bytecode, and redeem bytecode (of P2SH evaluations). This differs from the behavior of the existing operation limit (A.K.A. `nOpCount`), which resets its count to `0` prior to each evaluation stage.

#### Measurement of Stack-Pushed Bytes

For all operations which push to the primary stack, operation cost is additionally incremented by the byte length of the pushed stack item. See [Rationale: Limitation of Pushed Bytes](rationale.md#limitation-of-pushed-bytes).

This specification codifies the pushing behavior of each operation based on [`v27.0.0` of Bitcoin Cash Node](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/49ad6a9a95bda543926ec90d07e7bd266581c4d0/src/script/interpreter.cpp), an implementation written in 2008 by Satoshi Nakamoto and continuously improved by various contributors. Counting of pushed bytes are consistent with this implementation with the exception of `OP_ROLL` and the bitwise operations (`OP_AND`, `OP_OR`, and `OP_XOR`). See [Operation Cost by Operation](/operation-costs.md).

##### Bitwise operations

In addition to the aforementioned costs, the bitwise operations (`OP_AND`, `OP_OR`, and `OP_XOR`) are considered to push their result, even if an implementation performs bitwise operations in-place (e.g. for performance reasons).

##### OP_ROLL

In addition to the aforementioned costs, the operation cost of `OP_ROLL` is incremented by the numeric value indicating the depth of the roll, between `0` and `999` (the maximum-depth roll).

<details>
<summary>OP_ROLL Cost Example</summary>

For example, `<'a'> <'b'> <'c'> <2> OP_ROLL` (producing `<'b'> <'c'> <'a'>`) is incremented by `2` for a total cost of `100` (the base instruction cost), plus `1` (the byte length of `'a'`), plus `2` (the roll depth): `103`.

</details>

#### Arithmetic Operation Cost

To account for the cost of encoding VM numbers, the sum of all numeric output lengths with the potential to exceed `2**32` are added to the operation cost of all operations with such outputs: `OP_1ADD` (`0x8b`), `OP_1SUB` (`0x8c`), `OP_NEGATE` (`0x8f`), `OP_ABS` (`0x90`), `OP_ADD` (`0x93`), `OP_SUB` (`0x94`), `OP_MUL` (`0x95`), `OP_DIV` (`0x96`), `OP_MOD` (`0x97`), `OP_MIN` (`0xa3`), and `OP_MAX` (`0xa4`); e.g. given terms `a b -> c` (such as in `<a> <b> OP_ADD`), the operation cost is: the base cost (`100`), plus the cost of re-encoding the output (`c.length`), plus the byte length of the result (`c.length`), for a final formula of `100 + (2 * c.length)`. See [Rationale: Inclusion of Numeric Encoding in Operation Costs](rationale.md#inclusion-of-numeric-encoding-in-operation-costs).

To account for <code>O(n<sup>2</sup>)</code> worst-case performance, the operation cost of `OP_MUL` (`0x95`), `OP_DIV` (`0x96`), and `OP_MOD` (`0x97`) are also increased by the product of their input lengths, i.e. given terms `a b -> c`, the operation cost is: the base cost (`100`) plus the cost of re-encoding the output (`c.length`), plus the byte length of the result (`c.length`), plus the product of the input lengths (`a.length * b.length`), for a final formula of `100 + (2 * c.length) + (a.length * b.length)`.

#### Hash Digest Iteration Cost

All operations which increase the cumulative total of hash digest iterations must simultaneously increase the cumulative operation cost by the product of the additional iterations and the `Hash Digest Iteration Cost`. See [Rationale: Unification of Limits into Operation Cost](rationale.md#unification-of-limits-into-operation-cost).

The `Hash Digest Iteration Cost` is set to `64` for block validation (by consensus) and `192` for transaction relay ("standard") validation. See [Rationale: Selection of Hash Digest Iteration Cost](rationale.md#selection-of-hash-digest-iteration-cost).

#### Signature Checking Operation Cost

All operations which increase the cumulative total of `SigChecks` (as defined by the [2020-05 SigChecks specification](https://gitlab.com/bitcoin-cash-node/bchn-sw/bitcoincash-upgrade-specifications/-/blob/master/spec/2020-05-15-sigchecks.md)) must simultaneously increase the cumulative operation cost by the product of the additional sigchecks and `26000`. See [Rationale: Selection of Signature Verification Operation Cost](rationale.md#selection-of-signature-verification-operation-cost).

### Notice of Possible Future Expansion

While unusual, it is possible to design pre-signed transactions, contract systems, and protocols which rely on the rejection of otherwise-valid transactions that exceed current VM limits. Contract authors are advised that future upgrades may further expand VM limits by increasing allowable operation cost density, reducing the accounted cost of particular operations, or otherwise.

**This proposal interprets such failure-reliant constructions as intentional** – the constructions are designed to fail unless/until a possible future network upgrade in which such limits are increased, e.g. upgrade-activation futures contracts. See [Rationale: Inclusion of "Notice of Possible Future Expansion"](rationale.md#inclusion-of-notice-of-possible-future-expansion).

<details>

<summary>Notes</summary>

The security of a contract is the responsibility of the entity locking funds in that contract; funds can always be locked in insecure contracts (e.g. "Anyone-Can-Spend addresses"). This notice is provided to warn contract authors and explicitly codify a network policy: the possible existence of poorly-designed contracts will not preclude future upgrades from further expanding VM limits.

To ensure an otherwise-valid transaction will always fail when some limit is exceeded (in the rare case that such a behavior is desirable), that behavior must be either 1) explicitly validated or 2) introduced to the protocol or contract system in question prior to the activation of any future upgrade which expands the requisite limit.

A similar notice also appeared in [CHIP-2021-03: Bigger Script Integers](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md#notice-of-possible-future-expansion).

</details>

## Rationale

- [Appendix: Rationale &rarr;](rationale.md#rationale)
  - [Use of Explicitly-Defined Density Limits](rationale.md#use-of-explicitly-defined-density-limits)
  - [Exclusion of "Gas System" Behaviors](rationale.md#exclusion-of-gas-system-behaviors)
    - [Global-State Validation Architectures](rationale.md#global-state-validation-architectures)
    - [Stateless Validation Architecture](rationale.md#stateless-validation-architecture)
    - [Minimal Impact of Compute on Node Operation Costs](rationale.md#minimal-impact-of-compute-on-node-operation-costs)
  - [Non-Impact on "Data Storage" Costs and Incentives](rationale.md#non-impact-on-data-storage-costs-and-incentives)
    - [`OP_RETURN` Data Carrier Outputs](rationale.md#op_return-data-carrier-outputs)
    - [Data-Carrying Transactions](rationale.md#data-carrying-transactions)
    - [Indirect, Transaction "Padding" Incentives](rationale.md#indirect-transaction-padding-incentives)
  - [Retention of Control Stack Limit](rationale.md#retention-of-control-stack-limit)
  - [Use of Input Length-Based Densities](rationale.md#use-of-input-length-based-densities)
    - [Alternative: Transaction Length-Based Densities](rationale.md#alternative-transaction-length-based-densities)
    - [Alternative: UTXO-Length Increased Densities](rationale.md#alternative-utxo-length-increased-densities)
    - [Alternative: Transaction Fee-Increased Densities](rationale.md#alternative-transaction-fee-increased-densities)
  - [Selection of Input Length Formula](rationale.md#selection-of-input-length-formula)
  - [Hashing Limit by Digest Iterations](rationale.md#hashing-limit-by-digest-iterations)
  - [Selection of Hashing Limit](rationale.md#selection-of-hashing-limit)
  - [Exclusion of Signing Serialization Components from Hashing Limit](rationale.md#exclusion-of-signing-serialization-components-from-hashing-limit)
    - [Ongoing Value of OP_CODESEPARATOR Operation](rationale.md#ongoing-value-of-op_codeseparator-operation)
  - [Increased Usability of Multisig Stack Clearing](rationale.md#increased-usability-of-multisig-stack-clearing)
  - [Limitation of Pushed Bytes](rationale.md#limitation-of-pushed-bytes)
  - [Unification of Limits into Operation Cost](rationale.md#unification-of-limits-into-operation-cost)
  - [Selection of Operation Cost Limit](rationale.md#selection-of-operation-cost-limit)
  - [Selection of Base Instruction Cost](rationale.md#selection-of-base-instruction-cost)
  - [Inclusion of Numeric Encoding in Operation Costs](rationale.md#inclusion-of-numeric-encoding-in-operation-costs)
  - [Selection of Signature Verification Operation Cost](rationale.md#selection-of-signature-verification-operation-cost)
  - [Continued Availability of Deferred Signature Validation](rationale.md#continued-availability-of-deferred-signature-validation)
  - [Selection of Hash Digest Iteration Cost](rationale.md#selection-of-hash-digest-iteration-cost)
  - [Inclusion of "Notice of Possible Future Expansion"](rationale.md#inclusion-of-notice-of-possible-future-expansion)

## Tests & Benchmarks

This proposal includes a suite of functional tests and benchmarks to verify the performance of all operations within virtual machine implementations.

- [Appendix: Tests & Benchmarks &rarr;](tests-and-benchmarks.md#rationale)
  - [Testing](tests-and-benchmarks.md#testing)
  - [Benchmarks](tests-and-benchmarks.md#benchmarks)
  - [Evaluation of Results](tests-and-benchmarks.md#evaluation-of-results)

## Implementations

Please see the following reference implementations for additional examples and test vectors:

- C++:
  - [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/) – A professional, miner-friendly node that solves practical problems for Bitcoin Cash. [Merge Request !1874](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1874).
- JavaScript/TypeScript
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Pull Request #139](https://github.com/bitauth/libauth/pull/139).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Pull Request #101](https://github.com/bitauth/bitauth-ide/pull/101).
- Go:
  - [BCHD](https://bchd.cash/) – An alternative full node bitcoin cash implementation written in Go (golang). [OPReturnCode/bchd PR #1](https://github.com/OPReturnCode/bchd/pull/1)
- Java:
  - [Bitcoin Verde](https://bitcoinverde.org/) – Bitcoin Verde is a Java full-node implementation of the Bitcoin Cash protocol. Bitcoin Verde provides a block explorer, development library, and network implementation diversification. [Issue #24](https://github.com/SoftwareVerde/bitcoin-verde/issues/24)

## Evaluations of Alternatives

This proposal evaluates notable alternatives for each design decision made in the technical specification. Reviewed alternatives are enumerated here for ease of review:

- **Statically-Defined, Per-Input Limits** – Rather than this proposal's explicitly-specified limits, an alternative limits system could attempt to indirectly limit worst-case validation costs by specifying fixed per-input constant limits. This alternative would 1) create a minimum of ~244x variability in worst-case validation costs and 2) obfuscate the true limits such that worst-case costs are not predictable without exhaustive enumeration of optimally-packed attack cases. See [Rationale: Use of Explicitly-Defined Density Limits](./rationale.md#use-of-explicitly-defined-density-limits) and [Rationale: Use of Density-Based Limits](./rationale.md#use-of-density-based-limits) for details.

- **Financial Limitation of Computation (e.g. a "Gas" System)** – This proposal does not attempt to introduce a separate "Gas" system of pricing on computation because Bitcoin Cash already has a many orders-of-magnitude scalability advantage over global-state systems like Ethereum. Beyond preventing denial-of-service risks, Bitcoin Cash has no pressing need to measure or charge fees for computation. See [Rationale: Exclusion of "Gas System" Behaviors](./rationale.md#exclusion-of-gas-system-behaviors).

- **Unbounded or Contract-Length Bounded Control Stack Depth** – This proposal preserves the existing practical limit on Control Stack depth. Alternatively, this proposal could increase or remove this limit, requiring all implementations to carefully implement a particular optimization and potentially complicating future upgrades. See [Rationale: Retention of Control Stack Limit](./rationale.md#retention-of-control-stack-limit).

- **Transaction Length Based Densities** – Rather than basing density limits on input length, the density limit could be shared across the entire transaction. While safely allowing for higher limits in some cases, this approach would prevent contract authors from reliably predicting available limits in some cases, increase protocol complexity of parallelized validation, and increase the worst-case performance of maliciously-invalid transactions. See [Rationale: Use of Density-Based Limits](./rationale.md#use-of-density-based-limits) for details.

- **UTXO Length Addition to Density Control Length** – In addition to the computation limits afforded to contracts by this proposal, further allowance could be made based on the byte-length of the spent UTXO. However, this would increase the volatility of worst-case transaction validation and potentially incentivize would-be attackers to prepare attacks by first inflating the UTXO set. See [Rationale: Use of Density-Based Limits](./rationale.md#use-of-density-based-limits) for details.

- **Encoded Input Length Based Densities** – This proposal avoids tightly associating precise encoding semantics with the limit system by using an always-safe constant (`41`) rather than requiring implementations to measure or calculate the expected encoded length of inputs, simplifying implementations and eliminating a potential pitfall for contract developers. See [Rationale: Selection of Input Length Formula](./rationale.md#selection-of-input-length-formula).

- **Byte-Length Based Hashing Limits** – This proposal carefully limits hashing based on the internal number of hash digest iterations performed rather than a more naive, byte count-based limit, preventing an up to 55x magnification in worst-case validation costs. See [Rationale: Hashing Limit by Digest Iterations](./rationale.md#hashing-limit-by-digest-iterations).

- **Consensus-Only Hashing Limit** – This proposal establishes a lower, standard hashing limit at the asymptotic maximum density of hashing operations in plausibly non-malicious, standard transactions. While this standardness limit could be omitted, its inclusion improves the worst-case performance of Bitcoin Cash transaction and block validation by nearly an order of magnitude. See [Rationale: Selection of Hashing Limit](/rationale.md#selection-of-hashing-limit).

- **Internal Hashing Limits Within Signing Serializations** – Alternatively, this proposal could also require internally accounting for the cost of hashing within signing serialization components. However, such costs would need to be amortized across all of a transaction's signature checks to approximate their fixed, real-world cost. This would needlessly increasing protocol complexity, given both the ease of component caching and the presence of other limits on signature checking. See [Rationale: Exclusion of Signing Serialization Components from Hashing Limit](./rationale.md#exclusion-of-signing-serialization-components-from-hashing-limit).

- **Additional Limitations on `OP_CODESEPARATOR`** – If this proposal were to omit hashing limitations on signing serializations, additional limitations would be required to prevent abuse of `OP_CODESEPARATOR`. As `OP_CODESEPARATOR` remains useful, additional limitations would increase overall protocol complexity, and limitation of signing serialization hashing is otherwise prudent, this proposal instead avoid singling-out `OP_CODESEPARATOR` for special limitation. See [Rationale: Exclusion of Signing Serialization Components from Hashing Limit](./rationale.md#exclusion-of-signing-serialization-components-from-hashing-limit).

- **Additional Limitations on `OP_CHECKMULTISIG*`** – Because operation count currently limits the eccentric use of `OP_CHECKMULTISIG` for stack-dropping behavior, this proposal could attempt to place new restrictions on `OP_CHECKMULTISIG` to prevent expanded usage. However, because the unusual feature has been available to contract authors since Bitcoin Cash's 2009 launch, remains useful, and does not impact validation costs, this proposal does not attempt to apply new limitations for this case. See [Increased Usability of Multisig Stack Clearing](./rationale.md#increased-usability-of-multisig-stack-clearing).

- **Continuous Tracking of Total Stack Usage** – Instead of simply tracking total stack-pushed bytes, this proposal could limit memory usage by continuously tracking total usage and enforcing some maximum limit. However, this would increase implementation complexity, only implicitly limit memory bandwidth usage, and require additional limitations on linear-time operations. See [Rationale: Limitation of Pushed Bytes](./rationale.md#limitation-of-pushed-bytes).

- **Omit Base Instruction Cost** – this proposal could alternatively omit Base Instruction Cost, limiting the cost of all instructions to their impact on the stack. However, instruction evaluation is not costless – a nonzero base cost properly accounts for the real world overhead of evaluating an instruction and verifying non-violation of applicable limits. See [Rationale: Selection of Base Instruction Cost](./rationale.md#selection-of-base-instruction-cost).

- **Omit Numeric Encoding Cost** – this proposal could alternatively omit the cost of numeric encoding from [arithmetic operation cost](#arithmetic-operation-cost). However, if this proposal were to assume zero operation cost for encoding/decoding, this optimization would be required of all performance-critical VM implementations to avoid divergence of real performance from measured operation cost. See [Rationale: Inclusion of Numeric Encoding in Operation Costs](./rationale.md#inclusion-of-numeric-encoding-in-operation-costs).

## Risk Assessment

- [Appendix: Risk Assessment &rarr;](risk-assessment.md#risk-assessment)
  - [Risks \& Security Considerations](risk-assessment.md#risks--security-considerations)
    - [User Impact Risks](risk-assessment.md#user-impact-risks)
      - [Reduced or Equivalent Node Validation Costs](risk-assessment.md#reduced-or-equivalent-node-validation-costs)
      - [Increased or Equivalent Contract Capabilities](risk-assessment.md#increased-or-equivalent-contract-capabilities)
    - [Consensus Risks](risk-assessment.md#consensus-risks)
      - [Full-Transaction Test Vectors](risk-assessment.md#full-transaction-test-vectors)
      - [New Performance Testing Methodology](risk-assessment.md#new-performance-testing-methodology)
      - [`Chipnet` Preview Activation](risk-assessment.md#chipnet-preview-activation)
    - [Denial-of-Service (DoS) Risks](risk-assessment.md#denial-of-service-dos-risks)
      - [Expanded Node Performance Safety Margin](risk-assessment.md#expanded-node-performance-safety-margin)
    - [Protocol Complexity Risks](risk-assessment.md#protocol-complexity-risks)
      - [Support for Post-Activation Simplification](risk-assessment.md#support-for-post-activation-simplification)
      - [Continuation of Nonstandard Invalidation Precedent](risk-assessment.md#continuation-of-nonstandard-invalidation-precedent)
      - [Consideration of Possible Future Changes](risk-assessment.md#consideration-of-possible-future-changes)
  - [Upgrade Costs](risk-assessment.md#upgrade-costs)
    - [Node Upgrade Costs](risk-assessment.md#node-upgrade-costs)
    - [Ecosystem Upgrade Costs](risk-assessment.md#ecosystem-upgrade-costs)
  - [Maintenance Costs](risk-assessment.md#maintenance-costs)
    - [Node Maintenance Costs](risk-assessment.md#node-maintenance-costs)
    - [Ecosystem Maintenance Costs](risk-assessment.md#ecosystem-maintenance-costs)

## Stakeholders & Statements

[Stakeholder Responses & Statements &rarr;](stakeholders.md)

## Feedback & Reviews

- [`Raising the 520 byte push limit & 201 operation limit` – Feb 8, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/raising-the-520-byte-push-limit-201-operation-limit/282)
- [`CHIP: Targeted Virtual Machine Limits` – May 12, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/chip-targeted-virtual-machine-limits/437)
- [The BCH Podcast #122: VM Limits feat. Jason Dreyzehner](https://www.youtube.com/watch?v=Gy7uxpcyIl8)
- [Jason Dreyzehner on X: I proposed the Limits & BigInt CHIPs, Ask Me Anything](https://x.com/bitjson/status/1839361677759811611)
- [On r/btc: I proposed the Limits & BigInt CHIPs, Ask Me Anything –bitjson](https://old.reddit.com/r/btc/comments/1fq2hab/i_proposed_the_limits_bigint_chips_for_the_may/)
- [The Bitcoin Cash Podcast #130: Limits & BigInt CHIPs feat. Jason Dreyzehner](https://www.youtube.com/watch?v=dTMcgW1iHrU)
- [General Protocol Spaces (34): General Bull 34 of n - Jason Dreyzehner AMA](https://www.youtube.com/watch?v=xNQK3Ula2Ao)

## Acknowledgements

Thank you to the following contributors for reviewing and contributing improvements to this proposal, providing feedback, and promoting consensus among stakeholders:
[Calin Culianu](https://github.com/cculianu), [bitcoincashautist](https://github.com/A60AB5450353F40E), [Andrew#128](https://gitlab.com/andrew-128), [Fernando Pelliccioni](https://gitlab.com/fpelliccioni), [Mathieu Geukens](https://github.com/mr-zwets), [Joshua Green](https://github.com/joshmg), [OPReturnCode](https://github.com/OPReturnCode), [Jeremy](https://bitcoincashpodcast.com/), [Kallisti.cash](https://kallisti.io), [Corbin Fraser](https://corbinfraser.com/), [imaginary_username](https://gitlab.com/im_uname), [John Nieri](https://gitlab.com/emergent-reasons), [Jonathan Silverblood](https://gitlab.com/monsterbitar), [Josh Ellithorpe](https://github.com/zquestz), [John Moriarty](https://x.com/BitcoinOutLoud), [minisatoshi](https://minisatoshi.cash/), [Andrew Groot](https://github.com/thesquaregroot), [Tom Zander](https://github.com/zander), [Rosco Kalis](https://github.com/rkalis), [Richard Brady](https://github.com/rnbrady).

## Changelog

This section summarizes the evolution of this document.

- **v3.1.3 – 2025-05-08**
  - Clarify and correct technical overview ([#43](https://github.com/bitjson/bch-vm-limits/issues/43))
- **v3.1.2 – 2024-11-11** ([`ac0af7c6`](https://github.com/bitjson/bch-vm-limits/commit/ac0af7c6fb957cc8077089a598469740c563812b))
  - Clarify costing of signing serialization hashing ([#38](https://github.com/bitjson/bch-vm-limits/issues/38))
  - Correct list of operations with numeric outputs ([#39](https://github.com/bitjson/bch-vm-limits/issues/39))
- **v3.1.1 – 2024-10-01** ([`6b87d517`](https://github.com/bitjson/bch-vm-limits/commit/6b87d517081f2fdba6a50b8e7fb9147321def609))
  - Add [overview of benchmarking process and results](./tests-and-benchmarks.md)
  - Add [Risk Assessment](./risk-assessment.md)
  - Add latest test vectors and benchmarks ([#7](https://github.com/bitjson/bch-vm-limits/issues/7))
  - Add [Rationale: Use of Explicitly-Defined Density Limits](./rationale.md#use-of-explicitly-defined-density-limits) ([#28](https://github.com/bitjson/bch-vm-limits/issues/28))
  - Clarify explanation of hash digest iteration formula ([#29](https://github.com/bitjson/bch-vm-limits/issues/29))
  - Add [Rationale: Non-Impact on "Data Storage" Costs and Incentives](rationale.md#non-impact-on-data-storage-costs-and-incentives) ([#18](https://github.com/bitjson/bch-vm-limits/issues/18))
- **v3.1.0 – 2024-09-02** ([`35dc2c52`](https://github.com/bitjson/bch-vm-limits/commit/35dc2c5210bb34dc5255c0613a6665edff07d6c0))
  - Base densities on input length rather than transaction length ([#21](https://github.com/bitjson/bch-vm-limits/issues/21))
  - Include numeric encoding in operation costs ([#20](https://github.com/bitjson/bch-vm-limits/issues/20))
- **v3.0.1 – 2024-08-13** ([`929ef37`](https://github.com/bitjson/bch-vm-limits/commit/929ef37c6d5fb14736a62c3123904d80efc59b80))
  - Correct and clarify operation cost table ([#17](https://github.com/bitjson/bch-vm-limits/issues/17))
  - Clarify more differences between existing and upgraded behavior
- **v3.0.0 – 2024-08-06** ([`4eba48ea`](https://github.com/bitjson/bch-vm-limits/commit/4eba48ea4648a5ad39f40ff11bfebbe3459fca83))
  - Revise limits to be density based ([#8](https://github.com/bitjson/bch-vm-limits/issues/8))
  - Limit bytes pushed to the stack ([#10](https://github.com/bitjson/bch-vm-limits/issues/10))
  - Limit depth of control stack ([#11](https://github.com/bitjson/bch-vm-limits/issues/11))
  - Note accounting of P2SH redeem bytecode digest iterations ([#14](https://github.com/bitjson/bch-vm-limits/issues/14))
- **v2.0.0 – 2024-02-03** ([`e686b981`](https://github.com/bitjson/bch-vm-limits/commit/e686b981e14f9f69433682dda9df8b204c66b709))
  - Simplify stack memory limit calculation ([#6](https://github.com/bitjson/bch-vm-limits/pull/6))
  - Correct hashing benchmarks, update hashing limit ([#6](https://github.com/bitjson/bch-vm-limits/pull/6))
  - Propose for May 2025 Upgrade
- **v1.0.0 – 2021-05-12** ([`5b24b0ec`](https://github.com/bitjson/bch-vm-limits/commit/ba2785b1f38bdecd8d72d5236c31c6846165c141))
  - Initial publication

## Copyright

This document is placed in the public domain.
