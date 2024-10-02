# Risk Assessment

The following security considerations, potential risks, and costs have been reviewed to verify the safety and advisability of this proposal.

<details>

<summary><strong>Table of Contents</strong></summary>

- [Risks \& Security Considerations](#risks--security-considerations)
  - [User Impact Risks](#user-impact-risks)
    - [Reduced or Equivalent Node Validation Costs](#reduced-or-equivalent-node-validation-costs)
    - [Increased or Equivalent Contract Capabilities](#increased-or-equivalent-contract-capabilities)
  - [Consensus Risks](#consensus-risks)
    - [Full-Transaction Test Vectors](#full-transaction-test-vectors)
    - [New Performance Testing Methodology](#new-performance-testing-methodology)
    - [`Chipnet` Preview Activation](#chipnet-preview-activation)
  - [Denial-of-Service (DoS) Risks](#denial-of-service-dos-risks)
    - [Expanded Node Performance Safety Margin](#expanded-node-performance-safety-margin)
  - [Protocol Complexity Risks](#protocol-complexity-risks)
    - [Support for Post-Activation Simplification](#support-for-post-activation-simplification)
    - [Continuation of Nonstandard Invalidation Precedent](#continuation-of-nonstandard-invalidation-precedent)
    - [Consideration of Possible Future Changes](#consideration-of-possible-future-changes)
- [Upgrade Costs](#upgrade-costs)
  - [Node Upgrade Costs](#node-upgrade-costs)
  - [Ecosystem Upgrade Costs](#ecosystem-upgrade-costs)
- [Maintenance Costs](#maintenance-costs)
  - [Node Maintenance Costs](#node-maintenance-costs)
  - [Ecosystem Maintenance Costs](#ecosystem-maintenance-costs)

</details>

## Risks & Security Considerations

This section reviews the foreseeable security implications of the proposed changes to the Bitcoin Cash network. Key technical considerations include user impact risks, consensus risks, denial-of-service (DoS) risks, and risks to protocol complexity or maintenance burden of newly introduced behavior.

### User Impact Risks

All upgrade proposals must carefully analyze proposed changes for potential impacts to existing Bitcoin Cash users and use cases. Virtual Machine (VM) upgrades can impact node operators and blockchain indexers (and therefore payment processors, exchanges, and other businesses), software development libraries, wallets, decentralized applications, and a wide range of pre-signed transactions, contract systems, and transaction-settled protocols.

This proposal is designed to preserve backwards-compatibility along a variety of dimensions, minimizing user impact risks:

#### Reduced or Equivalent Node Validation Costs

By establishing all re-targeted limits at levels "rounded up" from their existing practical limits, this proposal minimizes the risk of increase to worst-case validation performance, preventing any increase in node operation costs. See [Selection of Operation Cost Limit](./rationale.md#selection-of-operation-cost-limit), [Selection of Base Instruction Cost](./rationale.md#selection-of-base-instruction-cost), [Selection of Hash Digest Iteration Cost](./rationale.md#selection-of-hash-digest-iteration-cost), and [Selection of Signature Verification Operation Cost](./rationale.md#selection-of-signature-verification-operation-cost). Further, this risk mitigation strategy has been empirically verified across multiple implementations using a wide range of [cross-implementation functional tests and performance benchmarks](./tests-and-benchmarks.md).

Finally, even if these precautions were to fail in preventing a performance issue or revealing an existing performance issue in a widely-used VM implementation, by design, this proposal expands an already [10x to 100x safety margin in performance-critical implementations](#expanded-node-performance-safety-margin).

#### Increased or Equivalent Contract Capabilities

Because the proposed, re-targeted limits are equal to or greater than their current practical equivalents, existing contract systems, decentralized applications, pre-signed transactions, and transaction-settled protocols have minimal risk of negative impact (in many cases, these users are the primary beneficiaries of increased Bitcoin Cash VM capabilities). By design, all currently-standard transactions remain valid following activation of this proposal.

Finally, this proposal highlights and evaluates the risk of impact to a smaller, hypothetical set of users who might have intentionally designed limit-failure-reliant constructions, see [Notice of Possible Future Expansion](/readme.md#notice-of-possible-future-expansion) and [Rationale: Inclusion of "Notice of Possible Future Expansion"](./rationale.md#inclusion-of-notice-of-possible-future-expansion).

### Consensus Risks

All network consensus upgrade proposals must account for consensus risks arising from incorrect or inconsistent implementation of consensus-critical changes. For Virtual Machine (VM) upgrades, consensus risks primarily apply to node implementations and other software which performs VM evaluation as part of transaction validation.

This proposal mitigates consensus risks via 1) an extensive set of full-transaction test vectors, 2) a new cross-implementation performance testing methodology, and 3) a 6-month early activation on `chipnet`.

#### Full-Transaction Test Vectors

To minimize the risk of inconsistencies between implementations, this proposal includes [over 36,000 cross-implementation functional tests and performance benchmarks](./tests-and-benchmarks.md) covering both the contents of the upgrade and of significant portions of pre-existing VM behavior.

In developing these test vectors, a workflow for test vector development has been established between [Libauth](https://github.com/bitauth/libauth) (a debugging-focused JavaScript implementation) and [Bitcoin Cash Node](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/) (C++) – the implementation most commonly relied on by Bitcoin Cash miners. In effect, this development brings the added reliability of [N-version programming](https://en.wikipedia.org/wiki/N-version_programming) to Bitcoin Cash's VM test vector development.

#### New Performance Testing Methodology

Though this proposal is designed to both [reduce worst-case node validation costs](#reduced-or-equivalent-node-validation-costs) and [expand the node validation performance safety margin](#expanded-node-performance-safety-margin), this proposal also standardizes a new performance benchmarking methodology to help detect performance irregularities and regressions in specific node implementations and VM-evaluating software.

In addition to purely functional tests, hundreds of carefully-designed worst-case performance benchmarks are integrated in the unified set of [tests and benchmarks](./tests-and-benchmarks.md), and a rigorous, cross-implementation performance testing strategy has been specified, with carefully selected metrics, file formats/structures, and an established methodology for [sharing results](./tests-and-benchmarks.md#evaluation-of-results) across implementations.

#### `Chipnet` Preview Activation

Finally, like all Bitcoin Cash Improvement Proposals (CHIPs), this proposal schedules activation on `chipnet` 6-months before activation on `mainnet`.

While many development teams and ecosystem stakeholders have reviewed this proposal prior to the November lock-in, only a smaller subset of highly-interested projects and parties (e.g. contract developers, decentralized application development teams, and node implementations) can speculatively allocate development time for complete integration testing with existing software systems.

By scheduling a 6-month early activation on `chipnet`, this proposal minimizes the cost of widest-possible integration testing – in a partially adversarial environment – in which software defects can be discovered and corrected prior to `mainnet` activation.

### Denial-of-Service (DoS) Risks

All network consensus upgrade proposals which alter system limits carry risks related to Denial-of-Service (DoS) attacks. In particular, modifications to VM limits could 1) exacerbate the worst-case performance of transaction or block validation for both expensive-but-valid cases and excessively-invalid cases, and/or 2) decrease the cost or increase the practicality of attempting a particular VM-related DOS attack.

#### Expanded Node Performance Safety Margin

Beyond other precautions taken by this proposal ([maximally-conservative limits](#reduced-or-equivalent-node-validation-costs) and [extensive performance benchmarking](./tests-and-benchmarks.md)) – denial-of-service risks are further reduced by an expanded margin of safety for transaction and block validation performance in hashing-expensive scenarios.

This proposal sets a standard hashing limit at `0.5` iterations per byte: the asymptotic maximum density of hashing operations in [plausibly non-malicious, standard transactions](/rationale.md#selection-of-hashing-limit). This limit is ~`7x` lower than the currently-standard ~`3.5` iterations per byte practical limit; as such, the standard-validation margin of safety in implementations with relatively-slow hashing performance (or a hashing-related performance regression) is increased `~7x` by this proposal. Additionally, though worst case nonstandard performance is [already limited by mining economics](./tests-and-benchmarks.md#standard-vs-nonstandard-vms), the similarly-improved margin of safety on nonstandard hashing reduces risks for some node operators and increases the predictability of worst-case block validation performance.

Finally, note that this proposal also maintains Bitcoin Cash's existing margin of safety for all other transaction and block validation scenarios. At a maximum block size of `32MB` (increasing by no more than 2x per year), and a worst-case standard validation performance ([`packed 1-of-3 ECDSA BMS, bottom slot`](vmb_tests/bch_2023_standard/core.benchmarks.signature-checking.bms-ecdsa.standard_stats.csv)) of `4` to `5` times the compute-density of the [baseline benchmark](./tests-and-benchmarks.md#baseline-benchmark) for performance-optimized implementations like [Bitcoin Cash Node](./tests-and-benchmarks.md#bitcoin-cash-node-c), even relatively low-powered **consumer hardware has excess compute capacity of `10x` to `100x` the worst-case computation requirements** of fully verifying all Bitcoin Cash transactions and blocks<sup>1</sup>.

Due to this node performance safety margin, in practice, any DOS attack attempting to create unusually-high CPU utilization across the network would likely be indistinguishable by most network participants from any other burst of low CPU-utilization transactions (e.g. transactions dominated by [`OP_RETURN` data carrier outputs](./rationale.md#op_return-data-carrier-outputs)). In this scenario, the more notable costs for honest network nodes remain bandwidth and storage costs (and likewise from the attackers' perspective, transaction mining fees). See [Rationale: Minimal Impact of Compute on Node Operation Costs](./rationale.md#minimal-impact-of-compute-on-node-operation-costs).

<details>

<summary>Notes</summary>

1. [Bitcoin Cash Node](./tests-and-benchmarks.md#bitcoin-cash-node-c) on typical consumer CPUs can verify the baseline benchmark at approximately `10k` transactions per second, per core, caching disabled (see [benchmarks](./tests-and-benchmarks.md)). Given a maximum block size of `32MB`, worst-case baseline-relative standard validation performance of `4` to `5`, and the baseline transaction length (`366` bytes), worst-case required performance is approximately `350k` to `440k` baseline validations (`(32,000,000 * 5)/366 = 349,726`; `(32,000,000 * 5)/366 = 437,158`). Given a typical single-core validation speed of approximately `10k tx/core/second`, worst-case block times of `5` to `10` minutes (`300` to `600` seconds), and parallelism between `2` and `8`, a very conservative safety margin for VM performance is on the order of `10` to `100` (worst case: `300 seconds per block / (~440k / (10k/s * 2 cores)) ~= 13.7x`; more typical case: `600 seconds per block / (~350k / (10k/s * 8 cores) ~= 137x`).

</details>

### Protocol Complexity Risks

All upgrade proposals must carefully analyze proposed changes for both immediate and potential future impacts on overall protocol complexity. This proposal has been reviewed to ensure that all changes are 1) minimal, 2) necessary, and 3) avoid creating technical debt, even if future upgrades were to further expand limits or expand the VM's capabilities in a number of analyzed scenarios.

#### Support for Post-Activation Simplification

This proposal carefully re-targets limits in backwards-compatible ways. By design, all transactions which were previously standard (acceptable for transaction relay) remain valid in this proposal's re-targeted system. As a result, significant portions of activation code and historical code used to enforce the old limits can be safely removed following activation.

#### Continuation of Nonstandard Invalidation Precedent

To minimize technical debt and worst-case nonstandard validation performance, this proposal invalidates two sets of currently-nonstandard transactions (transactions which are rejected by transaction relay validation, but accepted by block validation): 1) nonstandard transactions which violate the nonstandard hashing limit, and 2) nonstandard transactions with one or more input(s) which exceed their operation cost limit.

Nonstandard transactions of the first type are [demonstrably malicious within the current limit system](./rationale.md#selection-of-hashing-limit). However, a space of plausibly non-malicious contracts exists within the second type: contracts with spending code paths in which unlocking bytecode length is less than approximately 201 bytes and operation cost exceeds the input's limit (based on its [density control length](./readme.md#density-control-length)).

While this proposal could attempt to create limit exemption(s) for code paths in this plausibly non-malicious space, such exemptions would mislead users concerning longstanding precedent: nonstandard transaction validation is used to safely deprecate categories of previously-standard transactions (e.g. to remedy contract security or resource exhaustion risks) – only standard transactions should ever be expected to remain valid across network upgrades. E.g. [Pay-to-Script-Hash (P2SH)](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki) (2012), [`OP_CHECKLOCKTIMEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki) (2014), [Strict DER Signatures](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki) (2015), [`OP_CHECKSEQUENCEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) (2015), [`LOW_S` and `NULLFAIL`](https://github.com/bitcoin/bips/blob/master/bip-0146.mediawiki) (2016), [`OP_CHECKMULTISIG[VERIFY] NULLDUMMY`](https://github.com/bitcoin/bips/blob/master/bip-0147.mediawiki) (2016), [User Activated Hard Fork](https://reference.cash/protocol/forks/bch-uahf) (2017), [`BCH_2018_11`](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/2018-nov-upgrade.md), [`BCH_2019_05`](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/2019-05-15-schnorr.md), [`MINIMALDATA`](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/2019-11-15-minimaldata.md) (2019), and [`SigChecks`](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/2020-05-15-sigchecks.md) (2020), [CHIP 2021-01 Restrict Transaction Version](https://gitlab.com/bitcoin.cash/chips/-/blob/master/CHIP-2021-01-Restrict%20Transaction%20Versions.md), [CHIP-2022-02 CashTokens](https://github.com/cashtokens/cashtokens), and [CHIP-2022-05 P2SH32](https://gitlab.com/0353F40E/p2sh32/).

In practice, the nonstandard invalidation precedent has served a critical role in 1) reducing the impact of many contract security, consensus, and node resource exhaustion risks, 2) protecting real users across upgrades by ensuring a safe deprecation path for behavior that unintentionally relies on contract system or network vulnerabilities, and 3) enabling protocol enhancements that may have otherwise been impractical without significantly increased protocol complexity if previously-nonstandard validation behaviors were recast as "real-user" usage rather than as a deprecation and safety feature.

To preserve this important precedent for future upgrades, this proposal intentionally omits special considerations for any previously-nonstandard usage. By design, in specifying nonstandard validation behavior following the upgrade, only previously-standard validation can safely be considered within scope. By extension, it should be noted that any abusive behavior made nonstandard by this proposal is a candidate for full invalidation in future upgrades.

#### Consideration of Possible Future Changes

Finally, to minimize future protocol complexity, this proposal carefully reviews the potential impact of all re-targeted limits on a variety of possible future upgrades:

- **Increased stack item length limit** – The re-targeted limits prevent significant increases in worst-case performance following an upgrade in which the `10,000`-byte stack item length limit (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) is further increased. See [Rationale: Limitation of Pushed Bytes](./rationale.md#limitation-of-pushed-bytes).

- **Increased contract length limits** – The re-targeted limits prevent significant increases in worst-case performance following an upgrade in which the `10,000`-byte contract length limit (A.K.A. `MAX_SCRIPT_SIZE`) is increased or eliminated in favor of cumulative transaction size limits. See [Rationale: Use of Explicitly-Defined Density Limits](./rationale.md#use-of-explicitly-defined-density-limits), [Rationale: Selection of Base Instruction Cost](./rationale.md#selection-of-base-instruction-cost)

- **Increased transaction length limits** – The re-targeted limits prevent significant increases in worst-case performance following an upgrade in which the `100,000`-byte standard transaction length limit (A.K.A. `MAX_STANDARD_TX_SIZE`) or `1,000,000`-byte nonstandard transaction length limit (A.K.A. `MAX_TX_SIZE`) are increased. See [Rationale: Use of Explicitly-Defined Density Limits](./rationale.md#use-of-explicitly-defined-density-limits).

- **Increased number length limit** – The re-targeted limits immediately eliminate the need for a number length limit (A.K.A. `nMaxNumSize`). Following review by multiple VM implementations, a companion proposal was created to simultaneously remove the number length limit: [CHIP-2024-07-BigInt: High-Precision Arithmetic for Bitcoin Cash](https://github.com/bitjson/bch-bigint).

- **Addition of new opcodes** – Because the re-targeted limits create a unified "cost" metric for operations of significantly varying performance, new opcodes can be safely added to the VM by specifying only the new operation's cost formula, minimizing future upgrades' potential performance and protocol complexity impact. See [Rationale: Use of Explicitly-Defined Density Limits](./rationale.md#use-of-explicitly-defined-density-limits), [Rationale: Unification of Limits into Operation Cost](./rationale.md#unification-of-limits-into-operation-cost),

- **Compatibility with additional control flow structures** – By limiting the density of pushed bytes and the density of evaluated operations, this proposal's re-targeted limits remain effective even if future upgrades significantly expand the VMs control flow capabilities: definite or indefinite loops, pushed bytecode evaluation, word/function definition and evaluation, or other structures. Further, the worst-case performance of contracts using such control flow structures can be safely decoupled from the contract length limit, allowing greater flexibility, e.g. loops which safely evaluate more instructions than would otherwise be possible to fit within contract length limits. See [Rationale: Retention of Control Stack Limit](./rationale.md#retention-of-control-stack-limit), [Rationale: Limitation of Pushed Bytes](./rationale.md#limitation-of-pushed-bytes), and [Rationale: Selection of Base Instruction Cost](./rationale.md#selection-of-base-instruction-cost).

- **Increased operation density** – By establishing a relatively-high [Base Instruction Cost](./readme.md#base-instruction-cost), this proposal retains a similar ([within one order of magnitude](./rationale.md#selection-of-base-instruction-cost)) limit on operation density as under the existing limits. However, future upgrades may easily raise this limit by reducing the base instruction cost, requiring more careful optimization of instruction evaluation in all VM implementations while enabling a reduction in transaction sizes for operation-dense contracts. See [Rationale: Limitation of Pushed Bytes](./rationale.md#limitation-of-pushed-bytes) and [Rationale: Selection of Base Instruction Cost](./rationale.md#selection-of-base-instruction-cost).

- **Increased per-operation overhead** (selectively-decreased operation density) - Additionally, by establishing a relatively-high [Base Instruction Cost](./readme.md#base-instruction-cost), this proposal retains an ability for future upgrades to differentiate the overhead cost of existing "low-cost" operations (costs nearly equal to the base instruction cost) by up to two orders of magnitude, without increasing worst-case validation performance, by selectively lowering only the base instruction cost of operations to be sorted into the new, "faster" set of operations. See [Rationale: Selection of Base Instruction Cost](./rationale.md#selection-of-base-instruction-cost).

- **Relaxation of output standardness** – The re-targeted limits eliminate differences in worst-case performance between Pay-to-Script-Hash (P2SH) and non-P2SH contracts, simplifying potential future upgrades in which [output standardness is relaxed](https://bitcoincashresearch.org/t/relaxing-output-standardness/1391) to allow some custom, non-P2SH contracts within standard validation (e.g. such that output standardness could be determined only by UTXO length rather than by parsed UTXO contents). See [Rationale: Use of Explicitly-Defined Density Limits](./rationale.md#use-of-explicitly-defined-density-limits) and [Rationale: Use of Input Length-Based Densities](./rationale.md#use-of-input-length-based-densities),

## Upgrade Costs

This section reviews the costs of implementing the proposed changes.

### Node Upgrade Costs

This proposal affects consensus – all fully-validating node software must implement these Virtual Machine (VM) changes to remain in consensus. To minimize the cost of validating node implementation upgrades, this proposal includes a wide range of [cross-implementation functional tests and performance benchmarks](./tests-and-benchmarks.md).

Additionally, to minimize implementation costs, this proposal has been carefully designed to avoid requiring several potential optimizations in performance-critical VM implementations:

- **Optimized control stack** – This proposal allows – but does not require – VM implementations to implement an O(1) Control Stack. See [Rationale: Retention of Control Stack Limit](./rationale.md#retention-of-control-stack-limit).
- **Minimal-cost VM number encoding/decoding** – This proposal allows – but does not require – VM implementations to optimize the encoding or decoding of VM numbers between the consensus-enforced number encoding and whichever internal format is used by the implementation's high-precision arithmetic operations. This ensures that a wide variety of high-precision arithmetic libraries and built-in language features can be easily used in the development of performance-critical VM implementations. See [Rationale: Inclusion of Numeric Encoding in Operation Costs](./rationale.md#inclusion-of-numeric-encoding-in-operation-costs).
- **Batching of Schnorr signature verification** – This proposal allows – but does not require – VM implementations to optimize performance of Schnorr signature verification by deferring signature validation and batching the verification of Schnorr signatures. See [Rationale: Selection of Signature Verification Operation Cost](./rationale.md#selection-of-signature-verification-operation-cost) and [Rationale: Continued Availability of Deferred Signature Validation](./rationale.md#continued-availability-of-deferred-signature-validation).

### Ecosystem Upgrade Costs

Upgrade costs for the wider ecosystem – wallets, blockchain indexers, mining, mining-monitoring, and mining-administrative software, etc. are limited:

- This proposal **does not create new user-facing primitives** (e.g. [CashTokens, 2023](https://cashtokens.org/)) which might demand significant changes or additional features in wallets, block explorers, or other ecosystem software.

- This proposal **does not notably impact user or miner-visible metrics** (e.g. [Adaptive BlockSize Limit Algorithm, 2024](https://gitlab.com/0353F40E/ebaa#chip-2023-04-adaptive-blocksize-limit-algorithm-for-bitcoin-cash)) which might demand significant changes in software like block explorers, mining calculators, and proprietary mining software and tooling, as well as user-facing documentation and marketing materials referencing the well-known, Blocksize Limit metric.

While all Bitcoin Cash software systems must inevitably be upgraded to follow all consensus upgrades (e.g. a company must annually upgrade to the latest version of their preferred full node implementation), ecosystem upgrade costs for this proposal are essentially confined to specialized software, tooling, and documentation for contract developers. Further, as the fundamental aim of this proposal is to expand the power and flexibility of Bitcoin Cash's VM for these stakeholders, significant portions of these costs were already speculatively paid, in hopes of later activation, during the creation and review of this proposal.

## Maintenance Costs

All network upgrade proposals must evaluate their foreseeable impact on maintenance costs. Virtual Machine (VM) upgrades can increase the risks of [consensus divergence](#consensus-risks), [performance issues](#denial-of-service-dos-risks), and increased [implementation complexity](#protocol-complexity-risks). This section reviews foreseeable ongoing costs following activation of the proposed changes.

### Node Maintenance Costs

This proposal minimizes any negative impact on node maintenance costs:

- **Decreased or equivalent worst-case validation cost** – By design, this proposal establishes all re-targeted limits at levels which avoid any notable increase in worst-case validation cost, and in some cases, significantly decrease worst-case cost. This reduces ongoing node maintenance costs by reducing the potential impact of implementation-specific performance issues.

- **New functional tests and performance benchmarks** – This proposal also significantly improves the long-term maintainability of most node implementations by contributing both an enhanced set of [full transaction test vectors](#full-transaction-test-vectors) a new [cross-implementation performance testing methodology](#new-performance-testing-methodology) to help root out existing or future performance issues in specific implementations.

- **Potential increase in project maintenance resources** – By increasing the power and flexibility of Bitcoin Cash's VM for contract development, this proposal has the potential to attract further resources to node and VM implementations, both to 1) enhance performance of implementations primarily aimed at mining and/or business infrastructure, and 2) enhance the feature sets of alternative implementations aimed at contract development, decentralized application design, and/or specialized wallet management.

### Ecosystem Maintenance Costs

As with [ecosystem upgrade costs](#ecosystem-upgrade-costs), maintenance costs for the wider ecosystem – wallets, blockchain indexers, mining, mining-monitoring, and mining-administrative software, etc. are limited: this proposal **does not create new user-facing primitives**, and it **does not notably impact user or miner-visible metrics**. See [Ecosystem Upgrade Costs](#ecosystem-upgrade-costs).

Features of this proposal which are relevant to [node maintenance costs](#node-maintenance-costs) are also relevant to the rest of the ecosystem: 1) decreased or equivalent worst-case validation cost, 2) new functional tests and performance benchmarks, and a 3) potential increase in project maintenance resources.
