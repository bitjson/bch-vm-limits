# Risk Assessment

The following security considerations, potential risks, and costs have been reviewed to verify the safety and advisability of this proposal.

## Risks & Security Considerations

This section reviews the foreseeable security implications of the proposed changes to the Bitcoin Cash network. Key technical considerations include user impact risks, consensus risks, denial-of-service (DoS) risks, and risks to protocol complexity and ongoing maintenance burden of newly introduced behavior.

### User Impact Risks

All upgrade proposals must carefully analyze proposed changes for potential impacts to existing Bitcoin Cash users and use cases. Virtual Machine (VM) upgrades can impact node operators and blockchain indexers (and therefore payment processors, exchanges, and other businesses), software development libraries, wallets, decentralized applications, and a wide range of pre-signed transactions, contract systems, and transaction-settled protocols.

This proposal is carefully designed to preserve backwards-compatibility along a variety of dimensions, minimizing user impact risk.

#### Reduced or Equivalent Node Validation Costs

By carefully establishing all re-targeted limits at levels "rounded up" from their existing practical limits, this proposal minimizes the risk of increase to worst-case validation performance, preventing any increase in node operation costs. See [Selection of Operation Cost Limit](./rationale.md#selection-of-operation-cost-limit), [Selection of Base Instruction Cost](./rationale.md#selection-of-base-instruction-cost), [Selection of Hash Digest Iteration Cost](./rationale.md#selection-of-hash-digest-iteration-cost), and [Selection of Signature Verification Operation Cost](./rationale.md#selection-of-signature-verification-operation-cost). Further, this risk mitigation strategy has been empirically verified across multiple implementations using a wide range of [cross-implementation functional tests and performance benchmarks](./tests-and-benchmarks.md).

Finally, even if these precautions were to fail in preventing or revealing a significant performance issue in a widely-used VM implementation, by design, this proposal expands an already [10x to 100x safety margin in performance-critical implementations](#expanded-node-performance-safety-margin).

#### Increased or Equivalent Contract Capabilities

Because the proposed, re-targeted limits are equal to or greater than their current practical equivalents, existing contract systems, decentralized applications, pre-signed transactions, and transaction-settled protocols have minimal risk of negative impact (in many cases, these users are the primary beneficiaries of increased Bitcoin Cash VM capabilities). By design, all currently-standard transactions remain valid following activation of this proposal.

Finally, this proposal highlights and evaluates the risk of impact to a smaller, hypothetical set of users who might have intentionally designed limit-failure-reliant constructions, see [Notice of Possible Future Expansion](/readme.md#notice-of-possible-future-expansion) and [Rationale: Inclusion of "Notice of Possible Future Expansion"](./rationale.md).

### Consensus Risks

All network consensus upgrade proposals must account for consensus risks arising from incorrect or inconsistent implementation of consensus-critical changes. For VM upgrades, consensus risks primarily apply to node implementations and other software which performs VM evaluation as part of transaction validation.

This proposal mitigates consensus risks via 1) an extensive set of full-transaction test vectors, 2) a new cross-implementation performance testing methodology, and 3) a 6-month early activation on `chipnet`.

#### Full-Transaction Test Vectors

To minimize the risk of inconsistencies between implementations, this proposal includes [over 36,000 cross-implementation functional tests and performance benchmarks](./tests-and-benchmarks.md) covering both the contents of the upgrade and of significant portions of pre-existing VM behavior.

