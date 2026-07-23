# Self-hosting UT4MasterServer & connecting your own hubs/servers

A practical guide to running your own instance of this master server and pointing UT4
clients, hubs, and dedicated servers at it. This complements the user/account FAQ in the
main [README](../README.md); everything here is derived from the repo's own
`docker-compose.yml`, `appsettings.json`, and controller code.

## Architecture

The stack is defined in `docker-compose.yml`:

| Service | Host port | Purpose |
|---|---|---|
| `ut4masterserver` | `5000` → `80` | the master-server API (Epic-compatible endpoints) |
| `ut4masterserver-web` | `5001` → `8080` | web frontend (account management, statistics) |
| `mongo` | internal | database |
| `mongo-express` | `8888` → `8081` | Mongo admin UI — **development only, do not expose publicly** |

## 1. Running an instance

Development mode (as the README notes):

```bash
docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

The base `docker-compose.yml` already runs the API with `ASPNETCORE_ENVIRONMENT=Production`;
the bundled `docker-compose.override.yml` switches it to Development and rebuilds the web
frontend from the dev Dockerfile. For a real self-hosted instance, run the base compose and
supply **your own** override rather than the development one.

### Configuration — `UT4MasterServer/appsettings.json`

Under `ApplicationSettings`:

- **`DatabaseConnectionString`** / **`DatabaseName`** — MongoDB connection (default
  `mongodb://devroot:devroot@mongo:27017`, database `ut4master`). **Change the default
  `devroot` credentials before exposing an instance.**
- **`AllowPasswordGrantType`** (default `true`) — allows in-game username/password login
  (the "easy way" in the README FAQ). Set to `false` to require the exchange-code flow.
- **`ProxyClientIPHeader`** (`X-Forwarded-For`) and **`ProxyServers`** — used to recover the
  real client IP when the server runs behind a reverse proxy (see below). Set `ProxyServers`
  to your proxy's address on the Docker network.

## 2. TLS / reverse proxy (required for real clients)

UT4 talks to the master server over **HTTPS** — see `Files/Engine.ini`, where every service
uses `Protocol=https`. The API container serves plain HTTP on port `80` (`5000` on the host),
so a public instance needs a reverse proxy (nginx, Caddy, Traefik, …) that:

- terminates TLS for your domain,
- forwards requests to the `ut4masterserver` container, and
- sets `X-Forwarded-For` so the app sees real client IPs — this must match
  `ApplicationSettings:ProxyClientIPHeader`, and the proxy's address should be listed in
  `ProxyServers`.

## 3. Pointing clients, hubs, and servers at your instance

Use the existing **`Files/Engine.ini`** template: replace every
`<PUT_MASTER_SERVER_DOMAIN_HERE>` with your domain, then append it to the appropriate
`Engine.ini` (paths are in the README FAQ, for both clients and servers). This redirects the
game's `OnlineSubsystemMcp` service endpoints (`account`, `ut`, `entitlement`, `friends`, …)
to your master server.

## 4. How hubs and dedicated servers register

Hubs and dedicated servers authenticate with the OAuth **`client_credentials`** grant — a
"userless" session created in `SessionController.cs` — then create a matchmaking session and
heartbeat it via `MatchmakingController.cs`.

Worth knowing: the heartbeat handler contains a block, guarded by `#if false` (i.e. disabled
in the shipped build), that would otherwise reject long-lived `client_credentials` sessions
with a *"please use UT4UU"* error. **As shipped, hubs and servers register and heartbeat via
`client_credentials` with no client mod required** — that gate only takes effect if someone
compiles it in.

A successful registration is confirmed server-side by a log line such as:

```
RegisterServer: registered with master server OK (session <id>)
```

## 5. Gotcha: a hub behind NAT or containers advertises the wrong address

If a hub runs on the same host as the master server, in a container, or behind NAT, it can
register a bridge/loopback IP that other players cannot reach — the hub then appears in the
browser but is unjoinable. Set the address it should advertise, server-side:

```ini
[OnlineSubsystemUT]
ServerAddressOverride=<your.public.ip.or.domain>
```

(or pass `-ini:Engine:[OnlineSubsystemUT]:ServerAddressOverride=...` on the command line for
a packaged build). The server log line `RegisterServer: advertising serverAddress <addr>`
confirms what was published.

## Modern (UE5.8) clients

This master server is not limited to the original UE4.15 pre-alpha build. A community
UE4.15 → **UE5.8** port of UT4 has been validated end-to-end against an **unmodified**
instance of this server — account login, hub registration (`client_credentials`), the server
browser, and per-map content redirects all work with no master-server changes required. If
you're interested in a modern-engine client/server, see
[itpick/UnrealTournament](https://github.com/itpick/UnrealTournament).
