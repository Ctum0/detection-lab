# Known limitations

## L-001: auditd EXECVE telemetry not reaching Wazuh (open)

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

Further troubleshooting during the Flagship 2 Workstream 1 close-out ruled
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

### Resolution

**Fix: reboot `linux-victim`.** A full VM reboot clears the rate-limit
counter and cleanly reinitializes both `kauditd` and `auditd` — no config
changes are needed since the audit rule (`audit-wazuh-c`) and the agent's
`<localfile>` block were already correct. Not yet executed as of this
writeup.

### Workaround options (if the reboot doesn't fully resolve it)

1. **Short term:** validate DET-008/009 logic off-box — run the Sigma→SPL
   queries manually in Splunk against imported `ausearch` output, and
   record results in the det docs without flipping to VALIDATED.
2. **Alternative:** rewrite rules 100001/100010 against `full_log` PCRE2
   `<match>` (decoder-independent) if decoded fields prove unreliable —
   the converter already supports this pattern for sshd. Given the root
   cause is now known to be the daemon being dead rather than a decoding
   mismatch, this is unlikely to be needed.

### Next steps

- [ ] Reboot `linux-victim` (Sithum)
- [ ] Confirm `auditd`/`kauditd` are running clean post-reboot (`systemctl status auditd`, `ps` for `kauditd`)
- [ ] Re-run DET-008/DET-009 validation attacks (100001, 100010) once auditd is confirmed healthy (Sithum)
- [ ] Flip matrix + det-doc statuses to VALIDATED with evidence filenames
