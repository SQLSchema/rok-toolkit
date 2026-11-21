<sub>[RoK Toolkit](../../README.md) › [Docs](../README.md) › The Headless Method</sub>

<img src="../assets/headers/headless.svg" width="100%" alt="">

# The Headless Method

A standalone process that authenticates an account, opens a game session, and acts through the wire protocol.

| Layer | Page |
| --- | --- |
| Login and session | [Login and Initial Game State](01-login.md): methods, regional builds, build identity, the post-ack sequence, failure modes |
| Wire and cipher | [Wire Protocol and Cipher](02-wire-protocol.md): framing, the seed handshake, the keystream, envelopes, acknowledgements |
| Identity | [Identity and Fingerprint](03-identity.md): the device profile and its consistency surfaces |
| Message registry | [Message Registry](04-message-registry.md): MTs, directions, ack pairings, evidence labels |
| Actions | [Game Systems](game-systems/README.md): marches, city, claims, forts, reads, data gathering |

## Invariants

- Unreceived game state is unknown, never zero.
- Only state confirmation establishes completion.
- Transport acknowledgements have their own defaults: the [CommAck rule](02-wire-protocol.md#ordering-and-acknowledgements) treats an absent `Code` as `0`, which confirms the request.

---

<div align="center">

[↑ Documentation Map](../README.md) &nbsp;·&nbsp; [Login and Initial Game State →](01-login.md)

</div>
