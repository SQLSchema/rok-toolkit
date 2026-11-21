<sub>[RoK Toolkit](../../README.md) › [Docs](../README.md) › [Lua Runtime](README.md) › Reading the Client</sub>

<img src="../assets/headers/lua-runtime.svg" width="100%" alt="">

# Reading the Client

Read forms, literal table paths by domain, and configuration install files.

**On this page:** [Hard facts](#the-two-hard-facts) · [Read forms](#literal-read-forms) · [Governor profile](#governor-profile) · [Troops](#troops-and-commanders) · [City](#city) · [Bag and equipment](#bag-equipment-and-armaments) · [Alliance and chat](#alliance-mail-and-chat) · [Map](#map-and-kingdom-overview) · [Install files](#install-files) · [Failures](#failures-and-proof) · [Reload hold](#the-reload-hold) · [Gaps](#gaps)

Evidence labels:

- **MEASURED** = watched returning on a live client or observed on the wire
- **CODE** = read from the client's own code, not watched live
- **Enabled** = enabled for testing without a watched live return
- **UNKNOWN** = no evidence, never invented

## The two hard facts

> [!CAUTION]
> | Fact | Evidence | Rule |
> | --- | --- | --- |
> | Client Lua errors terminate the game, even in `pcall`. | **MEASURED** | No speculative calls or reads that can raise |
> | Lua 5.1 cannot check arity. Missing required arguments can terminate the client. | **MEASURED**, two live crashes | No new client getter calls in reads |

## Literal read forms

| Operation | Form | Result or limit |
| --- | --- | --- |
| Reach a global | `rawget(_G, "UserData")` | Check `type(...) == "table"` before reading its fields |
| Read a field | `rawget(t, "PlayerID")` | Only after `type(t) == "table"`; no `__index` |
| Walk a keyed table | `next(t, key)` | Only after a table check and `getmetatable(t) == nil`; impose an entry cap |
| Walk a list | `for i = 1, 64 do local row = rawget(t, i) if row == nil then break end end` | Queues, governors, home troops: at most 64 entries |
| Inspect outstanding requests | `rawget(rawget(t, i), "api")` | Check both tables first; `NetMessageMgr.callbacks` by index from 1 to 512 |
| Decode a default | `local n = rawget(row, "Pos"); if n == nil then n = 0 end` | Decoded protobuf messages only: `nil` means the field's declared default, not missing evidence |
| Compare client tables | `rawequal(a, b)` | Identity only; no `__eq` |
| Read configuration | `rawget(GameDatas, "x")` | **MEASURED:** `nil` on PC, because `GameDatas` is a lazy proxy; read `exceldata.bin` instead |

```lua
local function field(t, key)
  if type(t) ~= "table" then return nil end
  return rawget(t, key)
end

local user = field(_G, "UserData")
local governorId = field(user, "PlayerID")
local world = field(_G, "WorldMapData")
local marches = field(field(world, "MyChars"), "AllTroops")
local items = field(field(_G, "ItemData"), "MyItemList")
local item = field(items, 112)
local count = field(item, "itemCount")
```

```lua
local roles = field(field(_G, "AccountServerRoleData"), "RolesList")
local ids = {}
if type(roles) == "table" then
  for i = 1, 64 do
    local row = rawget(roles, i)
    local id = field(row, "PlayerId")
    if type(id) == "number" then ids[i] = id end
  end
end

local states = {}
if type(marches) == "table" and getmetatable(marches) == nil then
  local key = nil
  for i = 1, 64 do
    local row
    key, row = next(marches, key)
    if key == nil then break end
    states[i] = field(row, "state")
  end
end

local callbacks = field(field(_G, "NetMessageMgr"), "callbacks")
local apis = {}
if type(callbacks) == "table" then
  for i = 1, 512 do
    apis[i] = field(rawget(callbacks, i), "api")
  end
end
```

| Read boundary | Rule |
| --- | --- |
| Examples above | `field` is the local raw-read helper, not a client function; result tables are new local tables |
| Missing or wrong-shaped table | **UNKNOWN**, not zero, empty, completed or claimable |
| Decoded message defaults | Apply the declared default only after verifying the containing message table and field meaning |
| Ordinary state tables | Do not replace an absent field with zero merely because protobuf messages have defaults |
| Client objects | Raw-read verified Lua table fields only; no `#`, `pairs`, `ipairs`, `tostring` or engine-userdata field probes |
| Configuration objects | Engine-backed rows are not plain tables; no process-side configuration lookup or fallback through a getter |
| Execution | Lua chunk on the game's own thread through `lua_gettop`; batch related values in one chunk |
| Side effects | No server request, client getter, state mutation or window opening |
| Older getters | Only previously watched zero-argument getters; no new getter added; known crash-causing calls remain forbidden |
| Validation | Missing keys, wrong types and hostile metatables tested under Lua 5.1 and LuaJIT 2.1 before a live probe |
| Times | `GameTimer.frameServerTime` in epoch ms. Buff and event expiry: seconds unless noted |
| Dungeon and battlefield sessions | `DungeonMgr.curProcessSession` selects session-local state; home-city buffs, shield and AP are not substitutes |

## Governor Profile

| Domain | Read path | Values | Evidence |
| --- | --- | --- | --- |
| Governor Profile | `UserData.PlayerID` | Current governor id; 0 before login | **MEASURED** |
| Governor Profile | `UserData.Name` | Governor name | CODE, Enabled |
| Governor Profile | `UserData.Power` | Power | CODE, Enabled |
| Governor Profile | `UserData.Civilization` | Civilization id | CODE, Enabled |
| Governor Profile | `UserData.ServerID` | Current kingdom's server id; home kingdom number is server id + 1000 | **MEASURED** |
| Governor Profile | `PlayerInfoData.OriginalData[id].Player` | Profile fallback: `Name`, `Power`, `Civilization`, `Avatar` JSON with `avatar` and `avatarFrame` | CODE, Enabled; `id` is the governor id |
| Governor Profile | `PlayerInfoData.OriginalData[id].Alliance` | `Abbr`, `Name` | CODE, Enabled |
| Governor Profile | `PlayerInfoData.OriginalData[id].Fight` | `UnitKillScore`, kill points | CODE, Enabled |
| Governor Profile | `PlayerInfoData.OriginalData[id].KvkInfo` | `CurContribute`, `MaxContribute`, `WithContribute`, `PeakCnt` | CODE, Enabled |
| Governor Profile | `AchievementData.ShowcaseWall` | Achievement IDs at indices 1–5 after profile opens | CODE, Enabled |
| Governor Profile | `TempleData.titles[i]` | Kingdom title and holder; match `player.id` to the governor | CODE, Enabled |
| VIP | `VipData.VipLevel` | VIP level | CODE |
| VIP | `VipData.VipExp` | VIP points | CODE |
| VIP | `VipData.CanGetFreeBox` | Daily chest available | CODE |
| VIP | `VipData.CanGetFreePoint` | Daily points available | CODE |
| Resources | `CityBuildData.UserRss[kind]` | 1 Food, 2 Wood, 3 Stone, 4 Gold, 5 Gems, 9 Crystal, 18 Amber | CODE, Enabled |

## Troops and Commanders

| Domain | Read path | Values | Evidence |
| --- | --- | --- | --- |
| Troops | `TrainingData.UnitList[i]` | Home troops `unitId`, `unitCount`, indices 1–64 | CODE, Enabled |
| Troops | `WorldMapData.MyChars.AllTroops[troopId]` | `state`, `target`, `carrier`, `charId`, `pos`, `startTime`, `endTime`, `load`, `collected`, `collectStartTime`, `collectSpeed`, `loadRatio`, `isRally`, `IsAutoGather` | **MEASURED** |
| Troops | `WorldMapData.MyChars.AllTroops[troopId].troop.heros[i]` | `heroId`, `label`, `level`; both commanders | CODE; march read **MEASURED** |
| Troops | `WorldMapData.MyChars.TroopCount` | Client's march count | **MEASURED** |
| Troops | `WorldMapData.MyChars.CarriageIds` | Resource carriages also occupy march slots | CODE |
| Commanders | `HeroHandler.HeroMap[id].State` | 0 Nil, 1 Home, 2 Garrison, 3 Move | CODE; commander list **MEASURED** |
| Commanders | `HeroHandler.HeroMap` | Commander level, stars, state, trial status, formation, skills, active talents, treasures, worn equipment and armaments | CODE; expanded read Enabled |
| Commanders | `HeroHandler.HeroMap[main].PveKill` | Primary commander's barbarian kill tally | **MEASURED** |
| Troop Presets | `HeroHandler.TroopProfileData[idx]` | `heros`, `units`, `group`, `groupIdx`, `equips`, `tactics`; `idx = (group - 1) * 15 + groupIdx` | **MEASURED**; read existing data, never request a decode |
| Action Points | `PveData.AP` | Current AP; display rounds down | CODE |
| Action Points | `PveData.MaxAP` | Maximum AP | CODE |
| Army Expansion | `BuffData.BuffMap[12005]` | Nodes' `Source`, `Vf`, `Expire`; first live item node, VIP and other eligible sources | CODE, Enabled for the capacity read |
| March Speed | `BuffData.BuffMap` | Buff ids 6000 through 6004 | **MEASURED**, speed inputs |

| March value | Meaning |
| --- | --- |
| `state` | 1 Idle, 2 Move, 3 Attack, 4 Station, 5 Dead, 6 Gather, 7 Rally, 8 Reinforce, 9 Recover |
| `target` | Destination; resource node while heading out, own city while returning |
| `carrier` | Resource node while gathering; `resType`, `level`; gathering can have no `target` |
| `loadRatio` | **MEASURED:** 1000 on gems while gathering, 1 after walking on or home |
| March-slot floor | City Hall 1 through 4: 1; 5 through 10: 2; 11 through 16: 3; 17 through 21: 4; 22+: 5; separate KvK buff 5052 where applicable |
| Home troops | Unit type and tier from install table `training_Training`; missing home-troop evidence stays **UNKNOWN** |
| Buff expiry | `Expire == 0` permanent; otherwise compare expiry seconds to server time in seconds |

## City

| Domain | Read path | Values | Evidence |
| --- | --- | --- | --- |
| City Hall | `CityBuildData.CityViewList[WorldMapData.MyChars.CityId]` | City object's `lv`, `age`, `civilizationId` | CODE, Enabled |
| Buildings | `CityBuildData.CityViewList[WorldMapData.MyChars.CityId].buildingList[id]` | Building instance `id`, `type`, `lv`, `state`, `endTime`, `totalTime`; times in server ms | CODE, Enabled |
| Builders | `CityBuildData.BuildQueue[i]` | `id`, `buildingId`, `from`, `endTime`; `buildingId == ""` free; up to 8 builders in the building read | CODE, Enabled |
| Buildings | `CityBuildData.BuildingSchema` | Schema selecting the install's building rows | CODE, Enabled |
| City Hall | `CityBuildData.MyCityLevel` | Fallback City Hall level | CODE |
| Academy | `TechniqueData.ResearchProcessNode[i]` | `EndTime`, `TotalTime`, `Type`, `Level`; queue list capped at 64 | **MEASURED**, queue read |
| Training | `TrainingData.TrainingStatusList[i]` | `buildingId`, `unitId`, `unitCount`, `endTime`, `totalTime`; list capped at 64 | **MEASURED**, queue read |
| Hospital | `HealingData.HealingStatusList[1]` | `buildingId`, healing troop counts, `endTime`, `totalTime` | **MEASURED**, queue read |
| Training Expansion | `TrainingData.TrainingAdditionTimes` | Running training-expansion state; paired with `TrainingAdditionAdd` | **MEASURED**, item read |
| Training Expansion | `TrainingData.TrainingAdditionAdd` | Running training-expansion amount | **MEASURED**, item read |

| City value | Meaning |
| --- | --- |
| Building `state` | 1 Normal, 2 Upgrading, 3 Building, 4 NoExist |
| Building walk | Plain `buildingList` only; stop at 4000 entries and mark truncation |
| Queue time left | `ceil((endTime - GameTimer.frameServerTime) / 1000)` seconds |
| Builder expiry | `-1` means permanent; raw representation as `-1` versus uint64 sentinel remains **UNKNOWN** |
| Building levels and costs | Install rows, not client methods; level N+1's row gives the next upgrade's cost |

## Bag, Equipment and Armaments

| Domain | Read path | Values | Evidence |
| --- | --- | --- | --- |
| Bag | `ItemData.MyItemList[id]` | `itemId`, `itemCount`, `expireTime`; keyed bag, `next` on a plain table, cap 5000 | **MEASURED**; `expireTime` unit **UNKNOWN** |
| Bag | `ItemData.MyItemListInited` | `true` after the login bag reply | **MEASURED** |
| Choice Chests | `ItemData.CustomBoxContentConfig[tostring(id)]` | Existing decoded `Tick`, `ItemId`, `Loots` with `Type`, `SubType`, `Value`; no request from the read | CODE; convert only a validated scalar id |
| Equipment | `EquipData.m_aEquipments[i]` | `ID`, `cfgId`, `HeroId`, `Equip`; `HeroId == 0` spare | **MEASURED** |
| Armaments | `HeroFormationData.AllWarBooks[i]` | `id`, `cfgId`, `heroId`, affixes; full rolled-buff read not yet watched | **MEASURED**, list; expanded read Enabled |
| Blacksmith | `EquipData.m_aMaterialOrders[i]` | `Index`, `State`, `ItemId`, `Count`, `TotalTime`, `LeftTime`, `endTime`; up to 5 slots | CODE, Enabled |
| Peace Shield | `BuffData.Shield` | `shieldItemEndTime` | **MEASURED**, item read |
| Gathering Boosts | `BuffData.StackableBuffTargets` | Stackable targets, including 2000 | **MEASURED**, item read |
| Boosts | `BuffData.BuffMap[target]` | First node with `Source.sourceType == "item"`; `Expire` in seconds | **MEASURED**, item read |

| Material order `State` | Value |
| --- | --- |
| 1 | Idle |
| 2 | Waiting |
| 3 | Producing |
| 4 | Complete |

## Alliance, Mail and Chat

| Domain | Read path | Values | Evidence |
| --- | --- | --- | --- |
| Alliance | `AllianceData.AllianceBaseInfo` | `abbr`, `logo`, `lang`, `desc`, `member_num`, `member_max`, `power`, `gift_lv`, `owner`, `privileges` | CODE, Enabled |
| Alliance Members | `AllianceData.AllianceMemberList[i]` | `player`, `grade`, `title`, `join_time`, `loginTime`, `is_online`, board counts | **MEASURED**, list; expanded page read Enabled |
| Alliance List | `AllianceData.JoinBriefInfoList` | Existing list or search results; no search from the read | CODE, Enabled |
| Alliance List | `AllianceData.RecommendBriefInfos` | `id`, `abbr`, `name`, `lang`, `server`, `open`, `member_num`, `member_max`, `power`; at most 10 slots | **MEASURED**, list contents |
| Alliance Log | `AllianceData.AllianceLog[tab][Id]` | Tabs 1 through 3; `Id`, `RT`, `Ti`, `P1`, `P2`, `P3`, `P4` | CODE, Enabled |
| Alliance Technology | `AllianceResearchData.BattleResearchList[type]` | `level`, `start_time`, `end_time`, `total_time`, `science_point`, `skill_info` | CODE, Enabled |
| Alliance Technology | `AllianceResearchData.DevelopResearchList[type]` | Development research, same fields | CODE, Enabled |
| Alliance Technology | `AllianceResearchData.TerritoryResearchList[type]` | Territory research, same fields | CODE, Enabled |
| Alliance War | `RallyData.RallyList[i]` | `captainId`, `targetInfo`, `prepareEndTime`, `troopLimit`, `troops`, `marches`, `square` | CODE, Enabled |
| Alliance War | `RallyData.AllianceAlertList[i]` | Enemy march alerts | CODE, Enabled |
| Alliance Territory | `AllianceBuildData.AllianceBuildList` | `buildType`, `pos`, `buildProgress` with `max`, `remain` | **MEASURED**, list |
| Alliance Territory | `AllianceBuildData.AllianceMineList` | Alliance resource-point records | **MEASURED**, list |
| Alliance Holy Sites | `AllianceBuildData.AllianceStrongholdList` | Alliance's holy sites and positions | **MEASURED**, list |
| Alliance Shop | `AllianceShopData.BuyGoods[tab]` | `Id`, `Label`, `Num`, `Price`, `Item`; tabs 1 Speedups, 2 Resources, 3 Boosts, 4 Other, 5 Weekly | CODE, Enabled |
| Alliance Shop | `AllianceShopData.StockGoods[tab]` | Officer stock rows | CODE, Enabled |
| Alliance Shop | `AllianceShopData.WeekGoods` | Weekly rows; exact remaining-limit meaning of `Num` **UNKNOWN** | CODE, Enabled |
| Mail | `MailData.Datas[box]` | Newest-first `id`, `time` in microseconds, `type`, `box`, `receiver`, `unread`, `body`, `attachments`, `next` | CODE |
| Chat | `LimHelperLua.ConvQueue.hash_map[key]` | Conversation by `"<conv_type>_<conv_id>"`; `unread_count`, `mute`, `is_be_at`, `at_mgs`, `last_read_msg_id` | CODE, Enabled |
| Chat | `LimHelperLua.ConvQueue.queue` | Conversations, pinned first then newest; read cap 600 | CODE, Enabled |
| Chat | `LimHelperLua.ConvQueue.im_queues[key].msg_queue` | Oldest-first messages; read cap 150; `msg_id`, `sender_id`, `timestamp` in ms, `msg_type`, `content.text`, `extra`, `translateText`, `is_recalled` | CODE, Enabled |
| Chat | `[...]ChatSDKHandler.GroupChatID` | Kingdom chat type 4, Alliance 6, Lost Kingdom 1 through 3, Language 25 | CODE, Enabled |
| Language Chat | `LGIMHelper.LangChannelData` | Per-language `lang`, `name`, `list`; channel `index`, `conv_id`, `hot`; 64 languages, 64 channels each | CODE, Enabled |

| Social value | Rule |
| --- | --- |
| Chat cache | 150 messages per conversation normally; the game's open conversation may grow; reduced to 50 after leaving |
| Chat loading | Last 24 messages of rooms and groups and the 10 newest other conversations loaded at IM login |
| Chat reads | No messages sent, fetched, translated or marked read |
| Mail attachment status | 0 new, 1 granted; attachment `loot` records contain `Type`, `SubType`, `Value` |
| Alliance lists, log and boards | Only what the client fetched; an unopened screen can leave the table absent or unpopulated |

## Map and Kingdom Overview

| Domain | Read path | Values | Evidence |
| --- | --- | --- | --- |
| Map | `MapData.chars[id]` | Loaded objects; `charType`, `playerId`, `pos`, `allianceId`, `allianceInfo` | **MEASURED** |
| Map | `MapData.view` | Most recently applied view | **MEASURED** |
| Map | `MapData.charIDInGrid` | Objects belonging to that view | **MEASURED** |
| Map | `WorldMapData.ViewLevel` | Current map view level | **MEASURED** |
| Troops | `MapData.chars[id].path` | Own march route; points' raw `x`, `y`, else raw indices 1 and 2; maximum 64 points, 16 own marches | **MEASURED** |
| Troops | `MapData.chars[id].moveState` | 2 walking; own square has `charType == 1` and our `playerId` | **MEASURED** |
| Troops | `MapData.chars[id].pathIdx` | Current route index; same square has `speed`, `movePosTs`, `moveStartTs`, `targetId` | **MEASURED**; speed in world units per second, times in ms |
| Passes and Holy Sites | `StrategicMapData.WorldStrongholdList[i]` | Whole-kingdom sites and holder alliance ids; levels from install `stronghold_Stronghold` | **MEASURED** |
| Alliance Buildings | `StrategicMapData.AllianceBuildingList` | `charId`, `pos`, `buildType`, `allianceId`, `allianceAbbr` | **MEASURED** |
| Territory | `StrategicMapData.TerritoryOutlineList` | Alliance id, colour 1 through 16, active, inactive and building rings; Vector2 points read at raw indices 1 and 2 | **MEASURED** |
| Territory | `WorldMapData.AllTerritoryList` | `allianceId`, `allianceAbbr`; local cells, plain-table walk capped at 512 | **MEASURED** |
| Bookmarks | `BookmarkData.GroupingData.All[i]` | `Idt` with `GsId`, `X`, `Y`; `Comment`, `Tag`; indices 1 through 300, only own kingdom | **MEASURED**; loaded when the game's Bookmarks first opens |
| Kingdom Overview | `UIObjectManager.ServerListMainPageNew.originalKingdomListData[tab]` | Kingdom bars; tabs 1 Preparation, 2 Season 1, 3 Season 2, 3 Season 3, 5 Conquest, 6 Expedition | CODE, Enabled; screen must have opened |
| Kingdom Overview | `UIObjectManager.ServerListMainPageNew.kvkKingDomListData[tab]` | Lost Kingdom bars for the same six tabs | CODE, Enabled; screen must have opened |
| Lost Kingdom | `LostLandData.AllLostLandInfos` | `id`, `order`, `mode`, `state`, `openTime`, `closeTime` in ms, `lostLandLevel`, `members`, `hotBattleGrids` | CODE, Enabled |
| Governors | `AccountServerRoleData.RolesList[i]` | `PlayerId`, `ServerId`, `OriServerId`, `PlayerName`, `TownCenterLevel`, `Power`, `Avatar`, `Abbr`; index cap 64 | **MEASURED**; never read account keys, tokens or login IP |

| Map value | Rule |
| --- | --- |
| Coverage | Client-loaded views and observed messages only; unseen ground **UNKNOWN**, not an empty kingdom |
| Position | In-game X/Y = `floor(worldCoordinate / 6 + 0.5)`; distance in km = world units / 10 |
| Client object ids | Map keys use server char ids multiplied by 100 |
| Local positions | Subtract the kingdom's offset before treating client positions as wire coordinates |
| Raw object walk | Plain `MapData.chars` only; at most 3000 objects; truncated reads cannot prove disappearance |
| Kingdom Overview bounds | 2500 bars per tab, 4 kingdom records per row, 600 Lost Kingdoms, 32 members per Lost Kingdom, 64 governors |

### Watching the map

| Step | Literal client surface | Mechanics |
| --- | --- | --- |
| Arrival | `NetMessageMgr.HandleNetMsgV3(self, id, buffer, n, n)` | Decodes the incoming payload before dispatch |
| Wrapped decoded steps | `NetMessageMgr.ProcessAck(self, id, payload, n)` and `NetMessageMgr.DispatchAck(self, id, payload, ...)` | **MEASURED:** both receive the same decoded table |
| Per incoming message | `id`, `payload` | Note the message id and relevant scalar fields; suppress the duplicate observation with `rawequal(payload, lastPayload)` |
| Resource nodes | `payload.Rss[i]` on MT 1003 | `ResType`, `Level`, `Remain`, `Troops`; `Char.Id`, `Char.Pos.X`, `Char.Pos.Y`; territory alliance `AsInfo.Id` |
| Cities and troops | `payload.Castles[i]`, `payload.Squares[i]` on MT 1003 | City name, City Hall, alliance, position; barbarian or fort type, level and fighting state |
| Alliance buildings | `payload.Buildings[i]` on MT 1003 | Building type, position and holder; type 1 flag |
| Pass and holy-site holders | `payload.Passes[i]`, `payload.Fortresses[i]` on MT 1003 | `Char.AllianceId`, `Char.Alliance.Abbr` |
| Territory | `payload.AsTerritorys` on MT 1003 or MT 1136 | Cell positions and alliance; alliance 0 clears a cell |
| Removal | `payload.CharId` on MT 1022 | Object removed by the server |
| Forwarding | `return original(self, id, payload, ...)` | Passive wrapper passes the same arguments, payload and every return value through unchanged; no message dropped, queued, answered or rewritten |
| Observation bounds | MT 1003 and MT 1136 lists | 400 castles, 800 squares, 200 buildings, 64 passes, 64 holy sites, 64 territory blocks, 400 cells per message |
| Evidence boundary | Extra city, building and territory observations | CODE, hostile-message tests passed; not all extra branches watched live; message passage unchanged in tests |

## Events

| Domain | Read path | Values | Evidence |
| --- | --- | --- | --- |
| Event Calendar | `EventData.BeginStamp` | First calendar column's UTC epoch seconds | **MEASURED** |
| Event Calendar | `EventData.EventScheduleData[i]` | `EventId`, `Title`, `CalenderDesc`, `Icon`, `StartWDay`, `EndWDay`, `StartTime`, `EndTime`, `IsReward`, `Loots` | **MEASURED**; event times in seconds |
| Events | `EventData.EventSideListData[1][i]` | `EventId`, `Title`, `Icon`, `Tag`, `UIType`, `RedCount` | CODE |
| Events | `EventData.ServerEventData[id]` | Server event, tasks, exchanges, sub-events, rules and time window; `id` is a string, not a number | CODE; event messages **MEASURED** |
| Events | `EventData.PlayerEventData[id]` | Player tasks' `TaskStatus`, exchanges' `Cnt`, chosen rewards | CODE; event messages **MEASURED** |
| The Mightiest Governor | `EventData.ServerEventData[id].SubEvents[i]` | Stage with `OpenTime <= now < EndTime`; `ScoreRules`, `Tasks`, `RankPlayer` | CODE; event messages **MEASURED**; `UIType == "ORDER_BY_STAGE"` |
| The Mightiest Governor | `EventData.LeaderboardData[RankKey].SelfItem` | `Rank`, `Score`, `PreRank`; **UNKNOWN** until the game's board fetch | CODE, Enabled |
| The Mightiest Governor | `EventData.LeaderboardData[RankKey].List[i]` | `Id`, `Rank`, `Score`, `PreRank`, serialized `Value` | CODE, Enabled; event board not captured |
| Wheel of Fortune | `EventData.ServerEventData[id].Turntable[i]` | `Cycle`, `Loots`, `GemCost`, `Times5Cost`, `TotalTime`, `Odd`, `GachaId` | CODE, Enabled |
| Wheel of Fortune | `EventData.PlayerEventData[id].Turntable` | `CurrentTime`, `CurrentFree`, `CurrentHalf`, `PhaseLootCnt`, `AllTime`, `Selects`, `LeftBuyTimes`, `AllBuyTimes` | CODE, Enabled |
| Lucerne Scrolls | `BattlePassData.m_LevelInfo` | Current pass level and EXP | CODE; update messages **MEASURED** |
| Lucerne Scrolls | `BattlePassData.m_aRewardStatus` | Level rewards' `FreeGet`, `PayGet` | CODE; reward messages **MEASURED** |
| Lucerne Scrolls | `BattlePassData.m_aaMissionWeek[week]` | Weekly missions; weeks 1 through 5, `TaskStatus == -2` claimed | CODE |
| Lucerne Scrolls | `BattlePassData.m_aMisionSeason` | Season missions | CODE |
| Lucerne Scrolls | `BattlePassData.m_nDailyStatus` | 2 after daily EXP claimed | CODE |
| Council of Affairs | `FormationDispatchHandler.questList[i]` | `id`, `cfgId`, `startTS`, `duration`, `state`, `heros`; quest times in seconds | CODE; board messages **MEASURED** |
| Council of Affairs | `FormationDispatchHandler.dispatchDoneNum` | Dispatches used; paired with `dispatchMaxNum` | CODE |
| Council of Affairs | `FormationDispatchHandler.dispatchMaxNum` | Dispatch limit | CODE |
| Golden Kingdom | `RogueLikeData.IsInRglikeContext` | Boolean, inside the Golden Kingdom | CODE, Enabled; not watched live |
| Golden Kingdom | `RogueLikeData.CardList[i]` | Floor cards; decoded `Pos` and `State` default to 0 when raw `nil` | CODE, Enabled |
| Golden Kingdom | `RogueLikeData.CardInfo` | `Level`, `ReloadCount`, `IsFirstEnter`, `IsFirstEnterLast` | CODE, Enabled |
| Golden Kingdom | `RogueLikeData.MyTroops[i]` | `ID`, `BattleValue`, `UnitsValue`, `IsDead`, `PosIdx`, commanders | CODE, Enabled |
| Golden Kingdom | `RogueLikeData.BattleTreasures[i]` | Pending Blessing choice, `CfgId`, `Count` | CODE, Enabled |
| Golden Kingdom | `RogueLikeData.AllTreasures[i]` | Held Blessings | CODE, Enabled |
| Golden Kingdom | `RogueLikeData.AllItems[subType]` | Relics, `Value` count | CODE, Enabled |
| Golden Kingdom | `RogueLikeData.RgLikeCoin.Value` | Karaku Gold | CODE, Enabled |

## Install files

| Source | Literal table | Values |
| --- | --- | --- |
| `exceldata.bin` | `building_Building` | Building levels, schema overrides, enabled maxima, resource and item costs, prerequisites, base time and power |
| `exceldata.bin` | `buildingfunction_towncenter` | City Hall's `ArmyCnt` capacity and `MarchCnt` slots |
| `exceldata.bin` | `hero_HeroExp` | Flat capacity `Lead` by commander level |
| `exceldata.bin` | `training_Training` | Unit type, tier, names and base speed |
| `exceldata.bin` | `technique_technique` | Research levels and names |
| `exceldata.bin` | `items_items` | `Use`, `OpType`, `Target`, `OpValue`, `LastTime`, `MaxUseLimit`, `uselevel`; item names and descriptions |
| `exceldata.bin` | `buffcount_BuffCount` | Buff definitions and name keys; match `BuffId`, highest `Order` wins |
| `exceldata.bin` | `equipment_equipment`, `equipment_New_Equipment` | Equipment recipes, attributes, awakening and talent requirements |
| `exceldata.bin` | `stronghold_Stronghold` | Pass and holy-site kind and level |
| `exceldata.bin` | `quests_formation_dispatch_quest` | Council quest AP costs |
| `exceldata.bin` | `roguelike_Const`, `roguelike_Buff` | Golden Kingdom rules and floor buffs |

## Failures and proof

| Condition or level | Result |
| --- | --- |
| Lua error or timeout in a read | Disable only that read for this run |
| Read already off | Refuse immediately; never retry implicitly |
| Third failure since last manual re-enable | Disable across runs until manual re-enable |
| Several reads in one failing chunk | Switch all those reads off together |
| **Observed** | Watched returning on a live client; eligible for release |
| **Decompile** | From client code and mock-tested; never run live; not enabled |
| **Enabled** | Test-enabled, no live return observed, not release-eligible |
| **MEASURED** | Only the stated live or wire result, not every field or expanded read |
| **UNKNOWN** | Missing evidence or unresolved value. No invented paths or zero defaults |

## The reload hold

| Read path or condition | Value or rule |
| --- | --- |
| Governor switch, account switch, reconnect | Rebuilds the client's Lua world; every ordinary read refused throughout the reload |
| `UserData.PlayerID` | Reload identity read; no governor before login |
| `GameLoadingMgr.curStep` | `-1` means loaded only with a nonzero governor id; the module also initializes at `-1` |
| `GR_GameStates` | Raw start-up state; Tap to start is `StateTapToStart`, compared by `rawequal` |
| Exceptions | Only the switch flow's identity, Tap-to-start state and own teardown reads; no general getter or state walk |
| Session procedure | [Governors and reloads](04-governors.md) |

## Gaps

| In-game value | Evidence gap |
| --- | --- |
| Conduct score | Getter exists. Backing-table field **UNKNOWN** |
| Hospital wounded total and capacity | Only the healing queue verified. Client-side total fields **UNKNOWN** |
| VIP and profile expanded reads | CODE or Enabled where stated; no live-return claim added |
| Timed bag items | `ItemData.MyItemList[id].expireTime` exists; unit **UNKNOWN** |
| Builder expiry | `CityBuildData.BuildQueue[i].endTime` permanent representation remains unresolved |
| Golden Kingdom, Wheel and chat expanded state | Code-derived paths, not new live verification |

---

**Related:** [Invoking the client](03-client-calls.md).

<div align="center">

[← Attach and the Execution Hook](01-attach.md) &nbsp;·&nbsp; [↑ The Lua Runtime Method](README.md) &nbsp;·&nbsp; [Invoking the Client →](03-client-calls.md)

</div>
