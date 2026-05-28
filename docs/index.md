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
explicit. Selectors are one byte. Strings, floats, inheritance, generics
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
- No string literals. Use `uint` hashes.
- No floating point.
- No multi-contract source files. One `contract` per `.fou`.
- No optimizer. Each statement compiles to a fixed bytecode template.
- No constructor parameters. `init()` takes no args, runs once, no
  return value (gated by the flag at slot `2**256 - 1`).
