# Collector-MAPI-Telemetry

A service that connects to a central **mothership** over WebSocket to receive the **region
inventory** (which regions exist, plus their identity and health), connects to each of those
FlightOps MAPI regions and subscribes to four telemetry rooms per region
(`AirVehicleTelemetry`, `GimbalsTelemetry`, `MissionsTelemetry`, `UserMessages`), and
re-broadcasts an aggregated view — region inventory + per-region telemetry — to frontend
clients over a single WebSocket, once per second (stale regions are dropped from the live
telemetry feed).

> **Status — spec only.** This repo currently contains **only the OpenAPI specification** of
> the contract. There is no runtime yet: no `package.json`, no server, no MAPI connection
> code. The architecture below describes the *intended* service; the spec defines its
> authentication and output contract. Connection, auth, aggregation, and broadcast code are
> later stages.

---

## What exists today

| Path | What it is |
|------|------------|
| `multiRegionTelemetry.openapi-3.1.json` | The complete, self-contained **OpenAPI 3.1.0** spec: the `POST /api/login` auth endpoint and the WebSocket push contract, with every model inlined under `components.schemas`. |
| `docs/changes1.0.1-to-1.0.2.md` | Changelog of the latest schema changes. |

The bundle is self-contained and resolvable: **60 inline component schemas, 121 internal
`$ref`s, 0 broken.** MAPI models are vendored into it from `FlightOps-Mission-API` (the source
of truth) and tagged with `x-vendored-from` provenance.

---

## System Overview (intended)

```
   ┌──────────────────────┐        ┌──────────────┐     ┌──────────────┐
   │  Mothership          │        │ MAPI region 1│ ... │ MAPI region N│
   │  (region inventory)  │        └──────┬───────┘     └──────┬───────┘
   └──────────┬───────────┘               │ wss: 4 rooms each  │
              │ wss: inventory             ▼                    ▼
              │ (regionId, name,      ┌──────────────────────────────────┐
              │  host, health)        │     Collector-MAPI-Telemetry     │
              └──────────────────────▶│  mothership WS (inventory) +     │
                                      │  per-region WS clients → state → │
                                      │  1 Hz aggregate (drop stale)     │
                                      └────────────────┬─────────────────┘
                                                       │ wss: { regionsInventory:[…], regionsTele:[…] }
                                                       ▼
                                              ┌────────────────────┐
                                              │  Frontend clients  │
                                              └────────────────────┘
```

Inbound: a **mothership** WebSocket providing the region inventory (`regionId`, name, host,
health); per-region MAPI WebSocket rooms (telemetry); and MAPI REST `getConfiguration` (video
sensor URLs). Outbound: a single aggregated message (region inventory + per-region telemetry)
to all connected frontend clients, once per second.

---

## Authentication

Clients authenticate over REST, then present the token on the WebSocket.

1. `POST /api/login` with `{ "username", "password" }` → `200 { "token": "<JWT>", "expiresIn": <seconds> }`
   (or `401` with an `AuthError`).
2. On WebSocket connect, the client must send `{"token":"<JWT>"}` **within 10 seconds**.
   Invalid/expired token → the server disconnects.
3. The server also disconnects when the token reaches expiry. The client must obtain a new
   token and reconnect **before** expiry — use `expiresIn` to schedule renewal in advance.

---

## Output contract

The broadcast message (one per second), defined by `RegionsTelemetryOut`:

```jsonc
{
  "regionsInventory": [
    {
      "regionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "regionName": "grok-test-region1.qcc-mvp.grok-digital.com",
      "regionHost": "grok-test-region1.qcc-mvp.grok-digital.com",
      "health": "online",                       // "online" | "offline"
      "timestampLastChanged": 1771503605668
    }
  ],
  "regionsTele": [
    {
      "regionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",   // → matches a regionsInventory entry
      "AirVehicleTelemetry": [ { /* airVehicle fields */ "videoUrls": [ … ], "timestampLastChanged": 1771503605668 } ],
      "GimbalsTelemetry":    [ { /* GimbalTelemetryData fields */ "timestampLastChanged": 1771503605668 } ],
      "MissionsTelemetry":   [ { /* MissionTelemetryData fields */ "timestampMissionStarted": 1771503605668, "timestampLastChanged": 1771503605668 } ],
      "UserMessages":        [ { /* outUserMessage fields */ "timestampLastChanged": 1771503605668 } ]
    }
  ]
}
```

