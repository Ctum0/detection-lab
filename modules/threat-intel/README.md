# Threat Intelligence

**Status: NOT STARTED**

## Planned scope

Close the loop the other direction from SOAR: instead of alerts flowing
out to a human, IOCs flow back in to seed or tune detections.

- **Honeypots** — Cowrie/T-Pot sensors (see `shared/vm-inventory.md`, "TI
  feedback loop") generating real attacker IOCs from exposed low-value
  targets, not simulated data.
- **TIP integration** — MISP/OpenCTI for IOC storage, enrichment, and
  sharing, alongside the case-management work in the
  [SOAR](../soar/README.md) module.
- **Feedback into Detection Pipeline** — honeypot/sensor-sourced
  indicators enriching or seeding new Sigma rules in the
  [Detection Pipeline](../detection-pipeline/README.md) module's
  `detections/sigma/`, same CI/CD path every other detection goes
  through.

## Dependencies

- Feeds new rules into the same `detections/sigma/` -> CI -> CD pipeline
  proven by Detection Pipeline — no new deployment mechanism needed once
  this starts producing rules.
- No infrastructure stood up yet — see `shared/vm-inventory.md` (honeypot,
  TheHive/MISP/OpenCTI listed under SOAR / Threat Intelligence phase).
