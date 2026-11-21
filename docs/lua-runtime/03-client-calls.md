<sub>[RoK Toolkit](../../README.md) › [Docs](../README.md) › [Lua Runtime](README.md) › Invoking the Client</sub>

<img src="../assets/headers/lua-runtime.svg" width="100%" alt="">

# Invoking the Client

**MEASURED**

Call the function the game's own button calls, with the same arguments, after the game's own checks.

**On this page:** [Reading the calls](#reading-the-calls) · [Action families](#action-families) · [Gathering march](#dispatch-a-gathering-march) · [Barbarians](#search-and-attack-a-barbarian-together) · [Fort rally](#lead-or-join-a-fort-rally) · [Checks](#checks-before-each-invocation) · [Second password and gems](#second-password-and-gem-budget) · [Journal](#journal-fields) · [Source gaps](#source-gaps)

## Reading the calls

| Label | Meaning |
| --- | --- |
| **MEASURED** | Observed in the running client or a captured request/reply |
| **CODE** | Literal name or argument list recorded from the client's Lua, not proof that the invocation was watched live |
| **UNKNOWN** | These docs do not establish the callable name, full argument list or MT mapping |
| `cb` | The callback shape the game's own caller passes, including `nil` where it passes `nil` |
| Function tables | **CODE** unless a specific invocation is marked **MEASURED**; a named callable alone is not a watched send |
| Client boundary | The client sends the request and owns the session, login, heartbeat and cipher; no raw protocol send is substituted |

## Action families

### Marches

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Marches | Dispatch a gathering march | `WorldMgr:DispatchTroopReq(heros, units, 'dispatch_troop_1', nil, node * 100, false)` | [1012 → 1013](../headless/game-systems/02-marches.md#dispatch_gather-the-two-step-march) |
| Marches | Retarget a gathering march | `WorldMgr:MoveReq(troop, {x=0,y=0}, {x=0,y=0}, node * 100, false)` | [1007 → 1020](../headless/game-systems/02-marches.md#march_move-mt-1007-movereq) |
| Marches | Recall one standing march | `WorldMgr:ReceiveTroop(troopId, nil, nil)` at home | [1014 → 1, then 1005](../headless/game-systems/02-marches.md#recall_marches-mt-1014--mt-1019)<br><sub>accepted is not home</sub> |
| Marches | Resolve a custom march | `WorldMgr:DispatchTroopReq` receives the resolved heroes and units; a distinct client custom-march resolver is **UNKNOWN** | [1012 → 1013](../headless/game-systems/02-marches.md#dispatch_gather-the-two-step-march)<br><sub>no preset write</sub> |

### Barbarians

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Barbarians | Search one level | `RssSearchHandler:Search(RssSearchData.Type.Barbarian, level)` | [1178 → 1179](../headless/game-systems/03-forts.md#the-lead-sequence)<br><sub>with `NpcType = 1`, not the fort family's 2</sub> |
| Barbarians | Stage a march on open ground | `WorldMgr:DispatchTroopReq({heroes}, {units}, 'dispatch_troop_1', {x,y}, nil, false, nil, nil, cb)` | [1012 → 1013](../headless/game-systems/02-marches.md#_send_walkout-mt-1012-untargeted-walkout) |
| Barbarians | Attack with selected marches | `WorldMapHandler:SetStayAfterBattle(true)`, then `WorldMgr:MultiMoveReq(ids, {x=0,y=0}, {x=0,y=0}, target * 100, true, nil, cb, nil, false, nil, nil)`; restore the previous Stay value | [1017 → 1021](../headless/game-systems/02-marches.md#barb_wirebuild_multi_move-mt-1017-multimovereq-group-attack) |
| Barbarians | Retarget one march at a barbarian | `WorldMgr:MoveReq(troopId, {x=0,y=0}, {x=0,y=0}, targetId, isOffensive, nil, callback, session, isAllianceBuild, joinType, isAggressive)`; at home: session `nil`, alliance build `false`, no join type | [1007 → 1020](../headless/game-systems/02-marches.md#march_move-mt-1007-movereq) |
| Barbarians | Move a selected march to open ground | `WorldMgr:MoveReq(troopId, {x=0,y=0}, {x=sx,y=sy}, nil, false, nil, callback, session, nil, nil, aggressive or false)` | [1007 → 1020](../headless/game-systems/02-marches.md#march_move-mt-1007-movereq)<br><sub>no target, not offensive, `IdleStation = true`</sub> |

### Forts

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Forts | Precheck a rally target | `RallyHandler:RallyPrecheckReq`; complete arguments **UNKNOWN** | 1175 reach check, then 9997 precheck; these extra client calls are not field-tabled in [forts](../headless/game-systems/03-forts.md) |
| Forts | Create the rally | `RallyHandler:CreateRallyReq(heros, units, {}, lid, 300000, 'dispatch_troop_1', {}, nil, nil)` | [1100 → new march on 1005](../headless/game-systems/03-forts.md#fort_rally-create-mt-1100-createrallyreq)<br><sub>own rally appearing confirms acceptance</sub> |
| Forts | Join the captain's rally | `RallyHandler:ReinforceRallyReq({heros}, {units}, nil, captain, 'dispatch_troop_1', cb, nil, nil, nil, nil)` | [1108 → 1109, then 1105 roster](../headless/game-systems/03-forts.md#fort_join-reinforce-mt-1108--1109) |

### City

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| City | Upgrade a building | **UNKNOWN** client sender; `UpgradeBuildingReq` is a protocol message name, not an established callable | [205 → 206](../headless/game-systems/04-city.md#try_upgrade-mt-205-upgradebuildingreq) |
| City | Start research | **UNKNOWN** client sender; `TechniqueUpgradeReq` is a protocol message name | [252 → 253](../headless/game-systems/04-city.md#start_research-mt-252-techniqueupgradereq) |
| City | Collect finished research | **UNKNOWN** qualified client sender and argument list; the page names `TechniqueEndReq` without establishing its callable | [259 → 260](../headless/game-systems/04-city.md#start_research-mt-252-techniqueupgradereq) |
| City | Start training | `TrainingHandler:TrainingBeginReq(buildingId, unitId, count, false)` | [300 → 301, queue state 302](../headless/game-systems/04-city.md#train_troops-mt-300-trainingbeginreq) |
| City | Collect trained troops | `TrainingHandler:TrainingEndReq(buildingId, cb)` | [303 → 1, then 302 / 1234](../headless/game-systems/04-city.md#train_troops-mt-300-trainingbeginreq) |
| City | Start healing | `HealingHandler:HealingBeginReq(hospitalId, units, false, nil, nil, 0)` | [330 → 331, state 332 / 337](../headless/game-systems/05-claims.md#healing-mt-330--332--336) |
| City | Collect healed troops | `HealingHandler:HealingEndReq(status.buildingId)` | [336 → 1, then 332](../headless/game-systems/05-claims.md#healing-mt-330--332--336) |

### Blacksmith

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Blacksmith | Read production orders | `EquipHandler:RequestGetEquipMaterialOrdersStatus(id)` | [9750 → 9751](../headless/game-systems/04-city.md#material-production-mt-9750--9752--9755) |
| Blacksmith | Produce material | `EquipHandler:RequestEquipMaterialOrderAdd(id, slot, item, count, false)` | [9752 → 1, then order state](../headless/game-systems/04-city.md#material-production-mt-9750--9752--9755) |
| Blacksmith | Collect finished materials | `EquipHandler:RequestEquipMaterialOrderEnd(id)` | [9755 → 9756, then 103](../headless/game-systems/04-city.md#material-production-mt-9750--9752--9755)<br><sub>items, not resource balances</sub> |
| Blacksmith | Combine materials | `EquipHandler:RequestComposeEquipMaterial(id, item, count)` | MT mapping **UNKNOWN**; [city](../headless/game-systems/04-city.md) covers production only |
| Blacksmith | Dismantle materials | `EquipHandler:RequestDecomposeEquipMaterial(id, item, count)` | MT mapping **UNKNOWN** |
| Blacksmith | Forge equipment | `EquipHandler:RequestComposeEquipment(id, cfgId)` | 9700; ack **UNKNOWN** |
| Blacksmith | Refine equipment | `EquipHandler:RequestRefineEquipment(id, uid, false, {})` | MT mapping **UNKNOWN** |
| Blacksmith | Awaken equipment | `EquipHandler:RequestAwakeEquipment(id, uid)` | MT mapping **UNKNOWN** |
| Blacksmith | Dismantle equipment | `EquipHandler:RequestDecomposeEquipment(id, uid, false, cb)` | MT mapping **UNKNOWN** |

### Claims

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Claims: quests | Claim a quest reward | **UNKNOWN** client function; no quest module page establishes it | [143](../headless/game-systems/05-claims.md#claim_quests-mt-143-questgetrewardreq)<br><sub>acknowledgement **UNKNOWN** here</sub> |
| Claims: quest chests | Claim an activity chest | **UNKNOWN** client function | [144](../headless/game-systems/05-claims.md#claim_quests-mt-143-questgetrewardreq)<br><sub>separate from 143</sub> |
| Claims: VIP | Claim the daily chest | `VipHandler:ClaimDaily(function() end)` when `VipData.CanGetFreeBox` | [8602 → 8600](../headless/game-systems/05-claims.md#caravan--vip-mt-8057--8601--8602-and-the-shop-8603--8605) |
| Claims: VIP | Claim the exclusive chest | `VipHandler:ClaimPoint(function() end)` when `VipData.CanGetFreePoint` | [8601 → 103](../headless/game-systems/05-claims.md#caravan--vip-mt-8057--8601--8602-and-the-shop-8603--8605) |
| Claims: Tavern | Open a free silver, gold or material chest | `TavernHandler:Draw(pool, TavernDrawReq.Free, 1, function() end)` | [360 → 361, then 103](../headless/game-systems/05-claims.md#open_tavern_chest-mt-360-taverndrawreq)<br><sub>pool 1 / 2 / 3</sub> |
| Claims: conduct | Claim the monthly reward | `ReputationHandler:RequestCreditScoreMonthlyRewardInfo(cb)`; callback requests `ReputationHandler:RequestCreditScoreInfo()` | 2603 → 2604, then 103; refresh 2600 → 2601; **UNKNOWN** headless module field table |
| Claims: mail | Read all and claim one box | **UNKNOWN** named client sender; the documented Read all handler constructs the request in place | [7914 → 1, then item updates](../headless/game-systems/05-claims.md#mail-mt-7920--7921)<br><sub>no delete</sub> |
| Claims: mail | Mark unread alliance armory reports read | `AllianceArmoryReportRes:MarkAllAsRead()` / `MailHandler:SetRead(AllianceBox, unread)` | 7903 → **UNKNOWN** acknowledgement here; [mail reference](../headless/game-systems/05-claims.md#mail-mt-7920--7921) |
| Claims: bag chests | Open a random chest | `ItemHandler:UseItemReq(id, count, cb, true, bagItem)` | [100 → 1 / 103](../headless/game-systems/05-claims.md#item-use-mt-100-useitemreq)<br><sub>chest loot also arrives on 109</sub> |
| Claims: bag chests | Read a choice chest's rewards | `ItemHandler:FetchCustomBoxContent(id)` | MT mapping **UNKNOWN** |
| Claims: bag chests | Open the chosen reward | `ItemHandler:ExchangeCustomBox(id, count, Type, SubType, nil, cb)` | MT mapping **UNKNOWN** |
| Claims: Council of Affairs | Claim a finished quest or milestone chest | `FormationDispatchHandler:RequestDispatchReward(questId)`; `-1` for the chest | [4934 → 4935](../headless/game-systems/05-claims.md#the-council-boards) |
| Claims: Wheel of Fortune | Free or paid spin | `RequestWheel` is the named screen handler; `rollCall` is its local sender; complete arguments **UNKNOWN** | 1216 → 1, reward push 1217; **UNKNOWN** headless module field table |
| Claims: Wheel of Fortune | Claim an Extra Rewards chest | `RequestOpenChest_Wheel`; complete arguments **UNKNOWN** | 1218 → unknown ack (**UNKNOWN**) |
| Claims: Wheel of Fortune | Pick the featured commander | `EventHandler:FortuneWheelSelectHeroReq(eventData, RichManSlotParam, gacha_id, callback)` | 1298 → 1 |
| Claims: Wheel of Fortune | Buy Spin Tickets | **UNKNOWN** named client sender; the page constructs the request in place | 1378 → **UNKNOWN** acknowledgement here |
| Claims: battle pass | Daily EXP chest | `OnClickBtnBoxTop` is the named screen handler; callable owner, arguments and request-sender name **UNKNOWN** | 9807 → **UNKNOWN** acknowledgement here; [claims reference](../headless/game-systems/05-claims.md) |
| Claims: battle pass | Claim a finished mission | `RegisterBtnClaim` names the button-registration path, not an established reward sender; sender and arguments **UNKNOWN** | [9804 → 9805 / 9801 updates](../headless/game-systems/05-claims.md) |
| Claims: battle pass | Claim reached levels and owned paid-track rewards | `ClickBtnClaimAll` is the named screen handler; callable owner, arguments and request-sender name **UNKNOWN** | [9802 → 9808](../headless/game-systems/05-claims.md)<br><sub>optional max-level loot 9813, acknowledgement **UNKNOWN**</sub> |

### Alliance

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Alliance | Help all / request help | **UNKNOWN** client sender names; `AllianceHandler:IsBuildCanRequestHelp` is a gate, not the sender | [2045 / 2048](../headless/game-systems/05-claims.md#alliance-help-mt-2045--2048)<br><sub>healing-help route also records 2050</sub> |
| Alliance | Donate resources to research or a skill | `AllianceResearchHandler:ReqResearchDonate(info.type, info.level, const_pb.NormalDonate, kind, cb)`; kind 1 research, 2 skill | [2080 → 2081](../headless/game-systems/05-claims.md#tech-donate-mt-2080-astechdonatereq) |
| Alliance | Claim gifts | **UNKNOWN** client sender | [2067 → 2068](../headless/game-systems/05-claims.md#alliance-gifts-mt-2065--2066-open-2067-daily-supplies-2109) |
| Alliance | Buy / stock / recycle | `AllianceShopHandler:BuyGoodsReq(row, count, cb)` / `StockGoodsReq(row, count, cb)` / `AsRecycleReq(itemId, count, cb)` | 2097 → 2098 / 2095 → 2096 / 2125 → 1; **UNKNOWN** headless module field tables |
| Alliance | Refresh alliance lists | `AllianceHandler:RequestRecommendAllianceList(cb)` and `AllianceHandler:RequestJoinAllianceList(cb)` | 2023 → 2024; recommended list uses `Open = true`, general list `false` |
| Alliance | Search by tag | `AllianceHandler:SearchAlliance(tag, cb)` | 2017 → 2018 |

### Camera

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Camera | Bring a march into view and follow it | `WorldMgr:ViewTroop(troopId)` | **CODE**, live invocation **UNKNOWN**; **UNKNOWN** fixed MT mapping |
| Camera | Zoom out to Map | `WorldCamera:HK_SetCameraInfo({cameraDxf = 999999})` | Client may enter Map view and request 243 → 244; **UNKNOWN** headless module field table |
| Camera | Restore zoom, then centre | `WorldCamera:HK_SetCameraInfo({cameraDxf = before})`, then the same function with `{posx, posy}` | Do not combine centre and zoom restoration into one call; **UNKNOWN** fixed MT pair |

### Chat

| Family | To get | Client function or explicit gap | MT and headless field tables |
| --- | --- | --- | --- |
| Chat | Send text on the player's click | `LimHelperLua:TextSend(conv_type, conv_id, text, {})` | Separate IM link, not an ordinary game MT pair; conditional game forwarding 9911 → 9912 |
| Chat | Pull history on the player's click | `LimHelperLua:CheckAndRequestMsgs(conv_id, conv_type)` | Separate IM link; **UNKNOWN** ordinary game MT pair |

## Dispatch a gathering march

1. Check `WorldMgr.isEntered`, a free slot, the commanders at home, nonempty troops, a known city, a free node outside another alliance's territory, and `CanCollectRss(kind)`.
2. Build the route as `ReqPathLine` does and call `NetMessageMgr:SendRequest` for 1050, with the screen's callback shape; wait for 1051 or the refusal.
3. Call `EquipHandler:FetchEquipmentArchives(nil)` for 9726.
4. Set `WorldMapHandler:SetPendingAutoGather(...)` using the client's server, territory, kingdom-view and preference checks; the full argument list is **UNKNOWN** here.
5. Call `WorldMgr:DispatchTroopReq(heros, units, 'dispatch_troop_1', nil, node * 100, false)`.
6. Read 1013 and the new march state; **MEASURED**: five of five gathering dispatches reached the map, and retargeting was observed separately.

## Search and attack a barbarian together

1. Check the unlocked level, no 1178 in flight, the selected marches and their combined AP cost.
2. Call `RssSearchHandler:Search(RssSearchData.Type.Barbarian, level)`.
3. Read `RssSearchData.LocalData[key].Points` and its new `Tick`; the client orders the search results nearest the city first.
4. Stage each home march through the route preview, `EquipHandler:FetchEquipmentArchives(nil)` and the untargeted `WorldMgr:DispatchTroopReq` above.
5. Once the selected marches stand ready, preview each route, save the Stay setting, call `WorldMapHandler:SetStayAfterBattle(true)`, then call `WorldMgr:MultiMoveReq` once with all selected ids and restore Stay.
6. Read the accepted troop ids on 1021 and subsequent fight state; a successful invocation alone is not a kill.

## Lead or join a fort rally

1. Check alliance membership, castle, home-kingdom availability, unlocked fort level, AP, free march capacity and home commanders; for creation also check no rally already captained and the client's create cooldown.
2. For creation, follow the client's 1175 reach check with `RallyHandler:RallyPrecheckReq` (9997); reject a refused or already-rallied target.
3. Follow the screen's 1050 target selection and `EquipHandler:FetchEquipmentArchives(nil)` (9726).
4. Recheck the gates, then call `RallyHandler:CreateRallyReq(...)` with the exact argument list in the table; check the own rally appears.
5. For joining instead, check `CanJoinRally`, troop room, remaining muster time and AP; select the captain with 1050, fetch equipment archives, then call `RallyHandler:ReinforceRallyReq(...)` with the table's argument list.
6. Read the 1105 membership snapshot; use reward mail, not march motion, to establish a fort win.

## Checks before each invocation

| Gate | Fact or required read |
| --- | --- |
| In-flight request | `NetMessageMgr:HasRequest(api)` is the client's gate; one request of that path at a time |
| Callback arguments | Copy the client's caller exactly, including callback or `nil`; a wrong argument list can raise and terminate the client |
| Reply | `SendRequest(api, req, cb, expectedAck)` uses the named ack where supplied; without one, MT 1 carries the request's API and code |
| Timeout | Client timeout 15 s; an unanswered callback may remain in the pending list, so timeout is not permission to resend |
| Local checks | Recheck the current state at the invocation, not only at an earlier read |
| Timed city work | No `Seckill = true`; no gem instant-finish substitution |
| Training | Idle, normal-state building; unlocked exact tier; count within capacity; `GetTrainingCost(...).lackRes` empty |
| Healing | One job; no dungeon or battlefield; city not fighting; `GetHealingCost(units, nil, 0).lackRes` empty; collect only when `IsHealingFinished(status)` |
| Blacksmith production count | Client constant `One_Group_Material_Produce_Count`, default 10; install and documented order example give 15; live value on a given server **UNKNOWN**, not a universal fixed 15 |
| Blacksmith equipment | Normal mode, no pending forge/refine/awaken/talent selection, permission 7, costs available, no second-password prompt; no dismantling worn equipment |
| Claims | Claimable state first; no reward request merely because the function exists |
| Automation | Explicit Start; at most one action per step; locked actions run checks dry and record only what they would invoke |
| Session | One running client and one session; ending or reloading the client invalidates the old callable objects; [switch flow](04-governors.md) |

## Second password and gem budget

| Gate | Evidence | Result |
| --- | --- | --- |
| Second password | **MEASURED** tool boundary; client code names refusal 9273 `SecondPasswordNeedCheck` | Stop and report; never enter the password or automatically resend through the prompt |
| Gem budget | **MEASURED** tool boundary: player-set spending budget defaults to zero | No gem spending without the player's allowance; paid actions keep confirmation |
| VIP Shop | **CODE** resource-only mode, or gem mode within a player-set weekly budget | Recheck price, stock, level, funds and budget; 9273 stops buying until the player resets the plan |
| Wheel of Fortune | **CODE** only the free spin is automated | Paid spins and Spin Ticket purchases require the player's click and confirmed price |
| Tavern | **CODE** `Type = Free`, `Times = 1` | No gem or key draw |
| Blacksmith | **CODE** no Quick Forge or Quick Awaken | Stop before a second-password requirement |

## Journal fields

| Field | Records |
| --- | --- |
| Identity | Governor, action family and action |
| Intent | Target, arguments, pre-invocation guards, expected state |
| Invoked | Literal client function or request and supplied arguments |
| Response | Callback answer, reply/refusal code, post-call state, wait |
| Verdict | `As intended`, `Different`, `Refused`, `No answer`; dry runs: `Would send` |

## Source gaps

| Family | Still **UNKNOWN**, not inferred from an MT name |
| --- | --- |
| City | Building-upgrade sender; research-start sender; qualified research-collection callable and arguments |
| Quests | Quest-reward and activity-chest client senders; not established in this reference |
| Mail | Named Read all reward sender; request constructed inline |
| Alliance | Help-all, request-help and gift-opening client senders |
| Wheel of Fortune | Full `RequestWheel` / `RequestOpenChest_Wheel` signatures; named Spin Ticket purchase sender |
| Battle pass | Reward-sender names and complete handler signatures; `RegisterBtnClaim` not established as sender |
| Cross-module MT coverage | Blacksmith non-production actions, several event actions, alliance shop and camera calls lack corresponding headless field tables |

---

<div align="center">

[← Reading the Client](02-client-reads.md) &nbsp;·&nbsp; [↑ The Lua Runtime Method](README.md) &nbsp;·&nbsp; [Governors and Reloads →](04-governors.md)

</div>
