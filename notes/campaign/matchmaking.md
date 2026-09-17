# Campaign matchmaking

Build 5.3.0.103 uses the same Blaze GameManager `startMatchmaking` command for campaign and Arena, but the criteria are mode-specific. The campaign path sends `GameType=2`, `ExpectedPlayerCount=4`, and `TeamSize=0`; it also retains the selected one-based chain entry in `SelectedDifficulty`. Arena instead sends `GameType=3`, four players, and teams of two.

Dark Spin previously routed every request through the Arena-only validator. A campaign Matchmaker click therefore received an immediate system error before it could enter the native queue window. The shared queue now accepts the exact campaign tuple as a separate mode, keeps criteria—including the selected chain entry—as the compatibility key, and continues using unique authenticated session IDs and owner-checked cancellation.

When four compatible campaign entries are available, formation resolves the selected chain level, verifies that every member has unlocked it, creates one reserved chain game, assigns stable player slots, and publishes the ordinary complete Blaze setup roster followed by successful matchmaking results. Unsupported tuples remain rejected without reserving queue state.

The attached 0.5.23 report retains only the minute after the rejected click, so it contains no GameManager request frame. Its server trace does prove that no later matchmaking state existed during report submission. The mode mismatch is established independently by the canonical client constructor in `Game.c`: campaign sets a four-player capacity and no team size, while the former server validator required the Arena tuple unconditionally.
