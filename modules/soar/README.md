# SOAR & Automated Response

**Status: ✅ COMPLETE** — live end to end on `n8n.ctum.org`.

Wazuh alerts flow into an n8n workflow that filters, deduplicates,
enriches, asks an LLM for a short triage, and sends the result to Telegram.
The workflow stops there. A human reads the triage and decides what to do.

**Design principle: no autonomous response.** The pipeline never blocks an
IP, disables an account, or isolates a host. It produces a triage message
and a person acts on it. Automated containment is out of scope on purpose
until the triage quality is proven over a long period.

Workflow: **SOC Alert Triage** (n8n workflow ID `JZZ3TrGbgKEnV1Rp`, active).
The exported JSON lives at [`workflow-export.json`](workflow-export.json).

## Architecture

```text
Wazuh manager
  │  native "shuffle" integration hook
  │  filter: rule level >= 8, alert_format json
  ▼
POST /webhook/wazuh-alerts  (n8n)
  │
  ▼
[1] Webhook ──► [2] Severity Filter (threshold: see note)
                    │
                    ▼
               [3] Parse Alert (code)        extracts rule_id, rule_desc, level,
                    │                        agent_name, mitre, src_ip, file_hash,
                    │                        timestamp, dedup_key
                    ▼
               [4] Remove Duplicates v2      atomic, DB-backed, key = dedup_key
                    │
                    ▼
               [5] AbuseIPDB Enrichment      HTTP, reputation for src_ip
                    │
                    ▼
               [6] Merge (code, run once)    pulls enrichment itself, builds the
                    │                        LLM request body
                    ▼
               [7] AI Analysis               POST litellm /v1/chat/completions,
                    │                        model "Hermes Agent", 300 s timeout
                    ▼
               [8] Format Message (code)     strips markdown, builds Telegram HTML
                    │
                    ▼
               [9] Telegram Notify           sendMessage, parse_mode=HTML
                    │
                    ▼
            Human reads the triage and decides
```

## Node-by-node

| # | Node | Type | What it does | Why it is built this way |
|---|---|---|---|---|
| 1 | Wazuh Alert Webhook | Webhook (POST, path `wazuh-alerts`) | Receives the alert JSON from the Wazuh integration. | Single entry point; the path is the only Wazuh-side coupling. |
| 2 | Severity Filter | Filter | Passes items where `body.severity >= 3`. **Note:** the Wazuh integration already gates on level 8, so this node is a second, looser check on a different field. See the open items below. | Keeps empty or non-alert input out of the LLM budget. |
| 3 | Parse Alert | Code | Flattens the Shuffle payload into named fields and computes `dedup_key`. | Shuffle nests fields under `body.all_fields`, not `body.rule`. Parsing in one place means downstream nodes never care about the wrapper. |
| 4 | Dedup (2min) | Remove Duplicates (`removeItemsSeenInPreviousExecutions`) | Drops items whose `dedup_key` was seen in a previous execution (history 5000). The node name says "2min", but the window is set by the minute-bucketed key, not a timer; the name is misleading and should be renamed. | Database-backed and atomic, so concurrent executions cannot both pass. `dedup_key` is `rule_id` plus a minute-bucketed timestamp: a Wazuh double-post within the same minute collapses, a new attack minutes later fires again. |
| 5 | AbuseIPDB Enrichment | HTTP Request | Looks up `src_ip` reputation over the last 90 days. `neverError` is set so an API failure still produces a triage. | Enrichment is advisory; it must not block the pipeline. |
| 6 | Merge | Code (run once) | Reads Parse Alert's output and the enrichment result via `$('AbuseIPDB Enrichment').first()`, then builds `llm_body` as a JSON string. | Runs exactly once per alert. Explained in the war story: multiple inbound wires to one node cause one execution per branch. |
| 7 | AI Analysis | HTTP Request | Sends `llm_body` to the local LLM gateway. | Model is `Hermes Agent` via litellm. The 300 s timeout exists because the upstream can hang rather than fail. |
| 8 | Format Message | Code | Removes markdown (`**`, `__`, backticks, headers), turns bullets into `•`, and wraps sections in `<b>`/`<i>` HTML. | Telegram HTML mode is strict. Stripping markdown first avoids broken message rendering. |
| 9 | Telegram Notify | HTTP Request to the Bot API `sendMessage` | Sends the message to the analyst chat with `parse_mode=HTML`. Implemented as a raw HTTP call, not the Telegram node. | The human-in-the-loop endpoint. |

