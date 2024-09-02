# Operation Cost by Operation

**This table is provided for clarity, but it is not necessary for correct implementation of the proposal.** The provided `Operation Cost` formulas are derived from the proposal's [technical specification](./readme.md). The [test vectors](./tests-and-benchmarks.md) are also designed to ensure that all implementations correctly account for the cost of each operation.

The full, **standard operation cost**<sup>1</sup> for each executed operation<sup>2</sup> is provided below, accounting for [Measurement of Stack-Pushed Bytes](./readme.md#measurement-of-stack-pushed-bytes), [Arithmetic Operation Cost](./readme.md#arithmetic-operation-cost), [Hash Digest Iteration Cost](./readme.md#hash-digest-iteration-cost), and [Signature Checking Operation Cost](./readme.md#signature-checking-operation-cost).

| Opcode(s)                  | Codepoint(s)   | Notes                                                                     | Operation Cost                                                      |
| -------------------------- | -------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `OP_0`                     | `0x00`         |                                                                           | `100`                                                               |
| `OP_PUSHBYTES_N` (1-75)    | `0x01`-`0x4b`  | `OP_PUSHBYTES_2` is `102`, etc.                                           | `100 + N`                                                           |
| `OP_PUSHDATA_N`            | `0x4c`-`0x4e`  | `OP_PUSHDATA_2` of `521` is `621`, etc.                                   | `100 + length`                                                      |
| `OP_1NEGATE`               | `0x4f`         |                                                                           | `100 + 1`                                                           |
| `OP_1`-`OP_16`             | `0x51`-`OP_16` |                                                                           | `100 + 1`                                                           |
| `OP_NOP`                   | `0x61`         |                                                                           | `100`                                                               |
| `OP_IF`                    | `0x63`         |                                                                           | `100`                                                               |
| `OP_NOTIF`                 | `0x64`         |                                                                           | `100`                                                               |
| `OP_ELSE`                  | `0x67`         |                                                                           | `100`                                                               |
| `OP_ENDIF`                 | `0x68`         |                                                                           | `100`                                                               |
| `OP_VERIFY`                | `0x69`         |                                                                           | `100`                                                               |
| `OP_RETURN`                | `0x6a`         |                                                                           | `100`                                                               |
| `OP_TOALTSTACK`            | `0x6b`         |                                                                           | `100`                                                               |
| `OP_FROMALTSTACK`          | `0x6c`         | Stack: ` `; alt: `a` → stack: `a`; alt: ` `                               | `100 + a.length`                                                    |
| `OP_2DROP`                 | `0x6d`         |                                                                           | `100`                                                               |
| `OP_2DUP`                  | `0x6e`         | `a b` → `a b a b`                                                         | `100 + a.length + b.length`                                         |
| `OP_3DUP`                  | `0x6f`         | `a b c` → `a b c a b c`                                                   | `100 + a.length + b.length + c.length`                              |
| `OP_2OVER`                 | `0x70`         | `a b c d` → `a b c d a b`                                                 | `100 + a.length + b.length`                                         |
| `OP_2ROT`                  | `0x71`         | `a b c d e f` → `c d e f a b`                                             | `100 + a.length + b.length`                                         |
| `OP_2SWAP`                 | `0x72`         |                                                                           | `100`                                                               |
| `OP_IFDUP`                 | `0x73`         | `a` → `a` (false) **OR** `a` → `a a` (duplicate)                          | `100` (for false) **OR** `100 + a.length` (for duplication)         |
| `OP_DEPTH`                 | `0x74`         | `a b c ...` → `a b c ... d`                                               | `100 + d.length`                                                    |
| `OP_DROP`                  | `0x75`         |                                                                           | `100`                                                               |
| `OP_DUP`                   | `0x76`         | `a` → `a a`                                                               | `100 + a.length`                                                    |
| `OP_NIP`                   | `0x77`         |                                                                           | `100`                                                               |
| `OP_OVER`                  | `0x78`         | `a b` → `a b a`                                                           | `100 + a.length`                                                    |
| `OP_PICK`                  | `0x79`         | E.g. `a b c d 3` → `a b c d a` (N=3)                                      | `100 + a.length`                                                    |
| `OP_ROLL`                  | `0x7a`         | E.g. `a b c d 3` → `b c d a` (N=3)                                        | `100 + a.length + N` (where N is depth)                             |
| `OP_ROT`                   | `0x7b`         |                                                                           | `100`                                                               |
| `OP_SWAP`                  | `0x7c`         |                                                                           | `100`                                                               |
| `OP_TUCK`                  | `0x7d`         | `a b` → `b a b`                                                           | `100 + b.length`                                                    |
| `OP_CAT`                   | `0x7e`         | `a b` → `ab`                                                              | `100 + a.length + b.length`                                         |
| `OP_SPLIT`                 | `0x7f`         | `ab n` → `a b`                                                            | `100 + a.length + b.length`                                         |
| `OP_NUM2BIN`               | `0x80`         | `a n` → `b`                                                               | `100 + b.length`                                                    |
| `OP_BIN2NUM`               | `0x81`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_SIZE`                  | `0x82`         | `a` → `a b`                                                               | `100 + b.length`                                                    |
| `OP_AND`                   | `0x84`         | `a b` → `c`                                                               | `100 + c.length`                                                    |
| `OP_OR`                    | `0x85`         | `a b` → `c`                                                               | `100 + c.length`                                                    |
| `OP_XOR`                   | `0x86`         | `a b` → `c`                                                               | `100 + c.length`                                                    |
| `OP_EQUAL`                 | `0x87`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`) **OR** `100 + 1` (for `1`)                          |
| `OP_EQUALVERIFY`           | `0x88`         |                                                                           | `100 + 1`                                                           |
| `OP_1ADD`                  | `0x8b`         | `a` → `b`                                                                 | `100 + (2 * b.length)`                                              |
| `OP_1SUB`                  | `0x8c`         | `a` → `b`                                                                 | `100 + (2 * b.length)`                                              |
| `OP_NEGATE`                | `0x8f`         | `a` → `b`                                                                 | `100 + (2 * b.length)`                                              |
| `OP_ABS`                   | `0x90`         | `a` → `b`                                                                 | `100 + (2 * b.length)`                                              |
| `OP_NOT`                   | `0x91`         | `a` → `0` **OR** `1`                                                      | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_0NOTEQUAL`             | `0x92`         | `a` → `0` **OR** `1`                                                      | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_ADD`                   | `0x93`         | `a b` → `c`                                                               | `100 + (2 * c.length)`                                              |
| `OP_SUB`                   | `0x94`         | `a b` → `c`                                                               | `100 + (2 * c.length)`                                              |
| `OP_MUL`                   | `0x95`         | `a b` → `c`                                                               | `100 + (2 * c.length) + (a.length * b.length)`                      |
| `OP_DIV`                   | `0x96`         | `a b` → `c`                                                               | `100 + (2 * c.length) + (a.length * b.length)`                      |
| `OP_MOD`                   | `0x97`         | `a b` → `c`                                                               | `100 + (2 * c.length) + (a.length * b.length)`                      |
| `OP_BOOLAND`               | `0x9a`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_BOOLOR`                | `0x9b`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_NUMEQUAL`              | `0x9c`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_NUMEQUALVERIFY`        | `0x9d`         | `a b` → (none)                                                            | `100 + 1`                                                           |
| `OP_NUMNOTEQUAL`           | `0x9e`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_LESSTHAN`              | `0x9f`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_GREATERTHAN`           | `0xa0`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_LESSTHANOREQUAL`       | `0xa1`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_GREATERTHANOREQUAL`    | `0xa2`         | `a b` → `0` **OR** `1`                                                    | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_MIN`                   | `0xa3`         | `a b` → `c` (where `c` is minimum of `a` and `b`)                         | `100 + (2 * c.length)`                                              |
| `OP_MAX`                   | `0xa4`         | `a b` → `c` (where `c` is maximum of `a` and `b`)                         | `100 + (2 * c.length)`                                              |
| `OP_WITHIN`                | `0xa5`         | `a b c` → `0` **OR** `1`                                                  | `100` (for `0`), `100 + 1` for `1`                                  |
| `OP_RIPEMD160`             | `0xa6`         | `a` → `b` (Showing standard cost<sup>1</sup>)                             | `100 + (iterations * 192) + b.length`                               |
| `OP_SHA1`                  | `0xa7`         | `a` → `b` (Showing standard cost<sup>1</sup>)                             | `100 + (iterations * 192) + b.length`                               |
| `OP_SHA256`                | `0xa8`         | `a` → `b` (Showing standard cost<sup>1</sup>)                             | `100 + (iterations * 192) + b.length`                               |
| `OP_HASH160`               | `0xa9`         | `a` → `b` (Showing standard cost<sup>1</sup>)                             | `100 + ((iterations + 1) * 192) + b.length`                         |
| `OP_HASH256`               | `0xaa`         | `a` → `b` (Showing standard cost<sup>1</sup>)                             | `100 + ((iterations + 1) * 192) + b.length`                         |
| `OP_CODESEPARATOR`         | `0xab`         |                                                                           | `100`                                                               |
| `OP_CHECKSIG`              | `0xac`         | Null signature checks are `100` (Showing standard cost<sup>1</sup>)       | `100` (for `0`) **OR** `100 + 26000 + (iterations * 192) + 1`       |
| `OP_CHECKSIGVERIFY`        | `0xad`         | (Showing standard cost<sup>1</sup>)                                       | `100 + 26000 + (iterations * 192) + 1`                              |
| `OP_CHECKMULTISIG`         | `0xae`         | M-of-N; K=M for Schnorr, N for legacy (Showing standard cost<sup>1</sup>) | `100` (for `0`) **OR** `100 + (26000 * K) + (iterations * 192) + 1` |
| `OP_CHECKMULTISIGVERIFY`   | `0xaf`         | M-of-N; K=M for Schnorr, N for legacy (Showing standard cost<sup>1</sup>) | `100 + (26000 * K) + (iterations * 192) + 1`                        |
| `OP_NOP1`                  | `0xb0`         |                                                                           | `100`                                                               |
| `OP_CHECKLOCKTIMEVERIFY`   | `0xb1`         |                                                                           | `100`                                                               |
| `OP_CHECKSEQUENCEVERIFY`   | `0xb2`         |                                                                           | `100`                                                               |
| `OP_NOP4`                  | `0xb3`         |                                                                           | `100`                                                               |
| `OP_NOP5`                  | `0xb4`         |                                                                           | `100`                                                               |
| `OP_NOP6`                  | `0xb5`         |                                                                           | `100`                                                               |
| `OP_NOP7`                  | `0xb6`         |                                                                           | `100`                                                               |
| `OP_NOP8`                  | `0xb7`         |                                                                           | `100`                                                               |
| `OP_NOP9`                  | `0xb8`         |                                                                           | `100`                                                               |
| `OP_NOP10`                 | `0xb9`         |                                                                           | `100`                                                               |
| `OP_CHECKDATASIG`          | `0xba`         | Null signature check is `100` (Showing standard cost<sup>1</sup>)         | `100` (for `0`) **OR** `100 + 26000 + (iterations * 192) + 1`       |
| `OP_CHECKDATASIGVERIFY`    | `0xbb`         | (Showing standard cost<sup>1</sup>)                                       | `100 + 26000 + (iterations * 192) + 1`                              |
| `OP_REVERSEBYTES`          | `0xbc`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_INPUTINDEX`            | `0xc0`         | ` ` → `a`                                                                 | `100 + a.length`                                                    |
| `OP_ACTIVEBYTECODE`        | `0xc1`         | ` ` → `a`                                                                 | `100 + a.length`                                                    |
| `OP_TXVERSION`             | `0xc2`         | ` ` → `a`                                                                 | `100 + a.length`                                                    |
| `OP_TXINPUTCOUNT`          | `0xc3`         | ` ` → `a`                                                                 | `100 + a.length`                                                    |
| `OP_TXOUTPUTCOUNT`         | `0xc4`         | ` ` → `a`                                                                 | `100 + a.length`                                                    |
| `OP_TXLOCKTIME`            | `0xc5`         | ` ` → `a`                                                                 | `100 + a.length`                                                    |
| `OP_UTXOVALUE`             | `0xc6`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_UTXOBYTECODE`          | `0xc7`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_OUTPOINTTXHASH`        | `0xc8`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_OUTPOINTINDEX`         | `0xc9`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_INPUTBYTECODE`         | `0xca`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_INPUTSEQUENCENUMBER`   | `0xcb`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_OUTPUTVALUE`           | `0xcc`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_OUTPUTBYTECODE`        | `0xcd`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_UTXOTOKENCATEGORY`     | `0xce`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_UTXOTOKENCOMMITMENT`   | `0xcf`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_UTXOTOKENAMOUNT`       | `0xd0`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_OUTPUTTOKENCATEGORY`   | `0xd1`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_OUTPUTTOKENCOMMITMENT` | `0xd2`         | `a` → `b`                                                                 | `100 + b.length`                                                    |
| `OP_OUTPUTTOKENAMOUNT`     | `0xd3`         | `a` → `b`                                                                 | `100 + b.length`                                                    |

#### Notes

1. Standard and consensus ("nonstandard") operation costs differ only in the cost of hash digest iterations: `192` per iteration in standard validation, `64` per iteration in consensus validation.
2. All unexecuted operations cost the [Base Instruction Cost](rationale.md#selection-of-base-instruction-cost) of `100` per evaluation.
