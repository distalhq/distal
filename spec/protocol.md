# distal protocol, v1 (DRAFT)

Read `README.md` in this folder first. Byte layouts are in `apdu.md`. Status words are in `status-codes.json`.

## 1. Scope and fixed parameters

- Curve: secp256k1. Group order `n`. Generator `G`.
- Signatures: BIP340.
- **v1 supports exactly threshold `t = 2` and `n_cards = 4`.** A card rejects any other values with `UnsupportedParams`. `[design]`
- Card positions: each card has a **position** `p` in `0..3`. Positions 0 and 1 are the device cards. Positions 2 and 3 are the offline cards. `[design]`
- **FROST identifier of a card = `p + 1`.** ChillDKG gives participant `p` the share `f(p + 1)` `[verified: vss.py, secshare_for]`.
- One key slot per applet instance.
- Not in v1: anti-exfiltration, peer-checked PIN counters, HD derivation, any curve other than secp256k1, ChillDKG investigation and recovery.

Notation: `enc33(P)` is the 33-byte compressed SEC1 encoding. `enc65(P)` is the 65-byte uncompressed encoding. `x(P)` is the 32-byte big-endian X coordinate. `id32(i)` is the integer `i` as 32 bytes big-endian. `be4(i)` is `i` as 4 bytes big-endian. `||` is concatenation. Scalars are 32 bytes big-endian.

## 2. Hash functions

`TH(tag, m) = SHA256(SHA256(tag) || SHA256(tag) || m)`, the BIP340 tagged hash.

For signing, with `CTX` = the 28 ASCII bytes `FROST-secp256k1-SHA256-TR-v1`:

- `HF(tag, m)`: RFC 9380 `expand_message_xmd` with SHA-256, DST = `CTX || tag`, 48 output bytes, reduced mod `n`. With `D = DST || len(DST)` (one byte): `b0 = SHA256(64 zero bytes || m || 00 30 || 00 || D)`, `b1 = SHA256(b0 || 01 || D)`, `b2 = SHA256((b0 XOR b1) || 02 || D)`; the result is the first 48 bytes of `b1 || b2` as a big-endian integer mod `n`.
- `H1(m) = HF("rho", m)`. `H3(m) = HF("nonce", m)`.
- `H4(m) = SHA256(CTX || "msg" || m)`. `H5(m) = SHA256(CTX || "com" || m)`.
- `H2(m) = TH("BIP0340/challenge", m)` as a big-endian integer mod `n`. `SHA256("BIP0340/challenge")` = `7bb52d7a9fef58323eb1bf7a407db382d2f3f2d81bb1224f49fe518f6d48d37c`.

All of the above is `[verified]`: recomputed against the upstream vector by `docs/plans/reference/recompute_zf_tr_vector.py`.

ChillDKG uses its own tagged hashes with the prefix `BIP DKG/`. They are not restated here. Use the reference.

## 3. Card identity

- At `INIT` the card draws 32 random bytes `hsk` from its own random generator, repeating until `1 <= hsk <= n - 1`. This is the card's **host secret key** in ChillDKG's terms. It never leaves the card. It survives `WIPE`.
- `hpk = enc33(hsk·G)` is the card's **host public key** `[verified: hostpubkey_gen]`.
- The card must keep `hsk` as raw bytes, because ChillDKG feeds it to hash functions as well as using it for ECDH and for BIP340 signing `[verified: participant_step1 passes it as seed and deckey; certeq_participant_step signs with it]`.
- **Card fingerprint** = the first 10 bytes of `SHA256(hpk)`, shown as 20 hex characters in groups of 4. `[design]`

## 4. Lifecycle

States: `BLANK`, `READY`, `PAIRED`, `KEYED`, `LOCKED`.

| From | Command | Condition | To | Effect |
|---|---|---|---|---|
| BLANK | INIT | none | READY | Create `hsk`. Set the PIN. |
| READY | PAIR | PIN verified | PAIRED | Store the four host public keys and the card's position. |
| PAIRED | DKG_STEP FINALIZE | section 6 | KEYED | Store share, group key, public shares. |
| READY | RESHARE_STEP RECEIVE | section 8 | KEYED | Store share, group data and the peer list. |
| PAIRED, KEYED | WIPE | PIN verified | READY | Erase share, group key, public shares, peer list, position, nonce slot. Keep `hsk`, PIN, session counter. |
| any except BLANK | wrong PIN until no tries remain | none | LOCKED | Nothing else changes. |

