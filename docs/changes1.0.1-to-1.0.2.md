# Telemetry schema changes: 1.0.1 → 1.0.2

Changes to `multiRegionTelemetry.openapi-3.1.json`. Models are vendored from
`FlightOps-Mission-API` (source of truth; branch `internal_6.4.12`).

> ⚠️ This release contains **breaking changes** to `RegionTelemetry` and a
> **deliberate narrowing** of the supported mission types (the bundle no longer
> mirrors every MAPI mission type — see *Known divergences*).

---

## 1. Region identity moved to a top-level inventory (breaking)

Region identity (`regionHost` / `regionName`) was repeated on every
`RegionTelemetry`. It is now factored out into a single top-level inventory, and
each telemetry entry references its region by a stable UUID.

**`RegionsTelemetryOut`**
- Added `regionsInventory` (array, **required**) — a sibling of `regionsTele`.
  Lists every region the Collector knows about, including regions that are
  currently stale/absent from `regionsTele`.

**`RegionInventoryItem`** (new component) — all fields **required**:
| field | type | notes |
|-------|------|-------|
| `regionId` | string (uuid) | stable region identifier |
| `regionName` | string | from `regions.json` `name` |
| `regionHost` | string | from `regions.json` `host` |
| `health` | enum `online` \| `offline` | reachability as seen by the Collector |
| `timestampLastChanged` | number (epoch-ms) | when this entry last changed |

**`RegionTelemetry`** (breaking)
- **Removed** `regionHost` and `regionName`.
- **Added** `regionId` (string/uuid, **required**) — matches an entry in
  `regionsInventory`.

---

## 2. Mission types

### Added — AreaScan
- New mission body **`MAPI_AREA_SCAN_MISSION`** added to the
  `MissionTelemetryData.v1` `oneOf`, with enum values **`AreaScan`** and
  **`Patrol`** added to `type`.
- New supporting components (vendored): `FLIGHT_PARAMETERS`, `MAX_SLOPE_DEGREE`,
  `SCAN_ROUTE`, `SCAN_POINT_DATA`, `TASK_SCAN_MONITORING_ACTION`.

### Removed — unused mission types (intentional)
Mission types not used by this deployment were removed from the `oneOf` and the
`type` enum, and their now-unreferenced schemas were deleted:
- Mission types dropped: **Route, Point, WASP, Security, Dynamic**.
- 15 schemas deleted: `ROUTE_MISSION.v1`, `POINT_MISSION.v1`,
  `SECURITY_MISSION.v1`, `WASP_MISSION.v1`, `DYNAMIC_POINT_MISSIOM`,
  `POINT_OF_ROUTE_DATA.v1`, `AUTONOMOUS_POINT.v1`, `AUTONOMOUS_ACTION.v1`,
  `ACTION_CONDITIONS.v1`, `GIMBAL_ACITON_PARAMETERS`, `MOVE_ACTION_PARAMETERS`,
  `ON_ERROR_ACTION_PARAMETERS`, `SCAN_ACTION_PARAMETERS`,
  `WAYPOINT_ACTION_PARAMETERS`, `GEOPOINT_3D_SHORT.v1`.

### Final supported mission set
`oneOf`: `TARGET_TRACKING_MISSION`, `SHIP_MISSION.v1`, `DELIVERY_POINT_MISSION`,
`MAPI_PATROL_MISSION`, `MAPI_AREA_SCAN_MISSION`.
`type` enum: `Ship`, `ShipWithoutLanding`, `DeliveryPoint`, `TargetTracking`,
`AreaScan`, `Patrol`.

---

## 3. `MissionTelemetryItem` — mission-start timestamp

- Added **`timestampMissionStarted`** (number, epoch-ms, **required**):
  captured when a `missionId` first reaches status `Started`. It is fixed at
  that first start and does not change for the lifetime of the `missionId`
  (only reset when a new `missionId` appears).

---

## 4. `timestampLastChanged` clarified

- Reworded across all room items (`AirVehicleTelemetryItem`,
  `GimbalTelemetryItem`, `MissionTelemetryItem`, `UserMessageItem`): now states
  it is stamped by the Collector — the system that aggregates the rooms —
  recording when the item was last updated in the aggregated telemetry.
- Also present (with a region-specific wording) and **required** on
  `RegionInventoryItem`.

---

## 5. Cleanup & vendoring fixes

- Removed all **80 `x-stoplight`** editor annotations bundle-wide (inert
  metadata; no semantic effect).
- Vendoring correction: `TASK_SCAN_MONITORING_ACTION.metadata` `$ref` repointed
  to `TASK_METADATA` (upstream pointed at a non-existent
  `../Gimbal/TASK_METADATA.json`); recorded via `x-vendor-note`.

---

## Validation

- Valid JSON; **no dangling `$ref`s**; **60** components; **0** unused schemas.
- Examples: all three `/api/login` examples validate. The `/RegionsTelemetry`
  example is a deliberately minimal sample and does not fully satisfy the strict
  MAPI item schemas (pre-existing; left as an illustration).

## Known divergences / follow-ups

- **Narrower than MAPI:** the bundle no longer represents Route / Point / WASP /
  Security / Dynamic missions. If MAPI emits one of these in `MissionsTelemetry`
  it will not validate here. Re-sync with upstream would need to re-add them.
- **`regionId` is a UUID:** something (Collector config/logic) must assign each
  region a stable UUID for `regionId` to be populated; `regions.json` currently
  keys regions by `host`/`name`.
- The document `info.version` is still `"1.0"` — bump if you track the bundle
  version explicitly.
