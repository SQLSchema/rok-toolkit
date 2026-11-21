<sub>[RoK Toolkit](../../README.md) › [Docs](../README.md) › [Headless](README.md) › Message Registry</sub>

<img src="../assets/headers/headless.svg" width="100%" alt="">

# Message Registry

**On this page:** [Coverage](#registry-coverage) · [ID mapping](#id-mapping-and-unknown-payloads) · [Session and transport](#session-and-transport) · [City](#city-inventory-and-development) · [World map](#world-map-movement-and-action-points) · [Rallies](#rallies) · [Alliance](#alliance) · [Quests and mail](#quests-events-rewards-and-mail) · [Conflicts and limits](#registry-conflicts-and-limits) · [Appendix](#appendix-additional-observed-mts) · [Results and failures](#results-and-failures) · [Checks](#checks)

## Plain language

Main registry: engine-recognized MTs. Appendix: additional observed MTs, including unknown purposes.

## What you need first

> [!TIP]
> Look up the decoded [envelope’s](02-wire-protocol.md) decimal MT, not a field number, result code, or payload ID.

## Registry coverage

The engine registers 142 decimal message type IDs: 68 client-to-server (`c2s`) and 74 server-to-client (`s2c`). Protobuf schema 1.1.522 defines payload fields, not MT assignments or enums. Canonical names grouped by subject.

- `MEASURED` means observed
- `INFERRED` means the mapping rests on indirect evidence
- `UNKNOWN` means the purpose is not established

These are registry labels. `*`: no schema binding, not necessarily empty.

## ID mapping and unknown payloads

The version-1 registry is YAML: `schema: 1` and `messages:` (a sequence of mappings). A row starts with `mt` (unique integer), `name` (canonical identifier), `dir` (`c2s`, `s2c`, or `both`), and `message` (proto name or `null`). `aliases` are alternate names, not separate IDs. `status` rates the MT and `message_status` rates the payload binding: `MEASURED`, `INFERRED`, or `UNKNOWN` where applicable.

For example, MT 9's row starts `{mt: 9, name: HeartBeatReq, dir: c2s, message: HeartBeatReq, acks: [], refusal: [none]}`.

- `acks` lists successful reply IDs; `answers` is the reverse mapping (`[any]` for MT 1).
- `mt1_ok`, `refusal` (`mt1_echo`, `ack_code`, `silent`, `none`, or `unknown`), `push`, and `pushes_after` distinguish confirmations, refusals, and side effects.
- `wrapped` is observational, not a decoding rule.
- Outbound `domain` and `risk` classify requests; `emit_order` maps quoted field paths such as `'.'` (root) or `'.5.2'` (nested message) to lists of field numbers in send order.

The registry records evidence separately from the mapping. Observed game MTs outside it, such as MT 904 account buffs (`BuffNtf`), do not acquire a registered decoder.

- For a registered MT with a schema binding, the engine decodes the payload as that message, keeps unknown fields, and permits missing proto2 `required` fields.
- A missing registry row or failed typed parse increments the unknown-MT count and falls back to `Raw` top-level protobuf fields.
- A known row with no binding also uses `Raw` but does not increment that count.
- If even the raw parse fails, it returns the fields read before the fault and a malformed error.

> [!IMPORTANT]
> The live session ends on that error. A well-formed unknown MT does not end the session.

## Session and transport

- 1 `CommAck` (s2c): **MEASURED, binding INFERRED.** Echoes the request MT in `Api` and reports its result in `Code`.
- 2 `ClientStatusReport` (c2s): **MEASURED.** Periodic client status and network statistics, using the `HeartBeatReq` schema.
- 3 `ClientStatusAck` (s2c): **MEASURED.** Empty `HeartBeatAck` confirming MT 2.
- 9 `HeartBeatReq` (c2s): **MEASURED.** Empty keepalive sent without waiting for a reply.
- 14 `LoginReq` (c2s): **MEASURED.** Authenticates the game session using account, device, and version data. Fields: ([session](01-login.md#step-4-loginreq))
- 54 `LoginAck` (s2c): **MEASURED, binding INFERRED.** Accepts login; the payload is empty in the registry's PC captures.
- 161 `AccGetPlayerListReq` (c2s): **MEASURED.** Requests account characters and servers with `Acc`, `Udid`, `Lang`, and `Platform`.
- 162 `AccGetPlayerListAck` (s2c): **MEASURED.** Returns result, code, message, players, and servers.
- 8003 `TickNtf` (s2c): **MEASURED.** Server keepalive with `Tick`, `TickTS`, and `GsSendTS`.
- 8016 `TelemetryEventNtf`* (s2c): **MEASURED.** Named telemetry event with key/value details.
- 8562 `CipherSeedNtf`* (s2c): **MEASURED.** Initial cleartext frame with the `pf1` and `pf2` nonces.
- 9999 `MultiplexWrapper`* (s2c): **MEASURED.** Carries zlib-compressed envelopes and the uncompressed length.
- 61438 `MultiplexWrapperLarge`* (s2c): **MEASURED.** Same wrapper shape, observed with large single envelopes.

## City, inventory, and development

- 100 `UseItemReq` (c2s): **MEASURED.** Uses an item by ID and quantity.
- 102 `SpeedupReq` (c2s): **MEASURED.** Applies a quantity of speedup items to the selected task and building.
- 103 `ItemUpdateNtf` (s2c): **MEASURED.** Reports changed item IDs and their new totals.
- 104 `BagReq`* (c2s): **MEASURED.** Empty request for the entire inventory.
- 105 `ItemListAck` (s2c): **MEASURED, binding INFERRED.** Returns inventory item IDs and counts.
- 110 `PlayerInfoReq` (c2s): **MEASURED.** Queries a player profile by `PlayerId`, `GsId`, and `ViewerId`. Fields: ([reads](game-systems/01-state-reads.md#player-profile-mt-110--111))
- 111 `PlayerInfoAck` (s2c): **MEASURED.** Returns the player profile, including conduct in field 17, present on every profile read. Fields: ([reads](game-systems/01-state-reads.md#player-profile-mt-110--111))
- 120 `ProduceReapReq` (c2s): **MEASURED.** Collects production from the specified building.
- 121 `ResourceNtf` (s2c): **MEASURED.** Reports changed resource balances by type.
- 122 `ProduceReapAck` (s2c): **MEASURED.** Confirms a production collection request.
- 203 `GetBuildingListReq` (c2s): **MEASURED.** Requests a governor's building list. Fields: ([reads](game-systems/01-state-reads.md#city-and-buildings-mt-203--204))
- 204 `BuildingListInfo` (s2c): **MEASURED.** Returns building records and their owning governor. Fields: ([reads](game-systems/01-state-reads.md#city-and-buildings-mt-203--204))
- 207 `ResourceRefreshReq`* (c2s): **MEASURED.** Measured sends are empty and trigger an updated MT 121 resource push; a schema `f1` building-instance id is not confirmed on the wire.
- 211 `BuildingInfo` (s2c): **MEASURED.** Pushes an individual building's state.
- 300 `TrainingBeginReq` (c2s): **MEASURED.** Starts unit training and carries `Seckill` and `Discount` flags.
- 301 `TrainingBeginAck` (s2c): **MEASURED, binding INFERRED.** Acknowledges training start.
- 302 `TrainingStatusNtf` (s2c): **MEASURED.** Pushes training status and timing.
- 303 `TrainingEndReq` (c2s): **MEASURED.** Collects completed troops from a building.
- 305 `TrainingStatusReq` (c2s): **MEASURED.** Queries the training queue's status; the answer rides repeated MT 302 records. Fields: ([city](game-systems/04-city.md#train_troops-mt-300-trainingbeginreq))
- 330 `HealingBeginReq` (c2s): **INFERRED.** Starts healing selected units, with an instant-finish flag.
- 331 `HealingBeginAck` (s2c): **INFERRED.** Acknowledges healing start.
- 332 `HealingStatusNtf` (s2c): **MEASURED, binding INFERRED.** Pushes healing queue status; empty when nothing is healing.
- 336 `HealingEndReq` (c2s): **MEASURED.** Collects healed troops from a building.
- 360 `TavernDrawReq` (c2s): **MEASURED.** Requests a free, gem, or item draw from a tavern pool.
- 361 `TavernDrawAck` (s2c): **MEASURED, binding INFERRED.** Returns granted item IDs and quantities in field 1.
- 362 `TavernInfoReq` (c2s): **MEASURED, binding INFERRED.** Empty request for tavern pool information.
- 363 `TavernInfo` (s2c): **MEASURED.** Returns pool modes, free draws, and daily limits.
- 364 `TavernInfoNtf` (s2c): **MEASURED, binding INFERRED.** Pushes the updated `TavernInfo` payload, including after draws.
- 366 `HeroListReq` (c2s): **MEASURED, binding INFERRED.** Empty request for the account's commander roster. Fields: ([marches](game-systems/02-marches.md#commanders-query_hero_list-mt-366--367))
- 367 `HeroListAck` (s2c): **MEASURED.** Returns repeated `HeroInfo` records. Fields: ([marches](game-systems/02-marches.md#commanders-query_hero_list-mt-366--367))
- 450 `CourierCatalogReq`* (c2s): **MEASURED.** Requests the Courier Station catalog.
- 451 `CourierCatalogAck`* (s2c): **MEASURED.** Returns the catalog, with repeated goods in field 7.
- 452 `CourierBuyReq`* (c2s): **MEASURED.** Buys the item at a specified catalog slot.
- 453 `CourierRefreshReq`* (c2s): **MEASURED.** Requests a fresh catalog; subsequent refreshes can cost gems.
- 2088 `WarehouseProtectionNtf`* (s2c): **MEASURED.** Reports the raid-protected amount by resource type.
- 9750 `GetEquipMaterialOrdersStatusReq` (c2s): **MEASURED.** Queries material orders by building ID.
- 9751 `EquipMaterialOrderStatusNtf` (s2c): **MEASURED.** Returns order indices, states, item IDs, and counts.
- 9752 `EquipMaterialOrderAddReq` (c2s): **MEASURED.** Starts a material order with `Index`, `ItemId`, `Count`, and `Seckill`.
- 9755 `EquipMaterialOrderEndReq` (c2s): **MEASURED.** Collects completed material orders at a building.
- 9756 `EquipMaterialOrderEndAck` (s2c): **MEASURED.** Returns the collected materials as item records.

## World map, movement, and action points

- 907 `BuyPveAPReq` (c2s): **INFERRED.** Restores action points using the free or paid option.
- 908 `PveAPCost` (s2c): **MEASURED, binding INFERRED.** Reports `Cost`, `Target`, and whether the daily free option remains.
- 1001 `EnterWorldMapReq` (c2s): **MEASURED.** Empty request to enter or resynchronize the world map.
- 1002 `EnterWorldMapAck` (s2c): **MEASURED.** Returns initial castle, tick, and map dimensions.
- 1003 `WorldMapNtf` (s2c): **MEASURED.** Delivers map objects, including castles, squares, and resource nodes.
- 1004 `UpdateViewReq` (c2s): **MEASURED, binding INFERRED.** Updates the map viewport using position, level, and render mode.
- 1005 `PlayerTroopIndexNtf` (s2c): **MEASURED.** Pushes the full march index; `Action=2` deletes an entry. Fields: ([marches](game-systems/02-marches.md#the-march-index-mt-1005))
- 1007 `MoveReq` (c2s): **INFERRED.** Redirects a deployed troop using its ID and source and destination positions.
- 1008 `UnknownMoveAckCandidate1008`* (s2c): **UNKNOWN.** Unverified move-reply candidate; not seen in the registry captures.
- 1009 `MoveStopReq` (c2s): **INFERRED.** Stops a moving troop.
- 1012 `DispatchTroopReq` (c2s): **MEASURED.** Dispatches a troop to a target using the selected commanders and units.
- 1013 `DispatchTroopAck` (s2c): **MEASURED.** Returns `Code`, troop map data, and destination.
- 1014 `ReceiveTroopReq` (c2s): **MEASURED.** Recalls a troop by ID.
- 1019 `MultiReceiveTroopReq` (c2s): **INFERRED.** Recalls multiple troops.
- 1020 `MoveAck` (s2c): **INFERRED.** Acknowledges a deployed troop's move.
- 1024 `CastleUpdateNtf` (s2c): **MEASURED, binding INFERRED.** Pushes `MapCastle` and `Tick` updates, including garrison changes.
- 1027 `UnknownNodeClaimPush1027`* (s2c): **UNKNOWN.** Possible resource-node reservation push; a later capture measured a `MapRss` record in field 1.
- 1050 `CharMovePathReq` (c2s): **MEASURED.** Requests a path between source and destination objects or positions.
- 1051 `CharMovePathAck` (s2c): **MEASURED.** Returns the path as repeated `PosInfo` points.
- 1053 `UnknownMarchPush1053`* (s2c): **UNKNOWN.** March-related push of unknown purpose.
- 1083 `MapSquareMoveNtf`* (s2c): **MEASURED.** March route telemetry at ~4 Hz: `f1 MapSquareMove` (CharId, Move, TargetId), outer `f2 Tick`. Fields: ([marches](game-systems/02-marches.md#server-route-data-mapsquaref4-and-mt-1083))
- 1176 `SearchRssReq` (c2s): **INFERRED.** Searches for map resource nodes. Fields: ([marches](game-systems/02-marches.md#search_resource_nodes-mt-1176--1177))
- 1177 `SearchRssAck` (s2c): **INFERRED.** Returns map resource search results. Fields: ([marches](game-systems/02-marches.md#search_resource_nodes-mt-1176--1177))
- 1178 `SearchNPCReq` (c2s): **MEASURED.** Searches for NPC targets by type, level, value, and nearby position.
- 1179 `SearchNPCAck` (s2c): **MEASURED.** Returns search nodes, target type, level, and `ExtraCode`.
- 1234 `CastleDetailTroopNtf`* (s2c): **MEASURED.** Wounded/home troop snapshot, the same root structure as `1002.f19`: regular `f1[]` (`UnitId`, `BadHurt`), low/siege bucket `f4[]`. Fields: ([claims](game-systems/05-claims.md#healing-mt-330--332--336))
- 8031 `PVEInfo` (s2c): **MEASURED.** Pushes `AP`, `MaxAP`, `APSpeed` (float32), `BarLevel`, and the sync anchor. Fields: ([forts](game-systems/03-forts.md#action-points-mt-8030--8031-and-the-free-claim-907--908))
- 9820 `BytesKvGet` (c2s): **MEASURED, binding INFERRED.** Requests a byte value by `Key`, including saved troop loadouts.
- 9821 `BytesKvData` (s2c): **MEASURED.** Returns `Key` and the byte `value`.

## Rallies

- 1100 `CreateRallyReq` (c2s): **MEASURED.** Starts a rally against a target.
- 1101 `RallyListReq`* (c2s): **MEASURED.** Empty request for the rally list.
- 1102 `RallyListAck` (s2c): **MEASURED, binding INFERRED.** Returns the rally list, with an empty payload for an empty list.
- 1103 `CancelRallyReq`* (c2s): **MEASURED.** Empty request to cancel a rally; MT 1 carries the acknowledgement.
- 1104 `RallyPlayerTickNtf`* (s2c): **MEASURED.** Pushes player ID and a seconds counter alongside resolution mail; purpose INFERRED.
- 1105 `MapRally` (s2c): **MEASURED.** Pushes the full rally roster snapshot (captain, city object, target, members, windows). Fields: ([forts](game-systems/03-forts.md#rally-state-mt-1101--1102-and-the-mt-1105-push))
- 1108 `ReinforceRallyReq` (c2s): **MEASURED.** Sends commanders and units to reinforce a rally captain.
- 1109 `ReinforceRallyAck` (s2c): **MEASURED.** Acknowledges reinforcement and returns troop map data.
- 1112 `KickRallyReinforceReq` (c2s): **MEASURED.** Removes a reinforcement by player, captain, and troop ID.
- 1150 `RallyBriefReq`* (c2s): **MEASURED.** Rally brief query sent with the rally list request.
- 1151 `RallyBriefAck`* (s2c): **MEASURED.** Empty response to the rally brief query in captures.
- 9726 `RallyPreCreateReq`* (c2s): **MEASURED.** Empty preliminary request before creating a rally.
- 9727 `RallyPreCreateAck`* (s2c): **MEASURED.** Empty acknowledgement of MT 9726 in captures.

## Alliance

- 1126 `UnknownAllianceScreenReq1126`* (c2s): **MEASURED.** Empty request observed when opening the alliance screen.
- 1127 `UnknownAllianceScreenAck1127`* (s2c): **MEASURED.** Replies to MT 1126; payload purpose unconfirmed.
- 1231 `AsMineListReq`* (c2s): **MEASURED.** Empty request for the alliance mine list.
- 1232 `AsMineListAck` (s2c): **MEASURED, binding INFERRED.** Returns the alliance mine list.
- 2026 `AsDetailInfoReq` (c2s): **MEASURED, binding INFERRED.** Empty request for alliance details.
- 2027 `AllianceDetailAck`* (s2c): **MEASURED.** Returns alliance details, including roster and boards.
- 2045 `AsHelpDoReq` (c2s): **MEASURED.** Offers alliance help; captures set `IsAuto` to false.
- 2046 `AsHelpDoAck` (s2c): **MEASURED.** Returns helped entries, `HelpCnt`, and `MaxHelpCnt`.
- 2047 `AsHelpCan` (s2c): **MEASURED, binding INFERRED.** Pushes whether help is available and the help count.
- 2048 `AsHelpRequestReq` (c2s): **INFERRED.** Requests help for an activity by `Type`, `Id`, and `Level`.
- 2050 `AsHelpRequestAck`* (s2c): **MEASURED.** Replies to a help request; also observed empty at login.
- 2054 `AsWarehouseHarvestReq`* (c2s): **INFERRED.** Collects output from the alliance warehouse.
- 2065 `AsGiftReq` (c2s): **MEASURED, binding INFERRED.** Empty request for the alliance gift inventory.
- 2066 `AsGiftAck` (s2c): **MEASURED.** Returns alliance gift information and member gifts.
- 2067 `AsOpenGiftReq` (c2s): **MEASURED.** Opens alliance gifts by repeated gift ID.
- 2068 `AsOpenGiftAck` (s2c): **MEASURED, binding INFERRED.** Returns gift records and expired IDs; can also arrive as a push.
- 2077 `AsTechniqueReq`* (c2s): **MEASURED.** Empty request for alliance technology status.
- 2078 `AsTechnique` (s2c): **MEASURED, binding INFERRED.** Returns technology IDs and levels.
- 2080 `AsTechDonateReq` (c2s): **INFERRED.** Donates to alliance technology or skills.
- 2081 `AsTechDonateAck` (s2c): **INFERRED.** Returns points, funds, and donation cooldown details.
- 2092 `AsTechScreenReq`* (c2s): **MEASURED.** Empty request for technology screen data.
- 2094 `AsTechniqueListNtf`* (s2c): **MEASURED.** Returns the list of technology nodes.
- 2109 `AsDailySupplyClaimReq`* (c2s): **INFERRED.** Claims daily alliance supplies by supply ID.
- 2110 `AsDailySupplyClaimAck`* (s2c): **INFERRED.** Acknowledges the supply claim; payload undocumented.
- 2140 `AsActiveResearchNtf`* (s2c): **MEASURED.** Returns an active research node and timing.
- 4101 `UnknownAllianceScreenReq4101`* (c2s): **MEASURED.** Alliance-screen request with captured IDs of unknown meaning.
- 4102 `UnknownAllianceScreenAck4102`* (s2c): **MEASURED.** Replies to MT 4101; payload purpose unconfirmed.
- 8412 `AsHelpCountNtf` (s2c): **INFERRED.** Pushes help counts, including `MaximumCount`.

## Quests, events, rewards, and mail

- 140 `QuestListReq` (c2s): **MEASURED, binding INFERRED.** Empty request for the quest list. Fields: ([reads](game-systems/01-state-reads.md#quests-mt-140--141))
- 141 `QuestListNtf` (s2c): **MEASURED, binding INFERRED.** Returns the quest list. Fields: ([reads](game-systems/01-state-reads.md#quests-mt-140--141))
- 142 `QuestUpdateNtf` (s2c): **MEASURED.** Pushes changed `Quest` records.
- 143 `QuestGetRewardReq` (c2s): **MEASURED, binding INFERRED.** Claims a quest reward.
- 144 `QuestChestRewardReq` (c2s): **INFERRED.** Claims a quest chest by `ChestId`.
- 1206 `PlayerEventNtf` (s2c): **MEASURED, binding INFERRED.** Pushes event progress as repeated `PlayerEvent` records.
- 2600 `ConductReq`* (c2s): **MEASURED.** Empty request for conduct status.
- 2601 `ConductAck`* (s2c): **MEASURED.** Returns conduct score and whether a reward is claimable.
- 3026 `ExpeditionChestClaimReq`* (c2s): **INFERRED.** Claims an expedition chest.
- 3032 `ExpeditionShopBuyReq`* (c2s): **INFERRED.** Buys an expedition shop item by slot.
- 7909 `MailNtf` (s2c): **MEASURED, binding INFERRED.** Pushes a `MailEntity`, such as a rally result or refund. Fields: ([forts](game-systems/03-forts.md#outcomes-arrive-as-mail-not-on-the-march))
- 7920 `MailGetReq` (c2s): **MEASURED.** Requests a mail page by `Box`, `Cnt`, `FromTimeStamp`, and `Key`. Fields: ([reads](game-systems/01-state-reads.md#mail-mt-7920--7921))
- 7921 `MailGetAck` (s2c): **MEASURED.** Returns mail records and `LeftCount`. Fields: ([reads](game-systems/01-state-reads.md#mail-mt-7920--7921))
- 8057 `TribalCaravanCollectReq`* (c2s): **MEASURED.** Empty request to collect the tribal caravan reward.
- 8058 `TribalCaravanCollectAck`* (s2c): **MEASURED.** Acknowledges caravan collection; nested payload details are unconfirmed.
- 8600 `VipNtf` (s2c): **MEASURED.** Pushes VIP level, experience, and claim state.
- 8601 `VipGiftGetReq` (c2s): **MEASURED, binding INFERRED.** Empty exclusive VIP chest claim observed with an MT 103 inventory update.
- 8602 `VipFreePointGetReq` (c2s): **MEASURED, binding INFERRED.** Empty daily VIP chest claim observed with MT 8600 VIP state.

## Registry conflicts and limits

- MT 1's field 1 is `Api`, not the code. MT 161 is the player-list request, not a second key exchange. MT 8003 is `TickNtf`, not a separate server heartbeat type.
- MT 105 is the bag, not research. MTs 450/451 are the Courier Station. MT 103 pushes new item totals; MT 1 confirms MT 100 and MT 102. MT 207 triggers a resource refresh, not a building detail read. MT 8601 is the exclusive VIP chest and MT 8602 the daily chest.
- MT 1104 is not the cancel acknowledgement for MT 1103. MT 1105 field 4 is `CaptainCharId`, not a rally ID. MT 1112 kicks a reinforcement; the alternate MT 1114 assignment is unverified. MT 9727 is empty in captures; loadouts come via MT 9821.
- MT 2047 pushes help availability, not a heartbeat. MT 2088 reports warehouse protection, not timing. MT 8031 reports action points, not map position. Alliance help is on this socket (2045/2046; 2048/2050/8412 has mixed evidence). MT 1206's link to MT 2045 is not confirmed by the registry captures. MTs 2065 to 2068 are alliance gifts, not the unregistered MTs 1700/1711. MT 142 is a push; MT 143 claims the quest reward.
- MT 1012 has two candidate refusal channels (MT 1 and MT 1013). Batch presence is not fixed by MT. The extended length field is four bytes, not eight. MT 1004's captured PC `RenderMode` was 1; an older builder used 2. `Seckill` names the instant-finish flags in training, healing, and material orders.
- The registry chooses send order `4, 1, 2, 3` for MT 2080 and `1, 3, 2, 13, 4, 8, 10, 15, 9` for MT 1007 over other builder orders. Live payloads have fields absent from schema 1.1.522: MT 111 top-level 16 to 19; 1005 troop field 31; 1012 top-level 18; and 1013/1109 troop field 65.

## Appendix: additional observed MTs

> [!IMPORTANT]
> These values are outside the 142-row registry. Every row below has **no bound schema** in the engine. Where a name matches a protobuf type, it describes the candidate payload and does not install a decoder.

`fN` is protobuf field number N, and `[]` means repeated. **Measured** means the stated role was observed. **Inferred** means the role rests on indirect evidence. **Unestablished** means the traffic was seen but its role or mapping is not known. A `?` direction means the notes do not establish who sent it.

### Session, account and conduct

| MT | Direction | Name or label | Payload | Evidence |
| ---: | --- | --- | --- | --- |
| 8 | c2s | Analytics event | Analytics payload; field layout not established. | **Inferred** |
| 15 | s2c | Login state reply (`LoginAck` in notes) | Reply to MT 14 carrying account state, including City Hall level in `f35`; do not equate it with registered MT 54. | **Measured** |
| 22 | s2c | `SwitchFunctionNtf` | Feature switches including `formation_switch=1` for armaments. | **Measured** |
| 30 | c2s | AP sync request | The observed sync prompts MT 8031; payload meaning not established. | **Measured** |
| 904 | s2c | `BuffNtf` | Source key in `f1` and buff nodes with amounts and expiration times. | **Measured** |
| 930 | c2s | Login flood follow-up | Sent after the login burst; purpose not established. | **Unestablished** |
| 2603 | c2s | Claim conduct reward | Empty request. | **Measured** |
| 2604 | s2c | Conduct reward result | Observed reply: 36 bytes. | **Measured** |
| 8035 | c2s | Login flood follow-up | Sent after the login burst; purpose not established here. The [Canyon reads](game-systems/01-state-reads.md#canyon-panels-9606--9607-9608--9609-9613--9614) discuss 8035/8036 as saved-lineup transports. | **Unestablished** |
| 8603 | c2s | VIP shop catalog request | Field-mapped in [claims](game-systems/05-claims.md#caravan--vip-mt-8057--8601--8602-and-the-shop-8603--8605). | **Measured** |
| 8604 | s2c | VIP shop catalog response | Field-mapped in [claims](game-systems/05-claims.md#caravan--vip-mt-8057--8601--8602-and-the-shop-8603--8605). | **Measured** |

### City, inventory, research and commanders

| MT | Direction | Name or label | Payload | Evidence |
| ---: | --- | --- | --- | --- |
| 200 | c2s | Unlock builder | Slot ID string, payment selector and an item/count record; observed slot `"2"`. | **Measured** |
| 201 | c2s | Place building | `f1` building type, `f2` city-grid coordinate pair, `f3=0` observed. | **Measured** |
| 202 | s2c | Place building result | Building record with a newly assigned instance ID, level `0` and state `3` observed. | **Measured** |
| 205 | c2s | Upgrade building | `f1` instance ID string, `f2` flag `0` or `1`, `f3=0` observed. | **Measured** |
| 206 | s2c | Upgrade building result | Updated building record with timing and cost data when present. | **Measured** |
| 213 | c2s | Unplace building | `f1` instance ID string, `f2` coordinate pair `(-99,-99)` in the captured removal. | **Measured** |
| 214 | s2c | Unplace building result | Updated building record with position `(-99,-99)` and state `1` observed. | **Measured** |
| 219 | s2c | Builder roster | Repeated `f1` records with slot ID, assigned building or empty string, and source. | **Measured** |
| 250 | c2s | Research snapshot request | Requests the current technique state during login. | **Measured** |
| 251 | s2c | `TechniqueAck` | Research state with technique nodes. | **Measured** |
| 258 | s2c | `TechniqueNodeNtf` | Research node state update. | **Measured** |
| 337 | s2c | Healing status push | `HealingStatus` at the root rather than wrapped as in MT 331. | **Measured** |
| 365 | s2c | `HeroUpdateNtf` | One whole `HeroInfo`; a live XP level-up was not captured. | **Inferred** |
| 371 | s2c | Commander skill upgrade result | Skill upgrade response; complete shape not established. | **Inferred** |
| 403 | s2c | `HeroListUpdateNtf` | Repeated commander records in `f1`. | **Inferred** |
| 600 | c2s | Login-window request | Observed near the inventory exchange with a generic acknowledgment; purpose not established. | **Unestablished** |
| 1160 | s2c | Expiry-bearing update | Expiry sentinel `18446744073709551615` (2^64-1) means absent, not a timestamp. Exact payload role unestablished. | **Unestablished** |
| 5600 | s2c | Login-window message | Seen near the inventory snapshot; payload meaning not established. | **Unestablished** |
| 8030 | c2s | AP query (`PVEInfoReq`) | Empty request; provokes a fresh MT 8031 AP push. | **Measured** |
| 8045 | s2c | Login-window message | Seen near the inventory snapshot; payload meaning not established. | **Unestablished** |

### Alliance, chat, kingdoms and rankings

| MT | Direction | Name or label | Payload | Evidence |
| ---: | --- | --- | --- | --- |
| 220 | c2s | Full ranking request | Board key string (example form `Gs:<raw-server>:<board>`) and request selectors. | **Measured** |
| 221 | s2c | Full ranking reply | `f1[]` ranked rows and `f2` the caller's own row. | **Measured** |
| 227 | c2s | Ranking preview request | `f1[]` board-key strings; nine keys in the observed menu request. | **Measured** |
| 228 | s2c | Ranking preview reply | One first-place row per requested board. | **Measured** |
| 229 | s2c | Own ranking values | Caller values for nine boards, without rank positions. | **Measured** |
| 555 | c2s | Kingdom ruler request | Kingdom/server ID varint. | **Measured** |
| 556 | s2c | Kingdom ruler reply | Ruler and alliance summary, not the full alliance profile. | **Measured** |
| 557 | c2s | Immigration information request | Kingdom/server ID varint. | **Measured** |
| 558 | s2c | Immigration information reply | Kingdom ID, immigration gate and power cap details. | **Measured** |
| 560 | c2s | Additional kingdom information request | Kingdom/server ID varint. | **Measured** |
| 561 | s2c | Additional kingdom information reply | Per-kingdom scalars including a positively matched rising-kingdom marker in `f6`; other roles unresolved. | **Inferred** |
| 562 | c2s | Kingdom season request | Empty request for the global season table. | **Measured** |
| 563 | s2c | Kingdom season reply | Kingdom-to-season rows. | **Measured** |
| 591 | c2s | Global constants request | Empty request; intended feature not established. | **Unestablished** |
| 592 | s2c | Global constants reply | Ten scalars including a Unix timestamp; meanings not established. | **Unestablished** |
| 1700 | c2s | Unresolved service request | Observed ack status `13299`. | **Unestablished** |
| 1711 | c2s | Unresolved service request | Empty request acknowledged with status `13299`; alliance-help mapping was not established. | **Unestablished** |
| 2023 | c2s | Alliance list request | `f1 Lang`, `f2 Open=1`; the answer's repeated `f1` records carry id/name/abbr/members/power. | **Measured** |
| 2039 | c2s | Login-window request | Seen near the inventory exchange and generically acknowledged; purpose not established. | **Unestablished** |
| 2063 | c2s | Full alliance profile request | `f1` alliance ID varint. | **Measured** |
| 2064 | s2c | Full alliance profile | One nested record with identity, owner, membership and alliance attributes. | **Measured** |
| 3303 | c2s | Cross-kingdom event table request | Empty request. | **Measured** |
| 3304 | s2c | Cross-kingdom event table reply | Active events with map, story, slots and tiers. | **Measured** |
| 9911 | c2s | Send chat | Channel and decorated message record; event invitations use the same MT. | **Measured** |
| 9912 | s2c | Send chat acknowledgment | Small send acknowledgment; distinct from a chat-history response. | **Measured** |

### World map, marches and scouting

| MT | Direction | Name or label | Payload | Evidence |
| ---: | --- | --- | --- | --- |
| 1006 | s2c | Shield state push | `f1 CharId`, end `f2`, item window `f4/f5`, system window `f6/f7`. | **Measured** |
| 1017 | c2s | `MultiMoveReq` | Repeated march IDs with NPC target ID, attack flags and optional world-position pin in `f9`. | **Measured** |
| 1021 | s2c | `MoveAck` | Acknowledges grouped movement with returned march movement data. | **Measured** |
| 1022 | s2c | Coordinate update | Coordinate traffic mentioned as ambient; complete shape not established. | **Inferred** |
| 1137 | ? | March-adjacent message | Mentioned with MT 1138; the similar-looking fields are not troop-capacity limits. | **Unestablished** |
| 1138 | ? | March-adjacent message | Mentioned with MT 1137; payload role not established. | **Unestablished** |
| 1621 | c2s | Auto-scout request | Auto-scout command; the overview does not establish the field layout. | **Inferred** |
| 1622 | s2c | Auto-scout reply | Paired reply to MT 1621; the overview does not establish the field layout. | **Inferred** |

### Council, travel and expedition

| MT | Direction | Name or label | Payload | Evidence |
| ---: | --- | --- | --- | --- |
| 3025 | s2c | Expedition menu marker | Possible reward availability marker; zero does not prove an earlier claim. | **Inferred** |
| 3027 | s2c | `GetScenarioRewardAck` | `f1.f1[]` loot records as `{type, subtype, value}`. | **Measured** |
| 3028 | c2s | Expedition shop request | Empty request. | **Measured** |
| 3029 | s2c | Expedition shop reply | Shop slots in `f5[]` with slot ID in `f6`, not the item ID. | **Measured** |
| 4032 | c2s | Council quest dispatch | Timed quest assignments with quest IDs and the client-selected commander lists. | **Measured** |
| 4921 | s2c | Armament inventory push | New books in `f1[]`, including book stats and inscription IDs. | **Measured** |
| 4931 | s2c | Council board | Board, quest states, dispatch counters and gift points. | **Measured** |
| 4934 | c2s | `FormationQuestCommonReq` | Quest or milestone ID in `f1`; optional `OneClick` in `f2` absent in the examples. | **Measured** |
| 4935 | s2c | Council claim reply | Quest IDs in `f1[]`, optional loot in `f2[]` and books in `f3[]`. | **Measured** |
| 4939 | s2c | Council-related push | Twenty four-scalar `f1` records in one observation, not the gift-point counter. | **Unestablished** |
| 5160 | c2s | Travel state request | Empty request. | **Measured** |
| 5161 | s2c | Travel state reply | Partner ID, usage and max counts, unlock flag and luck counter. | **Measured** |
| 5162 | c2s | Select travel partner | `f1` commander template ID. | **Measured** |
| 5163 | s2c | Select travel partner reply | Empty acknowledgment. | **Measured** |
| 5164 | c2s | `TravelStartReq` | Batch count and travel mode, without a commander ID. | **Measured** |
| 5165 | s2c | Travel result | Loot, story entry IDs, armament books and luck increments. | **Measured** |
| 5176 | c2s | Unresolved council-window request | Empty payload. | **Unestablished** |
| 5177 | s2c | Unresolved council-window reply | Four scalar fields observed as `{1,1,0,0}`. | **Unestablished** |
| 7604 | s2c | Expedition menu update | First `f1[]` entry changes `f8` from `1` to `0` around the reward claim; exact semantics are not settled. | **Inferred** |

### Events, Canyon and battle pass

| MT | Direction | Name or label | Payload | Evidence |
| ---: | --- | --- | --- | --- |
| 533 | c2s | Open event tab | `f1` plaintext event-name string, `f2` observed as empty bytes. | **Measured** |
| 1201 | s2c | Event configuration | Event definitions containing item IDs, action keys and drop sources. | **Measured** |
| 1651 | c2s | Read own event tree | Empty request. | **Measured** |
| 1652 | s2c | Own event tree state | Tree state including claimable pool and counters. | **Measured** |
| 1653 | c2s | Claim own event tree | Empty request. Observed claims used the requester’s pool. | **Measured** |
| 1654 | s2c | Own tree claim result | Reward records of the same shape as visitor claims. | **Measured** |
| 1655 | c2s | Collect from another tree | One governor ID in the request; also used across paired kingdoms. | **Measured** |
| 1656 | s2c | Visitor collection result | Reward record with the running daily total in `f4`. | **Measured** |
| 1659 | c2s | Event-window request | Request listed with MT 1660; exact purpose not established. | **Unestablished** |
| 1660 | s2c | Event-window reply | Response listed with MT 1659; exact payload meaning not established. | **Unestablished** |
| 1661 | c2s | Read another event tree | One governor ID in the request. | **Measured** |
| 1662 | s2c | Another tree's state | Governor and tree details; several scalar meanings remain uncertain. | **Measured** |
| 1665 | s2c | Event tree state push | Same shape as MT 1652. | **Measured** |
| 1667 | s2c | Alliance event tree listing | `f1[]` per-governor rows with names, pool counts and other event state. | **Measured** |
| 1669 | s2c | Event tuning table | Plaintext names and scalar event limits, chances and pool values. | **Measured** |
| 1670 | c2s | Event kingdom pairing request | Two-byte request; selector meaning not established. | **Measured** |
| 1671 | s2c | Event kingdom pairing reply | Own and paired kingdom IDs in `f1` and `f2`; other scalars unresolved. | **Measured** |
| 1675 | c2s | Event-panel request | Listed with MT 1676; purpose not established. | **Unestablished** |
| 1676 | s2c | Event-panel reply | 1,022-byte panel reply; payload meaning not established. | **Unestablished** |
| 3743 | c2s | Event visit-key request | Empty request; the answer's short string is the event-tree visit key. | **Measured** |
| 3744 | s2c | Event visit-key reply | One eight-character string: the visit key for event-tree visits. | **Measured** |
| 7000 | c2s | Anniversary free reward claim | Requests the anniversary gift. | **Measured** |
| 7001 | s2c | Anniversary free reward reply | Repeated `{type,id,count}` reward records. | **Measured** |
| 9605 | s2c | Canyon attempt limits | `f1` normal attempts, `f4` normal cap, `f7` nested Lost Canyon attempts and cap. | **Measured** |
| 9606 | c2s | Canyon state request | `f1=0` Canyon or `f1=1` Lost Canyon. | **Measured** |
| 9607 | s2c | Canyon state reply | Returns the requested mode's state. | **Measured** |
| 9608 | c2s | Canyon leaderboard request | Asks for the event board. | **Measured** |
| 9609 | s2c | Canyon leaderboard | Ranked rows including the player's own row, not a list of opponents. | **Measured** |
| 9610 | c2s | Canyon battle request | Requests a fight against the selected opponent. | **Measured** |
| 9611 | s2c | Canyon battle reply | Rank in `f3`; `f12=1` win or `0` loss. | **Measured** |
| 9612 | c2s | Canyon lineup setter | Sends the starting lineup when no lineup is set. | **Measured** |
| 9613 | c2s | Canyon replay request | Requests the record of a prior Canyon battle. | **Measured** |
| 9614 | s2c | Canyon replay reply | Returns the battle replay. | **Measured** |
| 9615 | c2s | Canyon opponent selection | Default opponents required no selection in the observed flow. | **Measured** |
| 9616 | s2c | Canyon opponent selection reply | Confirms opponent selection. | **Measured** |
| 9801 | s2c | Battle-pass level state | Tier definitions in full snapshots and level/XP in short updates. | **Measured** |
| 9802 | c2s | Claim battle-pass tiers | Batched claim of available tier rewards. | **Measured** |
| 9804 | c2s | `BattlePassGetTaskRewardReq` | One challenge reward claim, with single challenge ID and its task category. | **Measured** |
| 9805 | s2c | Battle-pass task state | Matching task-state record; status `-2` observed after claiming. | **Measured** |
| 9808 | s2c | Battle-pass tier claim reply | Cumulative `f1[]` entries `{Level,FreeGet,PayGet}`. | **Measured** |
| 9875 | s2c | Event shop state | Shop offers, tier and spending progress with timed gates. | **Measured** |
| 9877 | c2s | Event shop purchase | Entry ID in `f2`, not the delivered item ID. | **Measured** |

## Results and failures

Appendix rows document observations, not send support or schema bindings.

- Preserve raw fields and direction.
- Missing fields are not zero.
- Capture position alone does not establish purpose.

## Checks

- Check the decimal MT against the registry before adding an appendix row.
- Verify capture direction and outer MT. For multiplexed traffic, check each inner envelope.
- Cite payload-role evidence. Adjacency alone does not establish a reply.

---

<div align="center">

[← Identity and Fingerprint](03-identity.md) &nbsp;·&nbsp; [↑ The Headless Method](README.md) &nbsp;·&nbsp; [Game Systems →](game-systems/README.md)

</div>
