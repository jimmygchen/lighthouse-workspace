---
name: verify
description: Verify a user-defined condition against remote beacon node(s) via SSH and Grafana
user_invocable: true
argument_description: "Any verifiable condition, e.g. 'check head is advancing for 5 mins', 'show me validator_count metric for 2 epochs', 'check for ERROR logs across all nodes'"
---

Verify a user-defined condition against remote beacon node(s). The condition can be anything observable — metrics, logs, beacon API, cross-node comparison, etc.

Condition: $ARGUMENTS

## Credential safety

**CRITICAL — NEVER violate these rules:**
- NEVER `echo`, `print`, `cat`, or log any variable loaded from `.env`
- NEVER interpolate credentials into output-producing commands
- To check if a var is set: `test -n "$VAR" && echo "set" || echo "unset"` — never print the value
- For Grafana auth: write a temporary netrc file instead of `curl -u`:
  ```
  NETRC=$(mktemp) && chmod 600 "$NETRC"
  echo "machine $(echo $GRAFANA_URL | sed 's|https\?://||;s|/.*||') login $GRAFANA_USER password $GRAFANA_PASS" > "$NETRC"
  curl --netrc-file "$NETRC" -s "$GRAFANA_URL/api/..."
  rm -f "$NETRC"
  ```
- Clean up netrc temp files in all code paths (success and failure)

## Key domain knowledge

**Slot/epoch math**: Auto-detect via `/eth/v1/config/spec` → `SECONDS_PER_SLOT`. Mainnet = 12s slots, 32 slots/epoch. 1 epoch = 384s (~6.4 min). 5 mins ≈ 25 slots.

**Observation window**: Timing starts ONLY after SSH tunnel is up AND beacon API responds with advancing head slot. Never count connection setup time.

**Condition patterns** (same as kurtosis-verify):
- *Absence* ("no missed slots", "no ERROR logs"): fail-fast on first violation, PASS only if the full window elapses clean.
- *Accumulation* ("stats on span X", "attestation performance"): collect data in intervals, aggregate and report at the end.
- *Per-slot/epoch reporting* ("print head slot every slot for 3 epochs"): query at each slot boundary (every `SECONDS_PER_SLOT`), not wall-clock intervals.

**Data source selection** — choose the right source for the query:
- *Beacon API*: real-time chain state — head, sync status, block contents, validator status
- *Metrics (Grafana)*: aggregated performance — rates, histograms, counters over time windows
- *Metrics (raw /metrics)*: simple current values only — gauges, counters (no PromQL functions)
- *Logs*: event-level detail — errors, warnings, specific code path execution
- *Traces*: TODO — Tempo support planned for future iteration

## Connection lifecycle

### 1. Gather connection details

**Step 1.0 — Parse inline connection params from `$ARGUMENTS`**:

If `$ARGUMENTS` contains `--host`, parse the following flags before any other step:
- `--host user@ip` — SSH host (required for inline mode)
- `--bn-port N` — beacon node port (default 5052)
- `--metrics-port N` — metrics port (default 5054)
- `--grafana-url URL` — Grafana endpoint (pre-tunneled by caller, use directly — no SSH tunnel needed for Grafana)
- `--grafana-user USER` — Grafana username (use for this session instead of loading from `.env`)
- `--grafana-pass PASS` — Grafana password (use for this session instead of loading from `.env`)
- `--log-source PATH` — log file path on remote host, or "journal" for journalctl
- Everything after a standalone `--` separator is the **verification condition**

If inline params are present:
1. Use them directly — **skip Step 1b** (interactive prompts)
2. Save host entry to `.remote-verify-config.json` (same format as Step 1c — merge/deduplicate with existing entries). Use defaults for any unspecified per-host fields (log_source=/var/log/lighthouse/beacon.log, journal_unit=lighthouse-bn, metrics_port=5054, sudo_logs=false). If `--log-source` is provided, use that instead of the default.
3. If `--grafana-url`, `--grafana-user`, `--grafana-pass` are provided, use them for this session — **skip `.env` loading** for Grafana in Step 2.
4. The verification condition is whatever follows `--` (not the original full `$ARGUMENTS`).
5. Proceed to SSH tunnel setup (Step 3) for beacon API only. If `--grafana-url` was provided, it is already tunneled — do not create an additional Grafana tunnel.

If `$ARGUMENTS` does NOT contain `--host`, continue with the existing interactive flow below.

---

**Step 1a**: Read `.remote-verify-config.json` from the workspace root. If it doesn't exist, `previous = null`.

**PROHIBITED — the agent must NEVER do any of these:**
- Ask "single or multiple nodes" as a question
- Invent, guess, or suggest example hostnames
- Check for existing SSH tunnels or control sockets before asking
- Run any Bash command before gathering connection details

