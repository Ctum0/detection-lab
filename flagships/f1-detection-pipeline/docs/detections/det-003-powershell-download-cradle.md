# DET-003 — Suspicious PowerShell Download Cradle

| Field | Value |
|---|---|
| Detection ID | DET-003 |
| Rule | Sigma `detections/sigma/powershell_download_cradle.yml`; Wazuh 100004; SPL `detections/splunk/powershell_download_cradle.spl` |
| Data source | Sysmon process_creation (SwiftOnSecurity config) via Wazuh agent |
| ATT&CK | T1059.001 — PowerShell |
| Severity | High |
| Status | VALIDATED 2026-09-27 |

## Logic

`Image endswith \powershell.exe / \pwsh.exe` AND `CommandLine` contains a download primitive (`DownloadString`, `DownloadFile`, `Net.WebClient`, `Invoke-WebRequest`, `Invoke-RestMethod`, `Start-BitsTransfer`) AND an execution primitive (`Invoke-Expression`, `IEX(`, `IEX `).

## Expected telemetry

Sysmon EID 1 whose `CommandLine` shows the full cradle, e.g. `IEX (New-Object Net.WebClient).DownloadString('http://...')`.

## Validation method

Host a benign text file on the attacker box and run a `DownloadString + IEX` cradle pointing at it from the Win10 victim; confirm the EID 1 and the alert.

Validated 2026-09-27 as part of the Flagship 2 Workstream 1 ART campaign
(`flagships/f2-adversary-ad-lab/docs/campaign-log.md`,
`flagships/f2-adversary-ad-lab/attack-tests/t1059-001-powershell-cradle.md`):
ran a `(New-Object Net.WebClient).DownloadString(...)` + `IEX` cradle
against a benign payload hosted on the attacker box from `windows-victim`.
Sysmon EID 1 shipped with the full cradle in `CommandLine`; custom Wazuh
rule 100004 `CTUM: PowerShell download cradle - fetch and execute pattern
[T1059.001]` fired at level 10. This is the same technique family as
DET-002 (encoded command) but a distinct execution pattern — the cradle
downloads and immediately executes in one line, whereas DET-002 catches
pre-staged/obfuscated payloads via `-EncodedCommand`.
Evidence: `shared/evidence/det003-rule100004-alert.png` (rule 100004,
level 10, 2026-09-27 18:51:56). Attack-terminal screenshot still to
capture.

## FP notes

Admin scripts pulling modules/updates then invoking them; deployment tooling with WebClient cradles. Tune with URL/domain allowlists.

## Investigation guidance

1. Extract the URL — is the domain/IP malicious? Fetch the payload safely.
2. Parent process and user context.
3. Did the download succeed (follow-on network/file telemetry)?
