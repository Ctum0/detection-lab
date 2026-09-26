# DET-011 — Clearing Windows Security Event Log

| Field | Value |
|---|---|
| Detection ID | DET-011 |
| Rule | Sigma: `detections/sigma/security_log_cleared.yml` |
| Data source | Windows Security log, Event ID 1102, via Wazuh agent |
| ATT&CK | T1685.005 — Disable or Modify Tools: Clear Windows Event Logs (Defense Impairment) |
| Severity | High |
| Status | UNTESTED |

## Logic

`EventID 1102`. The clearing action itself is logged before the log is wiped, so the event ships if forwarding is live.

## Expected telemetry

Security EID 1102 with `SubjectUserName` / `SubjectDomainName` identifying who cleared the log.

## Validation method

On the Win10 victim run `wevtutil cl Security` (or Clear-EventLog) as admin; confirm 1102 ships and the rule fires. Re-enable forwarding checks afterwards.

## FP notes

Authorised admins during lab rebuilds. In steady state there is no legitimate reason — treat unplanned hits as compromise indicators.

## Investigation guidance

1. Who cleared it (`SubjectUserName`)? Expected admin or attacker?
2. What happened just before the clear — pull forwarded copies (Wazuh/Splunk) for the missing window.
3. Look for concurrent defence-evasion: service stops, audit policy changes (4719).
