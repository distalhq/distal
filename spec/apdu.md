# distal APDU interface (DRAFT)

This file defines every byte between host and card. There are two profiles. They are never mixed in one build.

- **Profile M0**: the JCFROST interface, kept unchanged for the application demo.
- **Profile v1**: the interface for milestone M1 and later.

Notation is as in `protocol.md` section 1. `be2(x)` and `be4(x)` are big-endian integers of 2 and 4 bytes. All lengths are in bytes.

---

## Profile M0

Copied from JCFROST at commit `930515e496eea3e7c012ef26ffb072672f27b195` (`README.md`, `JCFROST.java`, `Consts.java`) `[verified]`. Short APDUs only. No chaining. No PIN.

| Name | CLA | INS | P1 | P2 | Command data | Response data |
|---|---|---|---|---|---|---|
| INITIALIZE | 00 | 00 | 00 | 00 | none | none |
| SETUP | 00 | 01 | `t` | `n` | card identifier (1), share (32), group key `enc65(P)` (65) | none |
| COMMIT | 00 | 02 | 00 | 00 | none | `enc33(D)` (33), `enc33(E)` (33) |
| COMMITMENT | 00 | 03 | identifier `j` | 00 | `enc65(D_j)` (65), `enc65(E_j)` (65) | none |
| SIGN | 00 | 04 | message length | 00 | message | share `z` (32) |
| RESET | 00 | 05 | 00 | 00 | none | none |
| GROUP_KEY | 00 | 06 | 00 | 00 | none | `enc33(P)` (33) |

Rules for M0:

- The share and group key in SETUP are already normalized to even Y by the host (`protocol.md` section 6.4 describes the same rule).
- COMMITMENT is sent once per signer, in ascending identifier order, the card's own included.
- One COMMIT allows one SIGN. A SIGN with no armed nonce returns `E_NONCE_NOT_ARMED` (`EE0A`).
- The debug behaviours of upstream (SETUP echoing data, COMMIT taking randomness) exist only in the test flavor, which runs only in the simulator.
- Status words for M0 are the entries with `"profile": "m0"` in `status-codes.json`.
- Upstream's status words start with `EE`, `F1`..`F5` and `FF`. ISO 7816 expects the first status byte to be `6x` or `9x`. Whether every card and reader passes the upstream values through unchanged is `[unverified]`. M0 keeps them because M0 does not change upstream's interface. If a card returns something else in an error case, record it in the M0-6 evidence; do not renumber.

---

## Profile v1

### 1. Framing

**Class byte.** `80` for a command that is complete or is the last block of a chain. `90` for a block that is followed by more blocks.

**Command chaining (host to card).** A command whose data is longer than 240 bytes is sent as several APDUs with the same INS, P1 and P2. Each block carries at most 240 data bytes. All blocks except the last use CLA `90`. The card appends each block to an input buffer and processes the command when the block with CLA `80` arrives.

- The input buffer holds at most 1024 bytes. A longer chain returns `ChainingError`.
- If INS, P1 or P2 changes inside a chain, the card returns `ChainingError`.
- After any error, and after any completed command, the input buffer is empty.
- A non-final block returns `9000` with no data.
- No protocol code sees a partial chain.

**Response framing (card to host).** Every successful response to a completed command is:

`be2(total) || chunk`

where `total` is the full length of the command's output and `chunk` is its first `min(total, 238)` bytes. A command with no output returns `00 00`.

If `total > 238`, the host reads the rest with FETCH. The output stays available until the next command that is not FETCH.

**FETCH.** CLA `80`, INS `7E`, P1 P2 = `be2(offset)`, no data. Returns up to 240 bytes of the pending output starting at `offset`, with no length prefix. An offset at or beyond `total` returns `BadData`. FETCH needs no PIN, because every command output is public or encrypted.

Extended-length APDUs are not used. `[design]`

**Errors.** A status word other than `9000` carries no data.

### 2. Instruction table

