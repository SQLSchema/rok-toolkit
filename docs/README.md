<sub>[RoK Toolkit](../README.md) › Documentation Map</sub>

# Documentation Map

Evidence labels throughout:

- **MEASURED**: directly observed in a capture or executed test.
- **INFERRED**: evidence-supported interpretation without direct confirmation.
- **UNKNOWN**: looked for, not found. A negative result means "not observed under these conditions", never "does not happen".

Subsidiary labels map onto these:

- registry **Unestablished** = UNKNOWN for purpose
- Lua runtime **CODE** = read from the client's own code, not watched live
- **REPORTED** = a third-party claim the artifacts do not confirm
- **Observed/Decompile/Enabled** grade Lua runtime reads by how they were verified

## Headless

A standalone process that authenticates and acts through a game session.

| Page | What it covers |
| --- | --- |
| [The Headless Method](headless/README.md) | Read-path-first construction. |
| [Login and Initial Game State](headless/01-login.md) | Login methods, regional builds, platform hosts, post-ack timing, failures. |
| [Wire Protocol and Cipher](headless/02-wire-protocol.md) | Framing, MT 8562 seed handshake, keystream and verification vector, envelopes, compression. |
| [Identity and Fingerprint](headless/03-identity.md) | Per-account DeviceProfile, field mappings, evidence limits. |
| [Message Registry](headless/04-message-registry.md) | 142 registered MTs: directions, ack pairings, evidence labels. |
| [Game Systems](headless/game-systems/README.md) | Action module index. |

### Action modules

| Page | What it covers |
| --- | --- |
| [State Reads](headless/game-systems/01-state-reads.md) | Explicit queries, the login flood, ranking data. |
| [Gathering and Marches](headless/game-systems/02-marches.md) | Two-step dispatch, searches, combat variant, recalls, hero roster. |
| [Fort Operations](headless/game-systems/03-forts.md) | Fort rallies: find, create, join, kick, cancel, AP. |
| [City Development](headless/game-systems/04-city.md) | Upgrades, placement, research, training. |
| [Collection and Claims](headless/game-systems/05-claims.md) | Claims, collection, mail, hospital, alliance help. |
| [Data Gathering](headless/game-systems/06-data-gathering.md) | Observation, recording, rankings MT fields. |
| [The RoK-Data Store](headless/game-systems/07-rokdata-store.md) | Record kinds, storage, API routes. |

## Lua Runtime

A desktop tool that attaches to the running client, reads state, and invokes client code.

| Page | What it covers |
| --- | --- |
| [The Lua Runtime Method](lua-runtime/README.md) | Method and comparison with headless. |
| [Attach and the Execution Hook](lua-runtime/01-attach.md) | Process discovery, elevation, attachment, agent invariants. |
| [Reading the Client](lua-runtime/02-client-reads.md) | Read constraints, literal table paths, install files. |
| [Invoking the Client](lua-runtime/03-client-calls.md) | Client functions by action family, invocation constraints. |
| [Governors and Reloads](lua-runtime/04-governors.md) | In-client switching, reload hold. |

---

<div align="center">

[↑ RoK Toolkit](../README.md) &nbsp;·&nbsp; [The Headless Method →](headless/README.md)

</div>