Latency from alert to Telegram message is roughly 2–3 minutes, most of it
spent in the AI step.

## Integration

**Wazuh side.** The manager uses a native integration block:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>https://n8n.ctum.org/webhook/wazuh-alerts</hook_url>
  <level>8</level>
  <alert_format>json</alert_format>
</integration>
```

This block is configured through the Wazuh dashboard config editor, not by
editing `ossec.conf` on the host. In single-node docker the live config
sits on a named volume. Edits to the host bind-mount never reached the
running container. See the war story.

**n8n side.** The workflow is the only consumer of `/webhook/wazuh-alerts`.
Telegram delivers to a single analyst chat. The bot token, the chat ID, the
AbuseIPDB key, and the litellm bearer token all live in n8n credentials,
not in the workflow JSON.

## Sample triage output

Illustrative only. The addresses come from RFC 5737 documentation ranges,
and the agent name and timestamp are invented for the example.

> **🔔 Wazuh alert: PowerShell with encoded command [T1059.001]**
> Rule 100005 · level 10 · agent `win-victim-01` · 2026-09-27 03:13 UTC
>
> <b>WHAT HAPPENED</b>
> `powershell.exe` launched with `-EncodedCommand`. The payload decodes to a
> short download-and-execute stub, source host `203.0.113.45`.
>
> <b>TECHNIQUE</b>
> T1059.001 — PowerShell. Obfuscated command-line execution.
>
> <b>SEVERITY</b>
> High. Encoded PowerShell from an interactive session is uncommon; the
> enrichment shows the source address has no abuse reports.
>
> <b>NEXT STEPS</b>
> • Decode the payload and check the URL it fetches.
> • Pull the parent process and logon session for `win-victim-01`.
> • Confirm whether the run was an authorised test.
>
> <b>FP LIKELIHOOD</b>
> Medium. Admin automation also uses encoded commands. Check against the
> change calendar before escalating.

## Lessons from the build

The full debugging story is in [`docs/war-story.md`](docs/war-story.md).
Seven stacked issues, including two of my own mistakes. Distilled
SOAR-specific lessons are in
[`shared/lessons-learned.md`](../../shared/lessons-learned.md#soar-build-lessons-module-4).

## Open items before publishing

- **Severity threshold mismatch.** The brief calls for level ≥ 8. The live
  Severity Filter checks `body.severity >= 3`, and `severity` is not the field
  Parse Alert reads (it uses `rule.level`). Either the filter is a no-op, or
  it depends on a field the payload may not carry. Needs a decision and a
  test with a real level-3 and a real level-8 alert before the README claims
  a threshold.
- **Node naming.** `Dedup (2min)` should be renamed to match its real
  behaviour.

## Re-importing the workflow

1. In n8n, open **Workflows → Import from file** and select
   `workflow-export.json`.
2. Every secret is exported as the literal text `REDACTED`. Fill them in
   before activating:
   - **AbuseIPDB Enrichment** — header `Key`: your AbuseIPDB API key.
   - **AI Analysis** — header `Authorization`: `Bearer <your litellm token>`.
   - **Telegram Notify** — the bot token in the URL path
     (`https://api.telegram.org/bot<token>/sendMessage`), and `chat_id` in the
     JSON body of the same node.
3. Activate the workflow. The Webhook node shows the production URL. Point
   the Wazuh integration `hook_url` at it.

## Related

- [Detection Pipeline](../detection-pipeline/README.md) — the source of the alerts.
- [Adversary Emulation](../adversary-emulation/README.md) — the attacks that
  produced the real alerts used to test this pipeline.
- `shared/architecture.md` — platform-wide data flow.
