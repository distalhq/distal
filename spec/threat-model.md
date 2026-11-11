# distal threat model (DRAFT)

Starting points: `distrust.md` section 5 and the threat tables in `PLAN.md`.

## 1. What is protected

1. The group signing key. It must never exist in one place.
2. Each card's share and host secret key.
3. The rule that one nonce produces at most one signature share.
4. The membership of the group: which four cards hold shares.

Not protected in v1: which messages get signed.

## 2. Who is assumed hostile

| Party | Assumed able to |
|---|---|
| The host (device firmware or computer), after setup | Read, drop, delay, reorder, replay and invent every message between cards. Read every PIN typed on it. Ask cards to sign anything once it has the PINs. |
| A thief with the device | Use both device chips, guess PINs up to the try limit, attack one chip physically. |
| One chip vendor, or one broken chip | Learn everything inside that one chip. |
| A finder of one offline card | Guess its PIN up to the try limit. |

Assumed honest: the host **during setup** (see section 4); the party that loads the applet onto the cards; at least one of the two chip vendors; the applet code itself.

## 3. Threats by milestone

| Threat | M0 | M1 | M2 |
|---|---|---|---|
| Host reads the key while signing (`distrust.md` 5a to 5c) | Mitigated during signing. **Not at setup**: the host deals the key. | Mitigated: key generation on the cards; the host sees public data and encrypted shares only | same |
| Host asks for a malicious signature (5e) | Not mitigated | Not mitigated | Not mitigated in phase 1 |
| Stolen device (5d) | No PIN | PIN per chip, 5 tries, then the chip locks for good; sweep available | plus peer-checked counters if built |
| Nonce reuse by replay, reorder or power loss | Mitigated by burn-before-output; simulator tests | Mitigated; card-made session numbers; power-cut rig | 10,000 cycles per vendor |
| Host replays or reorders protocol steps | Partly | Mitigated: the card owns all state (`protocol.md` 6.5, 7.4, 8.6) | same |
| Split view at pairing (cards disagree on the group) | No pairing | Detected by the group fingerprint check and by ChillDKG's certificate | same |
| Substituted cards at setup | n/a | **Not detected** in the prototype (section 4) | Factory record, checked on a second device; vendor attestation where offered |
| Rogue-key attack in key generation | No DKG | Mitigated by ChillDKG's proofs of possession | audited |
| Hidden Taproot script path in the group key | n/a | Mitigated by ChillDKG's tweak | same |
| Host or thief adds its own card by share repair | No repair | Mitigated: cards require an offline card's signature. The fingerprint check depends on a display the attacker does not control. | same |
| Backdoored chip leaks its share through nonces | Not mitigated | Not mitigated | Only if anti-exfiltration passes review |
| Side channels in the applet's arithmetic | Not mitigated | Not mitigated | Evaluated and stated; not claimed solved |
| One bug in the shared applet | Present | Present | Present, reduced by audit |
| Tampered applet installed | Not checked | Not checked | Per-vendor verdict |
| Both offline cards taken together | n/a | Stored apart; each has its own PIN | same |
| Recovery host reads offline-card PINs | n/a | Rule in `ceremony.md` section 5 | same |
| Both vendors colluding | Accepted | Accepted | Accepted |

## 4. Setup and substitution: the main limit

The cards make their own identity keys. Nothing inside the protocol can prove that a host public key belongs to a real chip.

- **What the fingerprint ceremony detects:** cards that were given different lists; a card that was not in the list the user recorded; any later change of membership that was not authorized.
- **What it does not detect:** a host that is malicious **during setup** and puts software keys it controls into the list in place of real cards. In the product the obvious case is malicious device firmware replacing its own two chips. All four fingerprints then agree, and the attacker holds two shares from the first moment.
- **What reduces it:** the factory record, checked on a second device, moves the attack to the factory or the delivery chain. Vendor attestation, where a vendor offers it to third-party applets, moves it to the vendor.
- **What remains:** setup on malicious firmware together with a tampered factory record is not detected. The party that loads the applet is trusted in any case, because users mostly cannot verify the installed applet.

This is no worse than a single-chip wallet, where malicious firmware at setup can generate a seed it already knows. The protection distal adds applies after an honest setup.

## 5. Known weaknesses of v1, stated plainly

1. **Share repair is new design** (`protocol.md` section 8). It has had no external review. Until it has, treat a repaired group as less trusted than a fresh one, and do not use repair with real funds.
2. **The PIN travels in clear** from host to card. A host or a tap on the link learns it.
3. **The two device chips count PIN tries separately.** Until peer-checked counters exist, a thief with the device gets up to 5 guesses on each chip, so up to 10 guesses at the device PIN.
3a. **A locked card cannot be unlocked.** Five wrong PINs on each of two cards by a visitor can remove two positions. Three such cards make signing impossible. This trades availability for simplicity.
4. **Replay of an old key-generation transcript is not possible in v1**, because v1 has no recovery command. If one is added later, that question must be reopened.
5. **The authorization for a repair can be reused** for the same position and the same new card. That only repeats the same repair to the same card.
6. **A stuck setup** (some cards keyed, some not) is handled by wiping all four. v1 does not use ChillDKG's recovery data.
7. **The session counter is 32 bits.** At its limit the card stops signing. That is far beyond the expected write endurance of the card `[unverified: endurance not measured]`.
8. **JCMathLib is not constant time.** Its authors say it is not for production. A timing leak in it applies to both vendors.
9. **The applet is one implementation on all four cards.** One logic bug affects all of them.
10. **No policy on what is signed.** A host with PINs signs anything.

## 6. Residual risks accepted for phase 1

- Both chip vendors colluding.
- A malicious host at setup combined with a tampered factory record.
- A thief who guesses the PIN within the try limit before the sweep confirms.
- Everything listed in section 5 that has no mitigation yet.
