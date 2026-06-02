---
title: Fourier
hide:
  - navigation
  - toc
---

# Fourier

A Rust-flavored DSL for WaveLedger smart contracts. Source files end in
`.fou`; the compiler (`fourier/compiler.py`) lexes, parses, and emits
stack-VM bytecode for the [WaveLedger VM](vm.md).

Contracts have native access to the chain's post-quantum primitives —
**ML-DSA-87** (FIPS 204) signature verification, **SHA3** (FIPS 202)
hashing, and **SLH-DSA-SHA2-128s** (FIPS 205) as an alternate signature
scheme — through precompiles at reserved low addresses. No
bytecode-level implementation of post-quantum cryptography is required
from contract authors.

New signature schemes are added through an on-chain registry:
`verify_sig(scheme_id, pk, msg, sig)` dispatches by identifier, so a
contract can opt into any registered scheme without recompilation. New
NIST-standardized primitives integrate as new precompile addresses and
new registry entries; the Fourier syntax and the VM ABI do not change.
See [Crypto agility](https://docs.fermi.world/concepts/agility/) on the
WaveLedger docs.

The official chain SDK
([`waveledger-sdk`](https://docs.fermi.world/sdk/) on
[PyPI](https://pypi.org/project/waveledger-sdk/) +
[npm](https://www.npmjs.com/package/waveledger-sdk)) compiles, deploys,
and calls Fourier contracts programmatically through the
[playground API](https://docs.fermi.world/api/playground/).

The design is deliberately compact. One parser pass and one codegen
pass. Every storage slot is explicit. Every call semantic is explicit.
Selectors are one byte. Inheritance and floating-point arithmetic are
not part of the language; fixed-point math is provided through Q64.64
builtins, and composition is provided through `library` blocks and
DELEGATECALL.

[Quick reference :material-arrow-right:](quick-reference.md){ .md-button .md-button--primary }
[Language reference :material-arrow-right:](language/index.md){ .md-button }

---

## Documentation map

<div class="grid cards" markdown>

-   :material-book-open-page-variant:{ .lg .middle } **Language**

    ---

    Every keyword, statement, and expression form, grounded in
    `fourier/parser.py` and `fourier/codegen.py`.

    [:octicons-arrow-right-24: Language reference](language/index.md)

-   :material-library:{ .lg .middle } **Standard library**

    ---

    Six contracts: Ownable, Pausable, ReentrancyGuard, PQ Multisig,
    Timelock, CryptoRegistry. Plus `safe_*` arithmetic builtins.

    [:octicons-arrow-right-24: Standard library](stdlib/index.md)

-   :material-code-braces:{ .lg .middle } **ABI and calldata**

    ---

    A 1-byte selector followed by 32-byte words. No Solidity
    function-ID hash. Variable-length arguments are not supported in
    the calldata ABI; `bytes` is supported as a return type.

    [:octicons-arrow-right-24: ABI specification](abi/index.md)

-   :material-cog:{ .lg .middle } **Compiler**

    ---

    `compile_source(text) -> bytes`. The compiler is invoked
    programmatically from Python or through the WaveLedger playground
    API.

    [:octicons-arrow-right-24: Compiler reference](compiler/index.md)

-   :material-file-document:{ .lg .middle } **Examples**

    ---

    Counter, ERC-20-style token, guestbook, and threshold-signature
    contracts. Annotated source for each.

    [:octicons-arrow-right-24: Examples](examples/index.md)

-   :material-chip:{ .lg .middle } **VM**

    ---

    The opcode catalogue, gas costs, and precompile addresses. The
    VM implementation lives in `vm/`; the reference mirrors it.

    [:octicons-arrow-right-24: VM reference](vm.md)

</div>

---

## Specification at a glance

| Property | Value |
|---|---|
| Source extension | `.fou` |
| Compiler entry | `fourier.compile_source(source: str) -> bytes` |
| Target | WaveLedger stack VM, 256-bit words |
| Selector width | 1 byte; `0x01` for the first `pub fn`, incrementing |
| Public-function cap | 255 per contract (one-byte selector space) |
| Calldata layout | `[selector:1][arg0:32][arg1:32]...` |
| Storage slot width | 256 bits |
| Mapping key derivation | `SHA3-256(key_word \|\| slot_word)` |
| Initialization guard slot | `2**256 - 1` (reserved) |
| Local memory base | `0x80` |
| Return memory slot | `0x40..0x60` |
| Reserved keywords | `contract storage event fn pub let if else while return emit map array struct true false uint address bool bytes library` |
| Builtins | `caller callvalue origin block_height timestamp balance sha3 require safe_add safe_sub safe_mul safe_div from_int to_int fmul fdiv safe_fmul safe_fdiv pack_sel call_b delegatecall_b staticcall_b lib_call verify_sig call len push pop` |
| Precompile addresses | `0x01` SHA3-512, `0x02` ML-DSA-87 verify, `0x03` SLH-DSA-SHA2-128s verify |

## Language scope

Fourier targets contract authoring on a post-quantum chain. The
following are part of the language:

- **String literals** (`"hello"`) — right-padded `uint`, max 32 bytes.
  See [types](language/types.md#string-literals).
- **`init` constructor parameters** — unpacked from the deploy
  transaction's `init_calldata`. See
  [contracts](language/contracts.md#init).
- **`library Foo { ... }` blocks** plus `Foo::method(addr, gas, args...)`
  call syntax — declarative library interfaces compiled to
  DELEGATECALL. See
  [cross-contract](language/cross-contract.md#library-blocks).
- **`lib_call`** — the lower-level DELEGATECALL builtin that `library`
  blocks desugar to. See
  [cross-contract](language/cross-contract.md#lib_call-library-delegatecall-sugar).
- **Multi-contract source files** — a `.fou` may declare multiple
  `contract` blocks. See
  [compiler / Python API](compiler/python.md).
- **Fixed-point arithmetic (Q64.64)** — `from_int`, `to_int`, `fmul`,
  `fdiv`, and the `safe_fmul` / `safe_fdiv` variants that revert on
  overflow. See [expressions](language/expressions.md#fixed-point-math-q6464).
- **Peephole optimizer** — folds `PUSH/POP` pairs and identity
  arithmetic, preserving control-flow structure. Enabled by default.

The following are not part of the language by design:

- Inheritance, traits, interfaces. Composition is provided through
  `library` blocks and address-targeted calls. See
  [stdlib](stdlib/index.md) for patterns.
- Floating-point arithmetic. Use Q64.64 fixed-point builtins.
- Generic types.
