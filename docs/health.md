# health.proto — Device Health Telemetry

`health.proto` defines the periodic heartbeat that every OpenPhase-speaking device emits. Cadence is typically once every 60 seconds. Consumers use it to confirm liveness, detect degraded operation, monitor cabinet environmentals, and track firmware/hardware revisions across a fleet.

## `DeviceHealth` — heartbeat

| Field | # | Type | Notes |
|-------|---|------|-------|
| `status` | 1 | HealthStatus | Overall device health classification |
| `uptime_s` | 2 | uint64 | Device uptime in seconds since last boot |

### Event statistics
| Field | # | Type | Notes |
|-------|---|------|-------|
| `events_decoded` | 10 | uint64 | Total events decoded since boot |
| `events_delivered` | 11 | uint64 | Total events successfully published to NATS |
| `events_pending` | 12 | uint64 | Events queued (backlog indicator) |
| `events_dropped` | 13 | uint64 | Events dropped due to buffer overflow |
| `discovery_packets` | 14 | uint64 | Unmapped SDLC bit transitions observed |

### System resources
| Field | # | Type | Notes |
|-------|---|------|-------|
| `cpu_temp_c` | 20 | float | CPU temperature (Celsius) |
| `memory_used_pct` | 21 | float | RAM utilization percentage |
| `buffer_used_pct` | 22 | float | Event buffer utilization percentage |
| `disk_used_pct` | 23 | float | eMMC storage utilization (local logging) |

### Cabinet environmental (J7 sensors on PRISM hardware)
| Field | # | Type | Notes |
|-------|---|------|-------|
| `cabinet_temp_c` | 25 | float | Cabinet temperature (DS18B20 on J7 pin 2) |
| `cabinet_door_open` | 26 | bool | Cabinet door status (J7 pin 3) |
| `last_door_open` | 27 | google.protobuf.Timestamp | Last door-open event timestamp |

### Communication health
| Field | # | Type | Notes |
|-------|---|------|-------|
| `sdlc_frames_received` | 30 | uint64 | Total SDLC frames captured (J1) |
| `sdlc_crc_errors` | 31 | uint64 | Frames with CRC failures |
| `sdlc_frame_rate_hz` | 32 | float | Current SDLC bus activity (frames/sec) |
| `can_frames_received` | 35 | uint64 | Total CAN-FD frames captured (J4) |
| `can_errors` | 36 | uint64 | CAN bus errors (bus-off, error frames) |
| `can_frame_rate_hz` | 37 | float | Current CAN bus activity (frames/sec) |
| `nats_connected` | 40 | bool | NATS connection status (J3/J8) |
| `nats_publish_errors` | 41 | uint64 | NATS publish failures since boot |
| `active_network_interface` | 42 | string | `"eth0"` (J1) or `"eth1"` (J8) when failover active |

### Version information
| Field | # | Type | Notes |
|-------|---|------|-------|
| `firmware_version` | 50 | string | Firmware version (e.g., `"2.1.3"`) |
| `decoder_version` | 51 | string | Event decoder version |
| `mapping_version` | 52 | string | Bit-to-event mapping table version |
| `hardware_revision` | 53 | string | Hardware revision (e.g., `"PRISM-v1.2"`) |

## `HealthStatus` enum

| Value | Name | Meaning |
|-------|------|---------|
| 0 | `HEALTH_STATUS_UNSPECIFIED` | Producer didn't set it |
| 1 | `HEALTH_OK` | All subsystems normal |
| 2 | `HEALTH_DEGRADED` | Operational but with non-fatal issues (high temp, packet loss, etc.) |
| 3 | `HEALTH_ERROR` | Critical fault — device may be missing data or offline |

## Notes

- Field numbers are intentionally non-contiguous (10, 20, 25, 30, 35, 40, 50) to leave room for future additions per category.
- The PRISM-specific cabinet sensor fields (J1/J3/J4/J7/J8 references) are appropriate for the OpenPhase v1 reference hardware. Other implementations with different I/O can leave them at zero/empty without breaking consumers.
- For fleet monitoring, consumers should alert on `events_dropped > 0` over any heartbeat interval (lossy transport or backlog) and on `nats_connected = false` for more than a couple of intervals (publish path broken).