Rules:

1. `LOCKED` is final for the applet instance. Only `STATUS` works. A locked card is treated as a dead card: it is replaced by share repair. There is no unblocking code in v1. `[design]`
2. PIN: 6 to 12 bytes. 5 tries. The try counter resets on a correct PIN. `[design]`
3. "PIN verified" lasts until the card is reset or the applet is deselected.
4. Every command that is not listed for the card's current state returns `WrongState` and changes nothing.
5. Every change to persistent secret data (share, peer list, nonce slot) is atomic: after a power cut the card holds either the old state or the new state, never a mix.
6. `STATUS` needs no PIN and returns no secret. It never returns a hash of the applet's own code.
7. `WIPE` keeps `hsk`, so a card's fingerprint never changes.

## 5. Pairing

Decision: fingerprint ceremony, plus a factory record, plus vendor attestation where offered. The cards implement only the first part. The factory record and its check live outside the cards (`ceremony.md`).

`PAIR` gives the card `t`, the count, and the ordered list of four host public keys.

The card checks:

- `t = 2` and count `= 4`, else `UnsupportedParams`;
- every key is a valid curve point, else `BadPoint`;
- no key appears twice, and its own `hpk` appears exactly once, else `BadData`.

The card's position `p` is the index of its own key in the list. The card stores the list and computes the **group fingerprint** `gf = params_hash(hostpubkeys, t)` exactly as the ChillDKG reference does: `TH("BIP DKG/params_hash", be4(t) || hpk_0 || hpk_1 || hpk_2 || hpk_3)` `[verified: params_hash]`. It returns `gf`.

The group fingerprint is shown to people as the first 10 bytes of `gf`, 20 hex characters in groups of 4. `[design]`

What pairing proves and does not prove is stated in `threat-model.md` section 4. In short: it lets people detect cards that disagree about who is in the group. It cannot tell a real chip from a software key.

## 6. Key generation

Key generation is ChillDKG at the pinned commit, with the cards as participants and the host as coordinator.

### 6.1 Mapping

| ChillDKG | distal |
|---|---|
| participant `i` | the card at position `i` |
| `hostseckey`, host public key | the card's `hsk`, `hpk` |
| `SessionParams(hostpubkeys, t)` | the list and `t` stored by `PAIR` |
| `participant_step1` | `DKG_STEP START` on the card |
| `participant_step2` | `DKG_STEP PEERS` on the card |
| `participant_finalize` | `DKG_STEP FINALIZE` on the card |
| `coordinator_step1`, `coordinator_finalize` | computed by the host |
| `random` and `aux_rand` arguments | 32 fresh bytes each from the card's random generator. If the 32 bytes are all zero, draw again. |
| `ParticipantMsg1`, `CoordinatorMsg1`, `ParticipantMsg2`, `CoordinatorMsg2` | sent unchanged as the command or response payload |

Message bytes are exactly what the reference's `to_bytes` produces. For `t = 2`, `n = 4` the lengths are: `ParticipantMsg1` 291, `CoordinatorMsg1` 681, `ParticipantMsg2` 64, `CoordinatorMsg2` 256 `[verified: computed from the len_bytes functions]`.

### 6.2 What the card must do

Everything `participant_step1`, `participant_step2` and `participant_finalize` do, including every check, in the same order. In particular `[verified: read from the reference]`:

- derive the session seed, the VSS coefficients, the proof of possession, the encryption nonce and the encrypted shares as `encpedpop.participant_step1` and `simplpedpop.participant_step1` do;
- in step 2: check its own public nonce and its own commitment in the coordinator message; verify the proof of possession of every other participant; decrypt its share; apply the Taproot tweak (`invalid_taproot_commit`); check its tweaked share against the tweaked public share; build `eq_input`; sign it (`certeq_participant_step`);
- in finalize: verify all four signatures in the certificate over `eq_input` (`certeq_verify`), its own included.

Requirements on primitives:

