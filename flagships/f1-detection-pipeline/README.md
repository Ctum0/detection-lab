# Flagship 1 — Detection Pipeline

**Status: COMPLETE** — ART campaign complete for Windows; 9/12 detections
validated end-to-end (attack → alert), Sigma → CI → CD → Wazuh pipeline
proven live.

Detection engineering as code: Sigma rules are the source of truth,
CI validates and auto-converts them to Splunk SPL, CD deploys the Wazuh
translation to the live manager on every push, and each detection is
proven by actually running the attack it's built to catch.

## Layout

```text
f1-detection-pipeline/
└── docs/
    ├── attack-matrix.md      ← all 12 detections at a glance
    ├── detections/           ← one doc per detection (det-001...det-012)
    ├── pipeline-demo/        ← DET-012 end-to-end CI/CD walkthrough
    └── known-limitations.md  ← open gaps (auditd ingestion, L-001)
```

This flagship authors into three platform-level shared resources rather
than owning them outright — `detections/` (Sigma source of truth, Wazuh
rules, generated SPL), `platform/converters/` (the Sigma→Wazuh converter),
and `platform/pipelines/` (Sigma conversion pipeline configs) all live at
the repo root, since Flagship 2's attack tests validate against the same
`detections/` tree. Only the write-ups genuinely specific to this
flagship — the matrix, per-detection docs, the pipeline demo, known
limitations — live under its own `docs/`.

## Pipeline

1. Author/edit rules in `detections/sigma/` (Sigma spec v2.1, ATT&CK tags
   `attack.tXXXX.XXX` + hyphenated tactic names — `sigma check` must stay
   clean).
2. CI (`Validate Sigma rules`) runs `sigma check` on every push touching
   `detections/sigma/**`.
3. CI auto-generates Splunk SPL via `sigma convert -t splunk
   --without-pipeline` into `detections/splunk/`, committed back
   automatically (`[skip ci]`, push only).
4. CD (`Deploy rules to Wazuh`, self-hosted runner) PUTs
   `detections/wazuh/custom_rules.xml` to the manager API on every push
   touching it, restarts the manager (single-node docker does not
   hot-reload rule files on PUT — see `docs/known-limitations.md` /
   `shared/lessons-learned.md`), then verifies (HTTP 200 + rule present).
   Proven end-to-end by DET-012, see `docs/pipeline-demo/`.
5. `ssh_success_after_failures.yml` is excluded from SPL conversion: its
   `temporal_ordered` correlation is unsupported by the Splunk backend.

## Metrics

- **Detections:** 12 (DET-001...DET-012) across 10 ATT&CK techniques —
  see `docs/attack-matrix.md`
- **Validated:** 9/12 — DET-001, DET-002, DET-003, DET-004, DET-005,
  DET-006, DET-007, DET-011, DET-012
- **Untested:** DET-008, DET-009 (blocked by the auditd ingestion gap,
  `docs/known-limitations.md` L-001), DET-010 (manual-deployment only,
  temporal correlation has no Wazuh equivalent)
- **Sigma rules:** 12 files, spec v2.1, `sigma check` clean
- **Wazuh rules:** 12 custom rules, IDs 100001-100012
- **Splunk SPL:** 11 auto-generated searches (DET-010 excluded)
- **Evidence:** indexed in `shared/evidence-index.md`, files in
  `shared/evidence/`

## Related

- [Flagship 2](../f2-adversary-ad-lab/README.md) — the attack side that
  validates these detections (ART campaigns, per-technique writeups)
- `shared/architecture.md` — platform-wide data flow
- `shared/lessons-learned.md` — distilled lessons from building this
