# OAuth authentication and desktop launch flow

## Purpose

This note defines the recommended authentication flow for launching Game
through a website-backed identity provider such as Discord OAuth. The website
authenticates the person, `darkspin.exe` coordinates the native desktop flow,
darkspin resolves the authenticated identity to a game account, and `fang.dll`
bridges the resulting launch credential into the legacy 5.3.0.103 Blaze login.

OAuth and browser handling belong in `darkspin.exe`, not inside the injected DLL.
Fang should remain a small compatibility bridge whose only authentication
responsibility is submitting an already-issued launch credential to the game.

## Recommended flow

Use the system browser with a temporary loopback listener owned by
`darkspin.exe`. The browser callback carries a short-lived, one-use authorization
code. It must not carry a reusable Discord token or the final launch JWT.

```text
darkspin.exe                       account website                 Discord                  darkspin
    |                                    |                           |                        |
    |-- listen on 127.0.0.1:random ------|                           |                        |
    |-- POST /api/desktop/start -------->|                           |                        |
    |<-- login URL + transaction ID -----|                           |                        |
    |                                    |                           |                        |
    |-- open system browser ------------>|-- OAuth authorize ------>|                        |
    |                                    |<-- OAuth callback --------|                        |
    |                                    |-- exchange OAuth code --->|                        |
    |                                    |<-- Discord identity -------|                        |
    |                                    |                           |                        |
    |                                    |-- resolve/link account --------------------------->|
    |                                    |<-- account subject -------------------------------|
    |                                    |                           |                        |
    |<-- GET /callback?code=ONE_USE_CODE-|                           |                        |
    |                                    |                           |                        |
    |-- POST /api/desktop/exchange ----->|                           |                        |
    |<-- short-lived launch JWT ---------|                           |                        |
    |                                    |                           |                        |
    |-- create suspended Game.exe --|                           |                        |
    |-- inject fang.dll -----------------|                           |                        |
    |-- pass JWT privately to DLL -------|                           |                        |
    |-- resume game ---------------------|                           |                        |
    |                                    |                           |                        |
    |        fang.dll submits token@local.invalid + JWT over Blaze -------------------------->|
    |<-------------------------- normal Blaze login/session response -------------------------|
```

## Why use a loopback callback

The launcher binds only to loopback using an operating-system-selected port:

```text
http://127.0.0.1:<random-port>/callback
```

This keeps the existing launcher process in control and avoids Windows
single-instance coordination or inter-process callback forwarding.

Discord's registered OAuth callback can remain a stable HTTPS URL on the
account website. After processing Discord's callback, the website redirects
the browser to the loopback URI that was bound to the desktop transaction.
The website must treat the loopback URI as structured transaction state, not
as an unrestricted redirect supplied by the browser.

## Responsibilities

### Account website

The website owns:

- Discord OAuth application credentials and callback handling.
- OAuth state, PKCE verifier/challenge state, nonce, and expiry.
- The mapping between a Discord identity and a darkspin account identity.
- Issuing one-use desktop authorization codes.
- Signing short-lived launch JWTs.
- A completion page telling the player that the browser may be closed.

Discord access and refresh tokens stay on the website backend. They must never
be returned to the launcher, Fang, Game, or darkspin's Blaze endpoint.

### `darkspin.exe`

The launcher owns:

- Binding a temporary HTTP listener to `127.0.0.1` on a random port.
- Creating a cryptographically random desktop transaction nonce.
- Asking the website to begin a desktop login transaction.
- Opening the returned HTTPS URL in the system browser.
- Waiting for the loopback callback with a bounded timeout.
- Verifying callback state before accepting its one-use code.
- Exchanging the code with the website over HTTPS.
- Passing the resulting launch JWT into the suspended game process.
- Clearing all local copies after Fang initialization.
- Falling back to the normal visible login page when authentication is
  cancelled, times out, or fails.

The launcher must bind specifically to `127.0.0.1`, not all interfaces. The
callback handler should accept one request, return a small completion page,
close the listener, and reject unexpected paths, methods, or state values.

### `fang.dll`

Fang owns only the legacy-client bridge:

