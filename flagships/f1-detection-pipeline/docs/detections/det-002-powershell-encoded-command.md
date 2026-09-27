# DET-002 — PowerShell Encoded Command Execution

| Field | Value |
|---|---|
| Detection ID | DET-002 |
| Rule | Sigma `detections/sigma/powershell_encoded_command.yml`; Wazuh 100005; SPL `detections/splunk/powershell_encoded_command.spl` |
| Data source | Sysmon process_creation (SwiftOnSecurity config) via Wazuh agent |
| ATT&CK | T1059.001 — PowerShell |
| Severity | High |
| Status | VALIDATED 2026-09-27 |

## Logic

`Image endswith \powershell.exe (or \pwsh.exe)` AND `CommandLine contains -EncodedCommand / -enc`. Single-event rule, no aggregation.

## Expected telemetry

Sysmon Event ID 1 with `Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` and a `CommandLine` containing the encoded flag plus a Base64 blob.

## Validation method

On the Win10 victim run `powershell -enc <base64 of whoami>` and confirm a Sysmon EID 1 ships to Wazuh/Splunk and the rule fires.

Validated 2026-09-27 (SIEM side): custom Wazuh rule 100005
`CTUM: PowerShell with encoded command [T1059.001]` fired on windows-victim
at 03:13, level 10 — first custom Sysmon rule to fire in the lab. Confirm
the triggering process was the planned test and not unexpected activity.
Evidence: `evidence/det002-rule100005-encoded.png`. Attack-terminal
screenshot still to capture.

## FP notes

SCCM/Intune and admin automation legitimately use encoded commands; allowlist known parent images or signing publishers after baselining.

## Investigation guidance

1. Decode the Base64 payload — what does it do?
2. Check parent process: who launched PowerShell?
3. Look for follow-on artefacts: file writes, network connections, persistence.
