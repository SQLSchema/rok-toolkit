<sub>[RoK Toolkit](../../../README.md) › [Docs](../../README.md) › [Headless](../README.md) › [Game Systems](README.md) › Data Gathering</sub>

<img src="../../assets/headers/game-systems.svg" width="100%" alt="">

# Data Gathering

RoK-Data uses a headless account to watch and record, with the same [login](../01-login.md), [wire](../02-wire-protocol.md), and session as a farming account. The Go [RoK-Data store](07-rokdata-store.md) normalizes observations for querying.

**On this page:** [Collection](#collection-is-not-a-special-mode) · [Observations](#what-the-account-observes) · [Rankings wire](#the-rankings-wire-mt-220-229) · [Delivery](#delivery-to-the-store) · [Consumer](#the-consumer)

## Collection is not a special mode

Collection reads the standard initial burst and every frame through disconnect. A gathering account can also farm gems on the same connection. These calls use [...]'s endpoints.

## What the account observes

| Source | Facts |
| --- | --- |
| Login flood | Profile (governor id, name, server, power, kills, kill points, deaths, alliance), city and buildings, marches, resources, commanders |
| MT 203 / 204 city read | Fresh building list, city hall level |
| MT 1004 / 1003 map read | Nearby nodes and map objects in a viewport |
| Rankings MT 220-229 | Ranked rows, previews, own board values |
| Alliance MT 2063 / 2064, 2026 / 2027 | Alliance profile, roster, boards |
| Events MT 1201, 1206, 3304 | Event configuration, progress, cross-kingdom tables |
| Profile reads MT 110 / 111 | Any player's profile, including conduct |

> [!IMPORTANT]
> Unobserved sections on this connection are `known: false`, never zero.

## The rankings wire (MT 220-229)

**MEASURED** · proto schema

**`RankRangeReq`** selects one board page by rank:

```text
f1 RankName (string, required)   the board key
f2 SelfId (int64, optional)      includes own placement
f3 From, f4 To (int64)           the rank window
f5 IdList (repeated int64)       or lookup specific ids
f6 UseCache (bool, optional)
```

**`RankListAck`**: `f1` repeated `RankItem`, `f2` optional own `RankItem`.

**`RankItem`** (one row):

| Field | Type | Meaning |
| --- | --- | --- |
| 1 `Id` | int64 | the subject (player, alliance, kingdom id) |
| 2 `Rank` | int64 | current position |
| 3 `Score` | int64 | the board's metric |
| 4 `Value` | bytes | the subject detail, one `RankData*` per board type |
| 5 `PreRank` | int64 | previous position |

**`RankBriefReq`** requests several boards at once: `f1` repeated `RankNames`, `f2` `SelfId`, optional `From`/`To`/`IdList`/`UseCache`/`WithTotal`. `RankBriefAck` returns repeated `RankBrief`, each with `RankName` (f1), `List` (f6), `SelfItem` (f7), `Total` (f8).

**`RankTopReq` / `RankTopAck`** provide top-of-board previews by `RankNames`.

**`RankTopNtf`** pushes an own-kingdom snapshot: `PlayerPower` f1, `PlayerKill` f2, `TownCenterLevel` f3, `AlliancePower` f4, `AllianceKill` f5, `AllianceFlag` f6, `ResCollect` f7, `Help` f9, `ScenarioTop` f10, `AchieveScore` f11, `PlayerKillScore` f12, `AllianceKillScore` f13.

Board type selects the `RankData*` submessage in `RankItem.Value`:

| Submessage | Key fields |
| --- | --- |
| `RankDataPlayer` | `Name` f1, `Avatar` f2, `AId`/`AName`/`Abbr` f3-f5, `TownCenterLevel` f8, `VipLvl` f9, `OriServerId` f11, `CountryId` f23, `Ext` f29 |
| `RankDataAlliance` | `Name` f1, `Abbr` f2, `Logo` f3, `OwnerId`/`OwnerName` f4/f5, `Power` f12, `TerritoryCnt` f26 |
| `RankDataServerPower` | `ServerId` f1 |
| `RankDataCountry` / `RankDataSide` | kingdom and season-side board payloads |

### Board keys and paging

String keys: `Gs:<raw-server>:<board>` for kingdoms, `As:` prefix for alliances. Displayed kingdom = raw server ID + 1000 (directly verified).

> [!CAUTION]
> The `From`/`To` window addresses rank positions, and a captured `f5` does not establish a reliable previous-rank value: compare two snapshots to measure movement rather than relying on `PreRank`.

Measured board names: `Gs` Power / KillScore / ResCollect / TownCenter / Scenario1 (Expedition) / Achieve; `As` Power / KillScore / Flag. On the 220 request, `f2` is the own governor id for `Gs` boards and the alliance id for `As`; observed `f3=0`, `f4=999`.

### Packed scores

Several boards pack the score and a time tie-break into one integer, with an exact inverse (**MEASURED**): `raw = displayed * 10^10 + (10^10 - unix_seconds_of_last_change)`, so `displayed = raw // 10^10` and `changed_at_s = 10^10 - raw % 10^10`. TownCenter, Scenario1, and Achieve use this form; equal scores rank by the earlier change.

> [!WARNING]
> The Alliance Fort Destroyer board instead uses `10^7` (`forts = raw // 10^7`; raw ordering reproduced both measured score ties), and its remainder is **not** an established timestamp.

Zero-score tail rows sort by a key absent from the message (**UNKNOWN**). Unpack per board.

### Cross-kingdom event joins

MT 3304 fields are distinct: event ID, map number, story bit, roster slots, schedule, ordinal, and tier. Join a profile’s current map/event context through the event table.

> [!WARNING]
> Map number is not event ID, and slot is not team.

## Delivery to the store

- The engine writes what it sees into a durable outbox before anything leaves: events and delivery rows in one atomic batch, disk-synced, delivered strictly in order per lane.
- Batching is 25 dispatch rows or 60 s, 50 log lines or 5 s.
- Retries use full jitter up to `min(5 min, 2 s × 2^n)` on transport errors, 401/408/425/429/5xx; other 4xx drop the row and count it.
- Every row carries an idempotency key so replaying a row has no effect.

## The consumer

The [RoK-Data service](07-rokdata-store.md) normalizes the delivered facts into its four record kinds and serves latest snapshots and history. A rank row off the wire maps to a `leaderboard` record: `event` = the board name, `subject_kind` from the board type, `subject_id` = `RankItem.Id`, `rank`, `score`, `observed_at` from the session clock.

---

<div align="center">

[← Collection and Claims](05-claims.md) &nbsp;·&nbsp; [↑ Game Systems](README.md) &nbsp;·&nbsp; [The RoK-Data Store →](07-rokdata-store.md)

</div>
