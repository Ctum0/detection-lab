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
└── .github/workflows/       ← CI later
```

- `detections/sigma/` — vendor-agnostic Sigma rules (source of truth)
- `detections/wazuh/` — Wazuh conversions
- `detections/splunk/` — Splunk SPL conversions
- `docs/` — architecture, VM inventory, per-detection docs
- `attack-tests/` — validation scenarios

## Author

Ctum0 (sithumsryt@gmail.com)
