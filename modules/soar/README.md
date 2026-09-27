# Flagship 4 — SOAR & Automated Response

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

- Consumes alerts produced by [Flagship 1](../detection-pipeline/README.md)
  (and eventually F2/F3).
- No infrastructure stood up yet — see `shared/vm-inventory.md` for planned
  assets (TheHive / MISP / OpenCTI listed as Flagship 4 / TI phase).
