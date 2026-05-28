# Documentation Followups

Things noticed while writing that should be filled in.

## Fourier
- [x] Document storage layout for struct fields (which slots they occupy
  relative to the struct's base slot) — covered in language/storage.md
- [x] Document nested-map storage key derivation — confirmed iterative
  `slot_i = SHA3(k_i || slot_{i-1})`; documented in language/storage.md
- [x] Document tuple destructuring rules — covered in language/statements.md
- [x] Document the codegen-level `pack_sel` builtin for hand-rolled
  calldata — covered in language/expressions.md and language/cross-contract.md
- [ ] Add a "common mistakes" page (address vs uint, init at deploy,
  storage slot conflicts, etc.)

## Stdlib
- [x] Document each stdlib contract's storage slot conventions so
  derived contracts know which slots are reserved — done for all six
  shipped contracts
- [x] Add usage examples for each stdlib contract — done

## Reference
- [x] Spec out the full VM opcode list with gas costs in one canonical
  page — see vm.md
- [x] Document the precompile address space (PQC verify, SHA3-512, etc.)
  — see vm.md
- [ ] Document Bohms (the gas-unit subdivision of WAVE)

## Site infrastructure
- [ ] Wire a "View on GitHub" link per page that goes to the source
  Markdown (currently `edit_uri: ""`)
- [ ] Set up real Algolia DocSearch when traffic justifies it (lunr
  local search ships out of the box for now)
- [ ] Add changelog section tied to compiler version
- [ ] Add OpenGraph images per page (auto-generated would be ideal)

## Discrepancies surfaced while writing the v1 docs

### Grammar vs implementation gaps

- `fourier/GRAMMAR.md` lists `bytes` as a return-only type, but the
  parser and codegen accept `bytes` as a function parameter type and a
  local-variable type as well. The grammar doc should be updated to
  match — see the `pack_sel` / `verify_sig` usage pattern in
  `stdlib/reentrancy_guard.fou` and `stdlib/multisig.fou`.
- `GRAMMAR.md` says "DELEGATECALL / library imports" are NOT supported
  in v1. The codegen actually implements `delegatecall_b` and
  `staticcall_b` builtins (since the v1 spec was written). The grammar
  doc should be updated.
- `GRAMMAR.md` lists `sha3(bytes)` as taking `bytes`. The codegen's
  `sha3` builtin only takes a single 32-byte `uint` word — for
  variable-length hashing, you'd need to call the SHA3-512 precompile
  directly. Update the grammar doc.
- `GRAMMAR.md` lists "Reentrancy guards (you write them by hand)" as
  unsupported, but `fourier/stdlib/reentrancy_guard.fou` ships as part
  of the stdlib. The intent matches — the stdlib is the "by hand"
  pattern, just packaged.

### Codegen behavior worth documenting more loudly

- `&&` and `||` do NOT short-circuit — both operands are always
  evaluated. This is in language/expressions.md but deserves
  louder warning prose for users coming from Solidity / Rust.
- The compiler's address-vs-uint enforcement only fires when BOTH
  sides have a known static type (locals with declared types).
  Storage reads return type `_` and silently pass — partial type safety.
- Private (`fn`, not `pub fn`) functions other than `init` emit no
  bytecode in v1; the parser accepts them but they're effectively dead
  declarations.

### Compiler ergonomics

- No standalone CLI binary. Currently documented as a `python -c`
  one-liner; ship a real entrypoint.
- No way to inspect a compiled contract's selector table from the
  compiler output (no metadata blob). Add a `--abi` flag once a CLI
  exists.

### Stubbed pages

- `examples/threshold.md` is a pattern walkthrough rather than a real
  shipped `.fou` file. If WAVE adopts threshold patterns as canonical,
  add a real example.
- `examples/guestbook.md` is the same — pattern walkthrough, not
  shipped source.

### Stdlib

- The shipped `Multisig` contract has no `init` — it expects an
  external setup step. Either add an `init` that takes no params and
  uses constants, or document an admin-add pattern.
- The shipped `Timelock` only supports single-arg proposals (one
  `p_arg: map[uint, uint]`). Multi-arg upgrade proposals need a
  custom Timelock variant or a calldata hash commit/reveal scheme.
- `CryptoRegistry` updates via Timelock need a two-arg proposal
  (scheme_id, precompile_addr) — Timelock currently can't carry both.
  Document the packing workaround or extend Timelock.
