# Evidence index

Screenshot evidence for detection validations, covering all flagships.
Files live in `shared/evidence/` under the exact filenames below.

Source screenshots live in `~/Pictures/Screenshots/` (Flagship 1, mostly)
and `~/Pictures/Screenshots/Flagship2/` (Flagship 2 — ART install, atomic
tests, rule firings); the Source column maps each one to its
`shared/evidence/` target name. Views: **attack** = attacker/victim
terminal doing the thing; **SIEM** = Wazuh alert/event view proving
detection. The end-to-end CI/CD demo (DET-012) lives separately in
`modules/detection-pipeline/docs/pipeline-demo/`.

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

## DET-002 — PowerShell Encoded Command (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det002-rule100005-encoded.png` | `Screenshot_20260927_031425.png` | windows-victim event: custom rule 100005 `CTUM: PowerShell with encoded command [T1059.001]`, level 10, at 03:13 — first custom Sysmon rule firing in the lab | SIEM view |
| Still to capture: `evidence/det002-encoded-attack.png` | — | The `powershell -enc …` run on windows-victim that produced it | Attack view — missing |

## DET-003 — PowerShell Download Cradle (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det003-rule100004-alert.png` | `Flagship2/Screenshot_20260927_185353.png` | Threat Hunting, `rule.id:100004`, 1 hit on windows-victim: `CTUM: PowerShell download cradle - fetch and execute pattern [T1059.001]`, level 10, 2026-09-27 18:51:56 | SIEM view |
| Still to capture: `evidence/det003-cradle-attack.png` | — | The `DownloadString`+`IEX` cradle run on windows-victim that produced it | Attack view — missing |

## DET-004 — LSASS Credential Dumping via comsvcs MiniDump (VALIDATED, evidence gap)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det004-lsass-92900-svchost.png` | `Screenshot_20260927_015031.png` | Threat Hunting filtered on `data.win.eventdata.targetImage:*lsass*` (2 hits): rule 92900 `Lsass process was accessed by C:\Windows\system32\svchost.exe with read permissions, possible credential dump`, level 12 | SIEM view — doubles as the FP case in `det-004`: stock 92900 fires on benign svchost read access, which is why custom rule 100003 keys on sensitive masks |
| Still to capture: `evidence/det004-comsvcs-attack.png` | — | ART T1003.001 `rundll32.exe comsvcs.dll, MiniDump` run on windows-victim | Attack view — missing |
| Still to capture: `evidence/det004-rule100003-alert.png` | — | Custom rule 100003 firing (×2 per campaign log) with GrantedAccess mask visible | SIEM view — **missing**; neither Screenshots folder contains a shot of `rule.id:100003` firing despite the det-doc/campaign-log recording it fired twice — capture on next lab session |

## DET-005 — New Local User Creation (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det005-net-user-backdoor.png` | `Screenshot_20260927_005426.png` | Admin PowerShell: `net user backdoor P@ssw0rd123 /add` → `The command completed successfully.` | Attack view |
| `evidence/det005-user-created-alerts.png` | `Screenshot_20260927_005437.png` | windows-victim events: 60109 `User account enabled or created`, 60110 `User account changed`, plus 92039 net.exe / 92033 PowerShell discovery hits | SIEM view |
| `evidence/det005-eid4722-drilldown.png` | `Screenshot_20260927_005642.png` | Document Details drill-down: EID 4722 `A user account was enabled`, Target `backdoor`, Subject `ctum` on `DESKTOP-2R9UM3Q` | SIEM view — field-level proof for the writeup |

## DET-006 — Scheduled Task Creation (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det006-rule100006-task.png` | `Screenshot_20260927_031700.png` | windows-victim event: custom rule 100006 `CTUM: Scheduled task created (4698) [T1053.005]`, level 7, at 03:16 — plus `Wazuh server started` (rule deploy restart) right above it | SIEM view |
| `evidence/det006-scheduled-task-60228.png` | `Screenshot_20260927_015819.png` | windows-victim events: stock `A scheduled task was created` (rule 60228) at 01:57 — the parent event custom 100006 chains off | SIEM view — supporting |
| Still to capture: `evidence/det006-schtasks-attack.png` | — | `schtasks /create /tn LabTest …` run on windows-victim | Attack view — missing |

## DET-007 — Suspicious Service Creation (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det007-rule100011-alert.png` | `Flagship2/Screenshot_20260927_184422.png` | Threat Hunting, `rule.id:100011`, 1 hit on windows-victim: `CTUM: Service created with suspicious binary path [T1543.003]`, level 10, 2026-09-27 18:43:53 | SIEM view |
| Still to capture: `evidence/det007-sc-create-attack.png` | — | ART T1543.003 run with `binary_path=C:\Temp\AtomicService.exe` (`-PromptForInputArgs`) on windows-victim | Attack view — missing |

