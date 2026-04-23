# faults.proto — Fault Snapshots

`faults.proto` carries a state snapshot at the moment of a fault. Includes the recent event history (the last ~100 events leading up to the fault) and the phase-state snapshot. Designed to be small enough to wrap into a single message — NOT a raw SDLC frame dump (too large, poor signal-to-noise).

## `FaultSnapshot`

| Field | # | Type | Notes |
|-------|---|------|-------|
| `fault_type` | 1 | FaultType | Fault classification |
| `fault_time` | 2 | google.protobuf.Timestamp | Precise fault timestamp |
| `recent_events` | 3 | repeated CompactEvent | Last ~100 events (~10-30 seconds typical) |
| `history_base_timestamp_ns` | 14 | int64 | Base timestamp for `recent_events.offset_ms` deltas |
| `phase_states` | 4 | repeated MovementState | Phase-state snapshot at fault time |
| `fault_description` | 10 | string | Human-readable description |
| `mmu_status_code` | 11 | uint32 | MMU fault code (if MMU-related) |
| `mmu_fault_register` | 12 | bytes | MMU hardware fault register dump |
| `conflict_bitmask` | 13 | bytes | Conflicting phase bitmask (if applicable) |
| `active_timing_plan` | 20 | uint32 | Timing plan ID active at fault time |
| `coordination_mode` | 21 | CoordinationMode | Coordination state at fault time |
| `sdlc_bus_health` | 22 | float | SDLC bus activity (0.0-1.0; 1.0=healthy) |

`recent_events` uses `CompactEvent` (from `ihr_events.proto`) so each event carries a millisecond delta from `history_base_timestamp_ns`. Reconstruct absolute time per event: `base_timestamp_ns + (offset_ms * 1_000_000)`.

`phase_states` uses `MovementState` (from `spat.proto`) for the per-movement signal phase state at the moment of the fault.

## `FaultType` enum

| Value | Name | Meaning |
|-------|------|---------|
| 0 | `FAULT_TYPE_UNSPECIFIED` | |
| 1 | `MMU_CONFLICT_MONITOR` | MMU detected conflicting greens (red-red failure) |
| 2 | `SDLC_TIMEOUT` | SDLC bus communication lost |
| 3 | `WATCHDOG_TIMEOUT` | Firmware watchdog triggered |
| 4 | `SECURITY_VETO` | Security system vetoed operation |
| 5 | `DECODER_ERROR` | Event decoder encountered unrecoverable error |
| 6 | `BUFFER_OVERFLOW` | Event buffer full (events being dropped) |
| 7 | `HARDWARE_FAULT` | Hardware fault (temp, voltage, etc.) |

## `CoordinationMode` enum

| Value | Name | Meaning |
|-------|------|---------|
| 0 | `COORDINATION_MODE_UNSPECIFIED` | |
| 1 | `COORDINATION_FREE` | Running free (no coordination) |
| 2 | `COORDINATION_COORD` | Coordinated to a pattern/cycle |
| 3 | `COORDINATION_PREEMPT` | Preemption active (overrides coordination) |
| 4 | `COORDINATION_FLASH` | Cabinet in flash mode |

## Notes

- For MMU conflicts (`FaultType.MMU_CONFLICT_MONITOR`), populate `mmu_status_code`, `mmu_fault_register`, and `conflict_bitmask` so downstream forensics can reconstruct the failure.
- For SDLC timeouts, `sdlc_bus_health` over the last few seconds is the most useful diagnostic.
- Fault snapshots are rare — bandwidth efficiency is less critical than for streaming. Producers can freely populate optional fields.
