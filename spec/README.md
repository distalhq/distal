# distal protocol specification

**Status: DRAFT v1, written 2026-11-11. Not reviewed. Not tagged.**

No one may implement from this folder until a person has reviewed it and tagged the commit `spec-v1`. Section 9 of `protocol.md` lists what the reviewer must check first.

| File | What it defines |
|---|---|
| `protocol.md` | Ciphersuite, key generation, signing, share repair, card state machine, rules |
| `apdu.md` | Every byte on the wire between host and card, for the M0 profile and for v1 |
| `status-codes.json` | Every status word. Applet and host constants are generated from this file. |
| `ceremony.md` | What people do: setup, storage, signing, recovery, sweep, card replacement |
| `threat-model.md` | What is protected, from whom, in which milestone, and what is not |
| `vectors/` | Test vectors read by both test suites (created by tasks M0-2 and M1-2) |
| `anti-exfil.md` | Does not exist yet. No code or vectors before it carries `Review: PASSED`. |

## Rules for this folder

1. A message format is defined in exactly one place: here, or in a pinned reference that this folder names. Nothing is defined twice.
2. Where this folder says "as the reference", the reference code at the pinned commit decides, and the shared vectors are the test. If text here disagrees with the pinned reference, the reference wins and the text is a bug to report.
3. Labels: `[verified]` means read from the pinned source or confirmed by recomputation. `[design]` means a choice made in this spec. `[unreviewed construction]` means new protocol design that needs external cryptographic review. `[unverified]` means unknown; do not rely on it.

## Pinned references

| Part | Reference | Pin |
|---|---|---|
| Signing | ZF `frost-secp256k1-tr` | version 3.0.0, git tag `frost-core/v3.0.0` |
| Key generation | ChillDKG reference, `python/chilldkg_ref/` | commit `a91896883f85b159415ecf298d5e844879af112d` (draft BIP, version 0.3.0-dev) |
| Share repair arithmetic | ZF `frost-core/src/keys/repairable.rs` | git tag `frost-core/v3.0.0` |
| Signature verification | bitcoin/bips `bip-0340/reference.py` | commit `f98774a68c23269d9bfa78c8d9c4671ebf4ceb69` |

There are two reference implementations: ZF for signing and repair, ChillDKG for key generation. Each protocol part has exactly one.

## Decisions recorded here (founder, 2026-11-11)

- DKG version: ChillDKG at the pinned commit.
- Pairing trust anchor: fingerprint ceremony, plus factory-recorded fingerprints checked on a second device, plus vendor attestation where a vendor offers it.