- BIP340 signing and verification with the tag prefix the reference passes (`schnorr_sign` and `schnorr_verify` in the vendored `secp256k1lab/bip340.py`).
- ECDH as `ecdh_libsecp256k1`: SHA-256 of the **compressed** shared point. So the card needs both coordinates of the shared point.
- Every received point must be checked to be on the curve and not the point at infinity where the reference rejects it. Every received scalar must be below `n` where the reference uses `from_bytes_checked`.

### 6.3 What v1 leaves out

- `participant_investigate`. If the share check fails, the card returns `DkgShareInvalid` and the whole setup is run again.
- `recover` and the recovery acknowledgment. A card that did not finalize is wiped and the setup is run again.

### 6.4 After a successful finalize

ChillDKG's output is `secshare`, `thresh_pk`, `pubshares`. They are already tweaked so that `thresh_pk` is safe to use as a Taproot output key `[verified: ChillDKG README, "Taproot-safe threshold public key"]`.

The card then **normalizes to even Y** `[design, required by the signing reference]`:

- if `thresh_pk` has odd Y: `s = n − secshare`, `P = −thresh_pk`, and every public share is negated;
- otherwise `s = secshare`, `P = thresh_pk`, public shares unchanged.

The card stores `s`, `P` and the four normalized public shares in one atomic step and becomes `KEYED`. All four cards reach the same `P`. The BIP340 public key of the group is `x(P)`.

Signing in section 7 applies **no further tweak**.

### 6.5 State and aborts

All intermediate DKG data lives in memory that is cleared on reset. So:

| Event | Result |
|---|---|
| `START` | Any earlier unfinished run is discarded. A new run begins with fresh randomness. |
| `PEERS` without a `START` in this power session, or `FINALIZE` without `PEERS` | `OutOfOrderStep` |
| Power loss or deselect before `FINALIZE` completes | The run is gone. The card is still `PAIRED`. |
| A check in step 2 or finalize fails | The status in `status-codes.json` for that check. The run is discarded. The card is still `PAIRED`. |
| Any `DKG_STEP` in `KEYED` | `WrongState`. A new key needs `WIPE` first. |

The host finishes setup only when all four cards report `KEYED` with the same `P` and the same `gf`. If any card differs, the host wipes all four and starts again.

## 7. Signing

Signing is ZF `frost-secp256k1-tr` 3.0.0 **without** the Taproot tweak (`round2::sign`, not `sign_with_tweak`), on the normalized share `s` and key `P`.

### 7.1 Card state for signing

- `ctr`: a persistent 32-bit session counter. It only increases.
- One persistent **nonce slot**: `EMPTY`, or `ARMED` with the session number and the two nonces.
- `signs`: a persistent 32-bit count of shares produced.

### 7.2 COMMIT (round 1)

Condition: `KEYED`, PIN verified.

1. Set the slot to `EMPTY`. This burns any earlier session.
2. `ctr = ctr + 1`. If `ctr` would overflow, return `Internal` and do nothing.
3. Draw `r1`, `r2`, 32 random bytes each, from the card's random generator.
4. `d = H3(r1 || s)`, `e = H3(r2 || s)`. `D = d·G`, `E = e·G`.
5. Store `d`, `e` and `ctr` in the slot and set it to `ARMED`, so that the slot is `ARMED` only if all of it was written.
6. Return `ctr`, `enc33(D)`, `enc33(E)`.

### 7.3 SIGN (round 2)

Condition: `KEYED`, PIN verified.

Input: session number `c`; the signer list, ascending by identifier, each entry `(j, D_j, E_j)`; the message `m`.

Checks, in this order. Each failure returns the named status and changes nothing unless stated:

1. Slot is `ARMED`, else `OutOfOrderStep`.
2. `c` equals the slot's session number, else `SessionSuperseded`.
3. Between 2 and 4 entries; identifiers in `1..4`, strictly ascending; the card's own identifier is present; else `BadData`.
4. Every `D_j`, `E_j` is a valid curve point, else `BadPoint`.
5. The card's own entry equals the `D`, `E` of its slot, else `CommitmentMismatch`.
6. Message length is between 1 and 64 bytes, else `BadData`. `[design]`

