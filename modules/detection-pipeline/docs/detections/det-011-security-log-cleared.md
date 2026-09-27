# DET-011 — Clearing Windows Security Event Log

| Field | Value |
|---|---|
| Detection ID | DET-011 |
| Rule | Sigma `detections/sigma/security_log_cleared.yml`; Wazuh 100007 (`if_sid 63103`); SPL `detections/splunk/security_log_cleared.spl` |
| Data source | Windows Security log, Event ID 1102, via Wazuh agent |
| ATT&CK | T1685.005 — Disable or Modify Tools: Clear Windows Event Logs (Defense Impairment) |
| Severity | High |
| Status | VALIDATED 2026-09-27 |

## Logic

`EventID 1102`. The clearing action itself is logged before the log is wiped, so the event ships if forwarding is live.

## Expected telemetry

Security EID 1102 with `SubjectUserName` / `SubjectDomainName` identifying who cleared the log.

## Validation method

On the Win10 victim run `wevtutil cl Security` (or Clear-EventLog) as admin; confirm 1102 ships and the rule fires. Re-enable forwarding checks afterwards.

Validated 2026-09-27 as part of the Adversary Emulation module's Workstream 1 ART campaign
(`modules/adversary-emulation/docs/campaign-log.md`,
`attack-tests/t1685-005-log-clearing.md`).
Two bugs surfaced before this one actually fired:

1. **Wrong parent (rule-authoring bug).** Rule 100007 originally anchored
   on `<if_group>windows_security</if_group>` — plausible, since 1102 is a
   Security-channel event, but wrong: on this Wazuh 4.14.8 ruleset the
   1102 "log cleared" event decodes under the specific stock rule
   **63103** ("The audit log was cleared"), not the generic
   `windows_security` group. `windows_security` never matched, so 100007
   silently never fired despite the base 1102 events shipping fine. Fixed
   by anchoring on `<if_sid>63103</if_sid>` instead — verified first with
   `wazuh-logtest` that 63103 decodes an EventID 1102 record before
   redeploying (see `../../../../detections/wazuh/DEPLOY-NOTES.md`).
2. **CD hot-reload bug (deploy pipeline, not this rule).** After fixing
   the parent and pushing, the deploy workflow reported a clean `HTTP 200`
   / `error: 0` from the Wazuh API PUT, but the rule still didn't fire on
   a re-test. Root cause: in this single-node docker deployment, PUTting
   `local_rules.xml` writes the file to disk but `analysisd` does not
   hot-reload it — a manager restart is required before an edited rule
   takes effect. `deploy-wazuh.yml` was fixed to `docker restart
   single-node-wazuh.manager-1` and verify agent connectivity after every
   deploy (see `shared/lessons-learned.md`; this bug affects every rule
   edit, not just this one, and was found here first).

With both fixed: ran Atomic Red Team test **T1685.005-4 "BlackCat
Ransomware Full Log Clear"** on `windows-victim`, which loops
`wevtutil.exe cl "<log>"` across dozens of event-log channels (including
Security) — ending on an expected failure clearing
`Microsoft-Windows-LiveId/Analytic` (`Access is denied`), which does not
affect the Security-log clear earlier in the loop. Security EID 1102
shipped, and custom Wazuh rule 100007 `Detection Platform: Windows Security event log
cleared (1102) [T1685.005]` fired at level 10 at 18:22:44. Evidence:
`shared/evidence/det011-wevtutil-attack.png` (attack view — the ART test
run), `shared/evidence/det011-rule100007-alert.png` (SIEM view — the
rule firing).

## FP notes

Authorised admins during lab rebuilds. In steady state there is no legitimate reason — treat unplanned hits as compromise indicators.

## Investigation guidance

1. Who cleared it (`SubjectUserName`)? Expected admin or attacker?
2. What happened just before the clear — pull forwarded copies (Wazuh/Splunk) for the missing window.
3. Look for concurrent defence-evasion: service stops, audit policy changes (4719).
