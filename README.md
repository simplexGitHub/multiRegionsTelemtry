# Collector-MAPI-Telemetry

A service that connects to multiple FlightOps MAPI regions, subscribes to four telemetry
rooms per region (`AirVehicleTelemetry`, `GimbalsTelemetry`, `MissionsTelemetry`,
`UserMessages`), and re-broadcasts an aggregated, per-region view to frontend clients over a
single WebSocket — once per second, with stale regions dropped.

> **Status — Stage 1 (spec only).** This repo currently contains **only the OpenAPI
> specification** of the broadcast contract plus the tooling that produced it. There is no
> runtime yet: no `package.json`, no server, no MAPI connection code. The architecture below
> describes the *intended* service; the spec defines its output contract. Connection, auth,
> aggregation, and broadcast code are later stages.

---

## What exists today

| Path | What it is |
|------|------------|
| `API/Telemetry Websockets/collectorSocketOut.json` | Main OpenAPI 3.0.0 spec: the WebSocket push contract (`$ref`s the output model). |
| `Models/Collector/*.json` | 7 authored schemas defining the `regionsTele` output envelope. |
| `Models/{AirVehicle,Gimbal,Mission,geo,generalServices,General,ELP,areaScan,MissionGeneralTypes}/` | 58 schemas vendored from `FlightOps-Mission-API` (the `$ref` closure of the four room item types), each tagged `x-vendored-from`. |
| `scripts/vendor-schemas.js` | Re-runnable tool that vendors the shared MAPI schemas (drift re-sync). |
| `docs/superpowers/specs/2026-06-01-collector-mapi-telemetry-openapi-design.md` | Design document. |

The spec is self-contained and Stoplight-resolvable: **66 JSON files, 95 `$ref`s, 0 broken.**

---

## System Overview (intended)

```
        ┌────────────────────────┐   ┌────────────────────────┐
        │  MAPI region #1         │   │  MAPI region #N         │
        │  (regions.json entry)   │   │  (regions.json entry)   │
        └───────────┬────────────┘   └───────────┬────────────┘
                    │ wss: AirVehicle/Gimbal/      │ (4 rooms each)
                    │ Mission/UserMessages rooms   │
                    ▼                              ▼
            ┌──────────────────────────────────────────┐
            │        Collector-MAPI-Telemetry           │
            │  per-region WS clients → state store →     │
            │  1 Hz aggregate (drop stale regions)       │
            └─────────────────────┬─────────────────────┘
                                  │ wss: { regionsTele: [...] }
                                  ▼
                       ┌────────────────────┐
                       │  Frontend clients  │
                       └────────────────────┘
```

Inbound: per-region MAPI WebSocket rooms (telemetry) + MAPI REST `getConfiguration` (video
sensor URLs). Outbound: a single aggregated `regionsTele` message to all connected frontend
clients, once per second.

---

## Output contract

The broadcast message (one per second). A region is **omitted entirely** when its newest
telemetry is older than the configurable staleness window. Defined by
`Models/Collector/RegionsTelemetryOut.json`.

```jsonc
{
  "regionsTele": [
    {
      "regionHost": "grok-test-region1.qcc-mvp.grok-digital.com",
      "regionName": "grok-test-region1.qcc-mvp.grok-digital.com",
      "AirVehicleTelemetry": [ { /* airVehicle fields */ "videoUrls": [ … ], "timestampLastChanged": 1771503605668 } ],
      "GimbalsTelemetry":    [ { /* GimbalTelemetryData fields */ "timestampLastChanged": 1771503605668 } ],
      "MissionsTelemetry":   [ { /* MissionTelemetryData fields */ "timestampLastChanged": 1771503605668 } ],
      "UserMessages":        [ { /* outUserMessage fields */ "timestampLastChanged": 1771503605668 } ]
    }
  ]
}
```

Each room is an **array** of the raw telemetry items exactly as MAPI sent them. Each item is
the MAPI item schema (`allOf`) with `timestampLastChanged` (epoch-ms of that item's last
change) merged in; AirVehicle items additionally carry `videoUrls`.

### Rooms

| Room | Item base schema (`allOf`) | Extra merged fields |
|------|----------------------------|---------------------|
| `AirVehicleTelemetry` | `Models/AirVehicle/airVehicle.v1.json` | `videoUrls`, `timestampLastChanged` |
| `GimbalsTelemetry` | `Models/Gimbal/GimbalTelemetryData.v1.json` | `timestampLastChanged` |
| `MissionsTelemetry` | `Models/Mission/MissionTelemetryData/MissionTelemetryData.v1.json` | `timestampLastChanged` |
| `UserMessages` | `Models/generalServices/outUserMessage.json` | `timestampLastChanged` |

The per-room item wrappers live in `Models/Collector/` (`AirVehicleTelemetryItem.json`,
`GimbalTelemetryItem.json`, `MissionTelemetryItem.json`, `UserMessageItem.json`).

### Video URL conversion

`videoUrls` are derived from each region's MAPI `getConfiguration` sensors. The sensor
`videoURL` is converted by these rules (see `Models/Collector/VideoUrl.json`):

1. swap scheme `http(s)://` → `rtsp://`
2. replace port `8889` → `8554`
3. strip any trailing `/`

Example: `https://host:8889/path/a/b/` → `rtsp://host:8554/path/a/b`

---

## OpenAPI conventions

- **OpenAPI 3.0.0**, matching `FlightOps-Mission-API` (the source of the vendored schemas).
- WebSocket messages are modeled as a `post` path with a `requestBody` and the marker
  `***THIS IS NOT A REST REQUEST, BUT WEBSOCKET***`, following the FlightOps house style.
  OpenAPI has no native WebSocket support; `servers[].url: "Websocket"` is a documentation
  placeholder, not a real server URL.
- Schemas are split across files and linked by relative `$ref` (Stoplight-compatible).

---

## Maintaining vendored schemas

The 58 files under `Models/` (excluding `Models/Collector/`) are copies of canonical schemas
from `FlightOps-Mission-API`. They will drift as upstream changes. To re-sync:

```bash
node scripts/vendor-schemas.js [path/to/FlightOps-Mission-API/Models]
```

The source root defaults to `/home/yuri/git/FlightOps-Mission-API/Models` and can also be set
via the `SRC_MODELS_ROOT` environment variable. The script resolves the full transitive `$ref`
closure of the four room item schemas, copies each file preserving its relative layout, and
stamps `x-vendored-from` provenance. Known upstream mispaths are corrected via its
`REF_REWRITES` map (recorded as `x-vendor-note` in the affected copies).

---

## Design Documents

| Document | Description |
|----------|-------------|
| `docs/superpowers/specs/2026-06-01-collector-mapi-telemetry-openapi-design.md` | Stage 1 design: file tree, output contract, vendoring strategy and outcome. |

---

## Not yet implemented (later stages)

- `config/regions.json` and `config/projconf.json` (region list + runtime tunables).
- Per-region auth + token renewal, WebSocket clients, aggregation, frontend broadcast server.
- The `convertHTTPS2RTSP` implementation (the rule is specified above).
- RTSP liveness checking; video URL `on`/`off` status.
