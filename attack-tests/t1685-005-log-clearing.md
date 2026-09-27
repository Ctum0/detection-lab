# T1685.005 — Clear Windows Event Logs (Disable or Modify Tools)

Validates: [DET-011](../modules/detection-pipeline/docs/detections/det-011-security-log-cleared.md)
(Wazuh custom 100007, `if_sid 63103`)

## Commands

Ran as admin on `windows-victim` — Atomic Red Team test **T1685.005-4
"BlackCat Ransomware Full Log Clear"**:

```powershell
Invoke-AtomicTest T1685.005 -TestNumbers 4
```

This atomic loops `wevtutil.exe cl "<log>"` across dozens of Windows
event-log channels (Application, System, Security, and many
provider-specific analytic/operational logs), matching what the BlackCat
ransomware family does for anti-forensics. It ends on an expected failure
clearing `Microsoft-Windows-LiveId/Analytic` (`Access is denied`) — a
permissions edge case on that specific channel, not a failure of the
technique; the Security log (the one this detection cares about) clears
earlier in the loop without issue.

## Expected telemetry

Security EID 1102 (`The audit log was cleared`) with `SubjectUserName` /
`SubjectDomainName` identifying who cleared it. The clearing event itself
is logged before the log is wiped, so it ships as long as forwarding is
live.

## Rules fired

- Custom Wazuh rule 100007 `CTUM: Windows Security event log cleared
  (1102) [T1685.005]`, level 10 — **after** fixing two bugs found during
  this test (full trail in the det-doc and `../modules/adversary-emulation/docs/campaign-log.md`):
  1. Rule was anchored on `<if_group>windows_security</if_group>`, which
     never matches this event; 1102 actually decodes under stock parent
     `63103`. Fixed to `<if_sid>63103</if_sid>`, verified against
     `wazuh-logtest` before redeploying.
  2. The CD pipeline's API PUT deploy reported success (HTTP 200,
     `error: 0`) but the manager doesn't hot-reload rules from a PUT
     alone in this single-node docker setup — needed an explicit manager
     restart, which `deploy-wazuh.yml` now does automatically after every
     deploy.

## Cleanup

None — log-clearing is the test itself; no further state to revert
(re-enable/verify forwarding is healthy afterward).

## Evidence

- `shared/evidence/det011-wevtutil-attack.png` — attack view, the ART test
  loop running and its final output line.
- `shared/evidence/det011-rule100007-alert.png` — SIEM view, rule 100007
  firing on `windows-victim` at 2026-09-27 18:22:44.

## Notes

This was the technique that surfaced both the rule-parenting bug and the
CD hot-reload bug — see `shared/lessons-learned.md` for the distilled
version of both. Treat any unplanned 1102 outside a scheduled test as a
compromise indicator (see the det-doc's investigation guidance).
