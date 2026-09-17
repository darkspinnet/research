# Client-local emergency action reset

Build 103 exposes the combat-input singleton through `sub_4E4E00` at RVA
`0xE4E00`. It returns `dword_1438CB0 + 80`, the same object Fang already reads
for active, response, queued, and retained-target diagnostics.

`sub_4D5EF0` at RVA `0xD5EF0` is the native whole-action-state reset. Its only
client callsite passes that exact singleton during level teardown. The routine
clears active, response, queued, deadline, current-action, and held-state
fields and resets the embedded action collection. It does not mutate campaign
progress, inventory, health, power, or account state.

Fang handles `/reset` in two coordinated paths:

- the original converted chat command still reaches Blaze and queues the
  authoritative server reset;
- Fang posts a private window message and calls `sub_4D5EF0` on the game window
  thread, allowing recovery even when the locked client emits no later RakNet
  gameplay packet.

The hook validates the singleton through its highest accessed byte before the
native call and traces both message posting and local reset completion.
