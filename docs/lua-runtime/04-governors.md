<sub>[RoK Toolkit](../../README.md) › [Docs](../README.md) › [Lua Runtime](README.md) › Governors and Reloads</sub>

<img src="../assets/headers/lua-runtime.svg" width="100%" alt="">

# Governors and Reloads

**On this page:** [Reload timings](#reload-timings) · [Client surface](#client-surface) · [Governor list](#read-the-governor-list) · [Switch governor](#switch-governor-on-the-current-account) · [Refusals](#refuse-or-fail-the-governor-switch) · [Stored account](#switch-stored-account-through-the-games-page) · [Tap to start](#tap-to-start-screen) · [Roster scan](#scan-the-roster-across-stored-accounts) · [Reload or restart](#reload-versus-tool-restart) · [Source gaps](#source-gaps)

## Reload timings

| Transition | Timing | Evidence |
| --- | --- | --- |
| Governor switch to new login | About 10 s | **MEASURED** |
| New governor login to loading done | About 18 s after login, not from switch | **MEASURED** |
| Stored-account switch through logout and restart | 15 to 20 s | **MEASURED** on valid stored accounts |
| Tap to start screen seen to tap | 1.5 to 4 s; once per screen | **MEASURED** |
| Before a governor switch | Stop loops, hold reads for 15 s, remove hooks before invoking the confirm call | **CODE**; 15 s is the client's request timeout |
| While rebuilding | Only the raw identity read and the guarded Tap to start screen path run; other reads refuse | **MEASURED** reload hold |
| After the new governor loads | Hold the loaded identity for 4 s before releasing ordinary reads; re-establish hooks from the new client objects | **CODE** |

## Client surface

| To get | Literal function or raw client field | Evidence / limit |
| --- | --- | --- |
| Governor list | `AccountServerRoleData.RolesList` | **MEASURED**; switch-screen list in game order |
| Governor-switch confirmation | `AccountServerRoleHandler:AccLoginPlayerReqUrl1(ServerId, PlayerId, AppUid, Region)` | **CODE** call; governor switching watched live |
| Current governor | `UserData.PlayerID` | **MEASURED**; zero until login, table absent during part of reload |
| Loading done | `GameLoadingMgr.curStep == -1`, with a nonzero current governor | **MEASURED** identity read; `-1` alone is insufficient |
| Account switch page | `LimPCHelperLua:AccountSwitchPageOpen()` | **CODE** call; page opening watched live; native call is `LimPCHelper.AccountSwitchPageOpen()` |
| Select a stored account | **UNKNOWN** named client callable; use the game's account chooser by unique exact display name | **MEASURED** selection flow; stored-list ordinal is not a login argument |
| Press Start on the account page | **UNKNOWN** named client callable; press the game's own Start control once | **MEASURED**; `sent` means the press was dispatched, not that login succeeded |
| Tap to start | The current screen button's `onClick:Invoke()` after its normal button checks | **MEASURED** click path; do not call `StateTapToStart:DoStartGame()` directly |

## Read the governor list

1. Wait until `AccountServerRoleData` and its `RolesList` exist; absent is not an empty roster.
2. Walk `RolesList` by index, up to 64 entries, and use `rawget` for each entry's fields; the array was plain but its entries carried metatables.
3. Keep the game's order: on sale, favourites, then power; do not sort a second independent list.
4. Read the target's `PlayerId`, `ServerId`, `Region`, name, kingdom, power, City Hall, `BanUntil` and sale state for selection and refusal checks.
5. Keep `AppUid` inside the client for the confirm call; never return `AppUid`, `Key`, `LastLoginIP` or the SDK token.

## Switch governor on the current account

1. Read the current governor and target list entry; refuse a current, unknown, banned or past-sale target before stopping or unhooking anything.
2. Refuse while disconnected, another switch or reload is in progress, or a required read is unavailable.
3. Stop the running loops and hold ordinary reads for 15 s so outstanding client requests can settle.
4. Remove the tool's hooks while still on the original governor; do not retain the old hooks for the rebuilt client.
5. Recheck the switch guards, then call `AccountServerRoleHandler:AccLoginPlayerReqUrl1(ServerId, PlayerId, AppUid, Region)` with that target entry's own fields, exactly as the game's confirm button does.
6. Follow the reload with the raw identity read every 2 s; type-check the tables before reading `UserData.PlayerID` and `GameLoadingMgr.curStep`.
7. On the Tap to start screen, wait 1.5 to 4 s, recheck the current screen and button, and invoke that button's `onClick:Invoke()` once.
8. Require the requested governor's nonzero identity and `curStep == -1`, then keep that loaded identity for 4 s.
9. Re-establish hooks from the rebuilt client, release ordinary reads, read and report the new governor's identity; leave every action loop stopped.

## Refuse or fail the governor switch

| Condition | Result |
| --- | --- |
| Target is the current governor | Refuse before changing anything |
| Target absent from the game's list | Refuse; no invented target entry |
| Target banned | Refuse before invocation |
| Target past being put on sale | Refuse before invocation; stricter than the game's static-sale check |
| No restart within 30 s | Failed; the confirm call may have been refused and the game shows its own error |
| Reload returns to the same governor or lands on a third governor | Failed, not a successful switch to the requested target |
| Requested governor not loaded within 3 minutes | Failed |
| Invocation answer absent | Follow identity anyway; the call may already have reached the game |
| A table disappears during reload | Hold; no client getter runs through a partially rebuilt world |

## Switch stored account through the game's page

1. Use an account already signed in once in this game installation; read names only and require a unique, nonblank display name.
2. Re-read the stored names immediately before selection; a changed order never changes which display name was requested.
3. Stop loops, hold reads and remove hooks before the switch can rebuild the client; keep the game's second-password gate.
4. Call `LimPCHelperLua:AccountSwitchPageOpen()` to open the game's account switch page, not its Add account sign-in page.
5. If the game requests its second password, stop and report; the player completes the prompt and chooses manually, with no automatic continuation.
6. Open the page's chooser, pick the unique exact display name, and verify that the page's displayed selection committed to that name.
7. Recheck that the game's Start control is present, visible, enabled and not blocked by a modal, then press it once; do not call a separate SDK login function.
8. Report `sent` only for that Start press, then follow logout, restart, the raw identity read and the Tap to start screen as in the governor-switch flow.
9. Read the resulting governor identity separately; the game's own screen is authoritative if login fails, including a stored account whose password changed.

| Page result or gate | Meaning |
| --- | --- |
| `opened` | The game's account switch page opened |
| `password` | The game asked for its second password; stop |
| `sent` | Start pressed, not logged in |
| Captcha or verification | Stop and leave the game's check to the player |
| Missing, duplicate, hidden, disabled, changed or modal-blocked controls | Refuse; do not guess another selection or press Start again |
| Account credentials | No password or token returned or supplied; only stored account display names |

## Tap to start screen

1. Read whether the current state is `StateTapToStart`, the screen is loaded, the splash is waiting and no exit animation has started.
2. After 1.5 to 4 s, check the button is still alive, interactable and active in the hierarchy.
3. Invoke the button's own `onClick:Invoke()` once; let its listener record `SplashFinish`, play the sound and animation, save the splash-completed state, and start loading when the animation ends.
4. If the player pressed first, do nothing; do not invoke any listener component or `DoStartGame()` directly.

| Reload origin | Tap rule |
| --- | --- |
| Tool-requested switch | Tap after the screen appears |
| Game-initiated opening or reload | Tap only while the player's Tap to start option is on |
| Same screen still present after a tap | No second tap |

## Scan the roster across stored accounts

1. Stop every action loop; refuse if a switch, reload or another scan is busy.
2. Record the current account by its known stored display name; if it is not known, record that returning to it cannot be guaranteed.
3. Select one opted-in stored account whose governor list has not been read; refuse unknown or non-opted-in accounts.
4. Run the account-page flow above, then wait for the raw identity read to show a loaded governor; `sent` alone is not sufficient.
5. Read `AccountServerRoleData.RolesList` with the same indexed raw read, preserving the game's order; the scan allows up to ten looks, 3 s apart.
6. Associate that list with the stored display name actually selected; keep governor ids, names, kingdom, power, City Hall, ban and sale state, not credentials.
7. Switch back to the starting account when its display name is known; otherwise leave the game on the visited account and report that explicitly.
8. Repeat one account at a time, switching back after each; leave action loops stopped.

## Reload versus tool restart

| Event | What resumes |
| --- | --- |
| Governor switch or game reload | No auto-start. Client objects replaced |
| Tool restart with the same governor | Only explicitly running work can resume through Start |
| Saved work belongs to another governor | Wait for its governor |
| Saved dry run whose gate closed, or live loop whose gate locked | Drop and report |
| Settings push or lagging phase display | Never starts a stopped loop |

## Source gaps

| Surface | **UNKNOWN** |
| --- | --- |
| Account chooser | A named client selection function; measured flow uses the game's display-name chooser |
| Account-page Start | Named client Start function. Measurement confirms the press, not login |
| Ban and sale thresholds | Exact raw predicates unresolved. No invented enum values |

---

<div align="center">

[← Invoking the Client](03-client-calls.md) &nbsp;·&nbsp; [↑ The Lua Runtime Method](README.md)

</div>
