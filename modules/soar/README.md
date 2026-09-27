# SOAR & Automated Response

**Status: NOT STARTED**

## Planned scope

Close the loop from alert to human decision with automation:

- **n8n** — orchestration layer for enrichment and case-creation workflows,
  triggered off Wazuh/Splunk alerts.
- **Enrichment** — automated lookups (IP/domain reputation, hash checks)
  attached to alerts before an analyst ever opens them.
- **TheHive** — case management: alerts promoted to cases, triage state,
  analyst notes, response actions tracked.
- Downstream of this: AI-assisted alert summarization and playbook
  suggestions, always with human-approved response (see
  `shared/architecture.md` for the platform-wide data flow this plugs into).

## Dependencies

- Consumes alerts produced by the
  [Detection Pipeline](../detection-pipeline/README.md) module (and
  eventually Adversary Emulation / Cloud & Identity Security).
- No infrastructure stood up yet — see `shared/vm-inventory.md` for planned
  assets (TheHive / MISP / OpenCTI listed as SOAR / Threat Intelligence
  phase).
