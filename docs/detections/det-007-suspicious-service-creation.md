# DET-007 — Windows Service Creation by Suspicious Path

| Field | Value |
|---|---|
| Detection ID | DET-007 |
| Rule | Sigma: `detections/sigma/suspicious_service_creation.yml` |
| Data source | Windows System log, Event ID 7045, via Wazuh agent |
| ATT&CK | T1543.003 — Windows Service |
| Severity | High |
| Status | UNTESTED |

## Logic

`EventID 7045` AND `ImagePath` contains temp/public/ProgramData/AppData paths, interpreter names (`powershell`, `pwsh`, `cmd.exe`) or script extensions (`.ps1`, `.vbs`, `.js`, `.hta`).

## Expected telemetry

System EID 7045 with `ServiceName`, `ImagePath` (suspicious location), `ServiceType`, `StartType`, `AccountName`.

## Validation method

Install a test service pointing at a Temp binary: `sc create LabSvc binPath= "C:\Temp\labtest.exe"`; confirm 7045 and the alert, then `sc delete LabSvc`.

## FP notes

Some third-party agents legitimately install from ProgramData/AppData — allowlist by `ServiceName` + publisher after review.

## Investigation guidance

1. Does the `ImagePath` binary exist? Check hash, signature, VT.
2. Which account does the service run as (`AccountName` = LocalSystem is worst case)?
3. Look for the dropper: how did the binary get there?
