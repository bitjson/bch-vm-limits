# Rationale

This section documents design decisions made in this specification.

<details>

<summary><strong>Table of Contents</strong></summary>

- [Use of Explicitly-Defined Density Limits](#use-of-explicitly-defined-density-limits)
- [Exclusion of "Gas System" Behaviors](#exclusion-of-gas-system-behaviors)
  - [Global-State Validation Architectures](#global-state-validation-architectures)
  - [Stateless Validation Architecture](#stateless-validation-architecture)
  - [Minimal Impact of Compute on Node Operation Costs](#minimal-impact-of-compute-on-node-operation-costs)
- [Non-Impact on "Data Storage" Costs and Incentives](#non-impact-on-data-storage-costs-and-incentives)
  - [`OP_RETURN` Data Carrier Outputs](#op_return-data-carrier-outputs)
  - [Data-Carrying Transactions](#data-carrying-transactions)
  - [Indirect, Transaction "Padding" Incentives](#indirect-transaction-padding-incentives)
- [Retention of Control Stack Limit](#retention-of-control-stack-limit)
- [Use of Input Length-Based Densities](#use-of-input-length-based-densities)
  - [Alternative: Transaction Length-Based Densities](#alternative-transaction-length-based-densities)
  - [Alternative: UTXO-Length Increased Densities](#alternative-utxo-length-increased-densities)
  - [Alternative: Transaction Fee-Increased Densities](#alternative-transaction-fee-increased-densities)
- [Selection of Input Length Formula](#selection-of-input-length-formula)
- [Hashing Limit by Digest Iterations](#hashing-limit-by-digest-iterations)
- [Selection of Hashing Limit](#selection-of-hashing-limit)
- [Exclusion of Signing Serialization Components from Hashing Limit](#exclusion-of-signing-serialization-components-from-hashing-limit)
  - [Ongoing Value of OP_CODESEPARATOR Operation](#ongoing-value-of-op_codeseparator-operation)
- [Increased Usability of Multisig Stack Clearing](#increased-usability-of-multisig-stack-clearing)
- [Limitation of Pushed Bytes](#limitation-of-pushed-bytes)
- [Unification of Limits into Operation Cost](#unification-of-limits-into-operation-cost)
- [Selection of Operation Cost Limit](#selection-of-operation-cost-limit)
- [Selection of Base Instruction Cost](#selection-of-base-instruction-cost)
- [Inclusion of Numeric Encoding in Operation Costs](#inclusion-of-numeric-encoding-in-operation-costs)
- [Selection of Signature Verification Operation Cost](#selection-of-signature-verification-operation-cost)
- [Continued Availability of Deferred Signature Validation](#continued-availability-of-deferred-signature-validation)
- [Selection of Hash Digest Iteration Cost](#selection-of-hash-digest-iteration-cost)
- [Inclusion of "Notice of Possible Future Expansion"](#inclusion-of-notice-of-possible-future-expansion)

</details>

## Use of Explicitly-Defined Density Limits

This proposal defines explicit limits on the overall density of computation required to validate both transactions and blocks, particularly of [arithmetic](./readme.md#arithmetic-operation-cost), [hashing](./readme.md#hashing-limit), and [signature checking](./readme.md#signature-checking-operation-cost) computations. These explicit limits predictably cap the worst-case cost of validating transactions and blocks across all possible contract constructions and transaction-packing schemes (see [Rationale: Use of Input Length-Based Densities](#use-of-input-length-based-densities) for details).

Alternatively, this proposal could continue to indirectly limit the worst-case validation costs of transactions and blocks by specifying a system of one or more constant limits applied at various stages of VM evaluation. At best, such **statically-defined, per-input limit schemes** obfuscate the true limits: determining the real-world, worst-case validation performance of such schemes requires exhaustive enumeration of optimally-packed attack cases. Worse, the precise construction of worst-case attacks becomes significant to real world node performance requirements; any contract construction discovery or network upgrade which reduces contract lengths can magnify an unpredictable set of worst case attacks: e.g. more abusive transactions may be packed into abusive blocks, more abusive inputs may be packed into abusive transactions, fewer abusive transactions are required to exhaust a node's compute capacity, etc.

Finally, because statically-defined, per-input limit schemes fundamentally apply the same limits to 41-byte inputs as they apply to 10,000-byte inputs, the baseline variability of compute requirements ranges from `1` to `243.90` times the worst-case validation (again, determinable only via exhaustive enumeration)<sup>1</sup>. In practice, this means that some non-malicious contracts (real users) will be limited – ostensibly to protect the network from abusive use of computation during transaction validation – after using less than 1% of the net computation afforded to intentionally-malicious contracts<sup>2</sup>.

<details>

<summary>Notes</summary>

1. `10000 / 41 = 243.902439...`
2. `1 / (10000 / 41) = 0.41%`

</details>

## Exclusion of "Gas System" Behaviors

This proposal preserves the Bitcoin Cash network's highly-scalable, **stateless validation architecture**. Bitcoin Cash contract validation remains stateless and cacheable:

1. Contracts can be verified using only data derived from spent UTXOs and the transaction itself,
2. A successful contract validation result can be cached without later re-validation, and
3. All invalid transactions can be quickly rejected/discarded, with minimal computation requirements for network nodes.

### Global-State Validation Architectures

The **global-state transaction validation architecture** of other, similarly-capable virtual machines – like the Ethereum Virtual Machine (EVM) – architecturally-prevent those systems from achieving the scalability of Bitcoin Cash:

1. Contracts can reference data derived from unrelated network state,
2. Contract evaluations cannot be safely cached – all global transactions must be re-validated in the precise order selected by each new block's miner/validator, and
3. Invalid transactions – both intentionally-abusive and unintentionally-invalid transactions (e.g. honest users underestimating "gas" fees) – exact an orders-of-magnitude greater computation cost on all network node operators than the equivalent cost on Bitcoin Cash network node operators.

In these less efficient validation architectures, maintaining acceptable transaction throughput performance requires both significant development effort and user-facing structural changes, like EVM's pay-per-compute, "gas" regime. With such gas systems, all structurally-valid transactions (even transaction which fail validation by unintentionally exceeding the allotted virtual-compute time purchased by the "gas" fee paid) are at least partially validated and included in the next block – regardless of whether or not the end user intended for their transaction to fail. This leads to:

1. **Poor user experiences** – users are forced to accept real-world financial losses for the unpredictability of gas fee markets and/or the technical failures of the network, nodes, contracts, and/or wallet systems.
2. **Financial limitation of contract capabilities** – because the per-computation impact of contracts on all network nodes is thousands of times higher than on scalable architectures like Bitcoin Cash, global-state systems are forced to charge a meaningful price for each additional computation. In many cases, this regime is so inefficient that it guides contract development away from algorithms which are more correct or provide better user experiences, and toward algorithms which are inexpensive to execute in the networks' virtual-compute pricing ("gas") system. Notably, the connection between virtual-compute pricing and real world performance is tenuous, with the real-world performance differences between various computations sometimes deviating from their charged "gas" prices by greater than an order of magnitude. The dominance of constant-product market makers (CPMMs) are an observable result of this phenomenon, despite other known constructions being superior for certain use cases (e.g. logarithmic market scoring rules).

### Stateless Validation Architecture

In contrast, this proposal preserves the superior user experience and scalability of Bitcoin Cash:

1. **Reliable user experience** – users never pay for un-mined, "failed" transactions, user transactions cannot be re-ordered by miners without user consent (via deliberate participation in reorder-able contract schemes), and unconfirmed transactions can be chained such that earlier transactions are guaranteed to be mined if a later transaction in the chain is mined (enabling additional miner-abuse resistance and censorship-resistance strategies).
2. **No-fee contract deployment and superior real-world privacy** – User wallets both “deploy” and “destroy” contracts incrementally as part of their usage, rather than calling monolithic, expensive-to-deploy, "global" contracts – improving both overall network throughput and real-world privacy.
3. **Reliably-low fees, even for complex contracts** – The per-computation contract validation costs for Bitcoin Cash node operators are thousands of times lower than such costs for node operators in global-state architectures (like Ethereum); existing limits already offer significantly more computation to most Bitcoin Cash contracts than are available to contracts on global-state systems.

### Minimal Impact of Compute on Node Operation Costs

In practice, computation costs of node operators are typically bounded for a given capacity, and actual utilization of computation has a relatively smaller impact on overall costs (e.g. electricity usage for a single system moving from 5% to 25% utilization). On the other hand, bandwidth and storage more commonly incur direct usage-based costs, with bandwidth commonly cited as the leading infrastructure cost of operation, followed by storage costs of increasing blockchain sizes (e.g. for archival nodes, indexers, and other Bitcoin Cash businesses and infrastructure operators).

Beyond preventing abusive contract constructions from creating denial-of-service risks to node operators by using orders of magnitude more computation than plausibly non-malicious contracts (see [Selection of Hashing Limit](#selection-of-hashing-limit) and [Risk Assessment: Denial of Service Risks](./risk-assessment.md#denial-of-service-dos-risks)) – Bitcoin Cash has no pressing need to measure computation in order to charge differing fees based on computation.

By design, this proposal reduces the overall worst-case computation costs of node operation, while significantly extending the universe of relatively inexpensive-to-validate contract constructions available to Bitcoin Cash contract developers. Further changes which might increase worst-case validation costs – like pay-for-compute schemes – are considered out-of-scope for this proposal (see [Use of Input Length-Based Densities](#use-of-input-length-based-densities) for details).

## Non-Impact on "Data Storage" Costs and Incentives

This proposal intentionally avoids impacting the costs and incentives surrounding usage of transactions for storage of arbitrary (often non-financial) data with respect to 1) standard validation rules regarding `OP_RETURN` data carrier outputs, 2) standard validation rules regarding all other data-carrying transactions, and 3) computation-related "data storage" incentives indirectly resulting from all other transaction validation limits.

### `OP_RETURN` Data Carrier Outputs

Following [CHIP-2021-03-12 Multiple OP_RETURNs](https://github.com/ActorForth/Auction-Protocol/blob/main/CHIP-2021-03-12_Multiple_OP_RETURN_for_Bitcoin_Cash.md), data carrier outputs are currently limited to 223 cumulative bytes (including encoding overhead), with a maximum content of 220 bytes. While this constant [appears to have been selected by mistake](https://bitcoincashresearch.org/t/raising-the-520-byte-push-limit-201-operation-limit/282/8), it remains the primary limit to accessible, on-chain data storage with multiple existing indexers and other well-supported infrastructure. As increasing this limit could increase indexing and maintenance costs across a variety of ecosystem software and infrastructure, this proposal considers such increases to be out of scope.

### Data-Carrying Transactions

Beyond the easily-accessible, well-indexed storage available via [`OP_RETURN` data carrier outputs](#op_return-data-carrier-outputs), the next meaningful limit on (less-accessible) data storage is the limit on maximum standard transaction length (A.K.A. `MAX_STANDARD_TX_SIZE`), currently `100,000` bytes. Up to this length, a variety of custom data storage protocols can be designed to pack data into transactions using a variety of constructions, with limits on individual inputs or push sizes exerting a minimal impact (generally less than 1%) on overall storage efficiency. Beyond `100,000` bytes, real-world costs [discourage usage of on-chain storage](https://bitcoincashresearch.org/t/raising-the-520-byte-push-limit-201-operation-limit/282/15) for most kinds of applications. (Note also that Bitcoin Cash's existing practical limits to data-carrying transactions are either equivalent or more conservative than most other networks resulting from bitcoin splits, i.e. BTC, BSV, and XEC.)

This proposal has no impact on the potential usage or efficiency of data-carrying transactions. In practice, most applications would remain best served by storing data off-chain and committing only the hash of such data into on-chain transactions (particularly in common, infrastructure-supported [`OP_RETURN` Data Carrier Outputs](#op_return-data-carrier-outputs)).

### Indirect, Transaction "Padding" Incentives

This proposal reduces indirect incentives for existing contracts to increase overall blockchain storage usage.

Because the existing statically-defined limits create arbitrary (i.e. inconsistent with respect to performance requirements) partitions in evaluation of existing contracts – requiring computations to be split across multiple inputs – the net effect of this proposal on all existing contract usage is to eliminate overhead (e.g. additional transactions and/or per-input encoding overhead) which is currently required to work around the existing, per-contract evaluation limits. For all currently-possible contracts, this **maintains or reduces resulting transaction sizes and overall blockchain data usage**.

However, while this proposal's re-targeted limits allow sufficient room for most contract developers to work without any consideration for VM limits, a subset of currently-possible-but-impractical contracts warrants further review:

- Within the existing limits, evaluations requiring high-density and/or high-precision arithmetic – e.g. emulation of post-quantum cryptography, zero-knowledge proofs, homomorphic encryption, etc. – must be split into multiple inputs, with intermediate results passed across evaluations to allow computation to continue (working around the existing limits). In practice, the excessive cost of contract development and security review limits production deployment of such multi-input systems. Worse, transactions resulting from such systems would require significant per-input and per-output overhead, reducing their efficiency and increasing their overall impact on blockchain storage usage.

- It could be argued that this proposal (coupled with [`CHIP-2024-07-BigInt`](https://github.com/bitjson/bch-bigint)) creates an incentive for these currently-theoretical, arithmetic-dense contracts to "pad" their [density control length](./readme.md#density-control-length)s with otherwise-unnecessary data in order to increase the maximum operation cost allocated to the input's evaluation. However, this argument reverses causation; in fact, these proposals so significantly reduce the overhead of arithmetic-dense contracts that many such contracts become efficient enough for realistic production development and deployment. (Note also that these are "economic use cases" related to securing or transacting BCH and/or CashTokens.<sup>1</sup>) In practice, these proposals produce an unequivocal improvement in the status quo with respect to overall blockchain storage efficiency, and they produce no meaningful change in the incentives surrounding "wasted" data usage. While further improvements are possible (see [Rationale: Use of Input Length-Based Densities](#use-of-input-length-based-densities)), this proposal considers further optimizations to be out of scope.

<details>

<summary>Notes</summary>

1. Even presuming a value judgement that "economic use cases" ought to be prioritized above equally-fee-paying "data storage use cases" (a statement on which this proposal remains neutral), the contracts in question can only reasonably be considered "economic use cases":

   - The purpose of the Bitcoin Cash VM is to adjudicate asset transfer rights without a central authority (for both BCH and CashTokens).

   - In this context, "data storage use cases" might be defined as any use cases in which this adjudication is not used. I.e. instead of relying on the VM for rights adjudication (and by extension, on-chain control of assets), off-chain systems simply use the network as a censorship-resistant messaging layer. Though data storage use cases can technically use the VM in purely-ceremonial ways (e.g. pretending to perform the same on-chain computations as have taken place in the off-chain system), if a system's consensus is reliant on an off-chain protocol, there remains little difference between committing inputs-plus-computation or fully-computed results to the chain (primarily transaction fee efficiency, where committing computed results will be less expensive than committing inputs and ceremonially re-performing the computation on-chain).

   - In contrast, any use case which _does_ require VM evaluation for adjudication of on-chain asset transfer rights (BCH and/or CashTokens), can only reasonably be described as an "economic use case", e.g. single-signature addresses, multi-party vaults, decentralized application treasuries, zero-knowledge proof covenants, sidechain bridges, etc. Within reasonable limits defined to protect network bandwidth, storage, or computation from abusive usage, further distinction between the qualifications of various VM-evaluated contracts as "economic" vs. "non-economic" becomes subjective/arbitrary.

</details>

## Retention of Control Stack Limit

This proposal avoids modifying the existing practical limit on control stack depth by introducing a specific `Control Stack Limit` in place of the current effective limit<sup>1</sup>.

While it is possible to implement an [O(1) Control Stack](https://github.com/bitcoin/bitcoin/pull/16902) such that deeply nested conditionals have no impact on transaction validation performance, requiring this optimization in all performance-critical VM implementations may have significant consequences for future protocol complexity, particular if future upgrades extend the VM's control flow capabilities. (E.g. [bounded loops](https://github.com/bitjson/bch-loops), switch or match expressions, word/function definition, etc.)

The existing control stack depth of `100` is already far in excess of all known usage. Additionally, given both the availability of the alternate stack and this proposal's replacement of the operation limit, it is possible to emulate practically any algorithm requiring excessive control stack depth with an equivalent, less-deeply nested implementation. As such, an increase in the existing limit on control stack depth is considered out of this proposal's scope.

<details>

<summary>Notes</summary>

1. The existing `201` operation limit prevents any currently-valid contract from requiring a control stack of depth greater than `100`, e.g. `<1> OP_IF <1> OP_IF <1> OP_IF ... OP_ENDIF OP_ENDIF OP_ENDIF`.

</details>

## Use of Input Length-Based Densities

This proposal limits both [hashing](readme.md#hashing-limit) and [operation cost](readme.md#operation-cost-limit) to maximum densities based on the approximate byte length of the input under evaluation (see [Rationale: Selection of Input Length Formula](#selection-of-input-length-formula)).

As is currently required within the existing system of limits, contract systems requiring additional computation following activation of this proposal may continue to rearrange or [break up expensive computations into multiple inputs](https://bitcoincashresearch.org/t/chip-2021-05-targeted-virtual-machine-limits/437/6). As this proposal attempts to avoid any impact to worst-case validation time – and future upgrades can deploy increases in operation cost limits – additional solutions for increasing per-input operation cost limits are considered out of this proposal's scope.

### Alternative: Transaction Length-Based Densities

Alternatively, this proposal could measure densities relative to the byte length of the full containing transaction, sharing a budget across all of the transaction's inputs. Such a transaction-based approach would provide contracts with the most generous possible computation limits given the transaction fees paid, allowing computationally-expensive inputs to also claim the computing resources purchased via the bytes of transaction overhead, outputs, and of other inputs. Additionally, if a future upgrade were to [relax output standardness](https://bitcoincashresearch.org/t/p2sh32-a-long-term-solution-for-80-bit-p2sh-collision-attacks/750/13#relaxing-output-standardness-1), transaction-based budgets would also offer non-P2SH (Pay to Script Hash) contracts a greater portion of the computing resources purchased via the transaction's other bytes, particularly for contracts which rely mainly on introspection for validation and include little or no unlocking bytecode (e.g. the ["Merge Threads" script in Jedex](https://github.com/bitjson/jedex)).

However, this proposal's input-based approach is superior in that it: 1) allows contract authors to reliably predict a contract's available limits regardless of the size and other contents of the spending transaction, 2) ensures that contract evaluations which do not exceed VM limits can be composed together in multi-input transactions without further regard for VM limits, 3) preserves the simplicity of parallelizing input validation without cross-thread communication, 4) more conservatively limits the worst-case validation performance of maliciously-invalid transactions (by failing the malicious transaction earlier), and 5) could be expanded into a transaction-based approach by a future upgrade if necessary.

### Alternative: UTXO-Length Increased Densities

Another alternative to offer greater limits to non-P2SH contracts would be to base densities on the byte length of both locking and unlocking bytecode, i.e. including the Unspent Transaction Output (UTXO) in the input's budget. However, this alternative approach would increase the volatility of worst-case transaction validation performance: the price of the UTXO's past bandwidth is paid by the previous transaction's mining fee, while the price of the UTXO's storage is paid only by the time-value of associated dust; if compute resources aren't strictly tied to costs in the present (like the current transaction's mining fee), the instantaneous computation requirements of transaction validation are also not bounded by limits in the present (e.g. slow-to-validate UTXOs may be stored up and evaluated in a smaller number of attack transactions). Finally, depending on the precise economics of particular attacks, limits enhanced by UTXO length may incentivize would-be attackers to inflate the UTXO set in advance of an attempted denial-of-service attack, increasing its cost-effectiveness. Instead, this proposal bases computation limits only on costs paid in the present, with the 41-byte minimum input length providing a reasonable minimum computation budget.

### Alternative: Transaction Fee-Increased Densities

Finally, this proposal could also increase the operation cost limit proportionally to the per-byte mining fee paid or based on the [declaration of "virtual" bytes](https://bitcoincashresearch.org/t/chip-2021-05-targeted-virtual-machine-limits/437/108), e.g. for fee rates of `2` satoshis-per-real-byte, the VM could allow a per-byte budget of `2000` (a `2.5` multiple, incentivizing contracts to pay the higher fee rate rather than simply padding the unlocking bytecode length). However, any allowed increase in per-real-byte operation cost also equivalently changes the variability of per-byte worst-case transaction validation time; such flexibility would need to be conservatively capped and/or interact with the block size limit to ensure predictability of transaction and block validation requirements. Additionally, future research based on real-world usage may support simpler alternatives, e.g. a one-time increase in the per-byte operation cost budget.

## Selection of Input Length Formula

The `Density Control Length` used by the [Hashing](readme.md#hashing-limit) and [Operation Cost](readme.md#operation-cost-limit) limits in this proposal approximates the encoded length of inputs by adding the length of the unlocking bytecode (A.K.A. `scriptSig`) to the constant `41`: the minimum possible per-input overhead of version `1` and `2` transactions<sup>1</sup>. This approach both simplifies implementations – by avoiding the need to either check the length of the serialized input or compute its expected length – and simplifies some contract development and deployment concerns by avoiding the 2-byte change in available budget as contract length increases from `252` bytes (encoded as `0xfc`) to `253` bytes (encoded as `0xfdfd00`).

Additionally, by avoiding direct use of encoded input length in limit calculations, the selected formula avoids directly tying the application of VM limits to the encoding format of the containing transaction. This ensures that contracts designed for use in version `1` and `2` transactions may retain equivalent limits even if evaluated within future transaction version(s) that more efficiently encode transaction inputs.

<details>

<summary>Notes</summary>

1. Outpoint Transaction Hash (32 bytes), Outpoint Index (4 bytes), `Unlocking Bytecode Length` (minimum 1 byte), `Unlocking Bytecode`, and Sequence Number (4 bytes): `32 + 4 + 1 + 4 = 41`.

</details>

## Hashing Limit by Digest Iterations

One possible alternative to the proposed [Hashing Limit](readme.md#hashing-limit) design is to limit evaluations to a fixed count of "bytes hashed" (rather than digest iterations). While this seems simpler, it performs poorly in an adversarial environment: an attacker can choose message sizes to minimize bytes hashed while maximizing digest iterations required (e.g. 1-byte messages).

In practice, because the performance cost of hashing a 55-byte message is equivalent to the performance cost of hashing a single-byte message, any hashing limit which fails to account for the internal block-digest behavior of the [Merkle–Damgård](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) hashing algorithm will be vulnerable to a 55x magnification in worst-case validation costs (see [Digest Iteration Count](./readme.md#digest-iteration-count)).

By directly measuring digest iterations, non-malicious contracts can be allowed to evaluate a higher density of hashing operations without increasing the worst-case validation requirements of the VM as it existed prior to this proposal.

## Selection of Hashing Limit

This proposal sets the standard (transaction relay policy) hashing density limit at `0.5` digest iterations per spending input byte. This value is the asymptotic maximum density of hashing operations in plausibly non-malicious, standard transactions within the current VM limits and instruction set: assuming no transaction encoding overhead and that all hashed material is maximally-padded (1-byte segments), the most efficient methods for reducing hashed material into a single result (without any further processing) requires 1 additional operation (e.g. `OP_CAT`) per hashing operation<sup>1</sup>.

Note that this standard limit is also well in excess of the highest density found in any standard transaction maximizing hashing within signature checking operations: `0.30` digest iterations per byte<sup>2</sup>.

For block validation, this proposal sets the consensus limit (A.K.A. "nonstandard") hashing density at `3.5` digest iterations per spending input byte. This limit exceeds the highest-known density of any currently standard transaction of `3.44` digest iterations per byte<sup>3</sup>. Ideally, a separate nonstandard limit could be avoided, but because currently-standard transactions can be designed to exceed the `0.5` limit, a higher nonstandard limit is necessary to avoid immediately invaliding any currently-standard transactions<sup>4</sup>. Future proposals could schedule an upgrade to consolidate these limits.

Alternatively, this proposal could set a consensus hashing limit without a lower standardness limit. However, hashing cannot be [reliably deferred like signature validation](#continued-availability-of-deferred-signature-validation); because the current practical limit is nearly an order of magnitude higher then theoretically useful, limiting the density allowed in standard validation significantly improves worst-case validation performance.

<details>

<summary>Notes</summary>

1. The two-round hashing operations (`OP_HASH160` and `OP_HASH256`) would be wasteful in this idealized scenario, as 1-byte inputs already require padding in the first round of hashing, so the idealized maximum density is 1 digest iteration per 2 opcodes. Note that while a future upgrade could increase this density by compressing the evaluating code (e.g. with [bounded loops](https://github.com/bitjson/bch-loops)) or allowing significant lengths of hashed material to be provided via UTXO bytecode (e.g. by relaxing output standardness), this example still 1) excludes the minimum `41` bytes of input overhead (allowing up to `1271` bytes hashed in `20` hash digest iterations), 2) significantly overestimates hashing density by assuming exclusively 1-byte input lengths rather than more common lengths (e.g. `20` or `32`), and 3) excludes overhead from the non-hashing operations performed in realistic contracts.
2. See benchmark `xfszef`.
3. See benchmark `lcennk`.
4. This proposal follows the [long-established](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/commit/a206a23980c15cacf39d267c509bd70c23c94bfa) process of restraining abusive usage by invalidating currently-standard malicious behavior via relay policy (A.K.A. "standardness"), and then only restricting the behavior from block validation (A.K.A. "consensus") after it has remained restricted for a significant period of time (e.g. [`NULLDUMMY`](https://github.com/bitcoin/bips/blob/master/bip-0147.mediawiki), [`NULLFAIL`](https://upgradespecs.bitcoincashnode.org/nov-13-hardfork-spec/), [`MINIMALDATA`](https://upgradespecs.bitcoincashnode.org/2019-11-15-minimaldata/), etc.).

</details>

## Exclusion of Signing Serialization Components from Hashing Limit

This proposal includes [the cost of hashing all signing serializations](readme.md#transaction-signature-checking-operations) in the proposed `Hashing Limit`, but it excludes the cost of any hashing required to produce the internal components of signing serializations (i.e. `hashPrevouts`, `hashUtxos`, `hashSequence`, and `hashOutputs`). This configuration reduces protocol complexity while correctly accounting for the real-world hashing cost of `coveredBytecode` (A.K.A. `scriptCode`), particularly in [preventing abuse of `OP_CODESEPARATOR`](https://gist.github.com/markblundeberg/c2c88d25d5f34213830e48d459cbfb44), future increases to maximum standard unlocking bytecode length (A.K.A. `MAX_TX_IN_SCRIPT_SIG_SIZE` – 1,650 bytes), and/or increases to consensus-maximum VM bytecode length (A.K.A. `MAX_SCRIPT_SIZE` – 10,000 bytes).

Alternatively, this proposal could also require internally accounting for the cost of hashing within signing serialization components. However, because components can be cached across signature checks – and for `hashPrevouts`, `hashUtxos`, `hashSequence`, across the entire transaction validation – such costs would need to be amortized across all of a transaction's signature checks to approximate their fixed, real-world cost. As signature checking is already sufficiently limited by `SigChecks` and operation cost, omitting hash digest iteration counting of signing serialization components reduces unnecessary protocol complexity.

### Ongoing Value of OP_CODESEPARATOR Operation

The `OP_CODESEPARATOR` operation modifies the signing serialization (A.K.A. "`SIGHASH`" preimage) used in later-executed signature operations by truncating the `coveredBytecode` component at the index of the last executed `OP_CODESEPARATOR` operation.

The `OP_CODESEPARATOR` behavior is useful for designing contracts in which a single key may provide signatures for use in multiple contract code paths. Without `OP_CODESEPARATOR`, a valid signature by the key could be re-used by an attacker in a manipulated version of the input's unlocking bytecode to unlock the contract following a different code path; if `OP_CODESEPARATOR` is executed at some point before any later signature checking operation, a valid signature for the intended code path will fail validation if the unlocking bytecode is manipulated to follow another code path.

Notably, the behavior of `OP_CODESEPARATOR` can be achieved more directly with `OP_CHECKDATASIG`: for each code path, the locking bytecode must simply require a different message to be signed by the data signature, preventing signatures from being reused across code paths. However, `OP_CODESEPARATOR` can be more byte-efficient in that it requires only one additional byte (the `OP_CODESEPARATOR` instruction) to differentiate the two signature preimages, while `OP_CHECKDATASIG` may require additional instructions or encoded data to differentiate each possible message.

While past proposals (for both Bitcoin Cash and similar networks) have called for deprecating or disabling `OP_CODESEPARATOR`, this proposal retains the feature while preventing excessive resource consumption, as `OP_CODESEPARATOR` is already available in the Bitcoin Cash instruction set, may have active users, and remains useful in some scenarios.

## Increased Usability of Multisig Stack Clearing

Prior to this proposal, the multisig operations (`OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY`) were limited in part via incrementing the operation count by the number of stack items provided as public keys. However, because public keys are only validated prior to their use in a signature check, it is also possible to use these operations to more efficiently drop items from the stack than via repeated calls to `OP_DROP`, `OP_2DROP`, or `OP_NIP`.

While this eccentric behavior was likely unintentional, it has been available to contract authors since Bitcoin Cash's 2009 launch, and it is generally the most byte-efficient method of dropping between 9 and 20 stack items (reducing transaction sizes for some contracts). Because this existing stack-clearing behavior is a useful feature and does not meaningfully impact transaction validation costs, this proposal treats zero-signature multisig operations equivalently to all other constant and linear time operations.

## Limitation of Pushed Bytes

This proposal limits the density of both memory usage and computation by limiting bytes pushed to the stack to approximately `700` per spending input byte (the per-byte budget of `800` minus the base instruction cost of `100`).

Because stack-pushed bytes become inputs to other operations, limiting the overall density of pushed bytes is the most comprehensive method of limiting all sub-linear and linear-time operations (bitwise operations, VM number decoding, `OP_DUP`, `OP_EQUAL`, `OP_REVERSEBYTES`, etc.); this approach increases safety by reducing the risk that any unexpected source of linear time complexity (e.g. due to a defect in a particular VM implementation) might create practically-exploitable performance issues (providing [defense in depth](<https://en.wikipedia.org/wiki/Defense_in_depth_(computing)>)) and reduces protocol complexity by avoiding special accounting for all but the most expensive operations (arithmetic, hashing, and signature checking).

Additionally, this proposal’s density-based limit caps the maximum memory and memory bandwidth requirements of validating a large stream of transactions, regardless of the number of parallel validations being performed<sup>1</sup>.

Alternatively, this proposal could limit memory usage by continuously tracking total stack usage and enforcing some maximum limit. In addition to increasing implementation complexity (e.g. performant implementations would require a usage-tracking stack), this approach would 1) _implicitly_ limit memory bandwidth usage and 2) require additional limitations on linear-time operations.

<details>

<summary>Notes</summary>

1. While the 10KB stack item length limit and 1000 item stack depth limit effectively limit maximum memory to 10MB plus overhead per validating thread, without additional limits on writing, memory bandwidth can become a bottleneck. This is particularly critical if future upgrades allow for increased contract or stack item lengths, [bounded loops](https://github.com/bitjson/bch-loops), word/function definition, and/or other features that reduce transaction sizes by increasing their operation density.

</details>

## Unification of Limits into Operation Cost

While this proposal maintains independent limits for both signature checking and hashing (via `SigChecks` and hash digest iterations, respectively), the measurements used to enforce each limit also contribute to the unified `Operation Cost Limit`. This configuration prevents maliciously designed transactions from maximizing validation cost across multiple disparate limits, reducing the potential disparity in worst case performance between honest and malicious usage.

## Selection of Operation Cost Limit

This proposal sets the operation cost density at `800` per spending input byte. Given a base instruction cost of `100`, this density ensures a net budget of `700` per spending input byte, a constant rounded up from the maximum effective limit prior to this proposal<sup>1</sup>.

<details>

<summary>Notes</summary>

1. Benchmark `d0kxwp` (`99966` bytes) pushes `62757175` bytes, approximately `627.79` bytes per spending input byte.

</details>

## Selection of Base Instruction Cost

To retain a conservative limit on contract operation density, this proposal sets a base operation cost of `100` per evaluated instruction, including unexecuted and push operations.

Given the [operation cost density limit of `800`](readme.md#operation-cost-limit), a base instruction cost of `100` ensures that the maximum operation density within spending inputs (8 per byte) remains within an order of magnitude of the current effective limit (approximately 1 per byte) resulting from the 201 operation limit.

Because base instruction cost can be reduced - but not increased - in future upgrades without invalidating contracts<sup>1</sup>, a base cost of `100` is more conservative than lower values (like `10`, `1`, or `0`). Additionally, a relatively-high base instruction cost leaves room for future upgrades to differentiate the costs of existing low and fixed-cost instructions, if necessary.

Alternatively, this proposal could omit any base instruction cost, limiting the cost of all instructions to their impact on the stack. However, instruction evaluation is not costless – a nonzero base cost properly accounts for the real world overhead of evaluating an instruction and verifying non-violation of applicable limits.

Finally, setting an explicit base instruction cost reduces the VM’s implicit reliance on bytecode length for limiting worst case validation performance. By explicitly limiting maximum operation density, future upgrades which increase the maximum allowable contract length or reduce transaction sizes by compressing contract code (e.g. [bounded loops](https://github.com/bitjson/bch-loops) or word/function definition) can be supported without incurring technical debt and/or producing de facto limit increases.

<details>

<summary>Notes</summary>

1. Note that pre-signed transactions, contract systems, and protocols can be specifically designed to operate on any observable characteristic of the VM, including limits. See [Notice of Possible Future Expansion](readme.md#notice-of-possible-future-expansion).

</details>

## Inclusion of Numeric Encoding in Operation Costs

The VM number format is designed to allow efficient manipulation of arbitrary-precision integers (Satoshi's initial VM implementation used the OpenSSL library's Multiple-Precision Integer format and operations), so sufficiently-optimized VM implementations can theoretically avoid the overhead of decoding and (re)encoding VM numbers. However, if this proposal were to assume zero operation cost for encoding/decoding, this optimization would be required of all performance-critical VM implementations to avoid divergence of real performance from measured operation cost.

Instead, this proposal increments the operation cost of all operations dealing with potentially-large numbers (greater than `2**32`) by the byte length of their numeric output (in effect, doubling the cost of pushing the output). This approach accounts for the cost of re-encoding numerical results from another internal representation as VM numbers – regardless of the underlying arithmetic implementation. (Note that the cost of decoding VM number inputs is already accounted for (in advance) by the comprehensive limiting of pushed bytes. See [Rationale: Limitation of Pushed Bytes](#limitation-of-pushed-bytes).)

Because operation cost can be reduced - but not increased - in future upgrades without invalidating contracts<sup>1</sup>, this proposal's approach is considered more conservative than omitting a cost for re-encoding.

<details>

<summary>Notes</summary>

1. Note that pre-signed transactions, contract systems, and protocols can be specifically designed to operate on any observable characteristic of the VM, including limits. See [Notice of Possible Future Expansion](readme.md#notice-of-possible-future-expansion).

</details>

## Selection of Signature Verification Operation Cost

In addition to the operation cost of hash digest iterations performed within signature checking operations, this proposal sets the operation cost of signature verification to `26000` per check. This constant is rounded down from the per-check budget available within current limits<sup>1</sup>.

By deriving this limit to be practically equivalent - but slightly more generous - than the existing SigChecks limit, this proposal ensures that all currently-standard transactions remain valid, while avoiding any significant increase in worst case validation performance vs. current limits.

Note that because Schnorr signatures can be batch validated, this proposal could reasonably have set another, lower cost for such signatures. However, as signature validation is generally the bottleneck in worst case validation performance of currently-standard transactions, it may be prudent to avoid extending higher limits for Schnorr signatures and to instead allow any such performance gains to simply improve average validation performance.

Additionally, for contracts to take advantage of any such operation cost discount for Schnorr signatures, the separate SigChecks limits would likely also need to be loosened for Schnorr signatures, and in both cases, a differentiated limit would essentially required all VM implementations to implement the batch verification optimization to avoid negatively impacting worst case performance.

Given these trade-offs, this proposal takes the more conservative approach of applying a fixed operation cost per signature check based only on the existing, worst case scenario.

<details>

<summary>Notes</summary>

1. Given the `SigChecks` standardness limit of 1 check per `33.5` bytes and a per-byte operation budget of `800`, the current implied cost of a signature check is `26800` (`33.5 * 800 = 26800`). Benchmark `l0fhm3` (`Within BCH_2023_05 standard limits, packed 1-of-3 bare multisig inputs, 1 output (all ECDSA signatures, first slot)`) demonstrates transaction validation performance similar to a worst-case scenario for this metric – it contains `2619` signature checks in `99988` bytes (`262`-byte signing serializations; 5 hash digest iterations per signature check), with a standard cost of `3867390` before accounting for signature checks and a remaining budget of approximately `28105` per signature check (`((99988 * 800) - 3867390 - (2619 * 5 * 192)) / 2619 ~= 28105.68`).

</details>

## Continued Availability of Deferred Signature Validation

By using the existing `SigCheck` metric in computing operation cost, this proposal retains the ability to defer signature validation (as described in the [2020-05 SigChecks specification](https://gitlab.com/bitcoin-cash-node/bchn-sw/bitcoincash-upgrade-specifications/-/blob/master/spec/2020-05-15-sigchecks.md)) until the rest of a contract has been evaluated. Notably, enforcement of the proposed hashing limit within signature checking operations can also be deferred until immediately prior to signature validation, though because other hashing operations must be evaluated during program execution, deferring either hashing or digest iteration estimation will not improve worst-case validation performance.

## Selection of Hash Digest Iteration Cost

For block validation (consensus), this proposal sets the operation cost of hash digest iterations to `64` (the hashing algorithm block size); for transaction relay (standardness policy) this proposal sets the operation cost of hash digest iterations to `192` (`64*3=192`). These values ensure that all possible standard transactions remain valid by consensus while accounting for the cost of hashing during standard validation.

Each VM-supported hashing algorithm – RIPEMD-160, SHA-1, and SHA-256 – uses a [Merkle–Damgård construction](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) with a `block size` of 512 bits (64 bytes); operation cost is essentially a count of bytes processed by any VM operation, so the hashing algorithm block size (`64`) is the most appropriate multiple upon which to base hashing operation cost calculations.

Under the existing limits, the highest possible hashing density of a standard transaction is `3.44` hash digest iterations per spending input byte<sup>1</sup>. Given a budget of `800` per byte, it is not possible to set a higher multiple of `64` for the consensus operation cost of hash digest iterations (e.g. `128`) without invalidating currently-standard transactions<sup>2</sup>. As such, this proposal sets the consensus operation cost of hash digest iterations to `64`.

To determine the standard operation cost of hash digest iterations, the results of multiple representative benchmarks can be compared across multiple independent VM implementations; in a variety of cases, given a [signature checking cost of `26000`](#selection-of-signature-verification-operation-cost), the closest multiple of `64` to real-world performance cost is `192`<sup>3</sup>.

<details>
<summary>Notes</summary>

1. Benchmark `lcennk`.
2. For example, benchmark `lcennk` (`99966` bytes) with a hash digest iteration cost of `128` results in a cumulative operation cost of `87910420`, or `879.40` per byte.
3. On one representative system, [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/), performs the baseline benchmark (`trxhzt`) at `11759.3` Hz (`2` sigchecks and `12` iterations) and a hashing-heavy benchmark (`lcennk`) at `10.6` Hz (`0` sigchecks and `344162` iterations). Given `11759.3 (2s + 12h) = 10.6 (344162h)`, each signature check costs about 149 hash digest iterations; calibrating for a signature check cost of `26000`, `26000 / 149.12 ~= 174`, the measured cost per digest iteration is approximately `174`. Likewise, [Libauth](https://github.com/bitauth/libauth), an unrelated JavaScript implementation using WebAssembly-based, software-only implementations of the relevant cryptographic algorithms, performs `trxhzt` at `2966.08` Hz and `lcennk` at `2.56` Hz. Given `2966.08 (2s + 12h) = 2.56 (344162h)`, each signature check costs about 142 hash digest iterations; calibrating for a signature check cost of `26000`, `26000 / 142.52 ~= 182`, the measured cost per digest iteration is approximately `182`.

</details>

## Inclusion of "Notice of Possible Future Expansion"

This proposal includes a [Notice of Possible Future Expansion](readme.md#notice-of-possible-future-expansion) to clarify established precedent in Bitcoin (Cash) protocol development: pre-signed transactions, contract systems, and protocols which rely on the non-existence of VM features must be interpreted as intentional. This precedent was established by Satoshi Nakamoto in [`reverted makefile.unix wx-config -- version 0.3.6`](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/commit/757f0769d8360ea043f469f3a35f6ec204740446) (July 29, 2010) with the addition of new opcodes `OP_NOP1` to `OP_NOP10`, and the precedent was reaffirmed by numerous upgrades to Bitcoin (Cash), e.g. [`OP_CHECKLOCKTIMEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki), [Relative locktime](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki), [`OP_CHECKSEQUENCEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki), the [2018 opcode restoration](https://upgradespecs.bitcoincashnode.org/may-2018-reenabled-opcodes/), [CHIP-2021-03: Bigger Script Integers](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md#notice-of-possible-future-expansion), and other upgrades.
