# Examples

Annotated references for the canonical Fourier source files shipped
in `fourier/examples/`.

- [**Counter**](counter.md) — minimal contract with init and a single
  mutating method. Source: `fourier/examples/counter.fou`.
- [**Token (ERC-20-flavored)**](token.md) — balances mapping,
  `transfer` with event emission. Source: `fourier/examples/token.fou`.
- [**Threshold (M-of-N)**](threshold.md) — pattern reference for an
  M-of-N approval contract. Built from the multisig stdlib primitives.
- [**Guestbook**](guestbook.md) — append-only message log using a
  storage array.

The first two correspond to working `.fou` files. The second two are
pattern references built from primitives in the language and stdlib;
v1 does not ship them as separate `.fou` files.

## Page structure

Each page:

1. Shows the full source.
2. Annotates each declaration and function.
3. Notes the resulting selector layout, storage slots, and any
   subtleties.
4. Lists variants.

For language-feature deep-dives, see the
[Language reference](../language/index.md). The examples here assume
familiarity with the [Quick reference](../quick-reference.md).