- Copy the inherited launch JWT and immediately clear its environment source.
- Use the 5.3.0.103 login-screen initialization callback and submit through the
  native Blaze login manager on that same UI thread once it reports ready.
- Submit the reserved identity `token@local.invalid` as `MAIL` and the JWT as
  `PASS`.
- Submit once and securely clear its token buffer.
- Do not also mutate login-controller submit flags. Build 103 can process those
  flags on the UI thread while the injected login-manager call is still in
  flight, producing two identical Blaze login requests on one connection.
- Do not invoke the login manager from a worker thread. Even without controller
  flags, the UI thread can observe the manager transition concurrently and
  produce the same duplicate request. A live client/server trace confirms the
  UI-thread call emits one 276-byte send and one command-40 request.
- Never open browsers, perform Discord OAuth, retain provider credentials, or
  decide which game account should be used.
- Fall back safely when the build-specific login hook cannot be validated.

### darkspin

darkspin owns:

- JWT signature, issuer, audience, lifetime, and not-before validation.
- Resolving the external subject to a canonical local account.
- Account provisioning or account-link policy.
- Ban, entitlement, role, and game-access decisions.
- Issuing the normal internal `AuthToken` used by later Blaze and HTTP traffic.
- Audit events that identify the transaction without recording credential
  contents.

The JWT establishes identity only. Inventory, progression, access grants,
moderator status, bans, and other game authority always come from darkspin's
database.

## Website API contract

Names are illustrative and may be adapted to the website's conventions.

### Start desktop authentication

```http
POST /api/desktop/start
Content-Type: application/json

{
  "callback": "http://127.0.0.1:49152/callback",
  "state": "launcher-generated-random-state",
  "code_challenge": "base64url-sha256-pkce-challenge"
}
```

Successful response:

```json
{
  "transaction_id": "opaque-server-transaction-id",
  "authorize_url": "https://accounts.example.com/oauth/discord/start?...",
  "expires_in": 300
}
```

The backend validates that `callback` uses plain HTTP, the literal loopback
address `127.0.0.1`, an allowed ephemeral port, and the exact callback path. It
stores the normalized callback against the transaction rather than reflecting
arbitrary input later.

### Browser completion callback

After Discord OAuth succeeds, the website creates a one-use desktop code and
redirects the browser:

```http
HTTP/1.1 302 Found
Location: http://127.0.0.1:49152/callback?code=ONE_USE_CODE&state=launcher-generated-random-state
```

The desktop code should contain at least 128 bits of randomness, expire within
approximately one minute, and be consumed atomically.

### Exchange desktop code

```http
POST /api/desktop/exchange
Content-Type: application/json

{
  "transaction_id": "opaque-server-transaction-id",
  "code": "ONE_USE_CODE",
  "code_verifier": "launcher-generated-pkce-verifier"
}
```

Successful response:

```json
{
  "launch_token": "header.payload.signature",
  "expires_in": 60
}
```

The exchange invalidates the code even under concurrent requests. Failed PKCE,
expired state, mismatched transaction IDs, and reused codes receive the same
generic rejection response.

## Identity model

Use Discord's stable user ID as the external subject. Do not key account
ownership by Discord display name or email because those values can change.

Recommended database relationship:

```text
external_identity
  provider          = "discord"
  provider_subject  = "123456789012345678"
  darkspin_account_id = 48291
```

Account linking should require an authenticated website session and an
explicit confirmation step. Automatically joining identities solely because
their email addresses match creates an account-takeover risk.

For first login, choose one explicit policy:

1. Automatically provision a new darkspin account for an unrecognized Discord
   subject.
2. Require the player to create or link a game account on the website first.

The second policy is more deliberate; the first gives a smoother onboarding
experience.

## Launch JWT

Recommended claims:

```json
{
  "iss": "https://accounts.example.com",
  "aud": "darkspin",
  "sub": "discord:123456789012345678",
  "email": "optional@example.com",
  "preferred_username": "Display Name",
  "iat": 1784160000,
  "nbf": 1784160000,
  "exp": 1784160060,
  "jti": "unique-launch-id"
}
```

