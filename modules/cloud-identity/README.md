# Flagship 3 — Cloud & Identity Security

**Status: NOT STARTED**

## Planned scope

Extend the detection-as-code approach from [Flagship 1](../detection-pipeline/README.md)
into cloud and identity control planes — the surfaces the on-prem/lab-VM
model of F1/F2 doesn't cover.

Candidate targets (TBD, pending scoping):

- **Microsoft Entra ID** — sign-in log anomalies, conditional access
  bypass attempts, app consent abuse, privileged role assignment.
- **AWS** — CloudTrail-driven detections (IAM privilege escalation paths,
  unusual API call chains, exposed credentials), GuardDuty triage.
- Sigma rules targeting cloud logsources (`product: azure`, `product: aws`)
  feeding the same CI validation pattern used in F1.

## Dependencies

- Builds on the Sigma-as-source-of-truth pipeline and CI patterns proven in
  [Flagship 1](../detection-pipeline/README.md).
- No infrastructure stood up yet — see `shared/vm-inventory.md` for planned
  assets.
