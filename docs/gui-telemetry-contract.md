# Firmware ↔ GUI Telemetry Contract

Status: draft for firmware sign-off  
Owner: Firmware + Dashboard team  
Last updated: 2026-08-17

## Contract rule

Firmware is authoritative. The GUI renders declared values and states; it must not
recalculate link scores, anomaly decisions, route reasons, thresholds, or timeouts.
Any field not present, invalid, or stale is displayed as unavailable/stale—not as zero.

## Transport and framing

The current demo transport is newline-delimited UTF-8 JSON over USB serial or the local
WebSocket bridge. WebSocket messages contain one JSON object equivalent to one serial
line; the bridge does not transform payloads.

| Property | Current contract |
|---|---|
| Serial baud | 115200 default; selectable by GUI |
| Serial framing | One JSON object per `LF` line; optional `CR` accepted |
| Encoding | UTF-8 |
| Maximum line | 4096 bytes; firmware must reject/truncate larger records safely |
| WebSocket | `ws://localhost:8765` default bridge endpoint |
| Checksum/CRC | TBD for binary transport; JSON phase currently has none |
| Protocol version | Required in `hello`; value TBD and must be frozen before hardware integration |
| Timestamp | Monotonic milliseconds since boot; `boot_id` distinguishes reboots |
| Sequence | Monotonic per source stream; gaps mean telemetry may be missing |

The binary frame described in the project checklist is a future transport, not a
requirement of the current JSON bridge. Do not silently introduce a second framing rule.

## Message registry

These names are reserved. Numeric IDs and exact frequencies require firmware sign-off.

| Type | Direction | Frequency | Purpose |
|---|---|---:|---|
| `HELLO` | FW → GUI | boot | protocol, firmware, node metadata, configuration |
| `HEARTBEAT` | FW → GUI | 1 Hz | liveness, uptime, sequence |
| `NODE_STATUS` | FW → GUI | on change / 1 Hz | role and health |
| `LINK_UPDATE` | FW → GUI | 2–5 Hz | authoritative link metrics |
| `ROUTE_UPDATE` | FW → GUI | on change | active and candidate routes |
| `PREDICTION` | FW → GUI | 2–5 Hz / on state change | degradation prediction and hysteresis |
| `SENSOR_STATUS` | FW → GUI | configurable | sensor value and detector output |
| `EVENT` | FW → GUI | immediately | significant events |
| `STATISTICS` | FW → GUI | 1 Hz | PDR and reliability aggregates |
| `ERROR` | FW → GUI | immediately | firmware errors |

Every message uses this envelope:

```json
{
  "protocolVersion": "TBD",
  "type": "LINK_UPDATE",
  "nodeId": "A",
  "bootId": "TBD",
  "seq": 104,
  "timestampMs": 123456,
  "payload": {}
}
```

Unknown message types and unknown fields must be ignored safely. A malformed line must
not terminate the telemetry stream. Firmware should emit `ERROR` for firmware-side
serialization failures; the GUI logs parse failures separately.

## Canonical identity and enums

Node IDs are strings `A`, `B`, `C`, `D`, and `S` for this demo. Firmware must use these
same IDs everywhere; the GUI must not map `1` to `A` implicitly. The node role enum is:
`SOURCE`, `RELAY`, `SINK`.

Frozen state vocabularies:

- Node: `ONLINE`, `STALE`, `OFFLINE`, `ERROR`
- Link: `UNKNOWN`, `HEALTHY`, `DEGRADING`, `UNHEALTHY`, `STALE`, `RECOVERING`
- Route: `ACTIVE`, `BACKUP`, `UNAVAILABLE`, `EXPIRED`
- Route reason: `LINK_DEGRADATION`, `LINK_FAILURE`, `STALE_NEIGHBOR`,
  `ROUTE_EXPIRED`, `PRIORITY_OVERRIDE`, `ROUTE_RECOVERY`, `MANUAL`, `UNKNOWN`
- Sensor: `NORMAL`, `SUSPECT`, `ANOMALY`, `FLATLINE`, `OUT_OF_RANGE`, `STALE`
- Severity: `INFO`, `WARN`, `ERROR`, `CRITICAL`

## Required payloads

### `HELLO`

Must contain `nodeId`, `nodeName`, `role`, `mac` (when available), `firmwareVersion`,
`protocolVersion`, and `config`. `config` must declare `heartbeatIntervalMs`,
`offlineTimeoutMs`, `routeTimeoutMs`, `tLow`, `tHigh`, `ewmaAlpha`, and telemetry rates.

### `HEARTBEAT`

```json
{"uptimeMs":752000}
```

