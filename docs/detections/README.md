# Detections docs

One markdown file per detection. Copy this template:

```markdown
# <Detection name>

- Rule: `detections/sigma/<file>.yml`
- Platforms: Wazuh / Splunk
- MITRE ATT&CK: Txxxx.xxx
- Log source: Sysmon Event ID xx
- Test: `attack-tests/<scenario>.md` or `.sh`

## Logic

What it detects and why.

## False positives

Known benign triggers and tuning.

## Validation

Steps + expected alert output.
```
