# Friends and association lists

## Current status

The friend system is only partially wired and is not yet proven end to end.
Build 103 bootstraps the social overlay with AssociationLists component `25`,
command `6` (`getLists`). Both attached and detached captures reach Darkspin
with `ALST`, `MXRC`, and `OFRC`, and Darkspin replies with `LMAP`. The runtime
also has durable per-user association storage and handlers for command `1`
(`addUsersToList`) and command `2` (`removeUsersFromList`).

The 2026-08-28 Rawr/Foo capture reaches command `1`, persists the requested
association, and then crashes Rawr on the returned mutation traffic. The later
captures separate the two unsafe cases: an unnormalized nonempty response and a
self-directed 71-byte membership notification. Darkspin now returns the
generated response using the canonical account identity, omits that local
notification, and reports an already-persisted repeat through the native
`MEMBER_ALREADY_IN_THE_LIST` error. Commands `3` (`clearLists`), `4`
(`setUsersToList`), `5` (`getListForUser`), `7` (`subscribeToLists`), and `8`
(`unsubscribeFromLists`) are still empty server stubs. Mutual friend-request,
acceptance, reciprocal-list projection, and visible client confirmation
therefore remain unverified or unimplemented.

## Client contract recovered so far

The executable registers `getFriendsAsJSON`, `followPlayer`, and
`unfollowPlayer` beside the lobby and blocked-player bindings. Its generated
AssociationLists proxy names the RPCs exactly as follows:

- `1`: `addUsersToList`
- `2`: `removeUsersFromList`
- `3`: `clearLists`
- `4`: `setUsersToList`
- `5`: `getListForUser`
- `6`: `getLists`
- `7`: `subscribeToLists`
- `8`: `unsubscribeFromLists`
- notification `1`: `NotifyUpdateListMembership`

The relevant generated DTOs are `ListMemberId`, `ListIdentification`,
`ListMemberInfo`, `ListMemberInfoUpdate`, `ListInfo`, `ListMembers`, `Lists`,
`UpdateListMembersRequest`, `UpdateListMembersResponse`, `GetListsRequest`, and
`UpdateListWithMembersRequest`. Before implementing more behavior, recover each
visitor's exact field order and wire type from `Game.c`; the Rooms failure
demonstrates that a structurally plausible but noncanonical TDF sequence can be
silently ignored by the generated forward-only decoder.

The `ListMembers` visitor is now exact: every `LMAP` list entry contains
`INFO` as a `ListInfo` struct (`LNM`, then `TYPE`), followed by the `MEML` list,
integer `OFRC`, and integer `TOCT`. Flattening `LNM` and `TYPE` directly into
the `ListMembers` entry let the social panel render an empty `MEML` while the
chat badge retained an invalid total count. The build-103 visitor at
`Game.c:2279985-2280070` proves the wrapper, order, and types.

The mutation DTOs are also exact. `UpdateListMembersResponse` visits the added
`LMID` vector followed by the removed `REM` vector. `ListMemberInfo` contains
`LMID` followed by `TIME`, and `ListMemberId` contains `BLID`, `PNAM`, `XREF`,
then `XTYP`. Most importantly, each notification `BIDL` entry is a
`ListMemberInfoUpdate`: its fields are an `INFO` wrapper around the complete
member followed by `LUPT`. The prior flattened `LMID`, `TIME`, `LUPT` entry was
not the generated notification contract.

The client owns the social indicator calculation. `sub_C2C640` retrieves
association list type `5`, iterates its members, and counts only users whose
UserSessions state equals online (`2`). It runs after the initial `getLists`,
after a friend presence event, and from the add/remove completion callbacks.
Consequently, Darkspin treats type `5` as the friend list and follows a
successful generated response with the friend's current UserSessions update.
The mutating client does not receive `NotifyUpdateListMembership` for its own
RPC; that duplicate path crashes build 103. `OFRC` remains the response page
offset and `TOCT` remains the total list size; neither field is the online badge
count.

The 2026-08-28 online-status capture proves both peers receive each other's
UserSessions command-2 identity and command-5 update. The update incorrectly
carried `FLGS=2`. Client accessor `sub_DFF280` returns offline unless bit zero is
set, and returns online only when bits zero and one are both set, so live users
must be projected with `FLGS=3` on every command-5 update. Login, presence,
lookup, and association refreshes now use that pair. Room membership previously
sent a later `FLGS=2` update for the same cached user and overwrote the online
state whenever friends met in the lobby; room visibility now uses the same
canonical online update. When a user's final authenticated Blaze session
disconnects, every remaining authenticated session receives `FLGS=1`. The
client retains that known user in the durable type-5 friend list but excludes
them from the online state and badge count. Disconnecting one of multiple live
sessions does not falsely mark the account offline. The packaged social page
loads each row image from `/game/service/png?account_id=<Blaze ID>`, including
offline rows. That endpoint resolves the durable public profile rather than
only active users, so logging out no longer replaces the account's selected
avatar with the generic blue placeholder; responses require revalidation so a
previous placeholder is not retained for five minutes.

Build 103's `cBlazeFriendsManager` listens for UserSessions extended-data
changes, verifies that the changed user belongs to association list type `5`,
and immediately publishes the row-update event consumed by the open Social
page. Area, activity, party-size, level, and progression changes therefore use
the existing cached identity: Darkspin sends command `5` to retain its online
flags followed by command `1` with the new presence. Repeating command `2`
(`userAdded`) for an already-known user replaced the cache entry while an open
association row could remain bound to the old object, leaving that row stale
until the page rebuilt it; identity-add notifications are now reserved for
login and first visibility.

## Next capture

Run Rawr and Foo together, add Foo once, and confirm that Rawr remains connected
and both clients project the membership change. Then remove Foo once to verify
the generated `REM` response and the same membership notification path.