The GUI marks a node online on receipt and stale/offline only using the firmware-declared
`offlineTimeoutMs`; it must preserve the last value as stale rather than replace it.

### `LINK_UPDATE`

```json
{
  "from":"A", "to":"B", "rssiDbm":-64, "rssiEwmaDbm":-62,
  "rssiSlopeDbPerSec":-1.8, "pdr":0.91, "pdrEwma":0.92,
  "stalenessMs":120, "linkScore":0.78, "state":"DEGRADING"
}
```

Firmware owns `linkScore` and `state`. Units: RSSI dBm, slope dB/s, PDR and score 0–1,
staleness milliseconds.

### `ROUTE_UPDATE`

```json
{
  "destination":"S",
  "active":{"hops":["A","B","S"],"score":0.82,"state":"ACTIVE"},
  "candidates":[
    {"hops":["A","B","S"],"score":0.82,"state":"ACTIVE"},
    {"hops":["A","C","D","S"],"score":0.91,"state":"BACKUP"}
  ],
  "trafficClass":"NORMAL",
  "reason":"LINK_DEGRADATION"
}
```

`trafficClass` is `NORMAL` or `PRIORITY`. A priority packet may use `hops:["A","S"]`
and must carry `reason:"PRIORITY_OVERRIDE"`; the GUI must not infer priority from hop
count.

### `PREDICTION`

Must include the authoritative `linkScore`, `predictionState`, `tLow`, `tHigh`, and
the current hysteresis state, plus the link metrics from `LINK_UPDATE` when available.
The GUI displays the state and thresholds; it does not reproduce the predictor.

### `SENSOR_STATUS`

Must include `sensorId`, `sensorType`, `value`, `healthState`, and `durationMs` for
`FLATLINE`. For MAD-Z, include `rawValue`, `baseline`, `mad`, `zScore`, and `threshold`.
The detector output is authoritative firmware data.

### `EVENT`

```json
{
  "eventType":"ROUTE_CHANGE", "severity":"INFO", "source":"A",
  "details":{"oldHops":["A","B","S"],"newHops":["A","C","D","S"],
             "reason":"LINK_DEGRADATION","leadTimeMs":232}
}
```

Required event types include `NODE_JOIN`, `NODE_LEAVE`, `LINK_DEGRADING`, `LINK_FAILURE`,
`ROUTE_CHANGE`, `ROUTE_RECOVERY`, `SENSOR_ANOMALY`, `SENSOR_FAILURE`, `PACKET_RETRY`,
`PACKET_DROP`, `PRIORITY_ROUTE`, and `ERROR`.

### `STATISTICS`

Firmware supplies `pdr`, transmitted/acknowledged/dropped counts, retry and duplicate
counts, sequence information, and end-to-end latency. PDR is always 0–1, latency is ms,
and counts are non-negative integers.

## Current rehearsal compatibility

Before the envelope is implemented, the dashboard accepts the existing flat demo object:

```json
{"linkAB":0.82,"route":"ABS","trafficType":"normal","flagC":"clean",
 "pdr":0.99,"rerouteLeadMs":143}
```

This is a temporary adapter shape only. `route` is `ABS` or `ACDS`; `AS` is allowed only
with `trafficType:"priority"`. The GUI must warn on unrecognized normal routes and must
not guess topology, thresholds, lead time, or anomaly state.

## GUI ↔ firmware commands

If control is enabled, commands use the same versioned envelope and typed payloads:
`REQUEST_STATUS`, `REQUEST_ROUTE_TABLE`, `REQUEST_LINK_TABLE`, `SET_DEMO_MODE`,
`RESET_STATS`, `START_TEST`, and `STOP_TEST`. Arbitrary serial strings are not part of
this contract.

## Integration acceptance checklist

- [ ] Firmware signs off protocol version, numeric message IDs, enums, units, rates,
  timeout semantics, and invalid/stale-data rules.
- [ ] `HELLO` displays every node's ID, role, firmware, protocol, and configuration.
- [ ] Heartbeat loss becomes stale/offline without fabricating zero values.
- [ ] Active, backup, priority, predictive reroute, timeout fallback, recovery, sensor
  anomaly/flatline, and reliability events are all represented by firmware messages.
- [ ] Sequence gaps and boot changes are visible in diagnostics.
- [ ] Walk-away and sudden-silence tests prove prediction and timeout are distinct.
- [ ] GUI replay tests use captured contract messages, not ad-hoc serial print strings.

## Sign-off

Firmware owner: ____________________  Date: __________  Protocol version: __________

GUI owner: _________________________  Date: __________  Contract revision: __________
