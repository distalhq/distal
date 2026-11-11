# distal ceremonies (DRAFT)

What people do, in order, and what each step protects. Card commands are named as in `apdu.md`.

Two settings appear throughout:

- **Prototype**: four smartcards, readers, and a computer running the host CLI. The only display is the computer's screen.
- **Product**: a wallet with its own screen and two embedded chips (positions 0 and 1), two offline cards (positions 2 and 3), and a second device such as a phone that can read the offline cards.

## 1. Fingerprints

- **Card fingerprint**: 20 hex characters from a card's host public key (`protocol.md` section 3). It never changes for the life of the applet on that card.
- **Group fingerprint**: 20 hex characters from the four host public keys and the threshold (`protocol.md` section 5). It changes only when a card is replaced.
- **Factory record** (product only): the four card fingerprints of one kit, recorded by the wallet maker when it loads the applet, and delivered with the kit on paper or engraved on the cards. `[design]`

Any card shows its own fingerprint and its group fingerprint through `STATUS`, without a PIN.

## 2. Setup

All four cards must be present, on a host the user trusts for this session.

1. `INIT` on each card. The user chooses a PIN per card. The offline cards' PINs must differ from the device PIN.
2. The host reads the four card fingerprints and shows them.
   - Product: the user checks on the second device that the two offline cards' fingerprints match the factory record, and that the wallet's screen shows the two chip fingerprints from the factory record.
   - Prototype: there is no factory record. The user writes the four fingerprints down.
3. `PAIR` on each card with the same ordered list. Each card returns the group fingerprint.
4. The host shows the group fingerprint. The user checks that all four cards report the same value. In the product the user reads it from the offline cards on the second device and compares with the wallet's screen. The user writes it down.
5. Key generation: `DKG_STEP START` on all four, then `PEERS` on all four, then `FINALIZE` on all four.
6. The host reads `STATUS` from all four cards. Setup continues only if all are `KEYED` with the same group key and group fingerprint. Otherwise the host wipes all four and returns to step 3.
7. A test signature with each of the six pairs of cards. Each is checked by the independent verifier.
8. The host shows the group's public key. The user records it with the group fingerprint.

If setup is interrupted at any point before step 6 succeeds, wipe all four cards and start from step 3. Nothing from the failed run is kept.

## 3. Storage

- Positions 0 and 1 stay in the device.
- Positions 2 and 3 go to two different places. Two offline cards in one place are a full signing quorum for whoever finds them and learns their PINs.
- The paper with the group fingerprint and public key is public information. Keep it with neither offline card, so that it stays available if one place is lost.
- PINs are not written on or kept with the cards.

## 4. Everyday signing

The device verifies the PIN on both of its chips, runs `COMMIT` on both, then `SIGN` on both, aggregates, and checks the signature. No offline card is needed.

If a signing session is interrupted, the host starts a new one with `COMMIT`. It never repeats `SIGN` for an old session.

## 5. Recovery

**One device chip is dead.** Bring one offline card. Sign with the living chip and that offline card. Then replace the dead chip (section 7).

**The device is lost or destroyed.** Bring both offline cards to a host the user trusts for this session. Sign with them. Then set up a new group on new cards and move the funds (section 6, without the urgency).

**Rule for PINs on a recovery host.** The host on which a PIN is typed can read it.

1. Recover on the device itself, or on a computer the user trusts for that session. The host says this before it asks for a PIN.
2. After any recovery or sweep on a general-purpose computer, change the PIN of each offline card that was used, at the next session on a trusted host. The host prompts for this.
3. A PIN without the card is useless. A malicious host with two cards inserted and both PINs can request any signature. v1 does not prevent that.

## 6. Sweep (the device was stolen)

A thief with the device holds two shares behind one PIN. Act before the PIN is guessed.

1. Set up a **new** group on four new cards (section 2). This creates a new key.
2. Bring both old offline cards. Build a transaction that spends everything from the old key to the new key.
3. Sign it with the two old offline cards (section 4's steps, on those cards).
4. Check the signature with the independent verifier. Broadcast the transaction.
5. When it has confirmed, `WIPE` the old offline cards.

Notes:

- Any two old shares can sign the sweep. If one device chip is still in the user's hands, it may be used with one offline card.
- If the thief signs first, the funds are gone. The PIN and its 5 tries are the only protection until the sweep confirms.
- The old key must never receive funds again.
- Transaction format: a Taproot key-path spend. The group key from key generation is used directly as the output key, with no further tweak, because ChillDKG already makes it safe for that use (`protocol.md` section 6.4). The sighash is computed as BIP341 specifies and is the 32-byte message given to `SIGN`. Which library builds the transaction, and its exact calls, are `[unverified]` and are fixed by the task that implements the sweep.
- Networks: signet or testnet only until the real-funds rule in the plan is met.

## 7. Replacing a card

Needed when a card is dead or locked. Not to be used when the old card may be in someone else's hands; sweep instead.

Present: the new card, two working cards of the group (the helpers), and at least one working offline card (the authorizer; it may be one of the helpers).

1. `INIT` on the new card with a new PIN. The host shows its card fingerprint.
2. **The user confirms the new card's fingerprint.** Product: against the fingerprint printed with the replacement card, read on the wallet's screen. Prototype: the user reads it from the host's screen, which is the weak point of the prototype.
3. `AUTHORIZE` on the offline card.
4. `HELP1` on both helpers, then `HELP2` on both helpers.
5. `RECEIVE` on the new card.
6. `ADOPT` on every other working card of the group. An offline card stored elsewhere adopts the next time it is brought out. Adopt replacements in the order they happened.
7. The host reads `STATUS` from every present card. The new card must show the same group key as the others, and every card that has adopted must show the same new group fingerprint. The user records the new group fingerprint.
8. A test signature with the new card and one other card, checked by the independent verifier.

Until step 8 passes, the host warns on every command that the position has no working card.

If the run fails at any step, the new card is still `READY`. Repeat from step 4. If it keeps failing, `WIPE` is not needed; the new card holds nothing.

## 8. Upgrading the applet

Installing a new applet version erases the card's share and its host secret key. So an upgrade is a card replacement:

1. Upgrade one card at a time.
2. Delete and install the applet on that card. It is now a new card with a new fingerprint.
3. Replace it into its old position (section 7).
4. Only then move to the next card.

Never upgrade two cards at once. Never upgrade both offline cards before the first one has been replaced back and tested.

## 9. What the user must keep

| Item | Secret? | Where |
|---|---|---|
| Device PIN | yes | memory |
| Each offline card's PIN | yes | memory, or separately from that card |
| Group fingerprint and public key | no | on paper, apart from the offline cards |
| Factory record (product) | no | with the paper above |
