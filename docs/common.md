# common.proto — Master Wrapper & Batching

`common.proto` defines the message envelope that every other OpenPhase payload travels inside, plus two batch wrappers for transport efficiency, and the `ClockSource` enum used to advertise time-sync quality.

## `IntersectionUpdate` — master envelope

The single envelope type for all per-event traffic over NATS JetStream / MQTT. Uses a `oneof` discriminator for type-safe routing on the consumer side.

| Field | # | Type | Notes |
|-------|---|------|-------|
| `intersection_id` | 1 | string | UUID or friendly site identifier (e.g., `"gdot-0142"`). NOT the J2735 numeric ID; that lives in `SpatUpdate.intersection_id`. |
| `ts` | 2 | google.protobuf.Timestamp | Nanosecond precision. Should reflect when the event occurred at the source, not when it was published. |
| `payload` | 3-10 | oneof | One of: `SpatUpdate`, `AtspmEvent`, `SecurityAlert`, `DiscoveryPacket`, `DeviceHealth`, `FaultSnapshot`, `ControllerConfig`, `ControllerStatus` |
| `clock_source` | 15 | ClockSource | Time-sync quality of the producing device. Critical for ATSPM analytics that require accurate inter-intersection timing. |

## `IntersectionUpdateBatch` — mixed-payload batch

Use when batching mixed payload types from one intersection (e.g., a SPaT update + a couple of ATSPM events + a health heartbeat for the same minute). Each row carries its own timestamp and discriminator.

| Field | # | Type | Notes |
|-------|---|------|-------|
| `intersection_id` | 1 | string | Common ID for all updates in the batch |
| `updates` | 2 | repeated IntersectionUpdate | Ordered by timestamp |

## `CompactEventBatch` — high-volume IHR event batch

Bandwidth-optimized for streaming the ATSPM "nervous system" (2M+ events/day per intersection) over cellular or low-bandwidth backhaul. Each event is ~4-8 bytes on the wire instead of ~40+ bytes for a full `IntersectionUpdate` envelope.

| Field | # | Type | Notes |
|-------|---|------|-------|
| `intersection_id` | 1 | string | Common ID for all events |
| `base_timestamp_ns` | 2 | int64 | Nanoseconds since Unix epoch. PTP precision. Use the EARLIEST event's timestamp as the base. |
| `clock_source` | 3 | ClockSource | Time-sync quality for this batch |
| `events` | 4 | repeated CompactEvent | Ordered by `offset_ms` |

`CompactEvent` lives in `ihr_events.proto`. Reconstruct the absolute time of any event with: `absolute_ns = base_timestamp_ns + (offset_ms * 1_000_000)`.

## `ClockSource` enum

Producers MUST set this field. Consumers can use it to weight, gate, or flag analytics outputs that depend on inter-device timing accuracy.

| Value | Name | Accuracy |
|-------|------|----------|
| 0 | `CLOCK_SOURCE_UNSPECIFIED` | Producer didn't set it (treat as unknown / lowest) |
| 1 | `CLOCK_PTP` | IEEE 1588 PTP — microsecond accuracy |
| 2 | `CLOCK_NTP` | NTP — typical 10-50 ms accuracy |
| 3 | `CLOCK_GPS` | GPS/GNSS-derived — microsecond accuracy |
| 4 | `CLOCK_POWER_LINE` | Legacy 60 Hz line sync — ~16 ms jitter, unreliable |
| 5 | `CLOCK_FREE_RUNNING` | No external sync — drift accumulates |

PTP-synced devices (e.g., SOMA hardware) should publish with `CLOCK_PTP`. Legacy NTP-only controllers should publish `CLOCK_NTP`. Anything sub-PTP is unsuitable for inter-intersection cycle alignment analytics.

## Wire-format tips

- Field numbers 11-14 are reserved (gap between payload oneof and clock_source). Do not reuse.
- Producers SHOULD prefer `CompactEventBatch` over per-event `IntersectionUpdate` for ATSPM streams that exceed ~10 events/second.
- Consumers MUST treat unknown `payload` oneof values as ignorable (forward compat for future v1.x payload additions).