| INS | P1 | Name | Needs PIN | Allowed in |
|---|---|---|---|---|
| 01 | 00 | STATUS summary | no | any |
| 01 | 01 | STATUS public shares | no | KEYED |
| 01 | 02 | STATUS peers | no | PAIRED, KEYED |
| 02 | 00 | INIT | no | BLANK |
| 03 | 00 | VERIFY_PIN | no | READY, PAIRED, KEYED |
| 04 | 00 | CHANGE_PIN | yes | READY, PAIRED, KEYED |
| 05 | 00 | PAIR | yes | READY |
| 06 | 00 | WIPE | yes | PAIRED, KEYED |
| 10 | 01 | DKG_STEP START | yes | PAIRED |
| 10 | 02 | DKG_STEP PEERS | yes | PAIRED |
| 10 | 03 | DKG_STEP FINALIZE | yes | PAIRED |
| 20 | 01 | SIGN_STEP COMMIT | yes | KEYED |
| 20 | 02 | SIGN_STEP SIGN | yes | KEYED |
| 30 | 01 | RESHARE_STEP AUTHORIZE | yes | KEYED |
| 30 | 02 | RESHARE_STEP HELP1 | yes | KEYED |
| 30 | 03 | RESHARE_STEP HELP2 | yes | KEYED |
| 30 | 04 | RESHARE_STEP RECEIVE | yes | READY |
| 30 | 05 | RESHARE_STEP ADOPT | yes | KEYED |
| 7E | offset | FETCH | no | any |

P2 is `00` for every command except FETCH. Any other INS returns `6D00`. Any other P1 or P2 returns `6A86`. A command in a state not listed returns `WrongState`. A command that needs the PIN without it returns `PinRequired`.

The card API named in the design (`dkg_step`, `sign_step`, `reshare_step`, `status`) is INS `10`, `20`, `30`, `01`. The other instructions are card management.

### 3. Commands

Lengths in parentheses. "Output" is the data after the `be2(total)` prefix.

**STATUS summary** (INS 01, P1 00). Input: none. Output (115):

| Field | Length | Values |
|---|---|---|
| version major, minor | 1, 1 | `01`, `00` |
| lifecycle | 1 | `00` BLANK, `01` READY, `02` PAIRED, `03` KEYED, `04` LOCKED |
| card profile | 1 | the install parameter (JCMathLib profile number) |
| position | 1 | `00`..`03`, or `FF` if not paired |
| t | 1 | `02`, or `00` if not paired |
| n | 1 | `04`, or `00` if not paired |
| PIN tries left | 1 | `00`..`05` |
| nonce slot | 1 | `00` EMPTY, `01` ARMED |
| session counter `ctr` | 4 | big-endian |
| sign count `signs` | 4 | big-endian |
| host public key `hpk` | 33 | zeros if BLANK |
| group fingerprint `gf` | 32 | zeros if not paired |
| group key `enc33(P)` | 33 | zeros if not KEYED |

**STATUS public shares** (INS 01, P1 01). Output (132): the four normalized public shares, `enc33` each, in position order.

**STATUS peers** (INS 01, P1 02). Output (132): the four host public keys, in position order.

**INIT** (INS 02). Input: the PIN (6 to 12). Output (33): `hpk`.

**VERIFY_PIN** (INS 03). Input: the PIN. Output: none. A wrong PIN returns `63Cx` where `x` is the number of tries left. With no tries left the card becomes LOCKED and returns `6983`.

**CHANGE_PIN** (INS 04). Input: the new PIN (6 to 12). Output: none.

**PAIR** (INS 05). Input (134): `t` (1), count (1), four host public keys (33 each) in position order. Output (32): `gf`.

**WIPE** (INS 06). Input: none. Output: none.

**DKG_STEP START** (INS 10, P1 01). Input: none. Output (291): `ParticipantMsg1`.

**DKG_STEP PEERS** (INS 10, P1 02). Input (681): `CoordinatorMsg1`. Output (64): `ParticipantMsg2`.

**DKG_STEP FINALIZE** (INS 10, P1 03). Input (256): `CoordinatorMsg2`. Output (33): `enc33(P)`, the normalized group key.

The four DKG messages are the byte strings produced by `to_bytes` in the ChillDKG reference at the pinned commit, for `t = 2`, `n = 4`. An input of any other length returns `BadData`.

**SIGN_STEP COMMIT** (INS 20, P1 01). Input: none. Output (70): `be4(ctr)`, `enc33(D)`, `enc33(E)`.

**SIGN_STEP SIGN** (INS 20, P1 02). Input:

