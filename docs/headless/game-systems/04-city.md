<sub>[RoK Toolkit](../../../README.md) › [Docs](../../README.md) › [Headless](../README.md) › [Game Systems](README.md) › City Development</sub>

<img src="../../assets/headers/game-systems.svg" width="100%" alt="">

# City Development

**MEASURED (proto schema)** throughout.

**On this page:** [Upgrades](#try_upgrade-mt-205-upgradebuildingreq) · [Placement](#create_building--place_all_buildings-mt-201) · [Research](#start_research-mt-252-techniqueupgradereq) · [Training](#train_troops-mt-300-trainingbeginreq) · [Builders](#use_builder_recruitment-mt-200-and-the-roster-mt-219) · [Autoscout](#set_autoscout-mt-1621--1622) · [Commanders](#commander-development-mt-391-378-375-395) · [Materials](#material-production-mt-9750--9752--9755) · [Council](#council-dispatch-and-travel-mt-4032-5162--5163-5164--5165) · [Canyon](#canyon-fights-mt-9610--9611-9612)

## `try_upgrade`: MT 205 `UpgradeBuildingReq`

**MEASURED** · proto schema

| Field | Type | Notes |
| --- | --- | --- |
| 1 `Id` | string, required | Building instance ID from MT 204 |
| 2 `Seckill` | bool, required | Instant-finish flag; captured sends are false |
| 3 `Discount` | bool, optional | |

**`UpgradeBuildingAck` (206):** carries the updated `BuildingInfo`, the shared eight-field record: `f1` instance id (string), `f2` type, `f3` position, `f4` level, `f5` state (2 working, 1 idle, 3 just placed), `f6` finish epoch millis, `f7` total job millis; `f6`/`f7` are absent when `f5` is 1.

> [!WARNING]
> The ack often arrives in MT 9999 with MT 1 not first. Unwrap every submessage to avoid false missing-ack reports.

On success an `AsHelpRequestReq` (alliance help, MT 2048) follows, carrying the building type as `SubType` f4.

- `try_free_speedup` (MT 215 `FreeSpeedupReq`): `f1` the string building instance id; **MT 216** is its selected acknowledgement.
- `try_use_speedup_item` (MT 102 `SpeedupReq`): see the table in [claims](05-claims.md).

## `create_building` / `place_all_buildings`: MT 201

`f1` = building type, `f2` = city-grid coordinate pair, `f3=0`. The `Pos` submessage encodes `f1=X`, `f2=Y` as **signed integer varints**. World positions elsewhere on the wire use float32. MT 202 returns the record with its assigned instance id.

`remove_obstacles` is MT 213 (instance id, the encoded `(-99,-99)` pair); the removal ack type is read from `214.f2.f2`.

## `start_research`: MT 252 `TechniqueUpgradeReq`

**MEASURED** · proto schema

```text
f1 = tech id (string bytes), f2 = Seckill 0, f3 = Discount 0
```

- `cancel_research` (MT 254, `f1` = id) refunds resources.
- `end_research` (MT 259, `f1` = id): research requires an explicit claim, unlike building upgrades which auto-finalize.
- `get_techniques` (MT 250, empty body) returns state via MT 251 `Technique` (`f1` repeated `TechniqueNode`: `f1 Id`, `f2 Type`, `f3 Level`, `f4 TotalTime`, `f5 LeftTime`, `f6 StartTime`) and MT 258 node pushes.

## `train_troops`: MT 300 `TrainingBeginReq`

**MEASURED** · proto schema

| Field | Type | Notes |
| --- | --- | --- |
| 1 `BuildingId` | string, required | Barracks/Stable/ArcheryRange/SiegeWorkshop instance id |
| 2 `UnitId` | int32, required | Must match the building: 1 Infantry@Barracks, 2 Cavalry@Stable, ... |
| 3 `UnitCount` | int64, required | |
| 4 `Seckill` | bool, required | Instant finish |
| 5 `Discount` | bool, optional | |

**`TrainingBeginAck` (301):** `f1` wraps **one** accepted `TrainingStatus`, limited to the queue just started.

**MT 302** carries the repeated queue records: `f1 BuildingId` (string), `f2 UnitId`, `f3 UnitCount`, `f4 LeftTime`, `f5 TotalTime`, optional source unit.

**`check_training_status` (empty MT 305, reply MT 302):** the explicit training-state query.

> [!IMPORTANT]
> Without a 302 reply, queue state remains unknown.

`collect_training` is MT 303 `TrainingEndReq` (`f1 buildingId`). The server clamps capacity.

## `use_builder_recruitment`: MT 200, and the roster: MT 219

The second builder has two payment forms.

- Item-funded: `f1` `"2"` (the builder slot), `f2=0`, `f3` an item block `{Id: 55, Cnt: 1}`; measured emission order `2, 3, 1`.
- Gem-funded: `f1` `"2"`, `f2=1`, no item block. Schema-defined, untested.

> [!CAUTION]
> A generic success does not prove the slot: verify an inventory decrease (MT 103/104) or resource deduction (MT 121).

**MT 219** is the builder-roster push, a **full replacement** of the slot list. Each slot includes its source and expiry.

## `set_autoscout`: MT 1621 / 1622

On: `Switch f1 = 1`. Off: empty body (absent switch). The reply parses `f1` switch state, `f2 max_time`, `f3 left_time`.

## Commander development: MT 391, 378, 375, 395

All take the commander id as `f1`.

- **`add_hero_exp` (391):** `f2` item id, `f3` count. This is the XP-item path, separate from generic MT 100 item use.
- **`hero_star_up` (378):** `f2` repeated sculpture ids, **unpacked** (one field occurrence per consumed piece). Implemented sculpture mappings: Advanced → 9996, Elite → 9997.
- **`add_talent_rank` (375) / `auto_fill_talent` (395):** `f2` talent id, `f3` page index 0. The roster (MT 367) carries the inputs: `f12 TalentPoints`, `f10` repeated pages, each page `f1 PageIdx` and `f3` talents `{Id, Level}`.

## Material production: MT 9750 / 9752 / 9755

Smithy materials are **items**: MT 103 updates balances, not MT 121.

| Operation | Request | Ack |
| --- | --- | --- |
| Query (pollable) | 9750 `{f1 building}` | 9751 slots: `{id, state, material, param, total-ms, remaining-ms}` at `f1`-`f6` |
| Queue an order | 9752 `{f1 building, f2 {slot, material, param, 0}}` | untyped |
| Claim | 9755 `{f1 building}` | 9756 `{item id, produced qty}` + MT 103 |

Observed material ids: 412 Leather, 413 Iron Ore, 414 Animal Bone, 417 Ebony. Request batch size equals current building output. Read it from 9751, not a constant.

## Council dispatch and travel: MT 4032, 5162 / 5163, 5164 / 5165

Assigning council quests: **MT 4032** with repeated `f1` assignments `{f3 cfg_id, f1 quest_id, f2 hero_id}` and outer `f2 = 1` (the one-click flag). Every slot is explicit, and server auto-fill was never demonstrated.

> [!CAUTION]
> Confirmation requires an echoed API 4032 inside an MT 1 with `Code = 1`. A plain MT 1 is insufficient.

**Travel:** partner selection sends `hero_id f1` (5162 → 5163); the travel batch sends `f1 = 10, f3 = 0` (5164 → 5165, ten outcomes). The reads side is in [claims](05-claims.md#the-council-boards).

## Canyon fights: MT 9610 / 9611, 9612

**Fight (9610):** `f1` target player, `f2` repeated slots `{f1 army, f2 position}`, variant flags `f3/f4`, Lost Canyon target rank `f5`. Each army: `f1[]` heroes `{id, label, level, awakened, formation}` at `f1`-`f5`, `f2` unit, `f3 = -1`.

**Defence (9612):** `f1[]` slots, `f2` event; acks on MT 1.

The 9611 result parses win `f12`, code `f13`, rank `f3`, signed rank delta `f5`. The panel reads are in [reads](01-state-reads.md#canyon-panels-9606--9607-9608--9609-9613--9614).

---

**Related:** [Invoking the Client](../../lua-runtime/03-client-calls.md).

<div align="center">

[← Fort Operations](03-forts.md) &nbsp;·&nbsp; [↑ Game Systems](README.md) &nbsp;·&nbsp; [Collection and Claims →](05-claims.md)

</div>
