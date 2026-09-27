# DET-006 — Scheduled Task Creation

| Field | Value |
|---|---|
| Detection ID | DET-006 |
| Rule | Sigma `detections/sigma/scheduled_task_creation.yml`; Wazuh 100006 (chains stock 60228); SPL `detections/splunk/scheduled_task_creation.spl` |
| Data source | Windows Security log, Event ID 4698, via Wazuh agent |
| ATT&CK | T1053.005 — Scheduled Task |
| Severity | Medium |
| Status | VALIDATED 2026-09-27 |

## Logic

`EventID 4698`. Every task registration is reviewed: task name, action, trigger and author.

## Expected telemetry

Security EID 4698 including `TaskName`, `TaskContent` (XML with Exec action/command), and `SubjectUserName`.

## Validation method

Create a test task with `schtasks /create /tn LabTest /tr calc.exe /sc onlogon`; confirm 4698 ships and the rule fires, then delete it.

Validated 2026-09-27 (SIEM side): custom Wazuh rule 100006
`Detection Platform: Scheduled task created (4698) [T1053.005]` fired on windows-victim
at 03:16, level 7, shortly after a `Wazuh server started` event (rule
deploy restart). Stock parent 60228 observed earlier the same night.
Evidence: `evidence/det006-rule100006-task.png`,
`evidence/det006-scheduled-task-60228.png`. Attack-terminal screenshot
still to capture.

## FP notes

Updaters, backup agents and legit admin tasks register frequently. Baseline common `TaskName` values before tightening.

## Investigation guidance

1. Parse `TaskContent`: what binary/args run, and on what trigger?
2. Is the action an interpreter (`powershell`, `cmd`, `wscript`) with obfuscated args?
3. Check task author vs expected admin accounts.
