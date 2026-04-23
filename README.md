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

| File | Domain | Docs | Description |
|------|--------|------|-------------|
| `common.proto` | Wrapper | [docs/common.md](docs/common.md) | Master message envelope (`IntersectionUpdate`), batch wrappers, `ClockSource` |
| `ihr_events.proto` | ATSPM | [docs/ihr_events.md](docs/ihr_events.md) | Indiana Hi-Resolution event log (`AtspmEvent`, `EventType`, `CompactEvent`) |
| `spat.proto` | SPaT | [docs/spat.md](docs/spat.md) | Signal Phase and Timing — live phase state for V2X/GLOSA (SAE J2735 compliant) |
| `ntcip.proto` | NTCIP MIB | [docs/ntcip.md](docs/ntcip.md) | Protobuf rendering of NTCIP 1202 v03A Actuated Signal Controller MIB |
| `health.proto` | Telemetry | [docs/health.md](docs/health.md) | Device health heartbeat — system resources, communication stats, environmental |
| `security.proto` | Security | [docs/security.md](docs/security.md) | Threat detection and alerts — spoofing, intrusion, conflict monitoring |
| `discovery.proto` | Discovery | [docs/discovery.md](docs/discovery.md) | Self-learning unmapped SDLC bit transitions for auto-configuration |
| `faults.proto` | Faults | [docs/faults.md](docs/faults.md) | Fault snapshots — state capture at time of fault with event history |

See **[docs/OVERVIEW.md](docs/OVERVIEW.md)** for the full protocol overview and how the schemas fit together.

---

## Package Structure

```
openphase/v1/
├── common.proto        # Master wrapper (IntersectionUpdate, batch)
├── ihr_events.proto    # ATSPM: Indiana Hi-Res event logging
├── spat.proto          # SPaT: live phase state (V2X/GLOSA)
├── ntcip.proto         # NTCIP 1202 v03A MIB object definitions
├── health.proto        # Device health heartbeat
├── security.proto      # Security alerts and threat detection
├── discovery.proto     # Self-learning bit discovery
└── faults.proto        # Fault snapshots
```

All messages use the `openphase.v1` package namespace.

---

## Design Principles

- **Fault snapshots** — last N events at time of fault
- **Master wrapper with `oneof`** — single envelope type for all message types
- **Versioned packages** — `openphase.v1`, `openphase.v2` for breaking changes
- **Standard types** — `google.protobuf.Timestamp` for nanosecond precision
- **SAE J2735 compliant** — SPaT message structure matches J2735 MovementState semantics
- **Indiana Hi-Resolution event log** — `EventType` in `ihr_events.proto` uses the canonical Indiana HiRes event codes (Sturdevant et al., INDOT/Purdue, November 2012)
- **NTCIP 1202 v03A MIB** — `ntcip.proto` is a protobuf rendering of the OID-addressed Actuated Signal Controller MIB (May 2019). Indiana HiRes and NTCIP 1202 are distinct standards; NTCIP is OID-addressed and does not define an event-code enumeration

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

**Document Version**: 1.2
**Last Updated**: 2026-04-22
**Maintainer**: OpenPhase Labs, LLC (under license from Heritage Grid, LLC, the IP holder)
