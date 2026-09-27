# Flagship 2 — Adversary Emulation & AD Lab

**Status: Workstream 1 (Windows campaign) COMPLETE**; Workstreams 2-4 not
started.

Adversary emulation against the lab's Windows and Linux victims — Atomic
Red Team-driven attack execution that validates the detections built in
[Flagship 1](../f1-detection-pipeline/README.md). Every "VALIDATED" status
in the F1 attack matrix traces back to a test run and writeup here.

## Workstreams

| Workstream | Scope | Status |
|---|---|---|
| W1 — Windows campaign | ART-driven validation of the Windows-side custom Wazuh rules (T1136.001, T1003.001, T1059.001, T1053.005, T1685.005, T1543.003) plus the existing SSH brute-force test | **COMPLETE** — see `docs/campaign-log.md` |
| W2 — Linux campaign | auditd-backed techniques (T1548.001, T1059.004) — currently blocked by the auditd ingestion gap, `../f1-detection-pipeline/docs/known-limitations.md` L-001 | NOT STARTED |
| W3 — Active Directory | Domain-joined attack paths once `dc-01`/`win-member-01` are stood up (see `shared/vm-inventory.md`) | NOT STARTED |
| W4 — Multi-stage campaigns | Chained TTPs across the kill chain, not single-technique tests | NOT STARTED |

## Layout

```text
f2-adversary-ad-lab/
└── docs/
    └── campaign-log.md  ← chronological W1 log: install saga, per-test
                             outcomes, bugs found
```

Technique writeups themselves live in the top-level `attack-tests/`, not
here — they're a shared/cross-flagship resource (every write-up maps back
to an F1 detection), same as `detections/` and `platform/`. This flagship
folder keeps only what's genuinely F2-specific: the campaign narrative.

## W1 close-out summary

- Installed Atomic Red Team on `windows-victim` (see the install saga in
  `docs/campaign-log.md`).
- Validated 7 techniques end-to-end (attack → telemetry → custom Wazuh
  rule fired): T1110.001, T1136.001, T1003.001, T1059.001, T1053.005,
  T1685.005, T1543.003 — writeups in the top-level `attack-tests/`.
- Found and fixed 2 rule-parenting bugs and 1 CD hot-reload bug along the
  way; distilled into `shared/lessons-learned.md`.
- T1548.001 and T1059.004 (Linux/auditd) remain untested — blocked
  upstream by the F1 auditd ingestion gap, not an ART/campaign issue.

## Related

- [Flagship 1](../f1-detection-pipeline/README.md) — the detections these
  attacks validate; see `docs/attack-matrix.md` there for live status
- `shared/evidence-index.md` — screenshot evidence for every test here
