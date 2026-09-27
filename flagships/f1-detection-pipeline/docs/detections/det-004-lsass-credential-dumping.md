# DET-004 — Windows Credential Dumping via LSASS Access

| Field | Value |
|---|---|
| Detection ID | DET-004 |
| Rule | Sigma `detections/sigma/lsass_credential_dumping.yml`; Wazuh 100003; SPL `detections/splunk/lsass_credential_dumping.spl` |
| Data source | Sysmon Event ID 10 (process_access) via Wazuh agent |
| ATT&CK | T1003.001 — LSASS Memory |
| Severity | Critical |
| Status | VALIDATED 2026-09-27 |

## Logic

`EventID 10` AND `TargetImage endswith \lsass.exe` AND `GrantedAccess` in (`0x1010`, `0x1410`, `0x1438`, `0x143a`, `0x1fffff`). Single-event rule.

## Expected telemetry

Sysmon EID 10 with `SourceImage` = dumping tool, `TargetImage` = `C:\Windows\system32\lsass.exe`, `GrantedAccess` = one of the sensitive masks, plus `CallTrace` showing the offending module.

## Validation method

Run a benign LSASS-handle probe (e.g. Sysinternals `procdump -ma lsass.exe` in test mode, or Atomic Red Team T1003.001 test) on the Win10 victim; confirm EID 10 and the alert. Coordinate first — AV/EDR may block.

Validated 2026-09-27 as part of the Flagship 2 Workstream 1 ART campaign
(`flagships/f2-adversary-ad-lab/docs/campaign-log.md`,
`flagships/f2-adversary-ad-lab/attack-tests/t1003-001-lsass-dump.md`): ran
the Atomic Red Team T1003.001 comsvcs.dll MiniDump test on `windows-victim`
— `rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump <lsass PID>
C:\Temp\lsass_dump.dmp full`, dumping LSASS memory via the
undocumented `comsvcs.dll` `MiniDump` export rather than a
dedicated dumping tool. Sysmon EID 10 shipped with `TargetImage`
`C:\Windows\system32\lsass.exe` and a sensitive `GrantedAccess` mask;
custom Wazuh rule 100003 `CTUM: Sensitive handle to LSASS - credential
dumping pattern [T1003.001]` fired twice at level 12 (rundll32 opens the
LSASS handle across two distinct access events for this technique).
Confirmed the firing process was the planned ART test, not unexpected
activity.

**Evidence gap:** no screenshot of custom rule 100003 firing exists yet in
`shared/evidence/` — checked both source screenshot folders and neither
contains a `rule.id:100003` hit, despite the campaign log recording it
fired twice. Only the earlier stock-92900 FP-case screenshot
(`shared/evidence/det004-lsass-92900-svchost.png`) is captured. Status is
kept VALIDATED on the strength of the campaign-log record, but capturing
`det004-comsvcs-attack.png` / `det004-rule100003-alert.png` is a
priority gap — see `shared/evidence-index.md`.

## FP notes

Endpoint security products legitimately open LSASS handles; baseline `SourceImage` values and exclude signed security tooling only.

## Investigation guidance

1. What is `SourceImage` — signed? Known tool? Check hash/VT.
2. Inspect `CallTrace` for unknown DLLs.
3. Hunt for follow-on use: lateral movement, new logons, DCSync-like traffic.
