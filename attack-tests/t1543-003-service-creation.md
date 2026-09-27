# T1543.003 — Windows Service Creation

Validates: [DET-007](../modules/detection-pipeline/docs/detections/det-007-suspicious-service-creation.md)
(Wazuh custom 100011)

## Commands

Ran the Atomic Red Team T1543.003 test on `windows-victim` (see the ART
install saga in `../modules/adversary-emulation/docs/campaign-log.md`), twice:

**Attempt 1 — ART default args (did not fire, by design):**

```powershell
Invoke-AtomicTest T1543.003 -TestNumbers <n>
```

Used ART's default `binary_path` input, which points somewhere under
`C:\AtomicRedTeam\...`.

**Attempt 2 — overridden binary_path (fired):**

```powershell
Invoke-AtomicTest T1543.003 -TestNumbers <n> -PromptForInputArgs
```

Prompted for input args and set `binary_path=C:\Temp\AtomicService.exe`
to match a realistic attacker staging path.

## Expected telemetry

System EID 7045 (`A service was installed in the system`) with
`ServiceName`, `ImagePath`, `ServiceType`, `StartType`, `AccountName`.

## Rules fired

- Attempt 1: EID 7045 shipped, but **rule 100011 correctly did not fire**
  — `C:\AtomicRedTeam\...` doesn't match the rule's suspicious-path
  pattern list (`\Temp\`, `\Users\Public\`, `\ProgramData\`, `\AppData\`,
  interpreter/script names). This is expected behavior, not a gap: the
  pattern list models where a real attacker stages a payload, not where
  a test framework happens to install itself.
- Attempt 2: EID 7045 shipped with `ImagePath` = `C:\Temp\AtomicService.exe`;
  custom Wazuh rule 100011 `CTUM: Service created with suspicious binary
  path [T1543.003]` fired at level 10.

## Cleanup

```powershell
sc delete LabSvc
```

(or whichever `ServiceName` the ART test registered — check
`Get-Service` / the 7045 event for the exact name before deleting).

## Evidence

- `shared/evidence/det007-rule100011-alert.png` — SIEM view, rule 100011
  firing on `windows-victim` at 2026-09-27 18:43:53.
- Attack-view screenshot (the `sc create`/ART invocation itself) still to
  capture — see `shared/evidence-index.md`.

## Notes

Good example of a detection working correctly by *not* firing on
benign/off-target activity — worth keeping both runs in the writeup
rather than only the one that fired, since the negative result is part of
what validates the rule logic.
