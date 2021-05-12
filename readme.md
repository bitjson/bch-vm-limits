# CHIP-2021-05-vm-limits: Targeted Virtual Machine Limits

        Title: Targeted Virtual Machine Limits
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2021-05-12
        Latest Revision Date: 2021-05-12

## Summary

This proposal replaces several poorly-targeted virtual machine (VM) limits with alternatives which protect against the same malicious cases while significantly increasing the power of the Bitcoin Cash contract system:

- the 520-byte stack element size limit is raised to 10,000 bytes.
- The 201 operation limit is removed.
- A new cumulative hashing limit is introduced, limiting contracts to 660 digest iterations per evaluation.
- A new stack memory usage limit is introduced, limiting contracts to 130,000 bytes of data on the stack.

This proposal **intentionally avoids modifying other properties of the current VM**:

- The limits on maximum standard input bytecode length (A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE` – 1,650 bytes), maximum VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE` – 10,000 bytes), maximum standard transaction byte-length (A.K.A. `MAX_STANDARD_TX_SIZE` – 100,000 bytes) and consensus-maximum transaction byte-length (A.K.A. `MAX_TX_SIZE` – 1,000,000 bytes) are not modified.
- The cost and incentives around blockchain “data storage” are not measurably affected.
- The worst-case transaction validation processing and memory requirements of the VM are not measurably affected.

## Deployment

Deployment of this specification is proposed for the May 2022 upgrade.

## Motivation

Bitcoin Cash contracts are strictly limited to prevent maliciously-designed transactions from requiring excessive resources during transaction validation. Two of these limits are poorly-targeted, unintentionally preventing valuable use cases.

- The **520-byte stack element size limit** (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) currently prevents items longer than 520 bytes from being pushed, `OP_NUM2BIN`ed, or `OP_CAT`ed on to the stack.
- The **201 operation limit** prevents contracts with more than 201 non-push operations from being broadcasted on the network or mined blocks.

Together, these limits:

