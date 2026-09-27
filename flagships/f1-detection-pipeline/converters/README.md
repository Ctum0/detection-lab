# converters/

## sigma_to_wazuh.py

Converts `detections/sigma/*.yml` into a single Wazuh custom rules file
(`detections/wazuh/custom_rules.xml`). Approach adapted from
[Alanv0303/Rule-converter](https://github.com/Alanv0303/Rule-converter) (MIT):
parse multi-doc Sigma YAML, translate selections to Wazuh checks, emit XML.
Extended for this repo with logsource-aware anchoring, Sigma→Wazuh field
mapping, MITRE blocks, `event_count` correlations, and an ID-mapping header.

Run from the repo root:

```bash
python3 flagships/f1-detection-pipeline/converters/sigma_to_wazuh.py
python3 flagships/f1-detection-pipeline/converters/sigma_to_wazuh.py \
    --sigma-dir flagships/f1-detection-pipeline/detections/sigma \
    --output flagships/f1-detection-pipeline/detections/wazuh/custom_rules.xml \
    --start-id 100001
python3 flagships/f1-detection-pipeline/converters/sigma_to_wazuh.py --anchor if_group   # Wazuh 4.9.0+
```

Requires: python3 + pyyaml. Note: `detections/wazuh/custom_rules.xml` has
been hand-tuned after generation (verified against the live Wazuh 4.14.8
ruleset — see its header comment), so re-running the converter overwrites
those fixes. Back up first, regenerate, then re-apply review deltas.

## Mapping decisions

Generator defaults (flags: `--anchor if_sid`, `--start-id 100001`):

| Sigma logsource | Generated anchor | Group |
|---|---|---|
| Sysmon `process_creation` (EID 1) | `<if_sid>61603</if_sid>` (stock 0595 parent) | `sysmon_event1` |
| Sysmon `process_access` (EID 10) | `<if_sid>61612</if_sid>` (stock 0595 parent) | `sysmon_event_10` |
| Windows `service: security/system` | `win.system.channel` + `win.system.eventID` fields | `windows_security` / `windows_system` |
| Linux `service: auth` (sshd) | `<match type="pcre2">` on full_log | `syslog,sshd` |
| Linux `service: auditd` | `<match type="pcre2">` on EXECVE full_log | `auditd` |

Hand-review deltas currently in `custom_rules.xml` (keep on regenerate):

- Sysmon rules use `<if_group>sysmon_event1</if_group>` /
  `<if_group>sysmon_event_10</if_group>` instead of `if_sid` (robust on
  4.9.0+, see Caveats).
- Auditd rules use decoded fields `audit.command` / `audit.args` under
  `<if_sid>80700</if_sid>` instead of full_log matches.
- Security rules chain stock parents where stable: 100002 under 60109,
  100006 under 60228; 100007 uses `if_sid` on `63103` (event-log-cleared
  parent, not `windows_security` — see `docs/known-limitations.md` /
  `shared/lessons-learned.md` for why); 100011 uses `windows_system` group.

- Sigma `Image|CommandLine|TargetImage|...` → `win.eventdata.*` equivalents;
  same-field OR-lists fold into one PCRE2 alternation (Wazuh ANDs fields).
- Levels: critical→12, high→10, medium→7, low/informational→5.
- Technique goes in `description` (`[T1059.001]`) plus a `<mitre>` block
  (tactic resolved from the rule's own tags, so MITRE restructures flow
  through — e.g. T1685.005 / Defense Impairment).
- Sigma `event_count` correlation → Wazuh `frequency` + `timeframe`
  (timespan parsed) + `if_matched_sid` + `same_source_ip`.
- `temporal*` correlations have no Wazuh equivalent → SKIPPED
  (`ssh_success_after_failures.yml`, noted in the XML header).

## Deploy

1. `wazuh-logtest` the generated file first.
2. CI (`.github/workflows/deploy-wazuh.yml`) PUTs it to the manager API and
   restarts the manager container — single-node docker does not hot-reload
   rule files on PUT alone (see `shared/lessons-learned.md`).
3. Tune: the sshd `full_log` match is intentionally broad — narrow to
   decoded fields once baselined. (Auditd rules already use decoded
   `audit.command` / `audit.args`, which is why the ingestion gap in
   `docs/known-limitations.md` blocks them.)

## Caveats

- Wazuh 4.9.0 broke `if_sid` chaining off level-0 sysmon parents at
  runtime (wazuh/wazuh#36029); use `--anchor if_group` there.
- Stock parent SIDs (61603/61612) follow wazuh-ruleset 4.x; verify against
  your manager's `0595-win-sysmon_rules.xml`.