Only `iss`, `aud`, `sub`, timing claims, and `jti` participate in security
decisions. Profile fields are hints for display or initial provisioning.

Production signing should be asymmetric, preferably Ed25519 or another
well-supported modern algorithm:

- The website holds the private signing key.
- darkspin holds only the public verification key.
- Keys have identifiers and support rotation.
- darkspin accepts only an explicit algorithm and expected key set.

The current local JWT implementation uses HS256 and a shared secret. That is
adequate for local development, but it means a compromised darkspin server could
mint website-equivalent launch tokens. Replace it before deploying a public
account service.

Launch JWTs should normally expire within 30 to 60 seconds. `jti` values may be
recorded until expiry to enforce one-use semantics. If retry behavior is
required for a lost Blaze response, bind the accepted `jti` to the same pending
desktop/game session rather than accepting it globally more than once.

## Launcher state machine

```text
Idle
  -> Listening
  -> BrowserOpened
  -> CallbackReceived
  -> CodeExchanged
  -> GameCreatedSuspended
  -> DLLInitialized
  -> GameRunning

Any pre-launch failure
  -> close listener
  -> clear code/JWT buffers
  -> offer Retry or Normal Login
```

Important behavior:

- Use one overall authentication timeout plus shorter HTTP timeouts.
- Support cancellation without leaving the loopback listener running.
- Do not start multiple browser flows for one launcher instance.
- Do not write authorization codes or JWTs to normal logs.
- Redact query strings in diagnostics because callbacks contain codes.
- Clear environment and memory copies as soon as the injected initializer has
  copied the credential.

## Failure behavior

| Failure | Launcher behavior |
| --- | --- |
| Browser cannot open | Show the HTTPS URL for manual copy and continue waiting |
| User cancels Discord | Close the transaction and offer Retry or Normal Login |
| Loopback port becomes unavailable | Cancel the transaction and create a new listener/transaction |
| Callback state mismatches | Reject request, retain timeout, never exchange its code |
| Desktop code expires or is reused | Clear state and begin a new transaction |
| JWT verification fails | darkspin returns the standard invalid-user response without claim details |
| External identity is not linked | Website directs the user through explicit linking/provisioning |
| Fang hook does not match 103 | Resume the game on the ordinary login page without submitting the token |

## Local development authentication

Local development should exercise the same launcher, JWT, Blaze, and account
resolution boundaries without requiring Discord OAuth. Do not add a Blaze
backdoor that trusts an arbitrary account name. Instead, run a local auth broker
that replaces only the external website and identity-provider interaction.

The proposed command is:

```powershell
darkspin auth --local
darkspin.exe --account developer@example.test
```

`darkspin auth` is a standalone development service. It binds only to loopback
and implements the same desktop start and exchange contract expected from the
production account website:

```text
darkspin.exe
  -> POST http://127.0.0.1:8090/api/desktop/start
     account = developer@example.test
  -> local mode approves the requested identity
  -> loopback callback with a one-use code
  -> POST http://127.0.0.1:8090/api/desktop/exchange
  <- short-lived development launch JWT
  -> normal Fang and darkspin Blaze authentication
```

The local broker skips Discord, not authentication. It still creates a signed,
short-lived credential containing the selected account identity. darkspin still
validates the signature, issuer, audience, and expiry before loading the local
account.

The `--local` flag is the security boundary. Without it, the auth command must
not accept a caller-provided identity without completing its configured OAuth
or account-link flow. Local mode is intentionally powerful: any local process
can request a launch credential for any existing development account.

Suggested command behavior:

- `darkspin auth --local` binds specifically to `127.0.0.1` and refuses to start
  if configured with a non-loopback listen address.
- `darkspin.exe --account <identity>` places the desired account identity in the
  desktop start request. This supports concurrent testing with multiple
  launcher processes and different accounts.
- Verify the actual TCP peer address. Never use `X-Forwarded-For` or another
  caller-controlled header to decide whether local mode applies.
- Require POST requests with JSON bodies, reject requests carrying a browser
  `Origin` header, emit no CORS headers, and validate the HTTP `Host` as the
  exact loopback listener. These checks reduce the chance that an unrelated
  website can drive the local development service through a browser.
