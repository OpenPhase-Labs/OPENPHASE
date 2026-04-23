# ntcip.proto — NTCIP 1202 v03A MIB Object Mapping

`ntcip.proto` is a protobuf rendering of the NTCIP 1202 v03A (May 2019) Actuated Signal Controller MIB. Each protobuf message corresponds to a SEQUENCE in the standard. OID paths are documented in proto comments for cross-reference.

NTCIP 1202 is OID-addressed via SNMP — it does NOT have a small event-code enumeration like Indiana Hi-Res does. High-resolution event logs live in [ihr_events.proto](ihr_events.md), not here.

## OID base

All objects are under the `asc` node:
```
1.3.6.1.4.1.1206.4.2.1
└── iso.org.dod.internet.private.enterprises.nema.transportation.devices.asc
```

Sub-groups under `asc.X`:

| OID | Group | Proto messages |
|-----|-------|----------------|
| `.1` | Phase | `PhaseEntry`, `PhaseTable`, `PhaseStatusGroup`, `PhaseControlGroup`, `PhaseOptions`, `PhaseStartup` |
| `.2` | Detector | `DetectorEntry`, `DetectorTable`, `DetectorStatusGroup`, `DetectorOptions`, `DetectorTravelMode`, `PedestrianDetectorEntry`, `PedestrianDetectorTable` |
| `.3` | Unit | `UnitParams`, `UnitControlMode`, `UnitFlashStatus`, `UnitAlarmStatus`, `UnitAutoPedClear`, `SpecialFunctionEntry`, `SpecialFunctionTable` |
| `.4` | Coordination | `CoordPatternEntry`, `CoordSplitEntry`, `CoordTable`, `CoordStatus`, plus enums: `CoordSplitMode`, `CoordCorrectionMode`, `CoordMaxMode`, `CoordForceMode`, `CoordLocalFreeStatus`, `PatternCoordSyncPoint`, `PatternMaxMode` |
| `.5` | Timebase | `TimebaseActionEntry`, `TimebaseTable`, `TimebaseStatus` (day-plan/schedule structures inherit from NTCIP 1201 Global MIB and are not redefined here) |
| `.6` | Preempt | `PreemptEntry`, `PreemptTable`, `PreemptControlBits`, `PreemptStatus`, `PreemptStateDetail`, `PreemptCurrentState` |
| `.7` | Ring | `SequenceEntry`, `RingControlGroup`, `RingTable`, `BarrierEntry`, `RingBarrierConfig` |
| `.8` | Channel | `ChannelEntry`, `ChannelTable`, `ChannelControlType`, `ChannelGreenType`, `ChannelStatusGroup` |
| `.9` | Overlap | `OverlapEntry`, `OverlapTable`, `OverlapType`, `OverlapStatusGroup` |

OIDs `.10` (Port 1, NEMA TS2 serial), `.12` (Cabinet), `.13` (ASC I/O Mapping), `.14` (SIU Port 1) are NOT modeled here. NTCIP 1202 v03A does NOT define a top-level Priority/TSP group — TSP is exposed through I/O mapping (`priorityRequest` / `priorityCheckout` input functions; `preemptActive` outputs).

## Composite messages

For bulk upload/download and snapshot publishing:

### `ControllerConfig` — full read-write parameter set
Wraps every config table: phases, detectors, unit, coordination, timebase, preempts, rings, channels, overlaps, special functions, ped detectors. Plus `RingBarrierConfig` (derived from phase ring assignments + sequences).

### `ControllerStatus` — full read-only registers snapshot
Wraps every status table at one moment: phase status/control groups, detector status, unit alarms, coord status, timebase status, preempt status, channel status, overlap status. Plus `snapshot_time` and `control_mode`.

## Field-level conventions

### Units
NTCIP 1202 mixes seconds and tenth-seconds based on the field. The proto comments mark every timing field explicitly: `<Unit>` of `second` or `tenth seconds`. Producers MUST follow the unit annotation — getting it wrong silently corrupts cycle-time analytics.

Common pitfalls (corrected in v1):
- `phaseWalk`, `phasePedestrianClear`, `phaseMinimumGreen`, `phaseMaximumInitial`, `phaseMaximum1/2/3`, `phaseTimeBeforeReduction`, `phaseTimeToReduce`, `phaseDynamicMaxLimit` — all in **SECONDS**, not tenths.
- `phasePassage`, `phaseYellowChange`, `phaseRedClear`, `phaseRedRevert`, `phaseAddedInitial`, `phaseReduceBy`, `phaseMinimumGap`, `phaseDynamicMaxStep` — **tenth seconds**.
- Detector `delay`/`extend` — **tenth seconds**. Detector `queue_limit`/`fail_time` — **seconds**. Detector `no_activity`/`max_presence` — **minutes**.
- Preempt `delay`/`min_duration`/`min_green`/`min_walk`/`enter_ped_clear`/`track_green`/`dwell_green`/`max_presence` — **SECONDS** (NOT tenths). Preempt v03 yellow/red clearance times — tenth seconds.
- Overlap `trail_green`/`walk`/`ped_clearance` — SECONDS. Overlap `trail_yellow`/`trail_red` — tenth seconds.

### Bitmask fields
Status/control group registers (e.g., `PhaseStatusGroup.greens`) are uint32 bitmasks where each bit corresponds to a specific phase/detector/channel within an 8-element group:
```
Bit N = (group_number * 8) - (7 - N)
```
For a group_number of 1: bit 7 = phase 8, bit 0 = phase 1.

Where the spec defines individual bit meanings (e.g., `phaseOptions`, `vehicleDetectorOptions`, `preemptControl`), the proto provides a sub-message that unpacks the bits into named bool fields (`PhaseOptions`, `DetectorOptions`, `PreemptControlBits`).

### OCTET STRING fields
Spec OCTET STRING fields where each octet is a phase number (e.g., `phaseConcurrency`, `preemptTrackPhase`, `overlapIncludedPhases`) are typed as `bytes` in the proto. Each byte carries one phase number (1-255).

## Standards reference

- **NTCIP 1202 v03A** (May 2019): "Object Definitions for Actuated Signal Controllers (ASC) Interface". Joint AASHTO/ITE/NEMA standard. The MIB file is available from NEMA at `ntcip@nema.org`.
- **NEMA TS 2**: Referenced throughout for parameter semantics (e.g., NEMA TS 2 Clause 3.5.3.1 for phase timing).
- Status of NtcipEventCode: removed. NTCIP 1202 is OID-addressed and does not define an event-code enumeration. See [ihr_events.proto](ihr_events.md) for the canonical event log.

## What this proto does NOT cover

- Day-plan/schedule structures from NTCIP 1201 Global MIB (`dayPlan`, `dayPlanEvent`, `schedulePlan`, `globalTime`).
- ASC I/O Mapping table (`.13` — vendor-specific input/output function assignment).
- Cabinet environmentals (`.12` — temperature/humidity sensor tables).
- SIU Port 1 / NEMA TS2 Port 1 serial communications (`.10`, `.14`).
- Volume/occupancy reporting (`.2.5` — derive from raw IHR detector events instead).
- ASC Block Objects (`.3.17+` — vendor-specific bulk transfer objects).

If you need any of the above, layer additional protos on top — don't put them here. This file is the canonical NTCIP 1202 v03A object definitions only.
