# Language

The Fourier language reference. Each page below covers one construct,
grounded in `fourier/parser.py` and `fourier/codegen.py`.

Recommended reading order for new readers:

1. [**Contracts**](contracts.md) — the top-level `contract { ... }`
   block. What goes inside and the `init` function.
2. [**Types**](types.md) — `uint`, `address`, `bool`, `bytes`, plus
   `map[K, V]` and `array[T]` for storage.
3. [**Storage**](storage.md) — `storage NAME: TYPE @ SLOT;` declarations
   and how slots are derived for mappings, arrays, and struct fields.
4. [**Functions**](functions.md) — `fn` vs `pub fn`, selector
   assignment, parameters, return types.
5. [**Statements**](statements.md) — `let`, assignment, `if`, `while`,
   `return`, `require`, `emit`.
6. [**Expressions**](expressions.md) — operators, precedence, builtins.
7. [**Events**](events.md) — `event` declarations, `emit` semantics,
   topic vs data layout.
8. [**Errors and reverts**](errors.md) — how `require` revertss, how
   sub-call failures propagate, the optional `bytes` revert payload.
9. [**Cross-contract calls**](cross-contract.md) — `call_b`,
   `delegatecall_b`, `staticcall_b`, and the one-word `call`.
10. [**Address type**](address.md) — semantics, sentinels, comparison
    rules, builtins that return addresses.

The full grammar lives at
[`fourier/GRAMMAR.md`](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software/blob/main/fourier/GRAMMAR.md)
in the repo. This reference is its more detailed, code-grounded
companion.
