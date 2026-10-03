# Cloud & Identity Security

**Status: SHELVED — until after the showcase.** Not started; scope below is planned, not in progress.

## Planned scope

Extend the detection-as-code approach from the
[Detection Pipeline](../detection-pipeline/README.md) module into cloud
and identity control planes — the surfaces the on-prem/lab-VM model of
Detection Pipeline / Adversary Emulation doesn't cover.

Candidate targets (TBD, pending scoping):

- **Microsoft Entra ID** — sign-in log anomalies, conditional access
  bypass attempts, app consent abuse, privileged role assignment.
- **AWS** — CloudTrail-driven detections (IAM privilege escalation paths,
  unusual API call chains, exposed credentials), GuardDuty triage.
- Sigma rules targeting cloud logsources (`product: azure`, `product: aws`)
  feeding the same CI validation pattern used in Detection Pipeline.

## Dependencies

- Builds on the Sigma-as-source-of-truth pipeline and CI patterns proven
  in the [Detection Pipeline](../detection-pipeline/README.md) module.
- No infrastructure stood up yet — see `shared/vm-inventory.md` for planned
  assets.
