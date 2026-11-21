<sub>[RoK Toolkit](../../../README.md) › [Docs](../../README.md) › [Headless](../README.md) › [Game Systems](README.md) › State Reads</sub>

<img src="../../assets/headers/game-systems.svg" width="100%" alt="">

# State Reads

Explicit post-login queries refresh state without changing it.

**On this page:** [World map](#world-map-snapshot-mt-1001--1002) · [City](#city-and-buildings-mt-203--204) · [Profile](#player-profile-mt-110--111) · [Mail](#mail-mt-7920--7921) · [Quests](#quests-mt-140--141) · [Map view](#map-view-mt-1004--1003) · [Bytes KV](#bytes-kv-mt-9820--9821) · [Alliance discovery](#alliance-discovery-mt-2023--2024-2017--2018) · [Bag](#bag-mt-104--105) · [Canyon](#canyon-panels-9606--9607-9608--9609-9613--9614) · [Rankings](#ranking-and-leaderboard-data)

| Read | MT pair | What it returns | Fields |
| --- | --- | --- | --- |
| City and buildings | 203 / 204 | Building records, levels, the march-slot floor | below |
| Hero roster | 366 / 367 | Owned commanders with level, state, gear count | [marches](02-marches.md) |
| Resource node search | 1176 / 1177 | Nodes with position, id, remaining/max yield | [marches](02-marches.md) |
| Barbarian search | 1178 / 1179 | Closest-first targets with ids | [marches](02-marches.md) |
| Map view | 1004 / 1003 | Map objects in a viewport | below |
| Player profile | 110 / 111 | Any player's profile, including conduct | below |
| Alliance detail | 2026 / 2027 | Roster, boards, alliance state | registry |
| Alliance discovery | 2023 / 2024, 2017 / 2018 | Open alliances by language; search by abbreviation | below |
| Training queues | 305 / 302 | Per-barracks training status | [city](04-city.md) |
| Bag | 104 / 105 | Inventory entries `{Id, Cnt}` | below |
| Tavern schedule | 362 / 363 | Pool modes, free-draw counters, next-free times | [claims](05-claims.md) |
| Canyon panels | 9606 / 9607, 9608 / 9609, 9613 / 9614 | Sunset/Lost Canyon rank, opponents, history | below |
| Rally list | 1101 / 1102 | Active rallies with captains and targets | [forts](03-forts.md) |
| Mail page | 7920 / 7921 | Mail records, resumable by `LeftCount` | below |
| Quest list | 140 / 141 | Quest progress records | below |
| Courier catalog | 450 / 451 | Station offers | registry |
| Bytes KV | 9820 / 9821 | Saved client blobs, including troop loadouts | below |

> [!IMPORTANT]
> Reading rules:
>
> - A value not yet received is unknown, never zero.
> - Pushes (MT 1005 march states, MT 103 item updates, MT 121 resource changes, MT 211 building changes) update the view asynchronously.
> - A read is the only proof of current state.
> - Freshness is connection-scoped: a disconnect keeps remembered values but makes march and slot evidence stale.
> - `Known` and `as-of` are per-domain, and reading a local snapshot is not a new game query.

## World map snapshot (MT 1001 / 1002)

The login flood's `EnterWorldMapReq` and its ack carry the map's foundation. The 1002 ack:

- `f1 Castle → f1 Char → f1 Id` (the castle's map **CharId**, in a different namespace from the governor PlayerId)
- `f3` the castle `Pos.{f1 X, f2 Y}`
- `f14 NpcLevelMax` the barbarian level gate
- `f19` the home army (`CastleDetailTroop → f1[]` units: `f1 UnitId`, `f2 total`, `f5 unavailable/hurt`; idle = total − unavailable)

> [!WARNING]
> A missing unit list must not overwrite a previously known roster with zero troops.

## City and buildings (MT 203 / 204)

**MEASURED** · proto schema

**`GetBuildingListReq` (203):**

| Field | Type | Notes |
| --- | --- | --- |
| 1 `PlayerId` | int64, required | The governor whose city |
| 2 `ViewerId` | int64, optional | Set when reading another player's city |

**`BuildingListInfo` (204):**

| Field | Type | Meaning |
| --- | --- | --- |
| 1 `List` | repeated `BuildingInfo` | The buildings |
| 2 `PlayerId` | int64 | Echo of the requesting governor |
| 3 `MaxProgress` | int32 | City progress |
| 4 `TownCenterUpgrade` | bool, optional | TC upgrade state |
| 5 `MaxTier` | `UnitMaxTier`, optional | Unlocked unit tiers per type |
| 6 `VipLvl` / 7 `VipShow` | int32 / bool | VIP display |
| 8 `UnlockBuildings` | repeated `UnlockBuilding` | Per-type unlock counts |
| 9 `IsUnlockKVK4` | bool | Season gate |
| 10 `LastCityReviewTime` | int64 | |
| 11 `FavoriteBuildings` | repeated `BuildingFav` | |
| 12 `BuildingSchema` | int32 | Schema version |

**`BuildingInfo`** (each entry in f1):

| Field | Type | Meaning |
| --- | --- | --- |
| 1 `Id` | string | The building **instance id** (the string MT 205 takes to upgrade it) |
| 2 `Type` | `BuildingType` enum | 1-77 core types; 200+ seasonal decorations |
| 3 `Pos` | `BuildingPos` | Grid position |
| 4 `Level` | int32 | |
| 5 `State` | `BuildingState` enum | Idle, upgrading, etc. |
| 6 `EndTime` | int64, optional | Unix time when the current job finishes |
| 7 `CostTime` | int64, optional | Total job duration |
| 8 `SkinId` | int32, optional | |

Key `BuildingType` values: 1 TownCenter, 2 Farm, 3 Sawmill, 4 Quarry, 5 SilverMine, 6 CityWall, 7 BuilderHut, 9 Barracks, 10 Stable, 11 ArcheryRange, 12 SiegeWorkshop, 13 Campus, 14 Hospital, 15 Storage, 16 AllianceCenter, 17 BuildingCastle, 18 Tavern, 21 CourierStation, 28 ScoutCamp, 31 Smithy, 42 Library. Obstacles: 22 Tree, 23 Rock. The full 1-77 enum plus seasonal 200+ exists in the schema.

For [march slots](02-marches.md), City Hall level is `Level` on `Type = 1`.

## Player profile (MT 110 / 111)

**MEASURED** · proto schema

**`PlayerInfoReq` (110):** `f1 PlayerId` (int64), `f2 GsId` (int32, the kingdom), `f3 ViewerId` (int64).

**`PlayerInfoAck` (111):**

| Field | Type | Meaning |
| --- | --- | --- |
| 1 `Player` | `PlayerBasic` | The profile itself |
| 2 `Alliance` | `AllianceBasic`, optional | Membership |
| 3 `Power` | `PlayerInfoPower` | `f1` troop, `f2` technology, `f3` building, `f4` commander; the four sum to `Player.f5` exactly (measured to the unit) |
| 4 `Fight` | `PlayerInfoFight` | `f1` kills, `f2` dead, `f3` unnamed counter (semantics **UNKNOWN**), `f4` defeats, `f5` victories, `f6` highest power, `f8` scout times, `f9[]` tier-kills, `f10[]` tier kill-points, `f11` total kill points. Tier points use weights T1..T5 = 0.2/2/4/10/20; the tier sum equals `f11` |
| 5 `Res` | `PlayerInfoRes` | `f5.f1` gathered resources, `f5.f2` resource assistance, `f5.f3` alliance-help times |
| 6 `Units` | repeated `Unit` | Home troops |
| 10 `Country` / 11 `Province` | strings | Origin |
| 12-14 | submessages | KvK, 5v5, alliance-battle achievements |
| 17 `Conduct` | int32 | The 0-100 conduct score, on every profile |

`PlayerBasic`: `f1 PlayerId`, `f2 Name`, `f3 Avatar`, `f4 ServerId` (**current map context**, which becomes a KvK event id; not the home kingdom), `f5 Power`, `f6 Kill`, `f7 TownCenterLevel`, `f8 Civilization` (verified: 4 France, 12 Vikings; other observed ids 1-14 unmeasured), `f9 VipLevel`, `f10 OriServerId` (**home kingdom**), `f11 KillScore`, `f14 AllianceName`, `f15 AllianceAbbr`, `f17 KvkStatus`.

`AllianceBasic`: `f1 Id`, `f2 Name`, `f3 Abbr`, `f4 Logo`, `f5 Lang`, `f6 Owner`, `f7 OwnerName`, `f8 Power`.

**Conduct status (MT 2600 → 2601).** Measured eight-byte reply: `f1` (unresolved), `f2` the 0-100 score, `f3` (unresolved), `f5` reward availability (1 claimable, 0 claimed). Claiming the monthly reward (2603 → 2604, plus the MT 103 inventory push) changes only `f5` from 1 to 0; the measured receipt carried item/quantity pairs 7996x68, 149x20, 37x590, 28x9 (scoped to that receipt).

## Mail (MT 7920 / 7921)

**MEASURED** · proto schema

**`MailGetReq` (7920):**

| Field | Type | Meaning |
| --- | --- | --- |
| 1 `Box` | string | The mailbox (rewards, reports, system...) |
| 2 `Cnt` | int32 | Page size |
| 3 `FromTimeStamp` | int64 | Pagination cursor: newest-first from this time |
| 4 `Key` | string | Continuation key from the previous page |

**`MailGetAck` (7921):** `f1` repeated `MailEntity`, `f2` `LeftCount` (int32). Page while `LeftCount` is nonzero.

`MailEntity` (abbreviated): `f1`-ish id fields, `f8 Box`, `f9 Type`, `f10 Timestamp`, `f20` repeated `MailAttachment Attachments` (what a claim collects), `f21 PrevBox`, `f31 ExpireTime`, `f35 GlobalMailId`, `f36 BatchId`, `f37 AutoReward`, `f44 IsReply`, `f53` repeated `MailReportAttack AttacksBody` (combat reports).

## Quests (MT 140 / 141)

**MEASURED** · proto schema

**`QuestListReq` (140):** empty body.

**`QuestListNtf` (141):**

| Field | Type | Meaning |
| --- | --- | --- |
| 1 `DailyQuests` | repeated `Quest` | |
| 2 `NormalQuests` | repeated `Quest` | |
| 3 `CompletedQuests` | repeated int32 | Claimed ids |
| 5 `Chests` | repeated `QuestChest` | Score chests (claimed via MT 144) |
| 6 `QuestScores` | int32 | Current score |
| 7 `NextRefreshTs` | int64 | Daily reset |
| 9 `KvkQuests`, 13 `ReflowQuests`, 14 `StrategicAchievedQuests` | repeated `Quest` | |

`Quest`: `f1 Id` (int32), `f2 Type` enum, `f3 Stat` enum (0 locked / 1 underway / 2 claimable / 3 complete), `f4 CurrentParam` (int64), `f5 New` (bool). Claim via MT 143 when claimable.

## Map view (MT 1004 / 1003)

**MEASURED** · proto schema

The 1002 snapshot carries the map's own geometry:

- dimensions `f5/f6` (7200/7200 measured)
- grid `f7/f8` (36/36)
- gridV3 `f9/f10` (720/720)
- territory grid `f11/f12` (18/18)
- territory radius `f13` (4)

Owner lookup: `floor(world / 18)` matched 87/87 measured owners (the neighboring rules scored 71/79); alliance flags occupy 3x3 cells centered on the floor cell (`1003.f6[].f6.f3/f7/f8` = radius 1, width 3, height 3).

**`UpdateViewReq` (1004):**

| Field | Type | Notes |
| --- | --- | --- |
| 1 `Pos` | `PosInfo`, required | Viewport center (world coordinates, 6 units per tile) |
| 2 `Level` | int32, optional | Zoom level; the gem sweep sends 0 |
| 3 `AgentId` | int32, optional | |
| 4 `IsGuest` | bool, optional | |
| 5 `RenderMode` | enum, optional | Captured PC value 1; an older builder used 2 |

**`WorldMapNtf` (1003):** map objects within the view, each family in its own field:

| Family | Field | Shape |
| --- | --- | --- |
| Castles | `f1` | |
| Squares (forts, camps) | `f2` | `MapSquare` |
| Resource nodes | `f5` | `MapRss` |
| Territory | `f17[]` | `MapAsTerritory` |

`MapAsTerritory`: `f1 AllianceId`, `f2 Color`, `f3 Abbr`, `f4[]` cells with integer `f1.f1/f1.f2` X/Y, `f4 Active`, optional `f5 Disable`/`f6 IsSys`; only active cells were measured.

`MapSquare`: `f1 Char`, `f2 PlayerId` (-2 marks an NPC-held square), `f13 NpcInfo` (barbarian or fort type and level), `f16 IsFighting`; wounded-at-home totals ride `f19`.

A gem sweep reads `1003.f5[]` with `ResType = 5`: node `f1 Char`, `f2 ResType`, `f3 Level`, doubles `f4 Max` / `f5 Remain` (reinterpret bits), occupying troops `f8[].f1.f1 TroopId`, `f15 EndTime`. Either the troop list or the end time can indicate occupancy.

Territory also rides the resource nodes: `MapRss.f11 OwnerAs.f1` is the owning alliance id, **distinct from the object's own alliance tag**; a missing `OwnerAs` on a complete, valid node means neutral, and malformed owner data is unresolved, never neutral.

> [!IMPORTANT]
> A map sighting is neither a full map nor proof the object remains.

## Bytes KV (MT 9820 / 9821)

**MEASURED** · proto schema

`BytesKvGet`: `f1 Key` (string). `BytesKvData`: `f1 Key`, `f2 value` (bytes). Troop loadouts are stored under the key `"TroopProfile"`: a repeated `TroopProfile` with `f1 Hero1`, `f2 Hero2`, `f3 Units`, `f4 Id`, `f7 Tab` (the color group: blue 1, red 2, yellow/green 3), `f8 Slot`, `f9 Tactics`. Canyon saved lineups use a different transport (9615/9616, 8035/8036) with composite keys `"<AsyncPVPAttack|AsyncPVPDefence|KVKPVPAttack|KVKPVPDefence>__<player_id>"`, separate from this generic KV.

## Alliance discovery (MT 2023 / 2024, 2017 / 2018)

**`ListAllianceReq` (2023):** `f1 Lang`, `f2 Open = 1`. **Search by abbreviation (2017):** `f1 Abbr` (string). Both acks share the record shape, repeated `f1`: `f1 Id`, `f2 Name`, `f3 Abbr`, `f8 Open`, `f10/f11` member count/max, `f12` applied, `f13` list power.

## Bag (MT 104 / 105)

Empty request. `105.f1[]` contains inventory counts `{f1 Id, f2 Cnt}`. The [courier catalog](../04-message-registry.md) (450/451) uses a separate protocol and lists items for sale.

## Canyon panels (9606 / 9607, 9608 / 9609, 9613 / 9614)

Sunset Canyon and Lost Canyon reads: one protocol, two events.

| Pair | Purpose |
| --- | --- |
| 9606 → 9607 | event state |
| 9608 → 9609 | leaderboard |
| 9613 → 9614 | battle replay |

Picking an opponent is 9615 → 9616 and setting a lineup is 9612, neither part of the read set. On 9606, `f1` selects the event: 0 Canyon, 1 Lost Canyon. Ticket counters live on the gate state 9605: Canyon attempts and cap at `f1`/`f4`, Lost Canyon at `f7.f1`/`f7.f3`; the root `f1` can read 0 while Lost Canyon still has attempts. The Lost Canyon shop request is empty. The opponent response lists `f1[]` with a robot flag `f2`.

## Ranking and leaderboard data

In-game, event boards and profile reads (MT 110/111, MT 1206 event pushes) carry rank and score for observed players and events on the wire. See [data gathering](06-data-gathering.md) for the MT 220-229 field breakdown. Out-of-game, the [RoK-Data store](07-rokdata-store.md) normalizes public game statistics into a timestamped record store with latest-snapshot routes; history is replay-only from the append log, with no query route.

---

<div align="center">

[↑ Game Systems](README.md) &nbsp;·&nbsp; [Gathering and Marches →](02-marches.md)

</div>
