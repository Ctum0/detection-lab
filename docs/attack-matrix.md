# Attack matrix

All 11 detections: technique → data source → artifacts → status.
Per-detection writeups live in `docs/detections/det-*.md`; screenshot
evidence is indexed in `docs/evidence.md`.

| ID | Technique | Tactic | Data source | Sigma file | Wazuh rule | SPL file | Status | Notes |
|---|---|---|---|---|---|---|---|---|
| DET-001 | T1110.001 Password Guessing | Credential Access | auth.log (sshd) via Wazuh agent | `ssh_bruteforce.yml` | 100008 (base) + 100009 (freq 6/10m); stock 5712 | `ssh_bruteforce.spl` | VALIDATED 2026-09-26 | Hydra burst → stock 5712 fired |
| DET-002 | T1059.001 PowerShell | Execution | Sysmon EID 1 via Wazuh agent | `powershell_encoded_command.yml` | 100005 | `powershell_encoded_command.spl` | VALIDATED 2026-09-27 | Custom 100005 fired lvl 10; attack screenshot pending |
| DET-003 | T1059.001 PowerShell | Execution | Sysmon EID 1 via Wazuh agent | `powershell_download_cradle.yml` | 100004 | `powershell_download_cradle.spl` | UNTESTED | — |
| DET-004 | T1003.001 LSASS Memory | Credential Access | Sysmon EID 10 via Wazuh agent | `lsass_credential_dumping.yml` | 100003 | `lsass_credential_dumping.spl` | UNTESTED | Stock 92900 observed firing on benign svchost read access — the FP case custom rule 100003 is built to exclude; custom rule itself not yet tested |
| DET-005 | T1136.001 Local Account | Persistence | Windows Security 4720 via Wazuh agent | `local_user_creation.yml` | 100002 | `local_user_creation.spl` | VALIDATED 2026-09-27 | `net user backdoor` → 60109 + EID 4722 observed |
| DET-006 | T1053.005 Scheduled Task | Persistence | Windows Security 4698 via Wazuh agent | `scheduled_task_creation.yml` | 100006 (chains stock 60228) | `scheduled_task_creation.spl` | VALIDATED 2026-09-27 | Custom 100006 fired lvl 7 after manager restart; attack screenshot pending |
| DET-007 | T1543.003 Windows Service | Persistence | Windows System 7045 via Wazuh agent | `suspicious_service_creation.yml` | 100011 | `suspicious_service_creation.spl` | UNTESTED | — |
| DET-008 | T1548.001 Setuid and Setgid | Privilege Escalation | Linux auditd EXECVE via Wazuh agent | `suid_privilege_escalation.yml` | 100010 | `suid_privilege_escalation.spl` | UNTESTED | Blocked: auditd ingestion gap (see `docs/known-limitations.md`) |
| DET-009 | T1059.004 Unix Shell | Execution | Linux auditd EXECVE via Wazuh agent | `linux_reverse_shell.yml` | 100001 | `linux_reverse_shell.spl` | UNTESTED | Blocked: auditd ingestion gap (see `docs/known-limitations.md`) |
| DET-010 | T1110.001 Password Guessing | Credential Access | auth.log (sshd), temporal correlation | `ssh_success_after_failures.yml` | SKIP (temporal_ordered unsupported) | — (excluded from SPL autogen) | UNTESTED | Manual deployment only; needs parsed `src_ip` |
| DET-011 | T1685.005 Clear Windows Event Logs | Defense Impairment | Windows Security 1102 via Wazuh agent | `security_log_cleared.yml` | 100007 | `security_log_cleared.spl` | UNTESTED | MITRE restructured 2026: ex-T1070.001 |
| DET-012 | T1098 (canary) | Execution | Sysmon EID 1 via Wazuh agent | `notepad_execution.yml` | 100012 | `notepad_execution.spl` | VALIDATED 2026-09-27 | Pipeline canary: commit -> CI -> deploy -> notepad run -> alert; see `docs/pipeline-demo/` |

Coverage: 10 distinct techniques across Credential Access, Execution,
Persistence, Privilege Escalation and Defense Impairment. 5/12 validated.
