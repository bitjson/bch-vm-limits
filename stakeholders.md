# Stakeholder Responses & Statements

<details>

<summary><strong>Table of Contents</strong></summary>

- [Stakeholder Responses](#stakeholder-responses)
  - [Nodes](#nodes)
  - [Wallets](#wallets)
  - [Projects](#projects)
  - [Industry](#industry)
- [Statements](#statements)
  - [Disapprove](#disapprove)
  - [Neutral](#neutral)
  - [Approve](#approve)

</details>

The goal of the [Cash Improvement Proposal (CHIP) process](https://blog.bitjson.com/bitcoin-cash-upgrade-2023/#why-chips) is to coordinate upgrades to Bitcoin Cash without a central authority by developing proposals in a collaborative, public process.

**CHIP maintainers are expected to actively seek out stakeholders and demonstrate widespread ecosystem approval.**

This includes approval from nodes, wallets, and other open source software projects, educational institutions and community initiatives, and industry actors like exchanges, miners, services, and other businesses.

## Stakeholder Responses

<details>

<summary>Pending</summary>

_Full table will be published and continuously updated from October 14 through November 14. For a past example, see [CashTokens Stakeholder Responses](https://github.com/cashtokens/cashtokens/blob/master/stakeholders.md#stakeholder-responses)._

</details>

## Statements

The following public statements have been submitted in response to this CHIP.

### Approve

The following articles have been published in support of this CHIP:

- [bitjson.com](https://bitjson.com/) (October 1, 2024): "[2025 Bitcoin Cash Improvement Proposals](https://blog.bitjson.com/2025-chips/)"

The following statements have been submitted in support of this CHIP.

> The VM Limits CHIP retargets Bitcoin Cash's Denial-of-Service limits to extend compute for real contracts by more than 100x while reducing worst-case node compute usage by 50%. By reducing overhead, the retargeted limits simplify contracts, reduce transaction sizes, streamline contract audits, and improve overall security.
>
> By improving contract efficiency, this upgrade also makes important use cases more practical, including post-quantum cryptography, stronger escrow and settlement strategies, zero-knowledge proofs, homomorphic encryption, and other crucial innovations for the future security and competitiveness of Bitcoin Cash.
>
> Finally, this upgrade raises the bar by contributing new tooling and a cross-implementation benchmarking methodology to continuously verify node performance. Beyond empirically verifying the safety and correctness of the upgrade, these tools will simplify development of new production-ready implementations, prevent regressions in existing implementations, and reduce the cost of verifying implementation-specific software updates.
>
> I'm confident in the extensive cross-implementation testing and verification performed for this CHIP, and I recommend activation in Bitcoin Cash's 2025 upgrade.
>
> —<cite>Jason Dreyzehner ([@bitjson](https://x.com/bitjson/status/1842044715622858846)), [Bitauth IDE](https://ide.bitauth.com), [Chaingraph](https://chaingraph.cash/), [Libauth](https://libauth.org/)</cite>

> I, Calin Culianu, believe the VM Limits and BigInt CHIPs should be included in the BCH upgrade for May 2025.
>
> I was excited about the VM Limits CHIP from at least February of 2024 and began considering it and trying out toy implementations of the earlier version of the CHIP since then. I subsequently worked with Jason a great deal with lots of back and forth in order to evolve the design, and I'm very confident in the final VM Limits CHIP design and rationale. I believe it will unlock a lot of extra capabilities for smart contracts on BCH, as well as plug some performance holes in terms of worst-case performance.
>
> In short: As a result of working on the VM Limits implementation for BCHN, I came to deeply understand its design and rationale and definitely endorse it. I think it is very well thought out and the idioms and costing concepts it invents give us new "knobs" to turn should we decide to turn up the script execution engine limits in the future. It is the right design, and it can take us into the future.
>
> Additionally – The BigInt CHIP came up as an offshoot of the VM Limits work. I believe bringing ~80k-bit integers to BCH script is a huge step forward as it unlocks many previously-impossible contract capabilities. While working on a proof of concept implementation of this CHIP, I was shocked at how fast BigInts are and how low the performance cost for allowing them truly is. It's a no-brainer in my mind to add this additional BigInt CHIP. What's more, VM Limits + BigInt work perfectly together and the costing scheme involved in VM Limits keeps the BigInt changes constrained, so no contract can abuse BigInts to create "poison" transactions or blocks that overuse CPU or memory when validating.
>
> As a Bitcoin Cash developer with many years of experience in this space, I endorse both the VM Limits and BigInt CHIPs for the upcoming May 2025 upgrade to BCH.
>
> —<cite>Calin Culianu, [Bitcoin Cash Node](https://bitcoincashnode.org/) contributor, [Fulcrum](https://github.com/cculianu/Fulcrum) lead developer, [Electron Cash](https://electroncash.org/) contributor</cite>

> The re-targeted VM Limits CHIP solves a real, pressing problem for BCH smart contract developers while also improving the network's security by preventing some worst-case malicious contracts. The research and testing that went into this CHIP is of outstanding quality and sets a standard for future proposals to aspire to.
>
> —<cite>Mathieu Geukens, [Cashonize](https://www.cashonize.com/) creator, [CashScript](https://www.cashscript.org/) developer, [Tokenaut](https://www.tokenaut.cash/) creator</cite>

> I fully support the `CHIP-2021-05-vm-limits: Targeted Virtual Machine Limits` proposal for Bitcoin Cash. This upgrade enables more advanced, efficient smart contracts by removing outdated restrictions while maintaining safeguards, allowing contract authors to focus on application logic rather than working around the limits.
>
> The increased stack element size, revised operation cost system, and targeted hashing limits are well-reasoned improvements enhancing BCH's capabilities without compromising security or performance, and the extensive rationale, benchmarks, and risk assessment provide confidence in its design.
>
> This upgrade is a crucial step in BCH's evolution as a powerful, flexible cryptocurrency platform, and I'm looking forward to designing contract systems leveraging these new features!
>
> —<cite>[bitcoincashautist](https://gitlab.com/0353F40E), Bitcoin Cash researcher & developer, [CHIP-2023-04 Adaptive Blocksize Limit Algorithm](https://gitlab.com/0353F40E/ebaa) Owner</cite>

> This proposal will unlock the next generation of smart contracts on Bitcoin Cash. Incredibly it unlocks Ethereum-level complexity with no sacrifice at the alter of global state. This will be a huge competitive advantage for Bitcoin Cash in the coming years. The conservative approach to the new limits along with the thorough benchmarking, risk assessment and documentation leave no doubt that we can safely adopt this CHIP in May, and I support doing so.
>
> While the VM Limits proposal unlocks Ethereum-level complexity on Bitcoin Cash, our developer experience (DX) must aim for simplicity. That includes straightforward manipulation of numbers, regardless of their size and without a need for libraries or extra operations. This BigInt proposal will deliver a simple and powerful DX with no compromise on security or performance and I therefore support its adoption."
>
> —<cite>Richard Brady, [Coinbooth](https://coinbooth.io)</cite>

> As a BCH builder and CashScript developer, I wholeheartedly support the VM limits and big integer upgrades proposed by Jason Dreyzehner. These changes are crucial for expanding Bitcoin Cash's smart contract capabilities and improving overall network efficiency.
>
> The density-based limits and removal of arbitrary number size restrictions will enable more complex on-chain applications, benefiting projects like [TokenStork.com](https://tokenstork.com/) and opening new possibilities for CashScript development. The rigorous testing and community-driven approach to these upgrades instill confidence in their implementation.
>
> I believe these enhancements will significantly contribute to Bitcoin Cash's growth as a versatile and powerful Web3 network, aligning well with the goals of increased Bitcoin adoption and usability that drive projects like [BitcoinCashSite.com](https://www.bitcoincashsite.com/).
>
> —<cite>George Donnelly, [Panmoni](https://www.panmoni.com/), [TokenStork](https://tokenstork.com/), [BCH Works](https://www.bitcoincashsite.com/), [BCH Latam](https://www.instagram.com/bchlatam/)</cite>

> As an ecosystem supporter and Fulcrum server operator, I wholly endorse CHIP 2021-05 Targeted Virtual Machine Limits and CHIP 2024-07 BigInt. I have no concern with my node's ability to keep operating effectively, and expect that projects I contribute to will be able to take full advantage of these improvements.
>
> Though a long period of discussion, it appears all major concerns I've seen have been addressed. Frankly, from all I've read, these limits are rather conservative!
>
> The massive work put in by Jason Dreyzehner, Calin Culianu, and bitcoincashautist is inspiring. The attention to detail, time put into addressing concerns, running massive testing suites, etc. I hope to see both CHIPs locked in November, and activated in 2025!
>
> —<cite>Alex ([@minisatoshi](https://x.com/_minisatoshi/status/1842239258985185735)), [minisatoshi.cash](https://minisatoshi.cash), Fulcrum server operator</cite>

> The VM Limits & BigInt CHIPs expand the power of Bitcoin script within BCH with minimal, if any downside. It's exciting to think about how it could potentially unlock new DeFi applications, and I appreciate the thorough risk assessment documentation that went into the CHIPs. Great work!
>
> —<cite>Jonald, [Electron Cash](https://electroncash.org/) developer</cite>

> Super excited by the new upgrades. VM Limits improvements and Big Integers are going to change the game completely. I have so many ideas for things we can do with these once live. Plus the prep work and test harnesses are top notch. Amazing what happens when devs work together.
>
> —<cite>Josh Ellithorpe ([@zquestz](https://x.com/zquestz/status/1838736315921371555)), [bitcoincash.org](https://bitcoincash.org/), [BCHN](https://bitcoincashnode.org/) contributor, [Electron Cash](https://electroncash.org/) contributor, Fulcrum server operator</cite>

> I express my sincere support towards implementation and activation of improved VM limits and bigint support in BCH.
>
> —<cite>[mainnet_pat](https://github.com/mainnet-pat), [mainnet.cash](https://mainnet.cash/), [TapSwap](https://tapswap.cash)</cite>

> Through divine intuition, I Luke Pryor The High Rabbi Of Bcash ($bch) support these two chips for the 2025 upgrade Cycle.
>
> —<cite>Luke Pryor ([@thelukepryor](https://x.com/thelukepryor/status/1842241008844902550)), [Life Labs HTMA](https://lifelabshtma.com/)</cite>

> On behalf of Bitcoin Out Loud, I’m endorsing CHIP-2021-05 VM Limits and CHIP-2024-07 BigInt for 2025 activation on the Bitcoin Cash network.
>
> From my point of view as a mildly technical enthusiast, each is a big step towards Bitcoin Cash taking the lead in decentralized finance.
>
> —<cite>[BitcoinOutLoud](https://www.youtube.com/@BitcoinOutLoud) ([@BitcoinOutLoud](https://x.com/BitcoinOutLoud/status/1842356896864362892))</cite>

> Based on the significant benefits these two CHIPs offer the Bitcoin Cash network and decentralised p2p cash, both of these CHIPs have my full endorsement for implementation for the May 2025 BCH upgrade.
>
> —<cite>FiendishCrypto ([@FiendishCrypto](https://x.com/FiendishCrypto/status/1842210048128303221))</cite>

> As a software developer in the #BitcoinCash ecosystem I have reviewed and endorse both VM Limits and the BigInt changes for activation in May 2025. All concerns I have raised have been adequately addressed and I believe the value provided clearly outweigh the cost and risks.
>
> —<cite>Jonathan Silverblood ([@monsterbitar](https://x.com/monsterbitar/status/1842491069562282121)), [BCH BULL](https://bchbull.com/) co-founder, [General Protocols](https://generalprotocols.com/) co-founder</cite>

> I support the VM Limit and BigInt CHIPs for Novemeber lock-in. MOAR scripting capability!
>
> —<cite>Sayoshi Nakamario ([@SayoshiHelpMe](https://x.com/SayoshiHelpMe/status/1842659595111850164)), [helpme.cash](https://helpme.cash), [badgers.cash](https://badgers.cash), [fundme.cash](https://fundme.cash), [CasualBCH Podcast](casualbch.cash) </cite>

> I formally support activation of the following two CHIPs for the Bitcoin Cash (#BCH💚) 2025 upgrade cycle:
>
> - CHIP-2021-05 VM Limits: Targeted Virtual Machine Limits
> - CHIP-2024-07 BigInt: High-Precision Arithmetic for Bitcoin Cash
>
> These two CHIPs dramatically increase the power of the BCH Script engine, while also making it significantly more efficient.
>
> The VM Limits CHIP implements a new script budgeting system that maximizes the resources available to a script without introducing any new worst-case scenarios or DoS vectors.
>
> This allows contract developers to reduce the complexity of their scripts while also affording them increased power and flexibility. Despite the increased flexibility, these scripts do not negatively affect the BCH economic model, as the budgeting system is measured against a standard P2PKH transaction as a baseline for worst-case validation times.
>
> The companionate BigInt CHIP enables the script engine to operate on integers of any magnitude (hundreds or thousands of digits). This allows contract and wallet developers access to bleeding-edge crypto technology such as quantum-resistant wallets, zero-knowledge-proof verification, and on-chain encryption.
>
> The benchmarking suite that was developed to R&D this CHIP is also quite impressive: it thoroughly tests every possible combination of arithmetic opcodes in order to prove that worst-case transaction validation times are not negatively impacted by the changes proposed in this CHIP.
>
> Overall, this CHIP (which started research in 2021, with the BigInt CHIP ending up as a "freebie" thanks to the new budgeting system) has a very clear analysis of benefits, risks, and implementation costs. On behalf of myself, [Selene Wallet](https://selene.cash/), [bch.ninja](https://bch.ninja/), and [XULU.TECH LLC](https://xulu.tech/), I couldn't be more excited to endorse activation of these two CHIPs for the November lock-in.
>
> —<cite>[Kallisti.cash](https://kallisti.cash/) ([@kzkallisti](https://x.com/kzkallisti)), [Selene Wallet](https://selene.cash/), [bch.ninja](https://bch.ninja/), [XULU.TECH LLC](https://xulu.tech/)</cite>

> I support BigInt and VM Limits CHIPs. It had me on stronger escrow contracts and post-quantum cryptography. BCH is making it nearly impossible for other coins to compete.
>
> —<cite>Bruno ([@brunopbch](https://x.com/Brunopbch/status/1842234652930773377)), translation (Brazilian Portuguese), merchant onboarding</cite>

### Disapprove

The following statements have been submitted from individuals and organizations that disapprove of this CHIP.

_(None)_

### Neutral

The following statements have been submitted from individuals and organizations that abstained from approving this CHIP.

_(None)_
