# DET-001 — SSH Brute Force (Password Guessing)

| Field | Value |
|---|---|
| Detection ID | DET-001 |
| Rule | Wazuh 5712 (stock); custom 100008 (base) + 100009 (freq 6/10m); Sigma `detections/sigma/ssh_bruteforce.yml`; SPL `detections/splunk/ssh_bruteforce.spl` |
| Data source | /var/log/auth.log (sshd) via Wazuh agent |
| ATT&CK | T1110.001 — Password Guessing |
| Severity | Level 10 |
| Status | VALIDATED 2026-09-26 |

## Logic
Multiple failed SSH authentication attempts from a single source
within the Wazuh brute-force window triggers rule 5712
("sshd: brute force trying to get access to the system").

## Expected telemetry
sshd "Failed password for ..." lines; aggregates by source IP.

## Validation
- Attack: hydra -l nonexistentuser -P rockyou.txt -t 4 <victim-ip> ssh
- Source: Parrot attacker machine
- Result: rule 5712 fired in Wazuh Threat Hunting
- Evidence: screenshots 04-hydra.png, 05-rule5712.png

## False-positive considerations
- Legitimate user mistyping password repeatedly
- Monitoring tools with stale credentials
- Consider source-IP allowlists for admin jump hosts

## Investigation guidance
1. Check source IP reputation (AbuseIPDB)
2. Did any attempt succeed afterward? (rule 5710/5715, successful login events)
3. Check targeted account validity
4. Timeline: first→last attempt, attempt rate

## Improvement notes
- Tune threshold vs FP rate
- Add Splunk SPL version
- Add Sigma rule for portability
