# Real estate tokenization

## The problem

A building is worth millions, takes months to sell, and its ownership
lives in a county deed office and a stack of paper. Tokenization turns
that single illiquid asset into fractional shares that many investors can
hold and transfer in seconds. That is the promise behind the tokenized
real estate market: fractional access, faster settlement, a global pool
of buyers.

Two things make it hard, and both are what this recipe handles. First, a
property share is a regulated security. You cannot let arbitrary
addresses hold or trade it. Only KYC-verified, eligible investors may,
and the issuer must be able to freeze transfers or revoke a holder when a
regulator requires it. Second, the arrangement is worthless unless
on-chain ownership is authoritative: the token has to point at the real
legal title, and the record of who owns what has to be impossible to
forge.

Ownership here is a signature. Property stays owned for decades, far
longer than the shelf life of today's signatures (RSA, ECDSA), which rest
on math a large quantum computer can break. The public keys needed to
forge them are already visible on-chain now, ready to be harvested. A
forged signature would let an attacker mint a fraudulent transfer and
move someone's shares, or forge a sale years after the fact, with the
fake indistinguishable from the real thing. For an asset meant to stay
owned across generations, that exposure is present the moment the shares
exist and lasts as long as the asset does. So every share transfer and every
compliance action here runs on WaveLedger's
[post-quantum stack](index.md#why-fourier-contracts-are-quantum-proof-by-default),
and the legal deed is anchored by its SHA3 fingerprint, keeping ownership
authoritative no matter when a quantum computer arrives, or whether one
already has.

## The contract

Source: `fourier/examples/real_estate.fou` (compiles and runs on the
WaveLedger VM).

```fourier
// RealEstateToken: compliance-gated fractional ownership of a property.
// Shares transfer only between KYC-verified investors, the legal title
// deed is anchored by its SHA3 fingerprint, and every ownership change is
// a post-quantum-signed transaction recorded permanently on-chain.
contract RealEstateToken {
    storage issuer: address @ 0;                  // sponsor who tokenized the property
    storage total_shares: uint @ 1;               // fixed supply = 100% of the property
    storage deed_hash: uint @ 2;                  // SHA3 fingerprint of the off-chain title deed
    storage property_ref: uint @ 3;               // off-chain registry / parcel identifier
    storage frozen: uint @ 4;                     // 1 = transfers halted (e.g. legal hold)
    storage shares: map[address, uint] @ 5;       // holder -> shares owned
    storage is_verified: map[address, uint] @ 6;  // holder -> 1 if KYC-verified

    event PropertyTokenized(issuer: address, total_shares: uint, deed_hash: uint);
    event InvestorVerified(investor: address);
    event InvestorRevoked(investor: address);
    event SharesTransferred(from: address, to: address, amount: uint);
    event DeedUpdated(deed_hash: uint);
    event FrozenSet(state: uint);

    // The deployer is the issuer and receives 100% of the shares to
    // allocate. `total` is the share count (e.g. 1000 = 0.1% granularity).
    fn init(total: uint, deed: uint, property: uint) {
        issuer = caller();
        total_shares = total;
        deed_hash = deed;
        property_ref = property;
        is_verified[caller()] = 1;
        shares[caller()] = total;
        emit PropertyTokenized(caller(), total, deed);
    }

    pub fn verify_investor(investor: address) {
        require(caller() == issuer);
        is_verified[investor] = 1;
        emit InvestorVerified(investor);
    }

    pub fn revoke_investor(investor: address) {
        require(caller() == issuer);
        is_verified[investor] = 0;
        emit InvestorRevoked(investor);
    }

    pub fn set_frozen(state: uint) {
        require(caller() == issuer);
        frozen = state;
        emit FrozenSet(state);
    }

    pub fn update_deed(new_deed: uint) {
        require(caller() == issuer);
        deed_hash = new_deed;
        emit DeedUpdated(new_deed);
    }

    pub fn transfer(to: address, amount: uint) -> uint {
        require(frozen == 0);
        require(is_verified[caller()] == 1);
        require(is_verified[to] == 1);
        let bal: uint = shares[caller()];
        require(bal >= amount);
        shares[caller()] = bal - amount;
        shares[to] = shares[to] + amount;
        emit SharesTransferred(caller(), to, amount);
        return 1;
    }

    pub fn shares_of(holder: address) -> uint {
        return shares[holder];
    }

    pub fn is_kyc(holder: address) -> uint {
        return is_verified[holder];
    }

    pub fn get_deed() -> uint {
        return deed_hash;
    }
}
```

## How it works

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `issuer` | `address` | Sponsor who tokenized the property; set in `init` |
| `1` | `total_shares` | `uint` | Fixed supply representing 100% of the property |
| `2` | `deed_hash` | `uint` | SHA3 fingerprint of the off-chain legal title |
| `3` | `property_ref` | `uint` | Off-chain registry or parcel identifier |
| `4` | `frozen` | `uint` | `1` halts all transfers (legal hold) |
| `5` | `shares` | `map[address, uint]` | holder → shares owned |
| `6` | `is_verified` | `map[address, uint]` | holder → `1` if KYC-verified |

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `verify_investor(address)` |
| `0x02` | `revoke_investor(address)` |
| `0x03` | `set_frozen(uint)` |
| `0x04` | `update_deed(uint)` |
| `0x05` | `transfer(address, uint) -> uint` |
| `0x06` | `shares_of(address) -> uint` |
| `0x07` | `is_kyc(address) -> uint` |
| `0x08` | `get_deed() -> uint` |

### Compliance is built into the transfer

A plain token lets anyone send to anyone. A property share cannot. The
transfer gate is three lines:

```fourier
require(frozen == 0);                  // no legal hold in effect
require(is_verified[caller()] == 1);   // sender is a KYC-verified investor
require(is_verified[to] == 1);         // recipient is a KYC-verified investor
```

Only addresses the issuer has verified can hold or receive shares, so
shares can never land with an unvetted party. Because `caller()` is
post-quantum-authenticated, the "is this really the issuer?" and "is this
really the share owner?" checks hold even against an adversary with a
quantum computer.

### Issuer controls for a regulated asset

Real securities need levers that ordinary tokens lack, and the issuer
holds them:

- `verify_investor` / `revoke_investor` manage the KYC allow-list.
  Revoking a holder blocks them from sending or receiving further shares.
- `set_frozen(1)` halts every transfer at once, for a court order,
  a dispute, or a corporate action. `set_frozen(0)` resumes trading.

### The token points at a real, unaltered deed

The legal title stays off-chain in the county record or a signed PDF.
`deed_hash` anchors that document's SHA3 fingerprint on-chain, so the
token provably references one specific deed. If the recorded title is
re-issued, `update_deed` (issuer only) rolls the fingerprint forward, and
the change is a permanent `DeedUpdated` event. Anyone can hash the deed
they were shown and confirm it matches what the token claims.

### Verified end-to-end

Deploying the contract and running a full issuance on the WaveLedger VM
confirms the compliance guards fire:

```text
holdings issuer/alice/bob     : 700 / 190 / 110
to non-KYC recipient blocked  : True
non-KYC sender blocked        : True
non-issuer verify blocked     : True
overspend blocked             : True
frozen halts transfer         : True
END-TO-END REAL ESTATE: PASS
```

## Driving it from your application

1. **Deploy** with its constructor arguments: total share count, the
   deed fingerprint (`sha3` of the title document), and a property
   reference. The deployer becomes the `issuer` and holds 100% of the
   shares.
2. **Onboard investors.** Run your KYC/AML off-chain, then call
   `verify_investor(address)` for each cleared investor.
3. **Allocate shares.** `transfer` shares from the issuer to investors
   during the primary sale. (Take payment on- or off-chain; see
   *Extending it*.)
4. **Let investors trade.** Verified holders `transfer` among themselves.
   Any transfer to or from an unverified address reverts automatically.
5. **Stay compliant.** `revoke_investor` removes a holder from the
   allow-list; `set_frozen(1)` pauses the whole book if a regulator or
   court requires it.

See the [ABI & calldata](../abi/index.md) reference for exact encoding
and the [Python](../compiler/python.md) compiler API to produce the
bytecode.

## Extending it

- **Rental income distribution.** Pay dividends pro-rata with the
  cumulative-per-share pattern: keep an `acc_income_per_share`
  accumulator that grows each time income is deposited, and a
  `reward_debt[holder]` that is settled on every balance change. Each
  holder then claims `shares[holder] * acc_income_per_share` minus their
  debt. This avoids iterating over holders (which no contract can do
  cheaply).
- **Forced transfer for recovery.** Add an issuer-only
  `force_transfer(from, to, amount)` for court-ordered reassignment or a
  holder who lost their keys, mirroring the controller operations in
  security-token standards like ERC-1400.
- **Lockups and caps.** Enforce Reg D holding periods with a
  `locked_until[holder]` timestamp, or per-investor ownership caps to
  respect concentration limits.
- **Multiple properties.** Key every map by a `property_id` to run a
  whole portfolio from one contract.
- **On-chain primary sale.** Let investors buy shares by sending WAVE
  with the call (`callvalue()`), transferring shares and forwarding
  proceeds to the issuer in the same transaction.
