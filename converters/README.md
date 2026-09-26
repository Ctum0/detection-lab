# converters/

## sigma_to_wazuh.py

Converts `detections/sigma/*.yml` into a single Wazuh custom rules file
(`detections/wazuh/custom_rules.xml`). Approach adapted from
[Alanv0303/Rule-converter](https://github.com/Alanv0303/Rule-converter) (MIT):
parse multi-doc Sigma YAML, translate selections to Wazuh checks, emit XML.
Extended for this repo with logsource-aware anchoring, Sigma→Wazuh field
mapping, MITRE blocks, `event_count` correlations, and an ID-mapping header.

```bash
python3 converters/sigma_to_wazuh.py
python3 converters/sigma_to_wazuh.py --sigma-dir detections/sigma \
    --output detections/wazuh/custom_rules.xml --start-id 100001
python3 converters/sigma_to_wazuh.py --anchor if_group   # Wazuh 4.9.0+
```

Requires: python3 + pyyaml. Regenerate (don't hand-edit) `custom_rules.xml`
after any Sigma change.

## Mapping decisions

| Sigma logsource | Wazuh anchor | Group |
|---|---|---|
| Sysmon `process_creation` (EID 1) | `<if_sid>61603</if_sid>` (stock 0595 parent) | `sysmon_event1` |
| Sysmon `process_access` (EID 10) | `<if_sid>61612</if_sid>` (stock 0595 parent) | `sysmon_event_10` |
| Windows `service: security/system` | `win.system.channel` + `win.system.eventID` fields | `windows_security` / `windows_system` |
| Linux `service: auth` (sshd) | `<match type="pcre2">` on full_log | `syslog,sshd` |
| Linux `service: auditd` | `<match type="pcre2">` on EXECVE full_log | `auditd` |

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
2. Copy to the manager: `/var/ossec/etc/rules/custom_rules.xml`, restart.
3. Tune: the auditd/sshd `full_log` matches are intentionally broad —
   narrow to decoded fields once baselined.

## Caveats

- Wazuh 4.9.0 broke `if_sid` chaining off level-0 sysmon parents at
  runtime (wazuh/wazuh#36029); use `--anchor if_group` there.
- Stock parent SIDs (61603/61612) follow wazuh-ruleset 4.x; verify against
  your manager's `0595-win-sysmon_rules.xml`.
