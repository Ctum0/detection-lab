# Adversary Emulation & AD Lab

**Status: Workstream 1 (Windows campaign) COMPLETE.** The AD phase (W3) is
shelved until after the showcase; W2 and W4 are not started.

Adversary emulation against the lab's Windows and Linux victims — Atomic
Red Team-driven attack execution that validates the detections built in
the [Detection Pipeline](../detection-pipeline/README.md) module. Every
"VALIDATED" status in its attack matrix traces back to a test run and
writeup here.

## Workstreams

| Workstream | Scope | Status |
|---|---|---|
| W1 — Windows campaign | ART-driven validation of the Windows-side custom Wazuh rules (T1136.001, T1003.001, T1059.001, T1053.005, T1685.005, T1543.003) plus the existing SSH brute-force test | **COMPLETE** — see `docs/campaign-log.md` |
| W2 — Linux campaign | auditd-backed techniques (T1548.001, T1059.004) — currently blocked by the auditd ingestion gap, `../detection-pipeline/docs/known-limitations.md` L-001 | NOT STARTED |
| W3 — Active Directory | Domain-joined attack paths once `dc-01`/`win-member-01` are stood up (see `shared/vm-inventory.md`) | SHELVED — post-showcase |
| W4 — Multi-stage campaigns | Chained TTPs across the kill chain, not single-technique tests | NOT STARTED |

## Layout

```text
adversary-emulation/
└── docs/
    └── campaign-log.md  ← chronological W1 log: install saga, per-test
                             outcomes, bugs found
```

Technique writeups themselves live in the top-level `attack-tests/`, not
here — they're a shared/cross-module resource (every write-up maps back
to a Detection Pipeline detection), same as `detections/` and
`platform/`. This module's folder keeps only what's genuinely specific to
it: the campaign narrative.

## W1 close-out summary

- Installed Atomic Red Team on `windows-victim` (see the install saga in
  `docs/campaign-log.md`).
- Validated 7 techniques end-to-end (attack → telemetry → custom Wazuh
  rule fired): T1110.001, T1136.001, T1003.001, T1059.001, T1053.005,
  T1685.005, T1543.003 — writeups in the top-level `attack-tests/`.
- Found and fixed 2 rule-parenting bugs and 1 CD hot-reload bug along the
  way; distilled into `shared/lessons-learned.md`.
- T1548.001 and T1059.004 (Linux/auditd) remain untested — blocked
  upstream by the Detection Pipeline module's auditd ingestion gap, not
  an ART/campaign issue.

## Related

- [Detection Pipeline](../detection-pipeline/README.md) — the detections
  these attacks validate; see `docs/attack-matrix.md` there for live
  status
- `shared/evidence-index.md` — screenshot evidence for every test here