- **`regionsInventory`** — every region the Collector knows about, received from the
  **mothership** over WebSocket, with its identity (`regionId`, name, host) and current
  `health`. A region present here but **absent from `regionsTele`** is known but not currently
  producing fresh telemetry.
- **`regionsTele`** — one entry per region currently producing fresh telemetry. Each entry
  identifies its region by `regionId` (a UUID that maps to a `regionsInventory` entry); it no
  longer carries `regionHost`/`regionName` directly.
- Each room is an **array** of the raw telemetry items exactly as MAPI sent them. Each item is
  the MAPI item schema (`allOf`) with `timestampLastChanged` (epoch-ms, stamped by the
  Collector when the item was last updated in the aggregate) merged in.

### Rooms

| Room | Item base schema (`allOf`) | Extra merged fields |
|------|----------------------------|---------------------|
| `AirVehicleTelemetry` | `airVehicle.v1` | `videoUrls`, `timestampLastChanged` |
| `GimbalsTelemetry` | `GimbalTelemetryData.v1` | `timestampLastChanged` |
| `MissionsTelemetry` | `MissionTelemetryData.v1` | `timestampMissionStarted`, `timestampLastChanged` |
| `UserMessages` | `outUserMessage` | `timestampLastChanged` |

The per-room item wrappers are the components `AirVehicleTelemetryItem`, `GimbalTelemetryItem`,
`MissionTelemetryItem`, `UserMessageItem`; region inventory entries are `RegionInventoryItem`.

`timestampMissionStarted` (on mission items) is captured when a `missionId` first reaches
status `Started`; it is fixed for the lifetime of that `missionId` and reset only when a new
`missionId` appears.

### Supported mission types

`MissionTelemetryData.v1.type` and its `oneOf` cover: **`Ship` / `ShipWithoutLanding`**
(`SHIP_MISSION.v1`), **`DeliveryPoint`** (`DELIVERY_POINT_MISSION`), **`TargetTracking`**
(`TARGET_TRACKING_MISSION`), **`Patrol`** (`MAPI_PATROL_MISSION`), and **`AreaScan`**
(`MAPI_AREA_SCAN_MISSION`).

> This set is intentionally narrower than MAPI: `Route`, `Point`, `WASP`, `Security`, and
> `Dynamic` missions were removed. If MAPI emits one of those in `MissionsTelemetry`, it will
> not validate against this contract. See `docs/changes1.0.1-to-1.0.2.md`.

### Video URL conversion

`videoUrls` are derived from each region's MAPI `getConfiguration` sensors. The sensor
`videoURL` is converted (see the `VideoUrl` component) by:

1. swap scheme `http(s)://` → `rtsp://`
2. replace port `8889` → `8554`
3. strip any trailing `/` (path otherwise preserved)

Example: `https://host:8889/path/a/b/` → `rtsp://host:8554/path/a/b`

---

## OpenAPI conventions

- **OpenAPI 3.1.0**, single self-contained bundle; all schemas inlined under
  `components.schemas` and linked by `#/components/schemas/...` `$ref`s.
- WebSocket messages are modeled as a `post` path (`/RegionsTelemetry`) with a `requestBody`
  and the marker `***THIS IS NOT A REST REQUEST, BUT WEBSOCKET***`, following the FlightOps
  house style. OpenAPI has no native WebSocket support; `servers[].url: "Websocket"` is a
  documentation placeholder, not a real server URL.
- `POST /api/login` is a genuine REST endpoint.

---

## Maintaining vendored schemas

Most components are copies of canonical schemas from `FlightOps-Mission-API` (source of truth;
vendored from branch `internal_6.4.12`). Each is tagged with an `x-vendored-from`
`{ repo, path, date }` block, and any upstream `$ref` mispaths corrected during vendoring are
recorded with an `x-vendor-note` (e.g. the non-existent `../Gimbal/TASK_METADATA.json` →
`TASK_METADATA`). They will drift as upstream changes; re-sync by re-vendoring the affected
schemas from the source repo and updating their provenance stamps.

---

## Design Documents

| Document | Description |
|----------|-------------|
| `docs/changes1.0.1-to-1.0.2.md` | Changelog: region inventory restructure, AreaScan mission, mission-type narrowing, timestamps, cleanup. |

---

## Not yet implemented (later stages)

- **Mothership WebSocket client** — connect, receive and track the region inventory feed
  (`regionId`, name, host, health) that drives `regionsInventory`.
- Per-region auth + token renewal, MAPI WebSocket clients, aggregation, frontend broadcast
  server.
- The `http(s)→rtsp` video URL conversion implementation (rule specified above).