Do NOT run ANY other tools before this step. No Bash commands. No tunnel checks. Nothing.

**Step 1b — First run** (no config file):

Output this exact text and wait for the user's reply. Do NOT use AskUserQuestion. Use defaults for all other settings (log source=/var/log/lighthouse/beacon.log, journal unit=lighthouse-bn, metrics port=5054).
```
Enter SSH host(s) to connect to (comma-separated for multiple, port optional — e.g. user@host1:5052, user@host2):
```
Parse each entry: if `:port` is present use that, otherwise default to `5052`. After the user replies, save to config and proceed directly to SSH tunnel setup.

**Subsequent runs** (config file exists) — exact JSON. Replace `<placeholders>` with actual values from config. Each host option label includes its port (e.g. `user@node1:5052`):
```json
{
  "questions": [
    {
      "question": "Select host(s) to connect to, or use Other to add new (format: user@host:port)",
      "header": "Hosts",
      "multiSelect": true,
      "options": [
        {"label": "<user@host1>:<port1>", "description": "Previously used"},
        {"label": "<user@host2>:<port2>", "description": "Previously used"}
      ]
    },
    {
      "question": "Log source",
      "header": "Logs",
      "multiSelect": false,
      "options": [
        {"label": "<previous-log-source> (Recommended)", "description": "Previously used"},
        {"label": "Other", "description": "Enter a different log source"}
      ]
    },
    {
      "question": "Metrics port",
      "header": "Metrics",
      "multiSelect": false,
      "options": [
        {"label": "<previous-metrics-port> (Recommended)", "description": "Previously used"},
        {"label": "Other port", "description": "Enter a different port"}
      ]
    }
  ]
}
```
Replace all `<placeholder>` values with actual values from config file. Only include questions relevant to the condition (max 4). Hosts is always included.

**Step 1c**: After user answers, save to `.remote-verify-config.json`:
```json
{
  "hosts": [
    {"host": "user@node1", "bn_port": "5052", "log_source": "/var/log/lighthouse/beacon.log", "journal_unit": "lighthouse-bn", "metrics_port": "5054", "sudo_logs": false},
    {"host": "user@node2", "bn_port": "5053", "log_source": "journal", "journal_unit": "lighthouse-bn", "metrics_port": "5054", "sudo_logs": true}
  ]
}
```
All settings are per-host. Merge selected previous hosts + any new "Other" input. Deduplicate by host. New hosts inherit defaults (bn_port=5052, log_source=/var/log/lighthouse/beacon.log, journal_unit=lighthouse-bn, metrics_port=5054, sudo_logs=false).

### 2. Load credentials from `.env`

Source `.env` from the workspace root **only if Grafana is needed** for the condition. If `.env` is missing or incomplete:
1. Check if `.env` exists; if not, run `cp .env.example .env && chmod 600 .env`
2. Tell the user: "Fill in Grafana credentials in `.env` with your editor, then re-run."
3. Stop — do not proceed without credentials.

### 3. SSH tunnels

Use SSH control sockets for clean management:
```
SSH_CONTROL="/tmp/remote-verify-%r@%h:%p"
ssh -fNL $LOCAL_PORT:localhost:$REMOTE_BN_PORT \
    -o "ControlMaster=auto" \
    -o "ControlPath=$SSH_CONTROL" \
    -o "ExitOnForwardFailure=yes" \
    $SSH_HOST
```

**Port selection**: Find a free port with:
```
python3 -c "import socket; s=socket.socket(); s.bind(('',0)); print(s.getsockname()[1]); s.close()"
```

**Multi-node**: Each host gets its own tunnel + control socket. Track mapping of `host → local_port` internally.

**If SSH hangs**: Likely waiting for a passphrase. Tell the user to check `ssh-agent` is running and key is added (`ssh-add -l`).

### 4. Detect network

Query `/eth/v1/config/spec` → extract `SECONDS_PER_SLOT`, `SLOTS_PER_EPOCH`. Confirm chain health (head slot advancing across 2 queries spaced 1 slot apart).

### 5. Teardown

On completion, ask user whether to keep or close SSH tunnels. Close via: `ssh -O exit -o "ControlPath=$SSH_CONTROL" $HOST`.

## Data source query patterns

### Beacon API (via tunnel)
- `curl -s "http://localhost:$LOCAL_PORT/eth/v1/node/syncing"`
- `curl -s "http://localhost:$LOCAL_PORT/eth/v2/beacon/blocks/{slot}"`
- `curl -s "http://localhost:$LOCAL_PORT/eth/v1/node/version"`

### Metrics via Grafana

First, detect Grafana version: `curl --netrc-file "$NETRC" -s "$GRAFANA_URL/api/health" | jq -r '.version'`

