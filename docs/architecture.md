# Architecture

Lab topology, log flow, and tooling for `detection-lab`.

## Topology

```text
[ Attacker VM ] ──> [ Victim VM (Sysmon) ] ──> [ Wazuh / Splunk ] ──> Detections
```

Update this diagram as the lab grows.

## Log sources

| Source | Ships to | Notes |
| ------ | -------- | ----- |
| Sysmon | Wazuh / Splunk | Main Windows telemetry |
| Windows Event Log | Wazuh / Splunk | Security / Sysmon channels |
| _add more_ | | |

## Detection pipeline

1. Write rule in `detections/sigma/` (preferred), then convert to `detections/wazuh/` / `detections/splunk/` as needed.
2. Document each detection in `docs/detections/<name>.md`.
3. Validate with a scenario script in `attack-tests/`.
