# Wazuh custom rules — deploy notes

`custom_rules.xml` is deployed automatically by CI (deploy-wazuh workflow)
to the Wazuh manager via the API (PUT /rules/files/local_rules.xml), which
then restarts the manager container — single-node docker does not
hot-reload rule files on PUT alone (see `../../shared/lessons-learned.md`).

IMPORTANT: keep this file free of XML comments — the Wazuh 4.14 API
rejects comment blocks containing HTML-escaped entities (&gt; &amp;)
with an internal error. Mapping info lives here instead.

Rule ID <-> Sigma rule mapping:
  100001 <-> linux_reverse_shell.yml (reverse shell, auditd)
  100002 <-> local_user_creation.yml (4720)
  100003 <-> lsass_credential_dumping.yml (Sysmon EID10) — VALIDATED firing
  100004 <-> powershell_download_cradle.yml (Sysmon EID1) — VALIDATED firing
  100005 <-> powershell_encoded_command.yml (Sysmon EID1) — VALIDATED firing
  100006 <-> scheduled_task_creation.yml (4698, parent 60228) — VALIDATED firing
  100007 <-> security_log_cleared.yml (1102, parent 63103) — VALIDATED firing
  100008 <-> ssh_bruteforce.yml base rule
  100009 <-> ssh_bruteforce.yml correlation (freq 6/600s)
  100010 <-> suid_privilege_escalation.yml (auditd)
  100011 <-> suspicious_service_creation.yml (7045) — VALIDATED firing
  100012 <-> notepad_execution.yml (Sysmon EID1, CI/CD canary) — VALIDATED firing
Skipped: ssh_success_after_failures.yml (temporal_ordered unsupported)

100007 parent fix (2026-09-27): originally anchored on
`<if_group>windows_security</if_group>`, which never fired — the 1102
"log cleared" event decodes under stock parent rule 63103, not the
generic windows_security group. Fixed to `<if_sid>63103</if_sid>`;
verified 63103 decodes eventID 1102 before redeploying. Full trail in
`../../flagships/f1-detection-pipeline/docs/detections/det-011-security-log-cleared.md`.
