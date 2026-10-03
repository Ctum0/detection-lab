# Tuning: false-positive suppressions

Two level-0 rules suppress known benign activity. Both are deployed from
`detections/wazuh/custom_rules.xml` and are documented here as detection
improvements, with the reasoning and the observed effect.

Level 0 means the rule matches and records the event but raises no alert.
Each suppression is a child of a stock rule, so the stock rule's logic is
unchanged. The suppression only applies when its own conditions also match.

## Rule 100020: PowerShell script-policy test files

**Stock rule:** 92213, which fires when an executable or script is dropped
into a Temp location.

**What it matches:** `win.eventdata.targetFilename` matching
`__PSScriptPolicyTest_[a-z0-9]+\.[a-z0-9]+\.ps1`.

**Why it is benign:** PowerShell writes these files into the user's Temp
directory at the start of every session, as part of its execution-policy
check. They are not payloads.

**Observed effect**

Sithum observed hundreds of 92213 fires during the ART campaign (327 or more
cited in AI triage analysis). After rule 100020 was deployed, those fires
fell to zero. Verified live: PowerShell launches produce no alert, while rule
100005 still fires normally on encoded PowerShell.

*Observation-based, not queried from the indexer.*

## Post-deploy verification

Both rules were checked on the manager after the deploy:

- `GET /rules` reports rule 100013 (level 0, enabled, `local_rules.xml`) and
  rule 100020 (level 0, enabled, `local_rules.xml`).
- The API lists each `local_rules.xml` rule twice, giving 28 entries for 14
  unique rules. This is a listing quirk of the 4.14.8 API for custom rule
  files. The container has one rules file, `wazuh-analysisd -t` reports no
  duplicate-rule warnings, and built-in rules show a single count. Behavioural
  evidence also confirms each rule loads once (see below).
- Encoded-PowerShell test: exactly one alert for rule 100005 (`firedtimes: 1`),
  and zero 92213 alerts.

The repository, the live manager and this document now describe the same rule
set.

## Rule 100013: LSASS access from the LSM path

**Stock rule:** 92900, which fires when a process reads LSASS memory. It
fires on benign reads by `svchost.exe`, which is why custom rule 100003 uses
a mask allow-list in the first place.

**What it matches:** the stock 92900 event where `GrantedAccess` is exactly
`0x101001` (no `PROCESS_VM_READ`) and the call trace includes `lsm.dll`.

**Why it is benign:** the Local Security Authority Manager loads `lsm.dll`
and touches LSASS with that mask during normal operation. The mask does not
include the read permission a credential dump needs.

**Observed effect:** no before and after figures are recorded yet for this
rule. The rule's effect should be measured the same way as rule 100020 before
the figures are quoted.

## What these rules do not cover

- Rule 100020 matches the filename pattern only. A malicious file given a
  name that fits the pattern would be suppressed too. The pattern is narrow
  and anchored to the PowerShell naming scheme, but it is not an identity
  check.
- Rule 100013 depends on the call trace containing `lsm.dll`. The call trace
  is recorded by Sysmon from the process that opened LSASS, so it is hard to
  fake from user mode. It is still worth watching for any dump that carries a
  matching trace, since 100013 would hide it.

## Deployment

Both rules live in `detections/wazuh/custom_rules.xml`. The CD pipeline PUTs
that file to the manager on each push that touches `detections/wazuh/**`. The
repo copy and the manager copy are kept identical, so a deploy does not
remove the suppressions.
