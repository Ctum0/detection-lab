# CTUM — Detection Engineering & Security Operations Platform

A detection engineering and security operations platform, built and
operated end to end: attacks executed in a controlled lab, telemetry
flowing into a dual-SIEM stack, detections managed as code with CI/CD,
purple-team validation against real attack execution, and — as the
platform grows — a threat-intel feedback loop and analyst-augmented SOAR
closing the loop back to human decision. Organized as a monorepo: one
folder per flagship initiative, with the detection library, attack-test
library, and pipeline tooling shared across all of them at the platform
level.

## Getting started

```bash
git clone git@github.com:Ctum0/ctum-platform.git
cd ctum-platform
```

## Lifecycle

```text
Attack (Parrot / Atomic Red Team)
  |
  v
Victims (Win10 + Sysmon, Ubuntu + auditd)     [Proxmox VMs]
  |
  v
Wazuh agents (TLS, port 1514)
  |
  v
Wazuh Manager (VPS, docker single-node)  -->  Wazuh Indexer + Dashboard
  |                        |
  |                        +--> (planned) Splunk forwarding
  v
Detections (Sigma as code -> CI -> CD -> Wazuh)
  |
  v
Alerts -> Investigation (Threat Hunting)
  |
  v
(planned) Threat intel feedback -> SOAR: enrichment -> AI summary ->
case mgmt -> human decision
```

Full platform architecture (network, hosts, remote access) is in
`shared/architecture.md`.

## Flagships

| Flagship | Focus | Status |
|---|---|---|
| [F1 — Detection Pipeline](modules/detection-pipeline/README.md) | Sigma→CI/CD→Wazuh, 12 rules, 9 validated | ✅ COMPLETE |
| [F2 — Adversary Emulation & AD Lab](modules/adversary-emulation/README.md) | ART campaign, AD domain (planned), purple-team loop | 🔨 W1 COMPLETE |
| [F3 — Cloud & Identity Security](modules/cloud-identity/README.md) | Entra ID / AWS monitoring, IaC scanning | ⬜ Not started |
| [F4 — Analyst SOAR](modules/soar/README.md) | Enrichment, AI-assisted triage, human-approved response | ⬜ Not started |
| [F5 — Threat Intel](modules/threat-intel/README.md) | Honeypots (Cowrie/T-Pot), MISP/OpenCTI, IOC feedback into F1 | ⬜ Not started |

F1's 9/12 (not 10/12) is the actual current count — see Metrics below for
exactly which three remain untested and why.

## Platform architecture

```text
                    modules/
        (F1 complete, F2 W1 complete, F3-F5 not started)
                         |
         each flagship's own docs/ (matrix, campaign
         logs, per-detection writeups — genuinely
         flagship-specific content only)
                         |
      -------------------+-------------------
      |                  |                  |
 detections/        attack-tests/       platform/
 (Sigma source    (one writeup per   (converters/,
 of truth, Wazuh   validated          pipelines/,
 + generated       technique —        workflows-docs.md
 Splunk SPL)        F1<->F2 bridge)    — shared tooling)
      |                  |                  |
      -------------------+-------------------
                         |
                     shared/
      (architecture, vm-inventory, evidence,
              evidence-index, lessons-learned)
```

`detections/`, `attack-tests/`, and `platform/` are cross-flagship
resources at the repo root, not owned by any one flagship — F1 authors
into `detections/` and `platform/`, F2 authors into `attack-tests/` (and
both consume `shared/`). This is deliberate: a new flagship never needs
its own copy of the Sigma pipeline or its own evidence index, it just
plugs into what's already here.

## How to add a detection

1. Write the rule in `detections/sigma/<name>.yml` (Sigma spec v2.1; run
   `sigma check` locally before pushing).
2. Push — CI validates and auto-generates the Splunk SPL.
3. Run `platform/converters/sigma_to_wazuh.py` to regenerate
   `custom_rules.xml`, hand-review the diff against the live Wazuh
   ruleset (see `platform/converters/README.md` for mapping decisions and
   caveats), then push — CD deploys and restarts the manager. Full
   CI/CD mechanics: `platform/workflows-docs.md`.
4. Add a per-detection doc in
   `modules/detection-pipeline/docs/detections/det-NNN-<slug>.md`
   (template: header table, Logic, Expected telemetry, Validation method,
   FP notes, Investigation guidance) and a row in
   `modules/detection-pipeline/docs/attack-matrix.md`.
5. Validate it: build a writeup in `attack-tests/`, run the attack,
   capture evidence into `shared/evidence/`, index it in
   `shared/evidence-index.md`, and flip the det-doc + matrix status to
   VALIDATED.

## New flagship

Add a folder under `modules/` with a `README.md` (purpose, scope,
status) and a row in the table above. Nothing else changes — it reads
from `detections/`, `attack-tests/`, `platform/`, and `shared/` the same
way F1 and F2 already do.

## Metrics

- **Detections:** 12 (DET-001…DET-012) across 10 ATT&CK techniques — see
  `modules/detection-pipeline/docs/attack-matrix.md`
- **Validated:** 9/12 (DET-001, DET-002, DET-003, DET-004, DET-005,
  DET-006, DET-007, DET-011, DET-012)
- **Untested:** DET-008/DET-009 — blocked by an auditd ingestion gap (see
  `modules/detection-pipeline/docs/known-limitations.md`); DET-010 —
  manual deployment only (temporal correlation, no Wazuh equivalent)
- **Sigma rules:** 12 files (spec v2.1, `sigma check` clean)
- **Wazuh rules:** 12 custom rules, IDs 100001–100012
- **Splunk SPL:** 11 auto-generated searches (DET-010 excluded)
- **Evidence:** indexed in `shared/evidence-index.md`, files in
  `shared/evidence/`

Distilled lessons from building this (Sigma/spec, Wazuh rules, Pipeline/CD,
Telemetry, ART) are in `shared/lessons-learned.md`.

## Repo layout

```text
ctum-platform/
├── README.md
├── modules/
│   ├── f1-detection-pipeline/
│   │   └── docs/                ← attack-matrix, per-detection writeups,
│   │                               pipeline-demo, known-limitations
│   ├── f2-adversary-ad-lab/
│   │   └── docs/                ← campaign-log.md
│   ├── f3-cloud-identity/       ← README only (not started)
│   ├── f4-soar/                 ← README only (not started)
│   └── f5-threat-intel/         ← README only (not started)
├── platform/
│   ├── converters/              ← sigma_to_wazuh.py
│   ├── pipelines/               ← Sigma conversion pipeline configs
│   └── workflows-docs.md        ← how CI + CD actually work
├── shared/
│   ├── architecture.md          ← platform-wide network/hosts/data flow
│   ├── vm-inventory.md          ← VM & asset inventory
│   ├── evidence/                ← screenshot evidence (all flagships)
│   ├── evidence-index.md        ← screenshot index
│   └── lessons-learned.md       ← distilled lessons across flagships
├── detections/
│   ├── sigma/                   ← source of truth (12 rules)
│   ├── wazuh/                   ← custom_rules.xml, DEPLOY-NOTES.md
│   └── splunk/                  ← CI-generated *.spl (never hand-edited)
├── attack-tests/                ← one writeup per validated technique
└── .github/workflows/           ← CI: validate + SPL autogen; CD: Wazuh deploy
```

## Author

Ctum0 (sithumsryt@gmail.com)
