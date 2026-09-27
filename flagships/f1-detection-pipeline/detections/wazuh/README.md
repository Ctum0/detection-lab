# Wazuh detections

Single hand-tuned file: `custom_rules.xml` (IDs 100001–100012), generated
from `../sigma/` by `../../converters/sigma_to_wazuh.py` then reviewed against
the live Wazuh 4.14.8 ruleset (see its header comment). Re-running the
converter overwrites the hand-tuning — back up first. Rule↔Sigma mapping
lives in `../../docs/attack-matrix.md`.
