# DET-004 — Windows Credential Dumping via LSASS Access

| Field | Value |
|---|---|
| Detection ID | DET-004 |
| Rule | Sigma: `detections/sigma/lsass_credential_dumping.yml` |
| Data source | Sysmon Event ID 10 (process_access) via Wazuh agent |
| ATT&CK | T1003.001 — LSASS Memory |
| Severity | Critical |
| Status | UNTESTED |

## Logic

`EventID 10` AND `TargetImage endswith \lsass.exe` AND `GrantedAccess` in (`0x1010`, `0x1410`, `0x1438`, `0x143a`, `0x1fffff`). Single-event rule.

## Expected telemetry

Sysmon EID 10 with `SourceImage` = dumping tool, `TargetImage` = `C:\Windows\system32\lsass.exe`, `GrantedAccess` = one of the sensitive masks, plus `CallTrace` showing the offending module.

## Validation method

Run a benign LSASS-handle probe (e.g. Sysinternals `procdump -ma lsass.exe` in test mode, or Atomic Red Team T1003.001 test) on the Win10 victim; confirm EID 10 and the alert. Coordinate first — AV/EDR may block.

## FP notes

Endpoint security products legitimately open LSASS handles; baseline `SourceImage` values and exclude signed security tooling only.

## Investigation guidance

1. What is `SourceImage` — signed? Known tool? Check hash/VT.
2. Inspect `CallTrace` for unknown DLLs.
3. Hunt for follow-on use: lateral movement, new logons, DCSync-like traffic.