- [cap maximum stack memory usage to ~130,000 bytes](#stack-memory-usage-limit), and
- cap the maximum processing requirements of the VM to [660 digest iterations of SHA-256 hashing](#hashing-limit).

While these limits have been sufficient to prevent Denial of Service (DOS) attacks by capping the maximum cost of transaction validation, their current design prevents many valuable contract use cases.

## Benefits

By replacing these limits with better-tuned alternatives, the Bitcoin Cash contract system can be made more powerful without sacrificing node validation performance.

### Enables More Advanced Contracts

The 201 operation limit prevents significant innovation by forcing contracts to limit functionality. The 520-byte stack element size limit prevents additional use cases because Pay-To-Script-Hash (P2SH) contracts must push their "redeem" bytecode to the stack prior to evaluation. The 520-byte stack element size limit also prevents the use of larger hash preimages in contracts which inspect parent transactions or utilize `OP_CHECKDATASIG`.

By increasing these limits, more advanced contracts can be supported.

### Enables More Efficient Contracts

The 520-byte limit sometimes forces contract authors to design less byte-efficient contracts in order to fit contract code into 520 bytes; rather than embedding small data elements directly in the contract code, authors are forced to pick and validate data from the unlocking bytecode, wasting transaction space. By removing this limit on P2SH contracts, many contracts can be made more efficient.

## Costs & Risk Mitigation

The following costs and risks have been assessed.

### Modification to Transaction Validation

Modifications to VM limits have the potential to increase worst-case transaction validation costs and expose VM implementations to Denial of Service (DOS) attacks.

**Mitigations**: this proposal [includes a series of benchmarks](#benchmarks) which measure worst-case validation performance and memory usage for contracts before and after the proposed upgrade. The newly proposed limits have been chosen to target performance and memory usage equivalent to the existing limits.

### Node Upgrade Costs

This proposal affects consensus – all fully-validating node software must implement these VM changes to remain in consensus.

These VM changes are backwards-compatible: all past and currently-possible transactions remain valid under these new rules.

### Ecosystem Upgrade Costs

Because this proposal only affects internals of the VM, standard wallets, block explorers, and other services will not require software modifications for these changes. Only software which offers emulation of VM evaluation (e.g. [Bitauth IDE](https://github.com/bitauth/bitauth-ide)) will be required to upgrade.

Wallets and other services may also upgrade to add support for new contracts which will become possible after deployment of this proposal.

## Technical Specification

The existing **`Stack Element Size Limit`** is modified, the **`Operation Limit`** is removed, and two new limits are introduced: a **`Hashing Limit`** and a **`Stack Memory Usage Limit`**.

### Increased Stack Element Size Limit

The existing 520-byte stack element size limit (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) is raised to 10,000 bytes.

> Note: this increases the maximum size of stack elements to be equal to the maximum allowed VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE`).

### Removal of Operation Limit

The existing 201-operation limit per evaluation (A.K.A. `MAX_OPS_PER_SCRIPT`) is removed.

> Note: after activation of this proposal, operation count no longer requires tracking in the VM. The below `Hashing Limit` and `Stack Memory Limit` cover all pathological constructions which might be prevented by the operation limit.

### Hashing Limit

A new limit is placed on `OP_RIPEMD160` (`0xa6`), `OP_SHA1` (`0xa7`), `OP_SHA256` (`0xa8`), `OP_HASH160` (`0xa9`), `OP_HASH256` (`0xaa`), `OP_CHECKDATASIG` (`0xba`), and `OP_CHECKDATASIGVERIFY` (`0xbb`) to prevent excessive hashing function usage.

Before any hashing function is called, the expected cost of the call – in terms of digest iterations – is added to a cumulative total for the evaluation. If the total required digest iterations exceeds `660`, the operation produces an error.

<details>
 <summary><b>Selection of <code>660</code> Digest Iteration Maximum</b></summary>

At a Maximum Digest Iteration of `660`, this `Hashing Limit` effectively replaces the existing `Stack Element Size Limit` and `Operation Limit`.

As of May 2021, the most computationally-expensive known contract is the [`Pre-Deployment Hashing Limit Benchmark`](#pre-deployment-hashing-limit-benchmark), which requires 3 operations to produce 10 SHA-256 digest iterations:

```
<0> <520> OP_NUM2BIN OP_HASH256 OP_DROP  // bytecode:   0x0002080280aa75
<1> <520> OP_NUM2BIN OP_HASH256 OP_DROP  // bytecode:   0x5102080280aa75
...
<65> <520> OP_NUM2BIN OP_HASH256 OP_DROP // bytecode: 0x014202080280aa75
```

Each `<N> <520> OP_BIN2NUM OP_HASH256 OP_DROP` line requires between 7-8 bytes and 3 operations. In each line, the first round of SHA-256 requires 9 digest iterations, and the second round of SHA-256 operates on the 32-byte result of the first, requiring one additional digest iteration. At a limit of 201 operations, 66 such lines of 10 digest iterations can be included in a valid contract, totalling 660 digest iterations.

</details>

#### Digest Iteration Count

The hashing limit caps the count of `message blocks` which may be processed by any hashing function within a single evaluation. This places an upper limit on the sum of bytes hashed (including padding).

Given a message length, digest iterations may be calculated with the following C function:

```c
int digest_iterations (int message_length) {
  return 1 + ((message_length + 8) / 64);
}
```

<details>
 <summary>JS</summary>

```js
const digestIterations = (messageLength) =>
  1 + (((messageLength + 8) / 64) | 0);
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
| 503                    | 8                 |
| 504                    | 9                 |
| 1015                   | 16                |
| 1016                   | 17                |
| 42231                  | 660               |
| 42232                  | 661               |

</details>

> Note: each VM-supported hashing algorithm – RIPEMD-160, SHA-1, and SHA-256 – uses a [Merkle–Damgård construction](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) with a `block size` of 512 bits (64 bytes), so the number of message blocks/digest iterations required for every message size is equal among all VM-supported hashing functions.

#### Single-Round Hashing Operations

Prior to calling their respective hashing functions, the `OP_RIPEMD160` (`0xa6`), `OP_SHA1` (`0xa7`), and `OP_SHA256` (`0xa8`) operations must compute the expected digest iterations for the length of the message to be hashed, adding to the evaluation's total digest iterations. If the new total exceeds the limit, the operation produces an error.

#### Double-Round Hashing Operations

Prior to calling their respective hashing functions, the `OP_HASH160` (`0xa9`), `OP_HASH256` (`0xaa`) operations must compute the expected digest iterations for the length of the message to be hashed.

> Note: double-round hashing operations pass the 32-byte result of their initial SHA-256 hashing round into their second round, requiring one additional digest iteration beyond the single-round `OP_SHA256`.

If the total of digest iterations exceeds the limit, the operation produces an error.

#### `OP_CHECKDATASIG` and `OP_CHECKDATASIGVERIFY` Operations

Prior to calling their internal SHA-256 operation, the `OP_CHECKDATASIG` and `OP_CHECKDATASIGVERIFY` operations must compute the expected digest iterations for the length of the message to be hashed. If the new total exceeds the limit, the operation produces an error.

### Stack Memory Usage Limit

A new limit is placed on all operations which push or modify elements on the stack: after each operation, the total cumulative byte-length of all stack elements may not exceed `130,000` bytes.

<details>
 <summary><b>Selection of <code>130,000</code> Byte Stack Memory Usage Limit</b></summary>

Prior to the this proposal, stack memory usage is limited by a combination of the (520-byte) stack element size limit and the (201) operation limit. The following contract demonstrates the largest possible (standard P2SH) memory consumption:

```
OP_1
<520-byte push>
<520-byte push>
<201-byte redeem bytecode>
--- // 201-byte P2SH lock:
OP_2DUP OP_3DUP OP_3DUP     // 10x 520-byte elements on the stack
OP_3DUP [...x77]            // 241x 520-byte elements on the stack
OP_2DROP [...x120]
OP_DROP                     // `0x01` on stack
```

At peak, this contract requires `125,320` bytes of memory to hold the raw bytecode contents of `242` stack items. This value is rounded up to the `130,000` constant to avoid invalidating any possible unspent P2SH transaction outputs.

</details>

#### Computing Stack Memory Usage

For consensus purposes, `stack memory usage` is the total byte-length of all stack element contents plus the count of stack elements.

For example, a stack of `['', '01', '0203']` contains 3 elements: element `0` is empty, element `1` contains one byte (`0x01`), and element `2` contains two bytes (`0x0203`). The memory usage of this stack is `6`: `3` bytes hold element values, and the stack depth is `3`.

> Implementation note: computing stack memory usage can become expensive in poorly-optimized stack implementations. To avoid Denial of Service (DOS) vulnerabilities, implementations should ensure that their [worst-case stack memory usage computation](#benchmark-stack-memory-check-at-maximum-depth) remains insignificant when compared with [worst-case hashing performance](#pre-deployment-hashing-limit-benchmark).

### Benchmarks

The following contracts are provided for testing worst-case implementation performance.

#### Pre-Deployment Hashing Limit Benchmark

This benchmark measures the worst-known transaction validation performance prior to deployment of this specification. This contract can also be evaluated by spending a standard P2SH output, and does not require miner participation.

```
<0> <520> OP_NUM2BIN OP_HASH256 OP_DROP  // bytecode:   0x0002080280aa75
<1> <520> OP_NUM2BIN OP_HASH256 OP_DROP  // bytecode:   0x5102080280aa75
...
<65> <520> OP_NUM2BIN OP_HASH256 OP_DROP // bytecode: 0x014202080280aa75
```

#### Post-Deployment Benchmarks

To remove the operation limit without affecting the worst-case transaction validation performance status quo, the following benchmarks must be validated faster than `Pre-Deployment Hashing Limit Benchmark`.

##### Post-Deployment Hashing Limit Benchmark

This is the post-deployment equivalent of `Pre-Deployment Hashing Limit Benchmark`:

```
<0> <10000> OP_NUM2BIN OP_HASH256 OP_DROP
<1> <10000> OP_NUM2BIN OP_HASH256 OP_DROP
<2> <10000> OP_NUM2BIN OP_HASH256 OP_DROP
<3> <10000> OP_NUM2BIN OP_HASH256 OP_DROP
<0> <2871> OP_NUM2BIN OP_HASH256 OP_DROP
```

##### Benchmark: Stack Memory Check at Maximum Depth

This benchmark ensures that validating stack memory usage (for the [`Stack Memory Usage Limit`](#stack-memory-usage-limit)) does not consume excessive resources:

```
OP_1 OP_DUP OP_2DUP OP_3DUP OP_3DUP      // 10 `0x01`s on the stack
OP_3DUP [...x330]                        // 1000 `0x01`s on the stack
OP_1ADD [...x9167]
OP_2DROP [...x499]
OP_DROP                                  // one `0x01` on stack
```

##### Benchmark: Excessive OP_IF

This benchmark ensures stack implementations remain efficient with abusive use of `OP_IF` after removal of the operation limit:

```
OP_1 OP_IF [...x2000]
OP_1 [...x1000]
OP_1 OP_ENDIF [...x2000]
```

> Note: an [O(1) Execution Stack](https://github.com/bitcoin/bitcoin/pull/16902) has been implemented in a derivative of the Satoshi client.

##### Benchmark: Excessive OP_ROLL

This benchmark ensures stack implementations remain efficient with abusive use of `OP_ROLL` after removal of the operation limit:

```
OP_1 OP_DUP OP_2DUP OP_3DUP OP_3DUP      // 10 `0x01`s on the stack
OP_3DUP [...x330]                        // 1000 `0x01`s on the stack
<999> OP_ROLL [...x9167]
OP_2DROP [...x499]
OP_DROP                                  // one `0x01` on stack
```

## Rationale

This section documents design decisions made in this specification.

### Hashing Limit of 660 Digest Iterations

In this proposal, the precise value of `660` digest iterations has been selected to simplify review – at this value, the new limit [does not increase the worst-case computational cost](#hashing-limit) of validating maliciously-designed transactions.

This limit is likely also adequate for highly-complex public covenants: a 32-level `HASH160` merkle tree of `64` byte leaves requires only `70` digest iterations; contracts can easily inspect multiple parent transactions and large merkle proofs within the same evaluation without reaching this pre-existing limit. (Note: in practice, these contracts will be limited by other factors, particularly the 1,650-byte maximum unlocking bytecode length, A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE`.)

### Hashing Limit by Digest Iterations

One possible alternative to the proposed [Hashing Limit](#hashing-limit) design is to limit evaluations to a fixed count of "bytes hashed" (rather than digest iterations). While this seems simpler, it performs poorly in an adversarial environment: an attacker can choose message sizes to minimize bytes hashed while maximizing digest iterations required (e.g. 1-byte and 56-byte messages).

For non-memoized VM implementations, a "bytes hashed" limit must be as low as 660 bytes to offer protection equivalent to the 660 digest iteration limit. However, by directly measuring and limiting digest iterations, non-malicious contracts can be allowed a limit of 42231 bytes hashed (efficiently) without increasing the worst-case validation requirements of the VM as it existed prior to this proposal.

### Exclusion of Additional Relay Policy Limits

By allowing larger stack elements, this proposal increases the efficiency of a hashing-limited transaction (see [`Post-Deployment Hashing Limit Benchmark`](#post-deployment-hashing-limit-benchmark)), increasing the approximate validation time for 32MB worst-case blocks from ~0.12s to ~0.60s on 2018 consumer CPUs<sup>1</sup>. As such, hashing-limited transactions remain an ineffective DOS strategy – in practice they are equivalent to simple block-stuffing DOS attacks (for which the existing mitigation is transaction fees and large-enough blocks). No further relay policy limits are necessary in the context of this proposal.

> Note: worst-case block validation times are less important on Bitcoin Cash than on other cryptocurrencies: if most miners validate transactions as they are received (and most blocks clear the "mempool"), most standard transactions in a block must have been previously validated by other miners, and the new block can be validated quickly (without revalidating previously-seen transactions). Worst-case transaction validation times are only a concern for miners intentionally mining non-standard transactions – if a mining peer receives and validates a competing block before their slow-to-validate block, the miner of the slow-to-validate block loses revenue.

<details>
 <summary>Calculations</summary>

The maximum-efficiency transaction employing the [`Pre-Deployment Hashing Limit Benchmark`](#pre-deployment-hashing-limit-benchmark) has 176 inputs: version (4 bytes), input count (1 byte, `0xb2`), 176x{ outpoint (36 bytes), unlocking bytecode length (3 bytes, `0xfd0b02`), unlocking bytecode (523 bytes), sequence number (4 bytes)}, output count (1 byte, `0x01`), value (8 bytes), locking bytecode length (1 byte `0x01`), locking bytecode (1 byte, `OP_RETURN`), locktime (4 bytes). Total transaction size: `4 + 1 + 176 * (36 + 3 + 523 + 4) + 1 + 8 + 1 + 1 + 4 = 99,636 bytes`. Total digest iterations: `176 * 660 = 116,160 digest iterations`. `116160/99636 ~= 1.166`; **attack costs 1 byte per ~1.166 SHA-256 digest iterations**.

The maximum-efficiency transaction employing the [`Post-Deployment Hashing Limit Benchmark`](#post-deployment-hashing-limit-benchmark) has 884 inputs: version (4 bytes), input count (3 bytes), 884x{ outpoint (36 bytes), unlocking bytecode length (1 byte, `0x46`), unlocking bytecode (72 bytes), sequence number (4 bytes) }, output count (1 byte, `0x01`), value (8 bytes), locking bytecode length (1 byte `0x01`), locking bytecode (1 byte, `OP_RETURN`), locktime (4 bytes). Total transaction size: `4 + 1 + 884 * (36 + 1 + 72 + 4) + 1 + 8 + 1 + 1 + 4 = 99,912 bytes`. Total digest iterations: `884 * 660 = 583,440 digest iterations`. `583440/99912 ~= 5.840`; **attack costs 1 byte per ~5.840 SHA-256 digest iterations**.

According to [AnandTech's CPU 2021 Benchmarks](https://archive.vn/PzlXO), most CPUs sold since 2015 perform SHA-256 (Linux, OpenSSL, multi-threaded) at over `2000 MB/s` for 8K message blocks: `2000 * 1000000 / 8000 * 126 = 31,500,000 digest iterations per second`. The `AMD Ryzen 5 2600` (a top-selling consumer CPU of 2018) performed at `19675 MB/s` for the same benchmark: `19675 * 1000000 / 8000 * 126 = 309,881,250 digest iterations per second`.

At a Bitcoin Cash block size of 32MB, a full block of `Pre-Deployment Hashing Limit Benchmark` transactions will include 321 transactions requiring `37,287,360 digest iterations`; **approx. ~1.2s on 2015 CPUs, ~0.12s on 2018 CPUs**.

At a Bitcoin Cash block size of 32MB, a full block of `Post-Deployment Hashing Limit Benchmark` transactions will include 320 transactions requiring `186,700,800 digest iterations`; **approx. ~5.9s on 2015 CPUs, ~0.60s on 2018 CPUs**.

 </details>

## Implementations

Please see the following reference implementations for additional examples and test vectors:

[TODO: after initial public feedback]

## Stakeholders & Statements

[TODO]

## Feedback & Reviews

- [`Raising the 520 byte push limit & 201 operation limit` – Feb 8, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/raising-the-520-byte-push-limit-201-operation-limit/282)
- [`CHIP: Targeted Virtual Machine Limits` – May 12, 2021 | bitcoincashresearch.org](https://bitcoincashresearch.org/t/chip-targeted-virtual-machine-limits/437)

## Copyright

This document is placed in the public domain.