## DET-011 — Clearing Windows Security Event Log (VALIDATED)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/det011-wevtutil-attack.png` | `Flagship2/Screenshot_20260927_182528.png` | Admin PowerShell on windows-victim: Atomic Red Team test **T1685.005-4 "BlackCat Ransomware Full Log Clear"** looping `wevtutil.exe cl "<log>"` across dozens of event-log channels (including Security), ending `Failed to clear log Microsoft-Windows-LiveId/Analytic ... Access is denied. Exit code: -1. Done executing test: T1685.005-4 BlackCat Ransomware Full Log Clear` | Attack view — confirms this ran as an ART atomic test, not a bare manual `wevtutil cl Security` |
| `evidence/det011-rule100007-alert.png` | `Flagship2/DET-011.png` | Threat Hunting, `rule.id:100007`, 1 hit on windows-victim: `CTUM: Windows Security event log cleared (1102) [T1685.005]`, level 10, 2026-09-27 18:22:44 — fired after the `if_sid 63103` parent fix | SIEM view |

## Supporting (no DET yet)

| Suggested file | Source | Shows | Story fit |
|---|---|---|---|
| `evidence/supporting-executable-drop-92217.png` | `Screenshot_20260927_010020.png` | Burst of rule 92217 `Executable dropped in Windows root folder` on windows-victim (00:58:35) | Attack-story context (payload drops); candidate for a future file-creation DET or DET-007 dropper analysis |

## Still to capture (no screenshots yet)

| Suggested file | DET | What to grab (attack + SIEM) |
|---|---|---|
| `evidence/det002-encoded-attack.png` | DET-002 | `powershell -enc …` run (SIEM side captured 2026-09-27, rule 100005 firing) |
| `evidence/det003-cradle-attack.png` | DET-003 | DownloadString+IEX cradle run on windows-victim (SIEM side captured 2026-09-27, rule 100004 firing) |
| `evidence/det004-comsvcs-attack.png` / `evidence/det004-rule100003-alert.png` | DET-004 | ART T1003.001 `rundll32 comsvcs.dll, MiniDump` run + custom rule 100003 firing — **neither view captured yet**, priority gap despite VALIDATED status |
| `evidence/det006-schtasks-attack.png` | DET-006 | `schtasks /create /tn LabTest …` run on windows-victim |
| `evidence/det007-sc-create-attack.png` | DET-007 | ART T1543.003 run with overridden `binary_path` on windows-victim (SIEM side captured 2026-09-27, rule 100011 firing) |
| `evidence/det008-chmod-suid-attack.png` / `evidence/det008-auditd-execve.png` | DET-008 | `chmod u+s /tmp/lab_suid` + auditd EXECVE record — blocked on L-001 (auditd dead on linux-victim, needs reboot) |
| `evidence/det009-revshell-attack.png` / `evidence/det009-auditd-execve.png` | DET-009 | Loopback `bash -i >& /dev/tcp/…` + EXECVE telemetry (loopback only) — blocked on L-001 |
| `evidence/det010-hydra-then-login.png` / `evidence/det010-correlation.png` | DET-010 | Hydra burst + one good login from same IP, then the temporal correlation firing |

## Misc / Unsorted (reviewed, not filed)

Screenshots reviewed in both source folders that don't map to a current
DET or lab asset. Never guessed a mapping for these — listed here instead
so they aren't lost or silently skipped on the next pass.

| Source | Why not filed |
|---|---|
| `1-VM Created.png` … `9 - New Rule if 100002.png` (9 files, Jul 26–Aug 2 2026) | **Earlier/superseded lab build** — hostnames `soc-wazuh-server`/`soc-win10-endpoint`/`soc-linux-endpoint`, Wazuh v4.14.6, and custom rules 100001/100002 with different descriptions/MITRE IDs than the current `custom_rules.xml`. Doesn't match current `shared/vm-inventory.md` (linux-victim/windows-victim, 192.168.1.112/113, Wazuh 4.14.8). Filing these under current DET filenames would misattribute evidence from a prior lab iteration. |
| `Screenshot_20260818_003616.png`, `_160018.png` | Personal Homarr/Portainer homelab dashboard (containers incl. `n8n`, `Wazuh`, `Hermes-Agent`) — infra-adjacent (n8n is the planned Flagship 4 tool) but not DET evidence. |
| `Screenshot_20260822_001505.png` | Meme/reaction image — irrelevant. |
| `Screenshot_20260830_195439.png` | Gym workout-set table — irrelevant. |
| `Screenshot_20260902_155357.png`, `_20260903_230719.png`, `_230727.png`, `_230735.png` | Personal chat/DM screenshots — irrelevant and personal content. |
| `Screenshot_20260904_095457.png` | Hosting-provider billing page (error state) — irrelevant. |
| `Screenshot_20260904_113028.png` | Contabo GmbH payment receipt — plausibly the VPS bill hosting the Wazuh manager, but a receipt, not detection evidence. |
| `Screenshot_20260906_190825.png` | Facebook comment/reaction screenshot — irrelevant. |
| `Screenshot_20260925_175156.png` | F1 race leaderboard — irrelevant. |
| `Screenshot_20260926_122744.png` | Unrelated AI-agent task-list UI — irrelevant. |

## Conventions

- `detNNN-<what>.png` — detection evidence, numbered by Detection ID.
- `lab-*` / `siem-*` — environment and overview shots.
- `supporting-*` — useful context not tied to a DET (yet).
- When a DET flips to TESTED/VALIDATED, update its Status in the
  corresponding `docs/detections/det-*.md` and keep filenames stable so
  writeups don't rot.