Then:

7. **Burn first.** Set the slot to `EMPTY` and `signs = signs + 1`, atomically, keeping `d` and `e` only in memory that is cleared on reset.
8. `com` = for each entry in order: `id32(j) || enc33(D_j) || enc33(E_j)`.
9. `prefix = enc33(P) || H4(m) || H5(com)`.
10. `rho_j = H1(prefix || id32(j))` for each `j`.
11. `R = sum over j of (D_j + rho_j·E_j)`.
12. `c2 = H2(x(R) || x(P) || m)`.
13. `lambda = product over j ≠ i of j / (j − i) mod n`, where `i` is the card's identifier.
14. `k = d + e·rho_i mod n`. If `R` has odd Y, `k = n − k`.
15. `z = k + lambda·s·c2 mod n`. Clear `d`, `e`. Return `z`.

Steps 8 to 15 are `[verified]` by recomputation against the upstream vector.

### 7.4 What this guarantees

- One `COMMIT` allows at most one `SIGN`. After step 7 the nonces exist only in volatile memory, so a power cut loses them and never repeats them.
- A second `SIGN` for the same session returns `OutOfOrderStep`.
- A new `COMMIT` burns the earlier session. A `SIGN` for the earlier session then returns `SessionSuperseded`.
- A power cut between `COMMIT` and `SIGN` leaves the slot `ARMED`. The `SIGN` for that session can still run once. This is needed when one reader is used for two cards.
- The host cannot choose or replay a session number, because the card creates it.

### 7.5 Aggregation (host)

`z = sum of the shares mod n`. The signature is `x(R) || z`. The host uses `frost::aggregate` from the reference, which also checks each share against the signer's public share and names a bad signer. The host builds the reference's public key package from `P` and the public shares that any card returns in `STATUS`.

Every signature is then checked by the independent BIP340 verifier before it is used.

### 7.6 Policy

The card signs any message a PIN-verified host presents. v1 has no display and no spending rules. This is stated in `threat-model.md`.

## 8. Share repair (card replacement)

**`[unreviewed construction]`** The arithmetic is the repairable threshold scheme as implemented in ZF `keys/repairable.rs` `[verified: read at the tag]`. The encryption of the values between cards, the authorization and the peer-list update are designed in this spec. They reuse ChillDKG's encryption primitive and BIP340 signatures and add no new primitive. They still need external cryptographic review before any use with real funds.

### 8.1 Roles

- **Recipient**: a new card in state `READY`, with host public key `hpk_new`. It takes the dead card's position `k`.
- **Helpers**: two `KEYED` cards at positions `a < b`, neither equal to `k`. v1 uses exactly two helpers. `[design]`
- **Authorizer**: one `KEYED` offline card (position 2 or 3), not at position `k`. It may also be a helper.

### 8.2 Authorization

