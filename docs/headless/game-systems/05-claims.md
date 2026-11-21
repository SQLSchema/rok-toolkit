<sub>[RoK Toolkit](../../../README.md) › [Docs](../../README.md) › [Headless](../README.md) › [Game Systems](README.md) › Collection and Claims</sub>

<img src="../../assets/headers/game-systems.svg" width="100%" alt="">

# Collection and Claims

All field layouts **MEASURED (proto schema)**.

**On this page:** [Quests](#claim_quests-mt-143-questgetrewardreq) · [Production](#collect_production-mt-120-producereapreq) · [Resource refresh](#resource-refresh-mt-207) · [Items](#item-use-mt-100-useitemreq) · [Speedups](#burn_speedups-mt-102-speedupreq) · [Mail](#mail-mt-7920--7921) · [Healing](#healing-mt-330--332--336) · [Tavern](#open_tavern_chest-mt-360-taverndrawreq) · [Caravan and VIP](#caravan--vip-mt-8057--8601--8602-and-the-shop-8603--8605) · [Alliance help](#alliance-help-mt-2045--2048) · [Membership](#alliance-membership-mt-2006-2026-2054) · [Tech donate](#tech-donate-mt-2080-astechdonatereq) · [Gifts](#alliance-gifts-mt-2065--2066-open-2067-daily-supplies-2109) · [Council](#the-council-boards) · [Lucerne Scrolls](#lucerne-scrolls-mt-9803--9801-9804--9805-9802--9808) · [Expedition](#expedition-mt-3026--3028--3032) · [Event shops](#event-shops-mt-9875--9877) · [Courier](#courier-shop-mt-452-refresh-453) · [Anniversary](#anniversary-claim-mt-7000) · [Shield](#shield-mt-100-with-state-on-1006--1002)

## `claim_quests`: MT 143 `QuestGetRewardReq`

`f1 Id` (int32, the quest id). Chest claims are MT 144 with `f1 ChestId` (int32).

## `collect_production`: MT 120 `ProduceReapReq`

| Field | Type | Notes |
| --- | --- | --- |
| 1 `Ids` | repeated string | Building instance ids; batch-collect in one request |

**`ProduceReapAck` (122):** `f1` repeated `ProduceItem` (per-building yield). A peek without collecting is MT 120's sibling `ProducePeekReq` with the same shape.

## Resource refresh: MT 207

Empty body; triggers an MT 121 `ResourceNtf` push with the new balances. The registry’s `f1` building-instance interpretation remains unconfirmed.

## Item use: MT 100 `UseItemReq`

| Field | Type | Notes |
| --- | --- | --- |
| 1 `Id` | int32, required | The item id |
| 2 `Cnt` | int64, required | Quantity |

Confirmed by MT 1; balances arrive as MT 103 `ItemUpdateNtf` (`f1` repeated `ItemInfo`: `f1 Id`, `f2 Cnt`, `f3 Expire`).

## `burn_speedups`: MT 102 `SpeedupReq`

| Field | Type | Notes |
| --- | --- | --- |
| 1 `Type` | enum | 1 Building, 2 Training, 3 Technique, 4 Healing, 5 KVKTechnique |
| 2 `Target` | string | The building/queue instance id |
| 3 `ItemId` | int32 | The speedup item |
| 4 `ItemCnt` | int64 | How many |
| 5 `Items` | repeated `SpeedItem` | Alternative item list form |

## Mail: MT 7920 / 7921

The full field tables are in [reads](01-state-reads.md#mail-mt-7920--7921): `Box`/`Cnt`/`FromTimeStamp`/`Key` paging, `LeftCount` continuation, `MailEntity` with `f20` attachments.

**Claiming (MT 7914):** `Read f3 = 1` before the repeated **string** mail ids `f1`; each attachment's `f20[].f2` carries the used status, which filters already-claimed rewards. The MailEntity id itself is string `f1`.

## Healing: MT 330 / 332 / 336

**`HealingBeginReq` (330):**

| Field | Type | Notes |
| --- | --- | --- |
| 1 `BuildingId` | string, required | The hospital |
| 2 `List` | repeated `Unit` | The wounded to heal: `f1 UnitId`, `f2 Count` |
| 3 `Seckill` | bool, required | Instant finish |
| 4 `LowUnitList` | repeated `Unit` | Separate low-tier batch |
| 5 `UsePiety` | bool, optional | |

**`HealingBeginAck` (331):** `f1` wraps one `HealingStatus`:

| `HealingStatus` field | Meaning |
| --- | --- |
| `f1 BuildingId`, `f3 List`, `f6 LowUnitsList` | Same batches as the request |
| `f4 LeftTime`, `f5 TotalTime` | **Milliseconds** |

MT 337 carries the same status at root and reaches zero before the separate 336 collection. MT 332's schema is a status **list** (`f1[]`); only empty 332 captures were observed (login and post-collection), never as wounds changed.

**`HealingEndReq` (336):** `f1 BuildingId`, collects the healed.

**Wounded snapshot (MT 1234):** the read for wounded-at-home totals (seeded by the map square's `1002.f19`):

| Bucket | Fields |
| --- | --- |
| Regular units `f1[]` | `f1 UnitId`, `f4 BadHurt` (wounded count) |
| Low-tier/siege units `f4[]` | `f1 UnitId`, `f2` wounded count |
| Root | `f5 MaxLowUnitCapacity` (50,000 observed) |

- Buckets are per-unit, not a siege-type rule: measured unit 4 sat in the low bucket with 102 wounded while unit 16 sat regular with 75
- 31 autonomous pushes summed to BadHurt 62, matching the battle mail's 62
- The waiting pool excludes units already accepted into an active heal: 1234 removes them at 330 acceptance, before collection

**Lua runtime healing help** rides the alliance help family: request 2048, server ack 2050.

## `open_tavern_chest`: MT 360 `TavernDrawReq`

| Field | Type | Notes |
| --- | --- | --- |
| 1 `Mode` | `HeroPoolMode` enum | Silver/Gold pool |
| 2 `Type` | enum | 1 Free, 2 Gem, 3 Item |
| 3 `Times` | int32 | Draw count |
| 4 `TargetServerId` | int32, optional | Cross-server pools |
| 5 `LabName` | string, optional | |

**`TavernDrawAck` (361):** `f1 Items`, `f2 OriItems`, `f3 ExgItems` (all repeated `ItemInfo`).

**Tavern readiness (MT 362 / 363):** empty request; `363.f1[]` per pool: `f1 Mode`, `f2 NextFreeTime` (epoch ms), `f3 FreeTimes`, `f4 DailyLeftTimes`, `f5 DailyMaxTimes`, `f6 Version`; `f2` at the root carries the legendary guarantee/selection. Modes: Silver, Gold, 3, 4. Check `FreeTimes` against `DailyLeftTimes`: it may not count remaining draws.

## Caravan / VIP: MT 8057 / 8601 / 8602, and the shop: 8603 / 8605

`TribalCaravanCollectReq` (8057) is an empty body and acks directly. Two independent, empty-body claims:

| Claim | Ack |
| --- | --- |
| MT 8602 daily VIP chest | 8600 (the pushed VIP state) |
| MT 8601 exclusive VIP chest | 103 |

The VIP shop: catalog 8603 → 8604 (`f1` the next-refresh block `{Next, Method, Value}`, `f2[]` items with slot/type/subtype/price/discount/amount/vip-points/weight/price-type at `f1`-`f9`); buy 8605 → 8606 with `SlotId f1`, `Type f2`, `SubType f3`, `Amount f4`, emitted `3,2,1,4`. The ack's `f1` is the updated slot and `Amount` is the remaining quota. Read the balance from the ack.

## Alliance help: MT 2045 / 2048

**`AsHelpDoReq` (2045, Help-All):** schema `f1 Ids` (repeated int64) and `f3 IsAuto`; the 17 captured bodies were exactly `18 00` (f3 IsAuto = false, no f1 list), answering 2046 with List/HelpCnt/MaxHelpCnt-compatible entries. Explicit-id sends and the minimal body are both observed forms.

**`AsHelpRequestReq` (2048):**

| Field | Type | Notes |
| --- | --- | --- |
| 1 `Type` | int32 | The activity kind: 1 building, 3 research, 4 healing |
| 2 `Id` | int64, **encoded as a string** | The activity instance |
| 3 `Level` | int32 | 0 on healing requests |
| 4 `SubType` | int32 | The building type on upgrade help; 0 on healing |

The tool **healing help** rides MT 2050, and the help-progress counter push is MT 8412 (`f1/f2` current/max, `f4/f5` activity type/id).

## Alliance membership: MT 2006, 2026, 2054

`join_alliance` uses 2006 with `f1` the alliance id; the application and actual membership are separate outcomes, confirmed by the 2026/2027 detail read. `claim_alliance_warehouse` uses 2054 with an empty body to harvest territory resources. It is answered by resource pushes on MT 121 (possibly wrapped).

> [!CAUTION]
> A clean response does not confirm collection.

## Tech donate: MT 2080 `AsTechDonateReq`

Send order `4, 1, 2, 3` over the builder's order. Ack MT 2081 returns points, funds, and cooldown (`f1`-range fields). The **skill variant** (`donate_alliance_skill`) sends `Kind f4 = 2` instead of 1, order `4,1,2,3`; its reply's skill node `f8` carries `Type, Level, StartTime, EndTime, ColdTime, EnergyPoint` at `f1`-`f6`. The officer recommendation read (2077/2078) feeds which node: `f1[]` tech nodes `{Type, Level}`, `f2` Recommend, `f6[]` skills, `f7` RecommendType (1 tech, 2 skill).

## Alliance gifts: MT 2065 / 2066, open 2067, daily supplies 2109

**List (2065, empty) → 2066:** `f1` repeated `GiftInfo`, `f3[]` member-gift records; each gift id at `f1`, reward fields `f4`-`f9` (already-opened history distinguishes from openable), rare markers `f11/f12`.

**Open (2067):** repeated `Ids f2` (**unpacked**), no AllianceId; ack 2068's nodes carry `f1` id, `f2` GiftId, `f5/f6` ItemId/count, `f8` points; `f3[]` lists expired ids. The same 2068 shape arrives unsolicited when gifts expire.

**Daily supplies (2109 → 2110 then MT 103):** `SupplyId f1`, `10000001` gems / `70000001` materials; one request per supply.

Refusals worth keeping:

- healing begin's instant-finish is `Seckill`
- the mail page is resumable via `LeftCount`
- quest chest claims (144) are separate from reward claims (143)

Healing order is 330 → 1234 → 121 → 331: the wounded snapshot and the absolute resource push land between request and ack. A measured one-unit heal cost 22 food + 22 wood on that account. Healing-help availability rides 2047 (`f1` flag, `f2` count, `f3` unresolved, `f4` id; binding inferred).

## The council boards

**Reads (5160/5161, 4930/4931, 5176/5177):** the first three requests are empty. The board detail `4931.f2`:

| `4931.f2` field | Meaning |
| --- | --- |
| `f2[]` quests | config/id/heroes/conditions/state/start/duration at `f1`-`f7` |
| `f3` / `f4` | dispatch max / done |
| `f5` / `f6` | gift threshold / points (threshold 15 measured; GiftPoint moved 14→15→0→7 across claims and resets, unchanged by dispatch) |

Travel state arrives on the 5161 reply:

| 5161 field | Meaning |
| --- | --- |
| `f2` | partner hero id |
| `f3` | travel time used |
| `f6` | gem-travel time used |
| `f7` | travel time max (20 observed) |
| `f8` | unlocked (1) |
| `f9` | luck number |

The 5177 reply is four unresolved council-window scalars (`{1,1,0,0}` observed), not travel state.

**Claims (4934 → 4935):** `f1` the quest id, **-1 the milestone chest** (the signed sentinel).

> [!WARNING]
> A normal claim's 4935 can return different or multiple quest ids, so read the result ids from the ack.

The 4935 loot `f3[]` carries armament books:

| Book field | Meaning |
| --- | --- |
| `f1` / `f2` | string instance id / HeroId |
| `f3[]` buffs | `{f1 type, f2 little-endian fixed64 double}` |
| `f4[]` / `f7[]` | affix ids / buff ids |
| `f5` / `f6` | lock / selected |
| `f8`-`f10` | attribute locks |
| `f11`-`f13` | rerandom count / guarantees |

> [!IMPORTANT]
> MT 4921 independently delivers the same instances; the two are one grant, not two.

The dispatch and travel actions are in [city](04-city.md#council-dispatch-and-travel-mt-4032-5162--5163-5164--5165).

## Lucerne Scrolls: MT 9803 / 9801, 9804 / 9805, 9802 / 9808

Snapshot 9803 (empty) → 9801:

| 9801 field | Meaning |
| --- | --- |
| `f1[]` | tier level and claimed entries |
| `f2` | level/XP (`f2.f1` level, `f2.f2` XP) |
| `f3` | season/end |
| `f4` / `f5` | weekly / season tasks |
| `f6[]` | XP-per-level threshold (`f1="Lv_experience"`, `f2` the threshold; 100 observed) |
| `f9` | MaxLootLevel (0 throughout, not a claimed watermark) |

Task state records arrive on 9805 (`f1[]`: TaskID `f1`, status `f2`, group `f3`, type `f4`, bonus XP `f5`), during ordinary progress as well as after claims; statuses are signed, -1 claimable and -2 claimed. A task's identity is the triple **type + task id + group id**.

Task claim 9804 → 9805: `f1` type, `f2` task id, `f3` group. XP awarded = `(newLevel - oldLevel) * Lv_experience + newExp - oldExp`; there is no direct XP scalar in 9804/9805 (measured: 1/90 → 2/40 is 50 XP, and sixteen claims took 1/0 → 11/50 = 1,050). Reset and max-level behavior are **UNKNOWN**.

Tier claim-all 9802 (`BattlePassGetLevelRewardReq`): `f1=0, f2=0` claimed all free-track tiers in one request each time measured.

> [!WARNING]
> The 9808 reply is **cumulative** claimed state (`f1[]`: level, free flag, paid flag; loot definitions live in the initial 9801 `f1[]`, keyed by level, with `f4 FreeLoot`): difference it against known claimed state, never count its entries as new rewards; claim-alls change no XP and emit no 9801.

Measured challenge awards (scoped):

| Type / id / group | XP |
| --- | --- |
| (1,1007,3) | 30 |
| (1,2013,3) | 50 |
| (1,3022,1) | 100 |
| (2,1003,0) | 400 |

## Expedition: MT 3026 / 3028 / 3032

**Daily chest (3026 → 3027):** `f1=0, f2=1`; a successful 3027 carries nested loot and **no status scalar**. The loot confirms success.

**Shop (3028 → 3029):** empty request; `f5[]` slots carry item id, count, price, limit, bought, slot id, limit count at `f1`-`f7`.

**Buy (3032):** send the **slot id** in `f1` (2 in the measured send). Each request makes one purchase and debits resource type 7 (Aethelflaed coins). The engine checks item 10990 in MT 103 to confirm.

Measured reconciliation:

| Fact | Value |
| --- | --- |
| Slot 2 | item 10990, price 1500, limit 1, limit count 3 |
| Three buys | each debited 1500 of resource 7 (4,500 total) |
| Each ack | MT 1 `{3032, 1}` |

## Event shops: MT 9875 / 9877

Catalog 9875: `f1` tier, `f6` **cumulative tier spend** tracked per account, `f7` the shop block; `f7.f2[]` entries carry id `f1`, goods `f2`, cost `f3`, stock `f4`, purchased `f5`, order `f7`; the tier threshold is `f7.f3` (root `f8`) and the currency `f9`. Measured thresholds: 315000 / 630000 / 99999999; one item offered at 130000 in one tier and 110000 in another (identity is tier-scoped). Buy 9877: `f2` the entry id, `f3` the quantity. A measured 2300-coin purchase reconciled exactly against `f6` and the coin balance.

## Courier shop: MT 452, refresh 453

Buying uses a **slot index**: `f1` is the slot id.

> [!WARNING]
> The item in a slot changes after a refresh, so enumerate (450/451) immediately before buying.

Refresh 453 is empty and answers 451. Catalog fields: refresh used/left/total `f3/f4/f5`, expiry `f6` in **milliseconds**, `f8` the refresh gem cost; goods `f7[]` carry id/category/reward/discount/cost-type/stock/list-price/**actual price** at `f1`-`f8`. Cost type is 5 gems or 1/2 food/wood; captured slot ids span 0-15. Use the discounted price for buy decisions. Refresh counters determine free eligibility even if price is zero.

## Anniversary claim: MT 7000

The event claim body carries an **ASCII letter id**, an additional selector, and the raw server id, all emitted non-numerically. Reply records distinguish currency from item rewards.

## Shield: MT 100 with state on 1006 / 1002

Activating a shield is ordinary item use; the state reads back elsewhere: **MT 1006** push carries `f1 CharId`, end `f2`, item window `f4/f5`, system window `f6/f7`; the world-map snapshot fallback reads `1002.f1` item end `f16`, system end `f26`, start `f29`, or the embedded shield at `1002.f16`.

---

**Related:** [Invoking the Client](../../lua-runtime/03-client-calls.md).

<div align="center">

[← City Development](04-city.md) &nbsp;·&nbsp; [↑ Game Systems](README.md) &nbsp;·&nbsp; [Data Gathering →](06-data-gathering.md)

</div>
