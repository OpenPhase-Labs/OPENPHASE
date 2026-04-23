# security.proto — Security Alerts and Threat Detection

`security.proto` carries security-relevant events: spoofing attempts, MMU conflict-monitor trips, intrusion alerts, and other threat indications. Consumers route these to SOC dashboards, alerting pipelines, or veto-logic decision systems.

## `SecurityAlert`

| Field | # | Type | Notes |
|-------|---|------|-------|
| `level` | 1 | AlertLevel | Severity classification |
| `source` | 2 | string | Free-form alert source (e.g., `"SNIFFER"`, `"PALO_ALTO"`, `"MMU_CONFLICT"`) |
| `description` | 3 | string | Human-readable description |
| `physical_fingerprint` | 4 | bytes | RF signal characteristics (for spoof detection) |
| `threat_type` | 10 | ThreatType | Structured threat classification (optional) |
| `attacker_signature` | 11 | string | Known attack pattern identifier (optional) |

## `AlertLevel` enum

| Value | Name | Action |
|-------|------|--------|
| 0 | `ALERT_LEVEL_UNSPECIFIED` | Producer didn't set |
| 1 | `INFO` | Informational — log only, no action |
| 2 | `WARNING` | Potential spoofing or anomaly — investigate |
| 3 | `CRITICAL` | MMU conflict, physical breach, or confirmed attack — page on-call |

## `ThreatType` enum

| Value | Name | Meaning |
|-------|------|---------|
| 0 | `THREAT_TYPE_UNSPECIFIED` | |
| 1 | `GPS_SPOOFING` | GPS signal manipulation detected |
| 2 | `TSP_REPLAY_ATTACK` | Transit signal priority request replay |
| 3 | `PREEMPT_SPOOF` | Emergency preemption spoof attempt |
| 4 | `MMU_CONFLICT` | Malfunction Management Unit detected conflicting greens |
| 5 | `NETWORK_INTRUSION` | Firewall/IDS alert (Palo Alto, etc.) |

## Notes

- `physical_fingerprint` is intentionally `bytes` — the format is producer-defined (RF spectral characteristics, signal-strength patterns, etc.). Consumers comparing fingerprints should agree on the encoding out-of-band.
- `source` is free-form to accommodate diverse third-party sources without proto changes. Producers should keep values stable across alerts (e.g., always `"PALO_ALTO"`, not sometimes `"PaloAlto"`).
- Add new threat types in `ThreatType` rather than overloading `description` — keeps consumers' filtering/routing simple.