Message: `A = prefix || gf || byte(k) || hpk_new`, where `prefix` is the ASCII string `distal/reshare auth v1` padded with zero bytes to 33 bytes (the same domain-separation pattern as ChillDKG's `certeq_message`), and `gf` is the current group fingerprint.

`AUTHORIZE` on the authorizer (PIN verified) checks that its own position is 2 or 3 and is not `k`, and returns a BIP340 signature over `A` with its `hsk` (standard `BIP0340` tags, fresh `aux_rand` from the card).

Before `AUTHORIZE` is called, the user confirms the recipient's card fingerprint (`ceremony.md`). The card cannot see that confirmation. What the cards enforce is the signature.

Every other card verifies the signature under the stored host public key at the authorizer's position, and rejects it unless that position is 2 or 3 and differs from `k`. A bad or missing authorization returns `ReshareRefused`.

### 8.3 Arithmetic (as the reference)

Identifiers are `position + 1`. For helper `i` with the other helper `j`, and target identifier `K = k + 1`:

- `zeta_i = (K − x_j) / (x_i − x_j) mod n`, with `x_i`, `x_j` the helpers' identifiers.
- Helper `i` splits `zeta_i·s_i` into two **deltas**. In the reference's order: the delta for the helper with the lower identifier is a fresh random scalar `r`; the delta for the helper with the higher identifier is `zeta_i·s_i − r`.
- Each helper's **sigma** is the sum of the delta it kept for itself and the delta it received.
- The recipient's share is `s_k = sigma_a + sigma_b mod n`.

`s_i` is the stored, normalized share. The result is the normalized share of position `k`.

The formulas in this subsection were checked numerically on 2026-11-11 for every position `k`, every pair of helpers, and for both normalized and non-normalized shares: `sigma_a + sigma_b` equals the share of position `k` in every case. That checks the arithmetic only. It says nothing about the security of sections 8.2, 8.4 and 8.5.

### 8.4 Encryption between cards

A value `v` sent from card `X` to card `Y` is sent as `(N, ct)`:

- `X` draws a fresh 32-byte secret `u` (valid scalar) and sets `N = enc33(u·G)`.
- `pad = TH("distal/reshare pad v1", ecdh(u, hpk_Y) || N || hpk_Y || ctx)` as an integer mod `n`, where `ecdh` is SHA-256 of the compressed shared point (as `ecdh_libsecp256k1`).
- `ct = v + pad mod n`.
- `Y` computes the same pad with its `hsk` and `N`, and `v = ct − pad mod n`.
- `ctx = byte(stage) || gf || byte(k) || hpk_new || byte(position of X) || byte(position of Y)`. `stage` is `01` for a delta and `02` for a sigma. For a sigma, the position of `Y` is `k`.

A fresh `u` is used for every value, so no pad is used twice.

### 8.5 Steps

1. `AUTHORIZE` on the authorizer. Output: the authorization signature.
2. `HELP1` on each helper (PIN verified). Input: `k`, `hpk_new`, the authorizer's position, the signature, the two helper positions. The helper verifies the authorization, computes its two deltas, keeps its own in volatile memory, and returns the other one encrypted to the other helper.
3. `HELP2` on each helper, in the same power session as its `HELP1`. Input: the encrypted delta from the other helper. The helper computes its sigma and returns it encrypted to `hpk_new`. It then clears its volatile repair data.
4. `RECEIVE` on the recipient (PIN verified). Input: `k`, the new list of four host public keys (with `hpk_new` at position `k`), the group key `P`, the four public shares, the two helper positions, and the two encrypted sigmas. The recipient checks that its own `hpk` is at position `k` and nowhere else, decrypts and adds the sigmas, and checks `s_k·G` against the public share for position `k`. If the check passes it stores everything in one atomic step and becomes `KEYED`. If it fails it returns `ReshareIncomplete` and stores nothing.
5. `ADOPT` on every other card of the group (PIN verified), whenever that card is next available. Input: `k`, `hpk_new`, the authorizer's position, the signature. The card verifies the authorization against its **current** group fingerprint and peer list, replaces the host public key at position `k`, and recomputes `gf`.

### 8.6 Rules

- `HELP2` without `HELP1` in the same power session returns `OutOfOrderStep`. A helper holds nothing about a repair after a reset.
- An interrupted repair leaves the recipient `READY`. Run it again from step 2. The authorization can be reused for the same `k` and `hpk_new`.
- The recipient has no way to check that the group data it is given is the real group. The user compares the recipient's group fingerprint and group key with those of the other cards after the run (`ceremony.md`).
- After the run, every card that has adopted shows the new group fingerprint. A card that has not adopted yet shows the old one. Signing works either way, because signing does not use the peer list.
- The repaired share equals the share the dead card held. If the old card may be in someone else's hands, do not repair. Sweep.
- If both offline cards are dead, repair is impossible. The device cards sweep to a new group.

## 9. What the reviewer must check before tagging `spec-v1`

1. Section 8 as a whole. It is new design.
2. Section 6.4: that negating the share, the key and the public shares after ChillDKG is consistent on all cards and with the signing reference.
3. Section 7.2 and 7.3: the order "burn, then compute", and that the session counter gives the replay protection claimed.
4. Section 4: no unblocking code, and `WIPE` keeping `hsk`.
5. That every `[design]` item is acceptable.
6. `apdu.md`: lengths and limits.

## 10. Reserved for later

- Anti-exfiltration. A status word is reserved. Nothing else is specified.
- Peer-checked PIN counters.
- More than one key slot.
- HD derivation.
