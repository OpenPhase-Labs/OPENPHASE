# OPENPHASE

**Open Protocol for Traffic Signal Communication**

OpenPhase is an open-source protocol specification for modern traffic signal infrastructure. It defines the Protobuf message schemas used for communication between all Traffic Ops ecosystem products over NATS JetStream, MQTT, and other messaging protocols.

---

## Overview

- Open protocol for traffic signal telemetry, control, and analytics
- Protobuf-based message schemas with NATS JetStream transport
- Versioned packages (`openphase.v1`) for schema evolution
- Compatible with nanopb (C, embedded) and standard Protobuf libraries (Python, Go, Java)
- Single source of truth — all product repos reference this repo

---

## Schema Domains

| File | Domain | Description |
|------|--------|-------------|
| `common.proto` | Wrapper | Master message envelope (`IntersectionUpdate`) with `oneof` discriminator |
| `spat.proto` | SPaT | Signal Phase and Timing — live phase state for V2X/GLOSA (SAE J2735 compliant) |
| `ihr_events.proto` | ATSPM | Indiana Hi-Resolution event logging — 2M+ events/day for analytics |
| `health.proto` | Telemetry | Device health heartbeat — system resources, communication stats, environmental |
| `security.proto` | Security | Threat detection and veto alerts — spoofing, intrusion, conflict monitoring |
| `discovery.proto` | Discovery | Self-learning unmapped SDLC bit transitions for auto-configuration |
| `faults.proto` | Faults | Fault snapshots — state capture at time of fault with event history |

---

## Package Structure

```
openphase/v1/
├── common.proto        # Master wrapper (IntersectionUpdate, batch)
├── spat.proto          # SPaT: live phase state (V2X/GLOSA)
├── ihr_events.proto    # ATSPM: Indiana Hi-Res event logging
├── health.proto        # Device health heartbeat
├── security.proto      # Security alerts and threat detection
├── discovery.proto     # Self-learning bit discovery
└── faults.proto        # Fault snapshots
```

All messages use the `openphase.v1` package namespace.

---

## Design Principles

- **Event-based, not raw capture** — no raw SDLC frame dumps (too much bus traffic)
- **Fault snapshots** — last N events at time of fault, not raw frame forensics
- **Master wrapper with `oneof`** — single envelope type for all message types
- **Versioned packages** — `openphase.v1`, `openphase.v2` for breaking changes
- **Standard types** — `google.protobuf.Timestamp` for nanosecond precision
- **SAE J2735 compliant** — SPaT message structure matches J2735 MovementState semantics
- **NTCIP 1202 compatible** — event codes map to industry-standard Purdue event numbers

---

## Breaking Change Policy

OpenPhase serves critical traffic infrastructure. To ensure interoperability and prevent fragmentation:

- **Additions**: Always allowed in the current version. Adding new messages, fields, or enum values to `openphase.v1` is non-breaking and encouraged.
- **Modifications**: Any change that alters or removes existing fields, renames messages, or changes field numbers requires a **new version folder** (e.g., `openphase/v2/`). The previous version remains available indefinitely.
- **Shadow versions are prohibited**: Vendors creating private modifications (e.g., a proprietary "v1.1" with altered field semantics) while distributing them as `openphase.v1` violates the MPL 2.0 license terms. Modified protocol files **must** be made available under the MPL 2.0 per Section 1.10 and Section 3.

This policy ensures that any device speaking `openphase.v1` can interoperate with any other device speaking `openphase.v1`, regardless of vendor.

---

## License

Mozilla Public License 2.0 (MPL-2.0)

See [LICENSE](LICENSE) for full text and [PATENTS.md](PATENTS.md) for patent grant details.

---

**Document Version**: 1.1
**Last Updated**: 2026-03-30
**Owner**: Protocol Engineering Team @ Heritage Grid, LLC
