# Recipes

Problem-first walkthroughs that show how to use Fourier to build
**quantum-proof workflows**. Not toy contracts, but the kind of
multi-party process a real application needs: a vote, a shipment, a
file's history.

Each recipe opens with the plain-language problem and why it has to
survive the arrival of quantum computers, then gives you a **complete,
tested Fourier contract** and shows how your application drives it.

- [**Quantum-proof voting**](voting.md): run an election or a
  governance vote where every ballot is authenticated, nobody can vote
  twice, and the tally can never be quietly rewritten.
- [**Logistics & chain of custody**](logistics.md): track an item as it
  changes hands so the record of *who held it, and when* can't be
  forged or back-dated, even decades later.
- [**Tamper-proof file registry**](file-registry.md): anchor a file's
  fingerprint and full version history on-chain so anyone can detect if
  a copy has been altered.

## The idea underneath all three

A blockchain isn't just a place to move money. It's a **shared,
tamper-proof rulebook** that parties who don't fully trust each other
can all rely on: it runs agreed-upon logic, records what happened
permanently, and checks the signature on every action automatically.

The interesting applications are workflows where you need proof of
*who did what, in what order, and that nobody rewrote history.* Every
one of those proofs is a digital signature, and today's signatures
(RSA, ECDSA) rest on math a large quantum computer can unravel. This
isn't fringe speculation. It's why NIST has standardized post-quantum
replacements and why governments and industry are already migrating.

Two things make the threat serious rather than distant. The exposure is
**present-tense**: the public keys an attacker needs are already visible
on every public ledger, ready to be harvested now and forged against
once the hardware matures. The damage is **retroactive and permanent**:
a record secured with breakable signatures can be rewritten years after
it was written, with no way to re-secure it. For a ledger meant to be
the source of truth, that is the whole game.

## Why Fourier contracts are quantum-proof by default

You do **not** have to write any cryptography to get this property.

- **Every call to a Fourier contract is a WaveLedger transaction, and
  every transaction is signed with a post-quantum signature scheme**
  (ML-DSA-87, NIST FIPS 204, today). Inside your contract, `caller()` is
  a **post-quantum-authenticated identity**. A check like
  `require(caller() == admin)` or `require(is_voter[caller()] == 1)` is
  already quantum-safe.
- **The whole stack is post-quantum, not only the signatures.**
  WaveLedger uses post-quantum primitives at every layer: **ML-DSA-87**
  (NIST FIPS 204) for signatures, **ML-KEM-1024** (FIPS 203) for key
  encapsulation (an account's very address derives from its ML-KEM key),
  and **SHA3-512** for block hashes, the Merkle tree, transaction IDs,
  and address derivation. Nothing in the chain relies on RSA or elliptic
  curves. These recipes foreground *signatures* only because authenticity,
  *who* acted, is what a vote, a custody trail, or a file history hinges
  on. That authenticity rests on a foundation that is post-quantum all
  the way down.
- **The scheme is not hard-wired. WaveLedger is crypto-agile.**
  ML-DSA-87 is the *current* default, not a permanent commitment. The
  chain keeps a governed on-chain registry of signature schemes (the
  [CryptoRegistry](../stdlib/crypto-registry.md), designed to sit behind
  a [Timelock](../stdlib/timelock.md)), and the VM already ships a second
  post-quantum scheme, SLH-DSA-SHA2-128s (NIST FIPS 205), alongside
  ML-DSA-87. New standards can be adopted, and weakened ones retired,
  **without a hard fork**. Contracts you write today keep working as the
  cryptography underneath them evolves, so build for "post-quantum," not
  for one specific algorithm.

!!! note "What this protects, and what it doesn't"
    These contracts protect **authenticity and integrity**: *who* acted,
    in *what order*, and that the record wasn't altered. They do **not**
    provide secrecy. Everything on-chain is public, so keep private data
    off-chain and anchor only a hash (see the
    [file registry recipe](file-registry.md)).

    Verifying signatures from keys that are **not** WaveLedger accounts
    (an external issuer, an offline device) uses the
    [`verify_sig`](../quick-reference.md#crypto-scheme-ids) precompile.
    That is a more advanced path with v1 constraints on passing large
    key/signature blobs. The recipes here use the built-in `caller()`
    authentication, which needs none of that.
