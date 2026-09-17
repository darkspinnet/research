# Go feature and boundary conventions

## Purpose

darkspin uses package-by-feature (vertical slice) boundaries. It does not use
top-level `domain`, `application`, and `persistence` folders. Add an abstraction
only where behavior or infrastructure is actually substitutable. The required
dependency direction is:

```text
HTTP / Blaze / RakNet adapter
            |
            v
feature operations, data, and rules -> feature-owned port <- adapter
```

For example, an account package owns `User`, account operations, and the
`UserRepository` port. `account/sqlite` and `account/postgres` implement that
port. The server composition root chooses and wires one implementation.
Infrastructure never owns game rules.

## Responsibilities

### Transport adapter

- Decode HTTP, XML, TDF, RakNet, or command-line data into a typed command.
- Resolve transport authentication into a domain identity.
- Call one application operation with the request context.
- Translate typed results/errors into the protocol response.
- Never mutate persistent profile fields or call a repository.

Transport-specific compatibility aliases such as `template_id` versus
`noun_id` stay here. The application receives one canonical noun ID.

### Feature operation

- Authorize the actor and locate the aggregate/session.
- Coordinate catalogs, clocks, repositories, and other ports.
- Invoke domain behavior.
- Own the persistence boundary and idempotency key.
- Return domain/application data rather than protocol DTOs.

`sporenet.UserManager` currently serves this role while the use cases are small.
Split it by capability when cohesion or test setup becomes poor, not merely to
create more packages.

### Feature data and rules

- Represent account, inventory, creature, squad, progression, and match rules.
- Protect invariants such as reward balance, ownership, capacity, lifecycle,
  cooldown, and result finality.
- Return descriptive sentinel or typed errors that callers can inspect with
  `errors.Is`/`errors.As`.
- Remain independent of XML, SQL, HTTP, Blaze TDF, and RakNet byte layouts.

### Port adapter

- Load and store aggregates; enforce storage uniqueness and concurrency.
- Translate storage errors into repository errors while preserving causes.
- Perform atomic replacement or database transactions.
- Never decide prices, rewards, unlock eligibility, game outcomes, or access.

The legacy file repository is an XML text-file adapter. SQLite and PostgreSQL
adapters implement the same feature-owned repository contract.

### Marshalling boundary

XML, JSON, Blaze TDF, RakNet packets, SQLite rows, and PostgreSQL rows are all
integration details. Feature types must not contain serialization tags or wire
field names. A transport decodes its request DTO into a typed feature command,
calls the feature operation, maps the result into an explicit allowlisted
response DTO, and only then marshals it.

Never marshal an aggregate or database row directly to a player. Internal data
may include credentials, moderation flags, anti-cheat observations, entitlement
provenance, pending transactions, and audit metadata; those fields are absent
from outbound DTOs by construction. Protocol trace redaction operates on the
DTO, not on an unrestricted aggregate.

## Persistent mutation shape

Every new persistent mutation should follow this sequence:

```text
decode command
  -> feature authorization
  -> business invariant check and mutation
  -> atomic repository commit
  -> publish result/event
  -> encode protocol response
```

Rejected commands produce no write. If a commit fails, the active aggregate
must be restored or discarded/reloaded before another command observes it.
End-game updates additionally require a unique match/result ID so retries cannot
grant progression or loot twice.

There is intentionally no general `Save` method exposed to transports. Each
application operation names the business action it commits, such as
`UnlockCreature` or eventually `ApplyMatchResult`.

## Idiomatic Go rules

- Organize packages by capability. Do not introduce generic technical-layer
  packages named `domain`, `application`, or `persistence`.
- Define small interfaces in the feature package that consumes them. Concrete adapters
  include compile-time interface assertions.
- Accept interfaces and return concrete implementations from constructors.
- Put `context.Context` first on operations that may perform I/O; never store it
  in a struct.
- Use zero values where meaningful and constructors where dependencies or
  invariants require validation.
- Wrap errors with a unique, one- or two-word operation tag using `%w`; use
  `errors.Is`/`errors.As` rather than comparing error strings. Longer scenario
  messages belong at the log, CLI, protocol, or user-facing handling boundary.
- Follow `AGENTS.md`: assign an error before its conditional instead of using an
  inline initializer.
- Keep locks private. Do not return pointers into mutable slices when the lock is
  released; introduce snapshots/read models as those paths are migrated.
- Prefer explicit command/result structs once a use case has more than a few
  same-typed parameters.
- Avoid package names such as `util`, `common`, or generic `service` when a
  capability name communicates ownership better.

## SOLID interpreted for Go

- **Single responsibility:** a protocol adapter translates; a feature operation
  coordinates; feature data enforces rules; a repository persists.
- **Open/closed:** add a PostgreSQL adapter or client build profile behind an
  existing port instead of adding database/build branches throughout handlers.
- **Liskov substitution:** repository contract tests must run against every
  adapter, including not-found, duplicate, cancellation, and atomicity cases.
- **Interface segregation:** HTTP and Blaze each declare only the application
  methods they use.
- **Dependency inversion:** application code owns repository ports; XML and
  PostgreSQL depend on those ports, never the reverse.

## Current migration status

- `UserRepository` is a feature-owned port that exchanges detached
  `UserRecord` values, separating durable profile data from active aggregates,
  locks, authentication sessions, and transports.
- `User.View` produces a detached player-visible projection without credentials,
  opaque legacy fields, associations, room/game IDs, or server session state.
- The HTTP account, creature, deck, settings, and inventory read paths project
  from `User.View` and explicitly allowlist the XML fields they marshal.
- `sporenet/sqlite` is the default standalone adapter. It maps normalized,
  adapter-owned SQL rows through `sqlx`, runs versioned migrations, enables WAL
  and foreign keys per connection, and commits each `UserRecord` atomically.
- SQLite is the only current profile adapter. `Creature` and `Part` carry no
  storage serialization tags; adapter-owned SQL rows handle persistence.
- Active users are indexed by username, account ID, and authentication token.
  Same-account login/logout is serialized independently of other accounts, so
  global presence lookups do not scan or globally block the online population.
- Blaze sessions are indexed by authenticated account ID, making targeted game
  and future global-chat notification fanout proportional to the recipient's
  connected sessions rather than the entire connected population.
- Game instances default to four players, allocate stable slots under a world
  lock, and atomically prevent one user from joining two worlds.
- HTTP and Blaze declare consumer-owned user-service interfaces.
- Registration defaults are applied by an application command, not HTTP.
- Creature unlock is a domain operation coordinated and persisted by the
  application layer. It rejects missing rewards without writing and rolls back
  if persistence fails.

Remaining direct field reads and the XML adapter's current location in the
`sporenet` package are migration debt. Once imported installations no longer
need compatibility fallback, that adapter can be removed. New mutation endpoints must
use this pattern, and existing
mutation acknowledgements should only be implemented after their
executable-confirmed commands and invariants are known.
