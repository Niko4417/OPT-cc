---
status: accepted
date: 2026-11-20
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/21
---

# Sign and encrypt every bundle with one key per installation

The bundle research signed a project memory bundle with the consultant's own SSH key. In V1 the export runs on AWS (ADR-0013), where that private key is not present. So each installation creates its own Ed25519 SSH key at install time and keeps it in AWS Secrets Manager in Frankfurt. This installation key signs the bundles the installation exports and decrypts the bundles sent to it.

**Signing.** Export signs `tagmanifest-sha256.txt` with `ssh-keygen -Y sign`. The tag manifest covers the payload manifest, which covers every file, so one signature covers the whole bag. Each installation has a list of trusted signers, and its consultant pastes an exporter's public key into that list. Import refuses an unsigned bundle, a bad signature and a signer that is not on the list.

**Encryption.** Every export is encrypted with `age` to the installation keys of the recipients the exporter names. The result is a `.pmb.age` file. There is no unencrypted export.

## Considered options

- The consultant's own SSH key, checked against their signing keys on GitHub. Rejected because the private key is on the consultant's laptop, not where the export runs.
- No signing in the proof of concept. Rejected because the glossary defines a project memory bundle as a signed file and ADR-0007 relies on that.
- Optional encryption. Rejected because it gives two code paths, and V1 would not test the one that client content needs.
- A second key per installation only for decryption. This is the stricter practice. Rejected because the consultants would exchange two keys in place of one.
- Sigstore keyless signing. Rejected because it needs an OpenID Connect identity at signing time.

## Consequences

- Two consultants exchange public keys in both directions before the first bundle. The exporter needs the recipient's key to encrypt, and the importer needs the exporter's key to verify.
- An installation that loses its key can no longer open bundles encrypted to it.
- The key store is specific to AWS. A later on-device or client-hosted installation keeps its key somewhere else, and the format does not change.
