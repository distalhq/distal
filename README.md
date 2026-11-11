# distal

Open threshold signing across secure elements from different vendors. Each chip holds one share of the private key and computes part of every signature, so the key is never assembled and no single chip is a single point of failure.

Site: [distal.run](https://distal.run)

## What is here

| Path | What it is |
|---|---|
| `spec/` | Protocol specification: FROST threshold Schnorr, key generation and resharing on the chips, APDU interface, ceremonies, threat model, status codes. |
| `applet/` | The JavaCard applet. Third-party code lives in `applet/vendor/` as submodules, with every change recorded in `applet/vendor/PATCHLOG.md`. |

The host library will be added here when it exists. The applet, host library and spec stay in one repository because they version together and share test vectors.

## Status

The specification is draft v1 and has not been externally reviewed. Some sections still point at internal planning notes that are not published; those pointers will be removed as the spec stabilizes. The anti-exfiltration design is new cryptography and does not ship before it has been reviewed.

## Licensing

No license has been chosen yet for Distal's own code, so until one is added all rights are reserved. Vendored code keeps its own licenses: [JCFROST](https://github.com/crocs-muni/JCFROST) and [JCMathLib](https://github.com/OpenCryptoProject/JCMathLib) are both MIT.

## Security

Please report vulnerabilities privately, not in public issues. See [SECURITY.md](https://github.com/distalhq/.github/blob/main/SECURITY.md).

## Clone

```bash
git clone --recurse-submodules https://github.com/distalhq/distal.git
```
