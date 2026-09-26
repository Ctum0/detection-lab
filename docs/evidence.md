# Evidence index

Screenshot evidence for detection validations. Sithum: drop the actual image
files into `evidence/` using the exact filenames below (no placeholders are
committed — this file is the index only).

Source screenshots live in `~/Pictures/Screenshots/`; the Source column maps
each one to its `evidence/` target name. Views: **attack** = attacker/victim
terminal doing the thing; **SIEM** = Wazuh alert/event view proving detection.

## Lab + SIEM overview (supporting)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/lab-endpoints-agents.png` | `Screenshot_20260926_195231.png` | Endpoints view: 2 active agents — `linux-victim` (Ubuntu 22.04.5, 100.109.150.66) and `windows-victim` (Win10 Pro 10.0.19045.2965, 100.116.117.32), v4.14.8, node01 | Lab view — proves the victim inventory behind every DET |
| `evidence/siem-threat-hunting-overview.png` | `Screenshot_20260926_195335.png` | Threat Hunting dashboard: 2,891 total alerts, 39 level-12+, 4 auth failures, 3 auth successes; alert-evolution spikes; MITRE donut incl. Password Guessing, PowerShell, Disable or Modify Too…, File Deletion, Account Discovery | SIEM view — lab is generating real technique telemetry |

## DET-001 — SSH Brute Force (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det001-hydra-attack.png` | `Screenshot_20260926_195548.png` | Parrot terminal running `hydra -l nonexistentuser -P rockyou.txt -t 4 192.168.1.112 ssh` (14M+ login tries queued) over a Wazuh events table streaming rule 5710 `sshd: Attempt to login using a non-existent user` | Attack view (+ SIEM context in background) |
| `evidence/det001-rule5712-bruteforce.png` | `Screenshot_20260926_200001.png` | Discover view: hydra at 76 tries/min over MPs; event table shows the 5710 stream **and** rule 5712 `sshd: brute force trying to get access to the system. Non existent user.` firing at 19:59:46 | SIEM view — the brute-force correlation firing |

## DET-004 — LSASS Credential Dumping (stock rule observed)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det004-lsass-92900-svchost.png` | `Screenshot_20260927_015031.png` | Threat Hunting filtered on `data.win.eventdata.targetImage:*lsass*` (2 hits): rule 92900 `Lsass process was accessed by C:\Windows\system32\svchost.exe with read permissions, possible credential dump`, level 12 | SIEM view — doubles as the FP case in `det-004`: stock 92900 fires on benign svchost read access, which is why custom rule 100003 keys on sensitive masks |
| Still to capture: `evidence/det004-procdump-attack.png` | — | Benign LSASS-handle probe (procdump/Atomic test) running on windows-victim | Attack view — missing |
| Still to capture: `evidence/det004-rule100003-alert.png` | — | Custom rule 100003 firing with GrantedAccess mask visible | SIEM view — missing |

## DET-005 — New Local User Creation (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det005-net-user-backdoor.png` | `Screenshot_20260927_005426.png` | Admin PowerShell: `net user backdoor P@ssw0rd123 /add` → `The command completed successfully.` | Attack view |
| `evidence/det005-user-created-alerts.png` | `Screenshot_20260927_005437.png` | windows-victim events: 60109 `User account enabled or created`, 60110 `User account changed`, plus 92039 net.exe / 92033 PowerShell discovery hits | SIEM view |
| `evidence/det005-eid4722-drilldown.png` | `Screenshot_20260927_005642.png` | Document Details drill-down: EID 4722 `A user account was enabled`, Target `backdoor`, Subject `ctum` on `DESKTOP-2R9UM3Q` | SIEM view — field-level proof for the writeup |

## DET-006 — Scheduled Task Creation

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det006-scheduled-task-60228.png` | `Screenshot_20260927_015819.png` | windows-victim events: `A scheduled task was created` (rule 60228) at 01:57 | SIEM view |
| Still to capture: `evidence/det006-schtasks-attack.png` | — | `schtasks /create /tn LabTest …` run on windows-victim | Attack view — missing |

## Supporting (no DET yet)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/supporting-executable-drop-92217.png` | `Screenshot_20260927_010020.png` | Burst of rule 92217 `Executable dropped in Windows root folder` on windows-victim (00:58:35) | Attack-story context (payload drops); candidate for a future file-creation DET or DET-007 dropper analysis |

## Still to capture (no screenshots yet)

| Suggested file | DET | What to grab (attack + SIEM) |
|---|---|---|
| `evidence/det002-encoded-attack.png` / `evidence/det002-sysmon-eid1.png` | DET-002 | `powershell -enc …` run + Sysmon EID 1 with encoded CommandLine |
| `evidence/det003-cradle-attack.png` / `evidence/det003-cradle-eid1.png` | DET-003 | DownloadString+IEX cradle run + EID 1 showing full cradle |
| `evidence/det007-sc-create-attack.png` / `evidence/det007-7045-alert.png` | DET-007 | `sc create LabSvc binPath= C:\Temp\…` + EID 7045 with suspicious ImagePath |
| `evidence/det008-chmod-suid-attack.png` / `evidence/det008-auditd-execve.png` | DET-008 | `chmod u+s /tmp/lab_suid` + auditd EXECVE record |
| `evidence/det009-revshell-attack.png` / `evidence/det009-auditd-execve.png` | DET-009 | Loopback `bash -i >& /dev/tcp/…` + EXECVE telemetry (loopback only) |
| `evidence/det010-hydra-then-login.png` / `evidence/det010-correlation.png` | DET-010 | Hydra burst + one good login from same IP, then the temporal correlation firing |
| `evidence/det011-wevtutil-attack.png` / `evidence/det011-1102-alert.png` | DET-011 | `wevtutil cl Security` + EID 1102 naming the operator |

## Conventions

- `detNNN-<what>.png` — detection evidence, numbered by Detection ID.
- `lab-*` / `siem-*` — environment and overview shots.
- `supporting-*` — useful context not tied to a DET (yet).
- When a DET flips to TESTED/VALIDATED, move its rows' Status in the
  corresponding `docs/detections/det-*.md` and keep filenames stable so
  writeups don't rot.
