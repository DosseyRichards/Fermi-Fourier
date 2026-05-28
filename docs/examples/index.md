# Examples

Walkthroughs of the canonical Fourier source files shipped in
`fourier/examples/`.

- [**Counter**](counter.md) — minimal contract with init and a single
  mutating method. Source: `fourier/examples/counter.fou`.
- [**Token (ERC-20-flavored)**](token.md) — balances mapping,
  `transfer` with event emission. Source: `fourier/examples/token.fou`.
- [**Threshold (M-of-N)**](threshold.md) — pattern sketch for an
  M-of-N approval contract. Built from the multisig stdlib primitives.
- [**Guestbook**](guestbook.md) — append-only message log using a
  storage array.

The first two are working `.fou` files; the second two are pattern
walkthroughs built from primitives that exist in the language and
stdlib (they don't ship as separate `.fou` files in v1).

## What you'll see

Each walkthrough:

1. Shows the full source.
2. Walks line-by-line through each declaration and function.
3. Notes the resulting selector layout, storage slots, and any
   subtleties.
4. Suggests next variants to try.

For language-feature deep-dives — what each construct means — go to the
[Language reference](../language/index.md). The examples here assume
you've at least skimmed the [Quick reference](../quick-reference.md).
