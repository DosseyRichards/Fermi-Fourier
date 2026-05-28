---
title: Fourier
hide:
  - navigation
  - toc
---

# Fourier

A Rust-flavored DSL for writing WaveLedger smart contracts. Source files
end in `.fou`; the compiler (`fourier/compiler.py`) lexes, parses, and
emits stack-VM bytecode that runs on the
[WaveLedger VM](vm.md).

One source file, one contract. One parser pass, one codegen pass, no
optimizer. Every storage slot is explicit. Every call semantics is
explicit. Selectors are one byte. Floats, inheritance, generics
are intentionally absent in v1.

[Quick reference :material-arrow-right:](quick-reference.md){ .md-button .md-button--primary }
[Browse the language :material-arrow-right:](language/index.md){ .md-button }

---

## What's in these docs

<div class="grid cards" markdown>

-   :material-book-open-page-variant:{ .lg .middle } **Language**

    ---

    Every keyword, every statement, every expression form. Grounded in
    `fourier/parser.py` and `fourier/codegen.py`.

    [:octicons-arrow-right-24: Read the language reference](language/index.md)

-   :material-library:{ .lg .middle } **Standard library**

    ---

    Six shipped contracts: ownable, pausable, reentrancy guard, multisig
    (PQC), timelock, crypto registry. Plus `safe_*` arithmetic builtins.

    [:octicons-arrow-right-24: Browse the stdlib](stdlib/index.md)

-   :material-code-braces:{ .lg .middle } **ABI and calldata**

    ---

    Calldata is a 1-byte selector followed by 32-byte words. No Solidity
    function-ID hash. No variable-length args in v1 (except `bytes`
    returns).

    [:octicons-arrow-right-24: ABI spec](abi/index.md)

-   :material-cog:{ .lg .middle } **Compiler**

    ---

    `compile_source(text) -> bytes`. That's the API. No CLI in v1 — use
    the Python entry directly or wrap it yourself.

    [:octicons-arrow-right-24: Compiler reference](compiler/index.md)

-   :material-file-document:{ .lg .middle } **Examples**

    ---

    Counter, ERC-20-style token, and walkthroughs of the two canonical
    `.fou` files shipped in the repo.

    [:octicons-arrow-right-24: Read the examples](examples/index.md)

-   :material-chip:{ .lg .middle } **VM internals**

    ---

    Every opcode, every gas cost, every precompile address. The VM is in
    `vm/` — these docs mirror that source.

    [:octicons-arrow-right-24: VM reference](vm.md)

</div>

---

## Five-minute orientation

| Property | Value |
|---|---|
| Source extension | `.fou` |
| Compiler entry | `fourier.compile_source(source: str) -> bytes` |
| Target | WaveLedger stack VM (256-bit words) |
| Selector width | 1 byte (`0x01` for the first `pub fn`, incrementing) |
| Calldata layout | `[selector:1][arg0:32][arg1:32]...` |
| Storage slot width | 256 bits |
| Mapping key derivation | `SHA3-256(key_word || slot_word)` |
| Init guard slot | `2**256 - 1` (reserved; collides → compile error) |
| Local memory base | `0x80` |
| Return memory slot | `0x40..0x60` |
| Reserved keywords | `contract storage event fn pub let if else while return emit map array struct true false uint address bool bytes` |
| Builtins | `caller callvalue origin block_height timestamp balance sha3 safe_add safe_sub safe_mul safe_div pack_sel call_b delegatecall_b staticcall_b verify_sig call len push pop` |
| Precompile addresses | `0x01` SHA3-512, `0x02` ML-DSA-87 verify, `0x03` SLH-DSA-SHA2-128s verify |

## What Fourier is not (v1)

- No inheritance, traits, or interfaces. Copy declarations from
  [stdlib contracts](stdlib/index.md) into your contract, or call them
  via address.
- No IEEE 754 floating point. Use the Q64.64 fixed-point builtins
  (`from_int`, `to_int`, `fmul`, `fdiv`) — see
  [expressions / fixed-point math](language/expressions.md#fixed-point-math-q6464).

Supported as of v1:

- **String literals** (`"hello"`) — desugar to a right-padded `uint`.
  See [types](language/types.md#string-literals).
- **`init` constructor parameters** — unpacked from the deploy tx's
  `init_calldata`. See [contracts](language/contracts.md#init).
- **`library Foo { ... }` blocks** plus `Foo::method(addr, gas, args...)`
  call syntax — declarative library interfaces compiled to DELEGATECALL.
  See [cross-contract / library blocks](language/cross-contract.md#library-blocks).
- **`lib_call`** — lower-level DELEGATECALL builtin that the `library`
  syntax desugars to. Useful when you don't want to declare the
  interface up front. See
  [cross-contract / lib_call](language/cross-contract.md#lib_call-library-delegatecall-sugar).
- **Multi-contract source files** — a `.fou` may declare multiple
  `contract` blocks. Use `compile_source_all(src)` for all bytecodes or
  `compile_source(src, name="...")` for one. See
  [compiler / Python API](compiler/python.md).
- **Fixed-point math (Q64.64)** — `from_int`, `to_int`, `fmul`, `fdiv`
  builtins for scaled decimal arithmetic on plain `uint` values. See
  [expressions](language/expressions.md#fixed-point-math-q6464).
- **Peephole optimizer** — runs by default. Current folds eliminate
  trivial `PUSH/POP` and identity-arithmetic patterns. Run
  `python3 scripts/bench_fourier.py` to measure on your contracts.
