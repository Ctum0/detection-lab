# Known limitations

## L-001: auditd EXECVE telemetry not reaching Wazuh (WONTFIX — abandoned)

Blocks validation of DET-008 (SUID, rule 100010) and DET-009 (reverse
shell, rule 100001). Both rules exist in Sigma, Wazuh XML and SPL, but no
auditd EXECVE events have been observed in the manager, so neither custom
rule can fire.

### Symptoms

- `chmod u+s` / loopback reverse-shell tests on `linux-victim` produce no
  `auditd`-sourced alerts in Threat Hunting / Discover.
- Rules 100001/100010 never fire; no `audit.command` / `audit.args` fields
  seen on any event.
- Windows (Sysmon/Security/System) and auth.log (sshd) pipelines are
  healthy — the gap is specific to the auditd path.

### Troubleshooting performed

1. **auditctl persistence** — rules added with `auditctl` worked until
   reboot, then vanished. Rules were moved to `/etc/audit/rules.d/*.rules`
   so they persist; reloaded and re-verified with `auditctl -l`.
2. **augenrules** — ran `augenrules --load` to compile
   `/etc/audit/rules.d/` into `/etc/audit/audit.rules` and restarted
   `auditd`; `ausearch` locally confirms EXECVE records are generated on
   the victim itself (generation works, forwarding doesn't).
3. **af_unix plugin path** — investigated the audisp `af_unix` plugin
   (forwards audit events over a unix socket to the Wazuh agent). Plugin
   present but events still not surfacing as decoded `audit.*` fields.
4. **localfile fallback** — tried reading `/var/log/audit/audit.log`
   directly via a `<localfile>` block on the agent; multiline EXECVE
   records arrive but don't decode into `audit.command`/`audit.args`,
   so rules 100001/100010 (written against decoded fields) still miss.

### Root-cause hypothesis (superseded — see 2026-09-27 update below)

The manager-side audit decoders (parent 80700, `audit.command` /
`audit.args`) expect the agent's native audit reader format, while the
agent is either (a) not enrolled in audit collection at all (missing /
disabled audit integration in `agent.conf` / `ossec.conf`), or (b)
shipping raw `audit.log` lines via syslog/localfile that match a generic
decoder instead of the audit decoder. The `ausearch`-works-locally result
points at collection/forwarding, not at auditd itself.

### Update 2026-09-27 — root cause found: auditd itself is dead, not a decoding/forwarding gap

Further troubleshooting during the Adversary Emulation module's
Workstream 1 close-out ruled
out decoding/config and pinned the fault on the `linux-victim` auditd
service itself:

1. **Config re-verified as correct.** `auditctl -l` confirms the audit
   rule is loaded with `-k audit-wazuh-c`, matching what the agent's
   `<localfile>` audit block in `ossec.conf` expects. The agent-side
   config was never the problem.
2. **Agent is reading, logcollector confirmed.** Wazuh agent logs show
   `ossec-logcollector` actively tailing the audit log path with no
   errors — the agent side of the pipe is healthy and waiting for data.
3. **`audit.log` is stale.** `/var/log/audit/audit.log` on `linux-victim`
   has not been written to since 2026-09-26 20:50 UTC, and its last line
   is a `DAEMON_END` record — auditd shut itself down and never restarted.
   Every EXECVE test since has produced nothing because there is no
   daemon running to generate the record, independent of collection,
   forwarding or decoding.
4. **auditd is dead, systemd won't restart it.** `systemctl status auditd`
   confirms the process is not running. `systemctl restart auditd` fails
   with `Start request repeated too quickly` — systemd's restart
   rate-limiter tripped after repeated failed restart attempts (from
   earlier troubleshooting sessions) and is now refusing to start the
   unit within its rate-limit window.
5. **`kauditd` kernel thread still lingering.** Even with the userspace
   `auditd` process dead, the `kauditd` kernel thread is still active
   (visible in `ps`), which can itself interfere with cleanly
   re-initializing audit on a plain `systemctl start` — the kernel-side
   audit subsystem needs a full reset, not just a userspace daemon
   restart, which a rate-limited `systemctl restart` doesn't achieve.
6. **Wazuh agent connection-lock episodes observed** in parallel during
   this window (agent briefly dropping and re-establishing to the
   manager) — noted as a contributing distraction during triage but not
   the root cause; the `audit.log` staleness and dead daemon predate and
   are independent of these episodes.

**Conclusion:** this was never a forwarding/decoding gap — collection
config, agent localfile config and the manager-side decoders were fine
all along. The actual fault is that `auditd` died on `linux-victim` on
2026-09-26 20:50 UTC and systemd's restart rate-limiter plus the
lingering `kauditd` thread are preventing a clean in-place recovery.

### Resolution attempted: reboot (did not resolve)

A full reboot of `linux-victim` was performed to clear the rate-limit
counter and reinitialise `kauditd`/`auditd`. The audit rule
(`audit-wazuh-c`) and the agent's `<localfile>` block were already
correct, so no config change was involved. The retry did not restore a
working audit pipeline.

### Decision: abandoned

L-001 is closed as **WONTFIX**. The reasoning, stated plainly:

- The remaining work (reinstalling or rebuilding the audit path, then
  re-validating two Linux rules) was judged to cost more time than it adds
  for this lab. Windows and SSH telemetry, which carry most of the detection
  value, were already validated.
- Execve telemetry is still captured on the host itself, and `ausearch`
  shows it, so the activity is not invisible. It just does not reach
  Wazuh.
- The question will be revisited only as part of a fresh VM rebuild, not as
  a standalone repair.

**Current status of DET-008 and DET-009:** the rules (100010, 100001) stay
deployed in `custom_rules.xml`, unvalidated. The matrix marks them
UNTESTED — BLOCKED, not VALIDATED. They are not claimed to work.

### Next steps

- [x] Reboot `linux-victim` — done, did not resolve
- [ ] Revisit only during a fresh VM rebuild (no separate task planned)

---

## L-002: built-in rule 92213 fires on PowerShell policy-test artefacts (resolved)

PowerShell writes `__PSScriptPolicyTest_*.ps1` files into the user's Temp
directory on every session start. Wazuh's built-in Temp-drop rule (92213)
fires on each one, which produced hundreds of alerts per day with no signal.

**Resolution:** rule 100020, a level-0 child of 92213, matches the
`__PSScriptPolicyTest_*.ps1` filename and suppresses the alert.

**Result:** before the rule, hundreds of 92213 alerts per day from this
source; after it, zero. Genuine Temp-drop detections were unaffected.

---

## L-003: stray `local_rules.xml.bak` in the rules directory (resolved)

A backup file `local_rules.xml.bak` sat inside `/var/ossec/etc/rules/`.
`analysisd` loads every XML file in that directory, so the backup could
redefine custom rules alongside the live file. It was the same class of
problem as the duplicate-rule-ID bug in the pipeline demo.

**Resolution:** the backup file was removed from the rules directory.
