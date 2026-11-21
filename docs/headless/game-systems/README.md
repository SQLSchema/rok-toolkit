<sub>[RoK Toolkit](../../../README.md) › [Docs](../../README.md) › [Headless](../README.md) › Game Systems</sub>

<img src="../../assets/headers/game-systems.svg" width="100%" alt="">

# Game Systems

The [message registry](../04-message-registry.md) indexes every MT used across the modules.

| Module | What it covers |
| --- | --- |
| [State Reads](01-state-reads.md) | Explicit queries: city, heroes, nodes, map, profile, mail, quests, KV; the login flood; ranking data |
| [Gathering and Marches](02-marches.md) | Two-step dispatch, searches, combat variant, recalls |
| [Fort Operations](03-forts.md) | Fort rallies: find, create, join, kick, cancel, AP |
| [City Development](04-city.md) | Upgrades, placement, research, training |
| [Collection and Claims](05-claims.md) | Claims, collection, mail, hospital, alliance help |
| [Data Gathering](06-data-gathering.md) | Ranking and map observations for the RoK-Data store, over the same login and wire protocol; rankings MT fields |
| [The RoK-Data Store](07-rokdata-store.md) | Record kinds, storage, API routes, operating limits |

## Invariants

- Merge updates and replace lists or outer-key domains with snapshots, including empty snapshots where specified.
- Unreported state stays unknown.
- Acceptance does not prove completion.

---

<div align="center">

[← Message Registry](../04-message-registry.md) &nbsp;·&nbsp; [↑ The Headless Method](../README.md) &nbsp;·&nbsp; [State Reads →](01-state-reads.md)

</div>
