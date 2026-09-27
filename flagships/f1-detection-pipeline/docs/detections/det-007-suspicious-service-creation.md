# DET-007 — Windows Service Creation by Suspicious Path

| Field | Value |
|---|---|
| Detection ID | DET-007 |
| Rule | Sigma `detections/sigma/suspicious_service_creation.yml`; Wazuh 100011; SPL `detections/splunk/suspicious_service_creation.spl` |
| Data source | Windows System log, Event ID 7045, via Wazuh agent |
| ATT&CK | T1543.003 — Windows Service |
| Severity | High |
| Status | VALIDATED 2026-09-27 |

## Logic

`EventID 7045` AND `ImagePath` contains temp/public/ProgramData/AppData paths, interpreter names (`powershell`, `pwsh`, `cmd.exe`) or script extensions (`.ps1`, `.vbs`, `.js`, `.hta`).

## Expected telemetry

System EID 7045 with `ServiceName`, `ImagePath` (suspicious location), `ServiceType`, `StartType`, `AccountName`.

## Validation method

Install a test service pointing at a Temp binary: `sc create LabSvc binPath= "C:\Temp\labtest.exe"`; confirm 7045 and the alert, then `sc delete LabSvc`.

Validated 2026-09-27 as part of the Flagship 2 Workstream 1 ART campaign
(`flagships/f2-adversary-ad-lab/docs/campaign-log.md`,
`attack-tests/t1543-003-service-creation.md`).

First attempt used the Atomic Red Team T1543.003 test with its default
`binary_path` (something under `C:\AtomicRedTeam\...`) — the service
registered, EID 7045 shipped, but **rule 100011 correctly did not fire**.
This is expected, not a bug: the rule's pattern list (`\Temp\`,
`\Users\Public\`, `\ProgramData\`, `\AppData\`, interpreter names, script
extensions) is threat-model-based — it targets paths an attacker would
realistically stage a payload in — not test-based, so it has no reason to
match ART's own install directory. Re-ran with `-PromptForInputArgs` to
override the input args and set `binary_path=C:\Temp\AtomicService.exe`,
matching the actual threat pattern; EID 7045 shipped with that `ImagePath`
and custom Wazuh rule 100011 `CTUM: Service created with suspicious
binary path [T1543.003]` fired at level 10.
Evidence: `shared/evidence/det007-rule100011-alert.png` (rule 100011,
level 10, 2026-09-27 18:43:53). Attack-terminal screenshot still to
capture.

## FP notes

Some third-party agents legitimately install from ProgramData/AppData — allowlist by `ServiceName` + publisher after review.

## Investigation guidance

1. Does the `ImagePath` binary exist? Check hash, signature, VT.
2. Which account does the service run as (`AccountName` = LocalSystem is worst case)?
3. Look for the dropper: how did the binary get there?
