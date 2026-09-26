# detection-lab

Detection engineering lab — Sigma rules as code, Wazuh + Sysmon detections, Splunk SPL autogen, and validation writeups.

## Getting started

```bash
git clone git@github.com:Ctum0/detection-lab.git
cd detection-lab
```

## Layout

```text
.
├── README.md
├── converters/              ← sigma_to_wazuh.py + notes
├── detections/
│   ├── sigma/               ← source of truth (12 rules)
│   ├── wazuh/               ← custom_rules.xml (IDs 100001–100012)
│   └── splunk/              ← CI-generated *.spl (11 files)
├── docs/
│   ├── architecture.md
│   ├── vm-inventory.md
│   ├── attack-matrix.md     ← all 12 detections at a glance
│   ├── evidence.md          ← screenshot index (files in evidence/)
│   ├── known-limitations.md ← open gaps (e.g. auditd ingestion)
│   ├── pipeline-demo/       ← DET-012 end-to-end CI/CD screenshots
│   └── detections/          ← one doc per detection (det-001…det-012)
├── evidence/                ← screenshot drops (indexed by docs/evidence.md)
├── attack-tests/            ← scenario scripts per test
├── pipelines/               ← helper configs
└── .github/workflows/       ← CI: validate + SPL autogen
```

- `detections/sigma/` — vendor-agnostic Sigma rules (source of truth)
- `detections/wazuh/` — hand-tuned `custom_rules.xml` generated from Sigma, then reviewed against the live manager
- `detections/splunk/` — CI-generated SPL conversions (never hand-edit; DET-010 excluded)
- `converters/` — Sigma→Wazuh converter script and mapping notes
- `docs/` — architecture, VM inventory, attack matrix, evidence index, limitations, per-detection docs
- `attack-tests/` — validation scenarios

## Pipeline

Sigma is the source of truth:

1. Author and edit rules in `detections/sigma/` (Sigma spec v2.1, ATT&CK tags `attack.tXXXX.XXX` + hyphenated tactic names, e.g. `attack.credential-access` — `sigma check` must stay clean).
2. CI (`Validate Sigma rules`) runs `sigma check` on every push touching `detections/sigma/**`.
3. CI auto-generates Splunk SPL via `sigma convert -t splunk --without-pipeline` into `detections/splunk/` (`{stem}.spl` per rule).
4. Generated `*.spl` files are committed back automatically (`[skip ci]`, push only — never on PRs), so the SPL in the repo is always deployable without hand-editing.
5. CD (`Deploy rules to Wazuh`, self-hosted runner) PUTs `detections/wazuh/custom_rules.xml` to the manager API on every push touching it, then verifies (HTTP 200 + rule present) — proven end-to-end by DET-012, see `docs/pipeline-demo/`.
6. `ssh_success_after_failures.yml` is excluded from conversion: its `temporal_ordered` correlation is not supported by the Splunk backend. Its detection doc (`docs/detections/det-010-ssh-success-after-failures.md`) covers manual deployment.

## Metrics

- **Detections:** 12 (DET-001…DET-012) across 10 ATT&CK techniques — see `docs/attack-matrix.md`
- **Validated:** 5/12 (DET-001 SSH brute force, DET-002 encoded PowerShell, DET-005 local user, DET-006 scheduled task, DET-012 notepad canary)
- **Sigma rules:** 12 files in `detections/sigma/` (spec v2.1, `sigma check` clean)
- **Wazuh rules:** 12 custom rules, IDs 100001–100012, in `detections/wazuh/custom_rules.xml`
- **Splunk SPL:** 11 auto-generated searches in `detections/splunk/` (DET-010 excluded — temporal correlation)
- **Evidence:** indexed in `docs/evidence.md`, files land in `evidence/`
- **Open limitation:** auditd ingestion gap blocking DET-008/009 — see `docs/known-limitations.md`

## Author

Ctum0 (sithumsryt@gmail.com)
