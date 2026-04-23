# discovery.proto — Self-Learning SDLC Bit Discovery

`discovery.proto` is the OpenPhase self-learning channel: when the device's SDLC bus sniffer observes a bit transition that isn't in its current bit-to-event mapping table, it emits a `DiscoveryPacket`. Backend cloud heuristics cross-reference these against controller FTP logs to auto-discover new mappings — useful for supporting controllers whose vendor-specific bit assignments aren't yet documented.

## `DiscoveryPacket`

| Field | # | Type | Notes |
|-------|---|------|-------|
| `sdlc_address` | 1 | uint32 | SDLC device address. `0x00-0x0F` = BIU, `0x20` = MMU. |
| `byte_index` | 2 | uint32 | Byte offset within the SDLC frame payload. |
| `bit_index` | 3 | uint32 | Bit position within the byte (0-7). |
| `old_state` | 4 | bool | Previous bit state |
| `new_state` | 5 | bool | New bit state after the transition |
| `frame_context` | 6 | bytes | Minimal surrounding-byte context — NOT the full raw frame (intentional, to keep packet size bounded). |
| `direction` | 7 | Direction | TX (controller → BIU) or RX (BIU → controller) |
| `suspected_event` | 10 | string | If a cloud heuristic match exists, the IHR event name (e.g., `"PHASE_2_GREEN"`). Empty if unmapped. |
| `confidence` | 11 | float | Confidence score (0.0-1.0) for the suspected mapping. |

## `Direction` enum

| Value | Name | Meaning |
|-------|------|---------|
| 0 | `DIRECTION_UNSPECIFIED` | Producer didn't set |
| 1 | `TX` | Controller → BIU (poll commands, output states) |
| 2 | `RX` | BIU → Controller (detector calls, status responses) |

## How it fits

A typical mapping-discovery loop:

1. Field device sees bit transition at (address, byte, bit) that isn't in its mapping table.
2. Device emits `DiscoveryPacket` with `suspected_event = ""` and `confidence = 0.0`.
3. Cloud-side learner correlates the timestamp with FTP-pulled high-resolution logs from the same controller, finds a likely IHR event match, and pushes an updated mapping table back to the device.
4. Future occurrences of that bit transition are decoded directly into `AtspmEvent` (in `ihr_events.proto`) instead of generating new discovery packets.

## Notes

- `frame_context` should be ~8-32 bytes — enough to disambiguate bit position within the frame structure, not enough to create a forensics dump. Producers SHOULD NOT send full raw frames (privacy, bandwidth).
- Consumers should rate-limit per (sdlc_address, byte_index, bit_index) — a misconfigured controller could otherwise flood discovery packets for every cycle of an unmapped bit.
