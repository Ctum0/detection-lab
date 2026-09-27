# CTUM — Detection Engineering & Security Operations Platform

A detection engineering and security operations lab, built and operated
end to end: attacks executed in a controlled environment, telemetry
flowing into a dual-SIEM stack, detections managed as code with CI/CD, and
every detection proven by actually running the attack it's built to
catch. Organized as a monorepo, one directory per flagship initiative.

## Getting started

```bash
git clone git@github.com:Ctum0/detection-lab.git
cd detection-lab
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
(planned) SOAR: enrichment -> AI summary -> case mgmt -> human decision
```

Full platform architecture (network, hosts, remote access) is in
`shared/architecture.md`.

## Flagships

| Flagship | Scope | Status |
|---|---|---|
| [F1 — Detection Pipeline](flagships/f1-detection-pipeline/README.md) | Sigma-as-code detections, Wazuh + Splunk translation, CI validation, CD auto-deploy | **COMPLETE** — 9/12 detections validated |
| [F2 — Adversary Emulation & AD Lab](flagships/f2-adversary-ad-lab/README.md) | Atomic Red Team-driven attack execution that validates F1's detections | **W1 (Windows campaign) COMPLETE**, W2-W4 not started |
| [F3 — Cloud & Identity Security](flagships/f3-cloud-identity/README.md) | Entra ID / AWS detections (TBD) | NOT STARTED |
| [F4 — SOAR & Automated Response](flagships/f4-soar/README.md) | n8n enrichment, TheHive case management | NOT STARTED |
| [Threat Intel](threat-intel/README.md) | Honeypot/sensor-sourced IOC feedback loop | NOT STARTED |

## Repo layout

```text
detection-lab/
├── README.md
├── flagships/
│   ├── f1-detection-pipeline/
│   │   ├── detections/          ← sigma/ (source of truth), wazuh/, splunk/ (generated)
│   │   ├── docs/                ← attack-matrix, per-detection writeups, pipeline-demo, known-limitations
│   │   ├── converters/          ← sigma_to_wazuh.py
│   │   └── pipelines/           ← Sigma conversion pipeline configs
│   ├── f2-adversary-ad-lab/
│   │   ├── attack-tests/        ← one writeup per validated technique
│   │   └── docs/                ← campaign-log.md
│   ├── f3-cloud-identity/       ← README only (not started)
│   └── f4-soar/                 ← README only (not started)
├── shared/
│   ├── architecture.md          ← platform-wide network/hosts/data flow
│   ├── vm-inventory.md          ← VM & asset inventory
│   ├── evidence/                ← screenshot evidence (all flagships)
│   ├── evidence-index.md        ← screenshot index
│   └── lessons-learned.md       ← distilled lessons across flagships
├── threat-intel/                ← README only (not started)
└── .github/workflows/           ← CI: validate + SPL autogen; CD: Wazuh deploy
```

## Pipeline (Flagship 1)

Sigma is the source of truth:

1. Author and edit rules in `flagships/f1-detection-pipeline/detections/sigma/`
   (Sigma spec v2.1, ATT&CK tags `attack.tXXXX.XXX` + hyphenated tactic
   names, e.g. `attack.credential-access` — `sigma check` must stay clean).
2. CI (`Validate Sigma rules`) runs `sigma check` on every push touching
   `flagships/f1-detection-pipeline/detections/sigma/**`.
3. CI auto-generates Splunk SPL via `sigma convert -t splunk
   --without-pipeline` into `.../detections/splunk/` (`{stem}.spl` per
   rule), committed back automatically (`[skip ci]`, push only — never on
   PRs).
4. CD (`Deploy rules to Wazuh`, self-hosted runner) PUTs
   `.../detections/wazuh/custom_rules.xml` to the manager API on every
   push touching it, restarts the manager (PUT alone does not hot-reload
   rules in single-node docker — see `shared/lessons-learned.md`), then
   verifies (HTTP 200 + rule present) — proven end-to-end by DET-012, see
   `flagships/f1-detection-pipeline/docs/pipeline-demo/`.
5. `ssh_success_after_failures.yml` is excluded from conversion: its
   `temporal_ordered` correlation is not supported by the Splunk backend.

## How to add a detection

1. Write the rule in `flagships/f1-detection-pipeline/detections/sigma/<name>.yml`
   (Sigma spec v2.1; run `sigma check` locally before pushing).
2. Push — CI validates and auto-generates the Splunk SPL.
3. Run `flagships/f1-detection-pipeline/converters/sigma_to_wazuh.py` to
   regenerate `custom_rules.xml`, hand-review the diff against the live
   Wazuh ruleset (see `converters/README.md` for mapping decisions and
   caveats), then push — CD deploys and restarts the manager.
4. Add a per-detection doc in `flagships/f1-detection-pipeline/docs/detections/det-NNN-<slug>.md`
   (template: header table, Logic, Expected telemetry, Validation method,
   FP notes, Investigation guidance) and a row in
   `flagships/f1-detection-pipeline/docs/attack-matrix.md`.
5. Validate it: build an attack-test writeup in
   `flagships/f2-adversary-ad-lab/attack-tests/`, run the attack, capture
   evidence into `shared/evidence/`, index it in `shared/evidence-index.md`,
   and flip the det-doc + matrix status to VALIDATED.

## Metrics

- **Detections:** 12 (DET-001…DET-012) across 10 ATT&CK techniques — see
  `flagships/f1-detection-pipeline/docs/attack-matrix.md`
- **Validated:** 9/12 (DET-001, DET-002, DET-003, DET-004, DET-005,
  DET-006, DET-007, DET-011, DET-012)
- **Untested:** DET-008/DET-009 — blocked by an auditd ingestion gap (see
  `flagships/f1-detection-pipeline/docs/known-limitations.md`); DET-010 —
  manual deployment only (temporal correlation, no Wazuh equivalent)
- **Sigma rules:** 12 files (spec v2.1, `sigma check` clean)
- **Wazuh rules:** 12 custom rules, IDs 100001–100012
- **Splunk SPL:** 11 auto-generated searches (DET-010 excluded)
- **Evidence:** indexed in `shared/evidence-index.md`, files in `shared/evidence/`

## Author

Ctum0 (sithumsryt@gmail.com)
