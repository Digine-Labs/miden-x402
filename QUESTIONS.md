## M2

Two notable design points (decisions that came up during the build and stuck):

- miden-client 0.14.8 with features = ["tonic"]. The GrpcClient type is feature-gated; default features don't expose it. Pinned 0.14 to match the public testnet (0.15 is unstable, not deployed).
- Recipient is verified via canonical P2ID storage, not by trust. The verifier confirms note.recipient().script().root() == P2idNote::script_root() and then parses the storage felts through P2idNoteStorage::try_from(&[Felt]) to extract the target AccountId. Anything that isn't a real P2ID note is rejected as NotP2id/InvalidP2idStorage. This closes the obvious "claim someone else's note paid me" attack.

## M3

- The load-bearing M3 choice is the header naming. We use Payment-Required / Payment-Signature / Payment-Response (TitleCase, no X- prefix) rather than the older Coinbase X-PAYMENT / X-PAYMENT-RESPONSE pair, because the x402 v2 spec and x402-rs v2 paygate both use the TitleCase forms. This matters for cross-SDK interop — Node/Python SDKs in M4/M5 must emit and parse exactly these names.