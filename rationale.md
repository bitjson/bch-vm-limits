## Rationale

This section documents design decisions made in this specification.

### Density-based Operational Cost Limit

The objective of this upgrade is to allow smart contract transactions to do more, and without any negative impact to network scalability.
With the proposed approach of limiting operational cost density, we can guarantee that processing cost of a block packed with smart contract transactions can't exceeed the cost of a block packed full of typical payment transactions (pay-to-public-key-hash transactions, abbreviated P2PKH).
Those kinds of transactions make more than 99% of Bitcoin Cash network traffic and are thus a natural baseline for scalability considerations.

Trade-off of limiting density (rather than total cost) is that transaction size may be intentionally inflated by users in order to "buy" more total operational budget for some input's script, in effect turning transaction's bytes into a form of "gas".
Transactions having such filler bytes still wouldn't negatively impact scalability, although they would appear wasteful.
These filler bytes would have to pay transaction fees just like any other transaction and we don't expect users to make these kinds of transactions unless they have economically good reasons, so this is not seen as a problem.
With the density-based approach, we can have maximum flexibility and functionality so this is seen as an acceptable trade-off.

What are the alternatives?

If we simply limited total transaction's operation cost, we'd still achieve the objective of not negatively impacting network scalability, but at the expense of flexibility and functionality: a big transaction would have as much operational cost budget as a small transaction, meaning it could not do as much with its own bytes, even when the bytes are not intentionally filler bytes.
To be useful, bigger transactions normally have to operate on more UTXOs or data, so they need to execute proportionally more operations than smaller transactions.
If we limited total operations, contract authors would then have to work around the limitation by creating chains of transactions to carry out the operations rather than packing all operations in one transaction - and that would result in more overheads and being relatively more expensive for the network to process while also complicating contract design for application developers.
This is pretty much the status quo, which we are hoping to improve on.

Another alternative is to introduce a kind of gas system, where transactions could declare how much processing budget they want to buy, e.g. declare some additional "virtual" bytes without actually having to encode them.
Then, transaction fees could be negotiated based on raw + virtual bytes, rather than just raw bytes.
This system would introduce additional complexity and for not much benefit other than saving some network bandwidth for those exotic cases.
Savings in bandwidth could be alternatively achieved on another layer: by compressing TX data, especially because filler bytes an be highly compressible (e.g. data push of 1000 0-bytes).

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

To retain a conservative limit on contract operation density, this proposal sets a base operation cost of `100` per evaluated instruction, including unexecuted and push operations.

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

This proposal includes a [Notice of Possible Future Expansion](#notice-of-possible-future-expansion) to clarify established precedent in Bitcoin (Cash) protocol development: pre-signed transactions, contract systems, and protocols which rely on the non-existence of VM features must be interpreted as intentional. This precedent was established by Satoshi Nakamoto in [`reverted makefile.unix wx-config -- version 0.3.6`](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/commit/757f0769d8360ea043f469f3a35f6ec204740446) (July 29, 2010) with the addition of new opcodes `OP_NOP1` to `OP_NOP10`, and the precedent was reaffirmed by numerous upgrades to Bitcoin (Cash), e.g. [`OP_CHECKLOCKTIMEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki), [Relative locktime](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki), [`OP_CHECKSEQUENCEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki), the [2018 opcode restoration](https://upgradespecs.bitcoincashnode.org/may-2018-reenabled-opcodes/), [CHIP-2021-03: Bigger Script Integers](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md#notice-of-possible-future-expansion), and later upgrades.
