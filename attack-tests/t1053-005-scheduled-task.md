# T1053.005 — Scheduled Task Creation

Validates: [DET-006](../modules/detection-pipeline/docs/detections/det-006-scheduled-task-creation.md)
(Wazuh custom 100006, chains stock 60228)

## Commands

Run on `windows-victim`:

```powershell
schtasks /create /tn LabTest /tr calc.exe /sc onlogon
```

## Expected telemetry

Security EID 4698 (`A scheduled task was created`) with `TaskName`,
`TaskContent` (XML with the Exec action) and `SubjectUserName`.

## Rules fired

- Stock 60228 `A scheduled task was created` — observed as the parent
  event.
- Custom Wazuh rule 100006 `Detection Platform: Scheduled task created (4698)
  [T1053.005]`, level 7 — fired shortly after a manager restart from an
  unrelated rule deploy (restart timing was coincidental, not required
  for this rule).

## Cleanup

```powershell
schtasks /delete /tn LabTest /f
```

## Notes

Ran as a manual `schtasks` command, not a named ART test. Evidence:
`shared/evidence/det006-rule100006-task.png`,
`shared/evidence/det006-scheduled-task-60228.png`.
