# The SOAR build: seven problems, in the order they hit

This is the record of building the Wazuh-to-Telegram triage pipeline. Each
section follows the same shape: what I saw, what I first believed, what was
actually wrong, what I changed, and what I took from it. Two of the seven
were mistakes I made myself, and they are written up as such.

---

## 1. The custom integration that did nothing

**Symptom.** I configured a Wazuh `custom-webhook` integration pointing at
the n8n webhook. Alerts were generated, the integration block was present,
and nothing arrived in n8n.

**What I first believed.** The webhook URL, the level filter, or the n8n
workflow was wrong. I checked each one.

**Actual root cause.** A `custom-*` integration needs a matching executable
in `/var/ossec/integrations/` with the same name. Wazuh runs that script
for each alert. Without it, the integration is silently skipped: no error
in the logs that points at the cause, just no traffic.

**Fix.** Switched to Wazuh's native `shuffle` integration, which ships with
the manager and needs no custom script.

**Lesson.** A silently skipped integration looks identical to a working one
that has no matching events. Verify the delivery end to end on a known alert
before debugging anything downstream.

---

## 2. The host config edits that never arrived

**Symptom.** I edited `ossec.conf` on the VPS host to set the shuffle hook.
The manager ignored it.

**What I first believed.** The file was not being read, or I had the wrong
section. I re-checked the XML several times.

**Actual root cause.** In single-node docker, the live config is on a named
docker volume, not the bind-mounted path I was editing. The container never
saw my host-side changes. Nothing was broken in the file itself; I was
editing a copy the container does not use.

**Fix.** Made the change through the Wazuh dashboard's configuration editor,
which writes to the volume the manager actually reads.

**Lesson.** When a container config looks correct but has no effect, find
out which file the running process reads before you check the contents. In
this setup, `docker inspect` on the mounts answers that in one command.

---

## 3. The payload that was one level deeper than I expected

**Symptom.** The Parse Alert node returned empty fields. `rule_id` was
undefined.

**What I first believed.** The Shuffle hook was sending a different schema
than the one I had read in the docs.

**Actual root cause.** The payload was valid. Shuffle nests the Wazuh alert
under `body.all_fields`, not directly under `body.rule`. My parser was reading
the right data one level too shallow.

**Fix.** Rewrote Parse Alert to read from `body.all_fields`, and flatten the
fields it needs into the item it passes downstream.

**Lesson.** Look at a real captured payload before writing a parser. The
docs described the outer envelope; only the captured request showed the
structure I actually had to parse.

---

## 4. The model that hung instead of failing

**Symptom.** The AI Analysis node timed out. Short test prompts to the same
model returned in about 4 seconds, so the model looked healthy.

**What I first believed.** The prompt was too long for the context window, so
I trimmed it. The timeouts continued.

**Actual root cause.** The upstream gateway (litellm) was flapping, and the
model `kimi-k3` could hang for a very long time on prompts of realistic size.
The "Say OK" tests passed because they were tiny. They did not exercise the
production path: a real triage prompt took 76 seconds or more, and sometimes
never returned.

**Fix.** Switched to the `Hermes Agent` model through the same gateway, which
completes reliably at production prompt size. Set an explicit 300 s timeout on
the HTTP node, because the failure mode is a hang, not an error.

**Lesson.** A health check with a tiny input proves almost nothing about a
production-size request. Test with the real payload size, and set explicit
timeouts on anything that calls a model.

---

## 5. The auth header I corrupted myself

**Symptom.** Requests to the LLM gateway started failing authentication after
an edit session.

**What I first believed.** The gateway had rotated its key.

**Actual root cause.** My own mistake. While editing, I pasted a 22-character
placeholder into the `Authorization` header in place of the real 67-character
bearer token. The gateway was fine. The header was wrong.

**Fix.** Restored the full 67-character token in the header.

**Lesson.** Secrets typed into a free-text field are easy to corrupt during
an edit session, and the failure looks like an upstream problem. When an
auth error appears right after an edit, compare the edited value to the
original before suspecting the service.

---

## 6. Deduplication: four attempts

Duplicates matter because Wazuh can post the same alert more than once, and
an analyst should not get the same message twice. This took four attempts.

**6a. Code-node memory.** The first version stored seen alert IDs in the
Code node's `staticData`. Two executions that arrived within the same
millisecond both read an empty store and both passed. That is a time-of-check
to time-of-use race, and `staticData` offers no atomicity for concurrent runs.

**6b. Remove Duplicates, wrong key.** I moved to n8n's Remove Duplicates node,
which is database-backed and atomic. The first configuration set
`dedupeValue` to `$json.body.rule_id`. That path does not exist. Parse Alert
outputs flat fields, so every item got an undefined key. This was my own
mistake: I referenced the payload shape from before problem 3 was fixed.
The node looked configured and did nothing useful.

**6c. "Value Is New" was too permanent.** With the key corrected, the node
was set to "Value Is New" mode. That mode remembers every value forever. The
first alert for a rule blocked every later alert for the same rule, so a
genuine new attack weeks later was silently dropped.

**6d. The final design.** The key is now `rule_id` plus a minute-bucketed
timestamp. A Wazuh double-post within the same minute collapses to one item.
A new attack a few minutes later produces a new key and fires again. History
is capped at 5000 entries.

**Lesson.** Deduplication has two different jobs: collapsing repeats within a
window, and not blocking a legitimate event later. A dedup key that is only
the rule ID does the second job badly. A key with a time bucket does both.

---

## 7. The doubled Telegram messages

This was the hardest bug, and it only appeared under real traffic.

**Symptom.** Every alert produced two identical Telegram messages. Synthetic
test payloads produced one.

**What I first believed.** The Telegram node was retrying, or the webhook was
being delivered twice. I checked both: neither was happening.

**Actual root cause.** The Merge node had two inbound connections, one
directly from Remove Duplicates and one from AbuseIPDB Enrichment. In n8n, a
node runs once for each arriving input branch. It does not wait and combine
inputs the way a SQL join would. So Merge ran twice, and every node after it
(AI Analysis, Format Message, Telegram Notify) ran twice.

Synthetic payloads only ever delivered data on one branch, so the second
run had nothing to pass through and looked like a single success. Real
attacks delivered both branches, which is why the duplication only showed up
on real alerts.

**Fix.** Rewrote Merge as a run-once Code node. It pulls the enrichment output
itself with `$('AbuseIPDB Enrichment').first()`, and I deleted the direct
Remove Duplicates → Merge connection. Now Merge has one input and runs once.

**Expected behaviour that looked like a bug.** An attack that triggers two
different rules, for example custom rule 100005 and built-in rule 92213,
correctly produces two messages, one per detection. That is the intended
result: each detection is a separate finding for the analyst. It is not
duplication. The distinction only matters once the single-run fix is in
place, because before it, the two-message case and the double-run bug looked
the same.

**Lesson.** In n8n, multiple inbound wires to a node mean multiple executions,
not merged inputs. This was the single biggest source of duplicate side
effects in the build. And test with real attacks end to end: the bug was
invisible to every synthetic test I wrote, and it appeared on the first real
run.

---

## What the seven problems have in common

Most were not exotic failures. They were quiet ones: an integration that
skipped without an error, a config file the container never read, a model
that hung rather than failed, a test that passed for the wrong reason, a
dedup key that was right for one case and wrong for another. In each case
the system did something plausible, and it took a specific check to show it
was not doing what I intended.

The checks that found the problems were the ones that used real input at
real size: a captured payload, a production-length prompt, and a real attack
rather than a synthetic one.