In developing these test vectors, a workflow for test vector development has been established between [Libauth](https://github.com/bitauth/libauth) (a debugging-focused JavaScript implementation) and [Bitcoin Cash Node](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/) (C++) – the implementation most commonly relied on by Bitcoin Cash miners. In effect, this development brings the added reliability of [N-version programming](https://en.wikipedia.org/wiki/N-version_programming) to Bitcoin Cash's VM test vector development.

#### New Performance Testing Methodology

Though this proposal is carefully designed to both [reduce worst-case node validation costs](#reduced-or-equivalent-node-validation-costs) and [expand the node validation performance safety margin](#expanded-node-performance-safety-margin), this proposal also standardizes a new performance benchmarking methodology to help detect performance irregularities and regressions in specific node implementations and VM-evaluating software.

In addition to purely functional tests, hundreds of carefully-designed worst-case performance benchmarks are integrated in the unified set of [tests and benchmarks](./tests-and-benchmarks.md), and a rigorous, cross-implementation performance testing strategy has been specified, with carefully selected metrics, file formats/structures, and an established methodology for [sharing results](./tests-and-benchmarks.md#evaluation-of-results) across implementations.

#### `Chipnet` Preview Activation

Finally, like all Bitcoin Cash Improvement Proposals (CHIPs), this proposal schedules activation on `chipnet` 6-months before activation on `mainnet`.

While many development teams and ecosystem stakeholders have reviewed this proposal prior to the November lock-in, only a smaller subset of highly-interested projects and parties (e.g. contract developers, decentralized application development teams, and node implementations) can speculatively allocate development time for complete integration testing with existing software systems.

By scheduling a 6-month early activation on `chipnet`, this proposal minimizes the cost of widest-possible integration testing – in a partially adversarial environment – in which software defects can be discovered and corrected prior to `mainnet` activation.

### Denial-of-Service (DoS) Risks

All network consensus upgrade proposals which alter system limits carry risks related to Denial-of-Service (DoS) attacks. In particular, modifications to VM limits could 1) exacerbate the worst-case performance of transaction or block validation for both expensive-but-valid cases and excessively-invalid cases, and/or 2) decrease the cost or increase the practicality of attempting a particular VM-related DOS attack.

#### Expanded Node Performance Safety Margin

Beyond other precautions taken by this proposal ([maximally-conservative limits](#reduced-or-equivalent-node-validation-costs) and [extensive performance benchmarking](./tests-and-benchmarks.md)) – denial-of-service risks are further reduced by an expanded margin of safety for transaction and block validation performance in hashing-expensive scenarios.

This proposal sets a standard hashing limit at `0.5` iterations per byte: the asymptotic maximum density of hashing operations in [plausibly non-malicious, standard transactions](/rationale.md#selection-of-hashing-limit). This limit is `7x` lower than the currently-standard `3.5` iterations per byte practical limit; as such, the standard-validation margin of safety in implementations with relatively-slow hashing performance (or a hashing-related performance regression) is increased `~7x` by this proposal. Additionally, though worst case nonstandard performance is [already limited by mining economics](./tests-and-benchmarks.md#standard-vs-nonstandard-vms), the similarly-improved margin of safety on nonstandard hashing reduces risks for some node operators and increases the predictability of worst-case block validation performance.

Finally, note that this proposal also maintains Bitcoin Cash's existing margin of safety for all other transaction and block validation scenarios. At a maximum block size of `32MB` (increasing by no more than 2x per year), and a worst-case validation performance ([`packed 1-of-3 ECDSA BMS, bottom slot`](vmb_tests/bch_2023_standard/core.benchmarks.signature-checking.bms-ecdsa.standard_stats.csv)) of `4` to `5` times the compute-density of the [baseline benchmark](./tests-and-benchmarks.md#baseline-benchmark) for performance-optimized implementations like [Bitcoin Cash Node](./tests-and-benchmarks.md#bitcoin-cash-node-c), even relatively low-powered **consumer hardware has excess compute capacity of `10x` to `100x` the worst-case computation requirements** of fully verifying all Bitcoin Cash transactions and blocks<sup>1</sup>.

Due to this node performance safety margin, in practice, any DOS attack attempting to create unusually-high CPU utilization across the network would likely be indistinguishable by most network participants from any other burst of low CPU-utilization transactions (e.g. transactions dominated by `OP_RETURN` data carrier outputs). In this scenario, the more notable costs for honest network nodes remain bandwidth and storage costs (and likewise from the attackers' perspective, transaction mining fees). See [Exclusion of "Gas System" Behaviors](./rationale.md#exclusion-of-gas-system-behaviors).

<details>

<summary>Notes</summary>

1. [Bitcoin Cash Node](./tests-and-benchmarks.md#bitcoin-cash-node-c) on typical consumer CPUs can verify the baseline benchmark at approximately `10k` transactions per second, per core, caching disabled (see [benchmarks](./tests-and-benchmarks.md)). Given a maximum block size of `32MB`, worst-case baseline-relative validation performance of `4` to `5`, and the baseline transaction length (`366` bytes), worst-case required performance is approximately `350k` to `440k` baseline validations (`(32,000,000 * 5)/366 = 349,726`; `(32,000,000 * 5)/366 = 437,158`). Given a typical single-core validation speed of approximately `10k tx/core/second`, worst-case block times of `5` to `10` minutes (`300` to `600` seconds), and parallelism between `2` and `8`, a very conservative safety margin for VM performance is on the order of `10` to `100` (worst case: `300 seconds per block / (~440k / (10k/s * 2 cores)) ~= 13.7x`; more typical case: `600 seconds per block / (~350k / (10k/s * 8 cores) ~= 137x`).

</details>