| Field | Length |
|---|---|
| session number | 4 |
| signer count `c` | 1 (2 to 4) |
| `c` entries, ascending by identifier: identifier (1), `enc65(D_j)` (65), `enc65(E_j)` (65) | 131 each |
| message length | 1 (1 to 64) |
| message | as given |

For two signers and a 32-byte message this is 300 bytes, so it is chained. Output (32): the share `z`.

Commitments are sent uncompressed so that the card does not compute square roots. The card must still check that each point is on the curve.

**RESHARE_STEP AUTHORIZE** (INS 30, P1 01). Input (34): position `k` (1), `hpk_new` (33). Output (64): the authorization signature.

**RESHARE_STEP HELP1** (INS 30, P1 02). Input (101): `k` (1), `hpk_new` (33), authorizer position (1), authorization signature (64), helper position `a` (1), helper position `b` (1), with `a < b`. Output (65): `N` (33), `ct` (32): this helper's delta for the other helper.

**RESHARE_STEP HELP2** (INS 30, P1 03). Input (65): `N` (33), `ct` (32): the other helper's delta for this helper. Output (65): `N'` (33), `ct'` (32): this helper's sigma for the recipient.

**RESHARE_STEP RECEIVE** (INS 30, P1 04). Input (430):

| Field | Length |
|---|---|
| position `k` | 1 |
| four host public keys, position order, `hpk_new` at `k` | 132 |
| group key `enc33(P)` | 33 |
| four public shares, `enc33` each | 132 |
| helper positions `a`, `b` | 2 |
| sigma from helper `a`: `N`, `ct` | 65 |
| sigma from helper `b`: `N`, `ct` | 65 |

Output (32): the new `gf`.

**RESHARE_STEP ADOPT** (INS 30, P1 05). Input (99): `k` (1), `hpk_new` (33), authorizer position (1), authorization signature (64). Output (32): the new `gf`.

### 4. Input validation, for every command

- A wrong total length returns `BadData` before anything else is looked at.
- A position outside `0..3`, an identifier outside `1..4`, or a count outside its range returns `BadData`.
- A point that is not a valid encoding of a curve point, or is the point at infinity where the reference rejects it, returns `BadPoint`.
- A scalar that is not below `n`, where the reference requires that, returns `BadData`.

### 5. Simulator bridge

Test tooling only. TCP on `127.0.0.1`, one port per simulated card. Each message in both directions is `be2(length)` followed by that many bytes. A request is one raw command APDU. A response is the response data followed by the two status bytes.

### 6. Not verified

- Whether each candidate card accepts the class bytes `80` and `90` for an applet without restrictions. `[unverified]` Test on each card model before relying on it.
- The memory each card model gives an applet for a 1024-byte input buffer, a pending output, and the DKG state. `[unverified]`

### 7. Test-only instruction (test flavor)

This instruction exists only in the test flavor of the applet. The test flavor runs only in the simulator and is never converted to a CAP file. In the release flavor, INS `F0` returns `6D00`, and a test proves it.

CLA `80`, INS `F0`, P2 `00`. No PIN. Chaining and response framing as in section 1.

| P1 | Name | Input | Effect |
|---|---|---|---|
| 01 | SET_HSK | host secret key (32) | Allowed in BLANK only. The next INIT uses this value as `hsk` instead of a random one. |
| 02 | PUSH_RANDOM | 1 to 240 bytes | Appends the bytes to a queue of at most 512 bytes. Every later draw of random bytes takes from the front of the queue until it is empty, then uses the simulator's random source. |
| 03 | FORCE_KEYED | position (1), share (32), `enc33(P)` (33), four public shares (33 each), four host public keys (33 each): 330 bytes | Allowed in READY only. Places the card in KEYED with exactly these values, already normalized. Used to test signing before pairing and key generation exist. |
| 04 | FAULT_AFTER_BURN | none | One-shot. The next SIGN stops with `Internal` immediately after the nonce slot is burned and before any share is returned. Used to test that a lost nonce is never reused. |

Output: none for all four.

Order of random draws that tests rely on: at COMMIT, the 32 bytes for `r1` and then the 32 bytes for `r2`. For key generation, the 32 bytes for `random` at START and the 32 bytes for `aux_rand` at PEERS. `[design]`