- darkspin rejects the resulting login normally if the requested account does
  not exist. The auth service does not silently substitute another account.
- Implement `/api/desktop/start` and `/api/desktop/exchange` with the same JSON
  shapes as the production website.
- Issue one-use desktop codes and launch JWTs even though all participants are
  local. This keeps replay and lifecycle behavior testable.
- Display an unmistakable development-mode banner and never silently fall back
  to an arbitrary account.
- Refuse `--local` in builds or deployments explicitly marked as production.
- Avoid reading or writing the live user repository from the auth process. The
  broker asserts the requested account identity; the running darkspin server owns
  repository access and decides whether that account exists.

For the current HS256 implementation, the auth broker and darkspin server may
share a development secret through an ignored local secret file or explicit
environment configuration. A future asymmetric implementation should let
`darkspin auth` own a development private key while the game server receives only
its public key.

## Launcher authentication endpoint

`darkspin.exe` should have a compiled default authentication URL represented by
a Go variable rather than a constant. CI can replace it through `-ldflags -X`
when producing a launcher for a particular community server. An illustrative
definition is:

```go
var authServiceURL = "http://127.0.0.1:8090"
```

An illustrative production build is:

```powershell
go build -ldflags "-X github.com/foo/darkspin/app/darkspin.authServiceURL=https://accounts.example.com" ./app/darkspin
```

Mage accepts the same build-time choice through the environment:

```powershell
$env:darkspin_AUTH_URL = "https://accounts.example.com"
mage build
```

The launcher deliberately has no runtime authentication-URL override. A
distributed `darkspin.exe` always contacts the service it was compiled for.

Non-loopback authentication URLs must use HTTPS. The compiled default is a
distribution convenience, not a trust decision: JWT issuer, audience, and
signature validation remain authoritative on darkspin.

Production builds point the same launcher protocol at their HTTPS account
website. Local builds point it at `darkspin auth --local`. This makes the auth
command a protocol-compatible development substitute rather than a second
authentication design.

A future server selector can select a server ID, authentication URL, and game
endpoint before authentication begins. The selected server ID should be bound
into the launch credential's audience or a dedicated signed claim so a token
issued for one server cannot be replayed against another. The launcher can then
pass the selected game endpoints to Fang through the same inherited
configuration channel currently used for launch authentication.

## Suggested implementation phases

### Phase 1: launcher and website handshake

- Add loopback listener lifecycle to `darkspin.exe`.
- Add `/api/desktop/start` and `/api/desktop/exchange` to the website.
- Open the system browser and render a completion page.
- Add `darkspin auth` as a loopback-only implementation of the same contract.
- Use a selected local account to validate callback, timeout, cancellation,
  JWT verification, and normal Blaze session creation.

### Phase 2: Discord OAuth and account mapping

- Add Discord OAuth state and PKCE handling to the website.
- Persist the provider-subject-to-account relationship.
- Add explicit first-login provisioning or account linking.
- Keep Discord credentials entirely server-side.

### Phase 3: production launch credentials

- Replace HS256 with asymmetric JWT verification.
- Add key identifiers and rotation.
- Add short expiry and `jti` replay handling.
- Add security-focused audit events without credential contents.

### Phase 4: client integration verification

- Confirm Fang's 5.3.0.103 login callback timing in a live run.
- Confirm a valid external subject reaches LoginPersona and normal HTTP account
  authentication.
- Confirm invalid, expired, and reused credentials return to a usable login UI.
- Confirm the JWT never appears in protocol traces, logs, crash messages, or
  the visible password field.

## Acceptance criteria

- A player can start `darkspin.exe`, authenticate through the system browser,
  and enter the game without typing a game password.
- Discord access and refresh tokens never leave the website backend.
- The browser callback contains only a one-use desktop authorization code.
- Callback state and PKCE are verified before a launch JWT is issued.
- darkspin resolves a stable external subject to a canonical local account.
- Fang submits the credential once and clears it afterward.
- Invalid authentication falls back safely without trapping the player in a
  broken or invisible login state.
- Account privileges and game state are loaded from darkspin, not trusted from
  JWT profile claims.
