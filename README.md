# detection-lab

Detection engineering lab — Sigma / Sysmon / Splunk / ELK experiments, detections and notes.

## Getting started

```bash
git clone git@github.com:Ctum0/detection-lab.git
cd detection-lab
```

## Layout

```text
.
├── README.md
├── detections/
│   ├── sigma/
│   ├── wazuh/
│   └── splunk/
├── docs/
│   ├── architecture.md
│   ├── vm-inventory.md
│   └── detections/          ← one doc per detection
├── attack-tests/            ← scenario scripts per test
└── .github/workflows/       ← CI: validate + SPL autogen
```

- `detections/sigma/` — vendor-agnostic Sigma rules (source of truth)
- `detections/wazuh/` — Wazuh conversions
- `detections/splunk/` — Splunk SPL conversions
- `docs/` — architecture, VM inventory, per-detection docs
- `attack-tests/` — validation scenarios

## Pipeline

Sigma is the source of truth:

1. Author and edit rules in `detections/sigma/` (Sigma spec v2.1, ATT&CK tags `attack.tXXXX.XXX` / `attack.taXXXX`).
2. CI (`Validate Sigma rules`) runs `sigma check` on every push touching `detections/sigma/**`.
3. CI auto-generates Splunk SPL via `sigma convert -t splunk --without-pipeline` into `detections/splunk/` (`{stem}.spl` per rule).
4. Generated `*.spl` files are committed back automatically (`[skip ci]`, push only — never on PRs), so the SPL in the repo is always deployable without hand-editing.
5. `ssh_success_after_failures.yml` is excluded from conversion: its `temporal_ordered` correlation is not supported by the Splunk backend. Its detection doc (`docs/detections/det-010-ssh-success-after-failures.md`) covers manual deployment.

## Metrics

- **Detections:** 11 (DET-001…DET-011) across 9 ATT&CK techniques — see `docs/attack-matrix.md`
- **Validated:** 2/11 (DET-001 SSH brute force, DET-005 local user creation)
- **Sigma rules:** 11 files in `detections/sigma/` (spec v2.1, `sigma check` clean)
- **Wazuh rules:** 11 custom rules, IDs 100001–100011, in `detections/wazuh/custom_rules.xml`
- **Splunk SPL:** 10 auto-generated searches in `detections/splunk/` (DET-010 excluded — temporal correlation)
- **Evidence:** indexed in `docs/evidence.md`, files land in `evidence/`
- **Open limitation:** auditd ingestion gap blocking DET-008/009 — see `docs/known-limitations.md`

## Author

Ctum0 (sithumsryt@gmail.com)
