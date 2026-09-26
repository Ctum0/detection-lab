# DET-008 — Linux SUID Binary Privilege Escalation Attempt

| Field | Value |
|---|---|
| Detection ID | DET-008 |
| Rule | Sigma: `detections/sigma/suid_privilege_escalation.yml` |
| Data source | Linux auditd EXECVE via Wazuh agent |
| ATT&CK | T1548.001 — Setuid and Setgid |
| Severity | Medium |
| Status | UNTESTED |

## Logic

`type EXECVE` AND `exe` is chmod AND (`a1`/`a2` contains `u+s`, `4755`, `4750` or `4777`). Single-event rule.

## Expected telemetry

Auditd EXECVE record: `exe=/usr/bin/chmod`, `a0=chmod`, `a1=u+s` (or `4755`), `a2=<target>`, with `uid/auid` identifying the actor.

## Validation method

On the Linux lab box run `touch /tmp/lab_suid && chmod u+s /tmp/lab_suid`; confirm the EXECVE record ships and the rule fires, then remove the file.

## FP notes

Hardening scripts and package post-installs set SUID deliberately; lab testing (coordinate first). Consider scoping out known config-management UIDs.

## Investigation guidance

1. Who ran it (`auid`, `uid`, ssh session)?
2. What file got SUID — attacker-controlled binary or system file?
3. Was the SUID binary executed afterwards?