**Discover Prometheus datasource** (cache the UID after first call):
```
curl --netrc-file "$NETRC" -s "$GRAFANA_URL/api/datasources" | jq '.[] | select(.type=="prometheus") | {uid, id, name}'
```

**Query** — use `/api/ds/query` (works on Grafana 9+):
```
curl --netrc-file "$NETRC" -s -X POST "$GRAFANA_URL/api/ds/query" \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [{
      "datasourceId": $DS_ID,
      "expr": "$PROMQL_EXPR",
      "refId": "A",
      "instant": true
    }],
    "from": "now-5m",
    "to": "now"
  }'
```

### Metrics via raw /metrics (fallback when no Grafana)

Create additional SSH tunnel to `$METRICS_PORT` (default 5054) on the remote host.
```
curl -s "http://localhost:$auto_metrics_port/metrics" | grep "$METRIC_NAME"
```

Supports simple value lookups and grep-style checks only. If the user requests PromQL functions (`rate()`, `histogram_quantile()`, etc.), warn that Grafana is needed and suggest configuring `GRAFANA_URL`.

### Traces (TODO)

Tempo support is planned for a future iteration. If the user requests trace-based verification, inform them that trace support is not yet available and suggest using metrics or logs as alternatives.

### Logs via SSH

**Auto-detect log source**: Use the log source provided by the user during setup (file path or journalctl).

**Timezone handling**: Log files and journalctl on the remote host use **UTC**. When the user specifies a relative time (e.g. "5 hours ago"), convert to a UTC timestamp before querying. Use `date -u` to compute the cutoff:
```
UTC_SINCE=$(date -u -d '5 hours ago' '+%Y-%m-%d %H:%M:%S' 2>/dev/null || date -u -v-5H '+%Y-%m-%d %H:%M:%S')
```
(The first form works on Linux, the fallback on macOS. Run this locally, then pass the UTC timestamp to the remote command.)

**File-based logs**:
- Recent lines: `ssh $HOST "tail -n 1000 $LOG_PATH"`
- Time-bounded: `ssh $HOST "awk '\$0 >= \"$UTC_SINCE\"' $LOG_PATH | tail -n 500"` — assumes log lines start with a sortable timestamp (ISO 8601 or similar). If log format is different, use `grep` with the date prefix instead.
- Pattern search: `ssh $HOST "grep '$PATTERN' $LOG_PATH | tail -n 100"`
- Time + pattern: `ssh $HOST "awk '\$0 >= \"$UTC_SINCE\"' $LOG_PATH | grep '$PATTERN' | tail -n 100"`

**Journalctl logs**:
- Recent: `ssh $HOST "journalctl -u lighthouse-bn --since '$UTC_SINCE' --no-pager -o cat"`
- Pattern search: `ssh $HOST "journalctl -u lighthouse-bn --since '$UTC_SINCE' --no-pager -o cat | grep '$PATTERN'"`
- The systemd unit name defaults to `lighthouse-bn` but can be overridden by the user during setup.

**Sudo auto-detection**: If a log command returns "Permission denied", retry once with `sudo` prepended. If sudo works, remember it and use `sudo` for all subsequent log commands in this session. Save `"sudo_logs": true` to the config file so future runs use sudo from the start.

**IMPORTANT: Never use `tail -f`** — it will hang the bash tool (2-minute timeout, no streaming). Instead, poll with `tail -n N` or `journalctl --since` at each observation interval.

**Multi-node**: Query each host separately. Aggregate results across hosts for cross-node checks.

## Resilience

- **Transient HTTP errors (5xx)**: Retry up to 3 times with 5s backoff.
- **SSH tunnel death**: Before each query cycle, verify tunnel with `ssh -O check`. If dead, attempt to re-establish once. If that fails, abort verification with a clear error.
- **Grafana down**: If Grafana returns errors for 3 consecutive cycles, warn the user and continue with available data sources (beacon API, logs via SSH).

## Parallel checks

If the user's condition contains multiple independent checks (e.g. "check head is advancing AND check for ERROR logs AND show me validator_count metric"), run them in parallel using the Agent tool:

1. Set up SSH tunnels first (this must happen in the main agent before spawning subagents).
2. Spawn one Agent per independent check, passing the tunnel details (local port, host) in the prompt. Each subagent runs its check and returns PASS/FAIL with evidence.
3. Collect results and present a unified report.

This also applies to multi-node checks — if the same condition needs to run on multiple hosts, spawn one agent per host.

## Progress and output

Print progress every ~30s. On completion, report **PASS** with evidence or **FAIL** with what was expected vs observed. Ask user whether to keep or tear down SSH tunnels.

## TODO

- [ ] Tempo/trace support (direct URL and/or via Grafana datasource proxy)
- [ ] Elasticsearch support for log querying across nodes
