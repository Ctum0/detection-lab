# DET-005 — New Local User Creation

| Field | Value |
|---|---|
| Detection ID | DET-005 |
| Rule | Sigma: `detections/sigma/local_user_creation.yml` |
| Data source | Windows Security log, Event ID 4720, via Wazuh agent |
| ATT&CK | T1136.001 — Local Account |
| Severity | Medium |
| Status | UNTESTED |

## Logic

`EventID 4720`. No AD in the lab yet, so every 4720 is a local account creation on the Win10 victim.

## Expected telemetry

Security EID 4720 with `TargetUserName`, `TargetDomainName` (host), and `SubjectUserName` (who created it).

## Validation method

Run `net user labtest P@ssw0rd! /add` on the victim; confirm 4720 ships and the rule fires, with `SubjectUserName` matching the operator.

## FP notes

Legitimate admin provisioning and installers creating service accounts. Correlate with change tickets; alert on off-hours creation.

## Investigation guidance

1. Who created the account (`SubjectUserName`/logon ID)?
2. Was it added to privileged groups (look for 4732/4728 after)?
3. Any logons with the new account (4624)?
