# DET-003 — Suspicious PowerShell Download Cradle

| Field | Value |
|---|---|
| Detection ID | DET-003 |
| Rule | Sigma `detections/sigma/powershell_download_cradle.yml`; Wazuh 100004; SPL `detections/splunk/powershell_download_cradle.spl` |
| Data source | Sysmon process_creation (SwiftOnSecurity config) via Wazuh agent |
| ATT&CK | T1059.001 — PowerShell |
| Severity | High |
| Status | UNTESTED |

## Logic

`Image endswith \powershell.exe / \pwsh.exe` AND `CommandLine` contains a download primitive (`DownloadString`, `DownloadFile`, `Net.WebClient`, `Invoke-WebRequest`, `Invoke-RestMethod`, `Start-BitsTransfer`) AND an execution primitive (`Invoke-Expression`, `IEX(`, `IEX `).

## Expected telemetry

Sysmon EID 1 whose `CommandLine` shows the full cradle, e.g. `IEX (New-Object Net.WebClient).DownloadString('http://...')`.

## Validation method

Host a benign text file on the attacker box and run a `DownloadString + IEX` cradle pointing at it from the Win10 victim; confirm the EID 1 and the alert.

## FP notes

Admin scripts pulling modules/updates then invoking them; deployment tooling with WebClient cradles. Tune with URL/domain allowlists.

## Investigation guidance

1. Extract the URL — is the domain/IP malicious? Fetch the payload safely.
2. Parent process and user context.
3. Did the download succeed (follow-on network/file telemetry)?
