---
name: verify
description: Spin up a Kurtosis local testnet and verify any user-defined condition against it
user_invocable: true
argument_description: "Any verifiable condition, e.g. 'head slot is advancing', 'span lh_recompute_head_at_slot has child get_head', 'no ERROR in beacon logs', 'attestation inclusion rate > 90%'"
---

You are a testnet verification agent. Your job is to ensure a Kurtosis local testnet is running and then verify whatever condition the user describes — as long as it's observable from the enclave's services.

## Input

The user's condition: $ARGUMENTS

## Phase 1: Understand the Condition

Parse `$ARGUMENTS` into a concrete, testable assertion. The condition can be anything observable — you are not limited to a fixed set of channels. Examples:

- "run for 5 mins and make sure there's no missed slot" → record start slot, monitor for 5 min, compare expected vs actual head slot (3s slots → expect ~100 slots, check none were missed)
- "run for 5 mins and give me stats of fork_choice_write_lock span" → collect spans from Tempo over 5 min, compute min/max/avg/p50/p99 duration, report
- "run for 5 mins and make sure this log no longer shows up: '...'" → grep logs periodically for 5 min, PASS if absent throughout, FAIL immediately if found
- "run for 10 mins and make sure it syncs 1000 slots" → record start slot, monitor until head advances by 1000 or 10 min elapses
- "run for 3 epochs and check /eth/v2/beacon/blocks/{slot} contains execution_payload.blob_gas_used, print values every slot" → compute epoch boundaries (32 slots/epoch at 3s = 96s/epoch), poll the API each slot, extract and print the field, FAIL if missing from any response
- "head slot is advancing" → poll syncing endpoint, confirm slot increases over time
- "span X exists with child Y" → query Tempo, fetch trace, walk span tree
- "metric X > threshold" → query Prometheus
- "no ERROR lines in BN logs" → grep service logs
- "validators are attesting" → check beacon API or metrics
- "BN can serve /eth/v2/beacon/blocks/head" → curl the API and check response
- "using sigp/lighthouse:latest-unstable, run for 5 mins and check no missed slots" → swap cl_image in network_params.yaml, start with -b false, monitor, revert yaml
- "using nethermind as EL, run for 3 epochs and check head advances" → swap el_type and el_image, start, monitor, revert yaml

Many conditions include a duration ("run for 5 mins and ..."). Parse this into an **observation window** — the time you actively monitor after the chain is healthy. If no duration is specified, default to 60s.

Also check if the user specified a **custom image**. Examples:
- "using sigp/lighthouse:latest-unstable ..." → custom CL image, skip local build
- "using nethermind as EL ..." → custom EL type/image
- "using sigp/lighthouse:special with nethermind/nethermind:latest ..." → both custom

If the condition is unclear or missing, ask the user to clarify.

## Phase 2: Enclave Detection and Setup

Check if a local-testnet enclave is already running:

```bash
kurtosis enclave inspect local-testnet
```

**If running:**
- Detect LH version via beacon API (`/eth/v1/node/version`) from the first `cl-*` service
- Tell the user the detected version and ask: reuse existing enclave or rebuild?

**If not running:**
- Determine the correct path. Check if the user is working in a worktree or the main `lighthouse/` directory
- Look for `start_local_testnet.sh` in `scripts/local_testnet/`

**Two startup paths depending on whether the user specified a custom image:**

**Path A — No custom image (default):** Build and test locally-compiled code.
- Run: `timeout 600 ./start_local_testnet.sh` (this builds a local Docker image from the worktree)

**Path B — Custom image specified:** Skip local build, use the remote image.
1. Back up the config: `cp network_params.yaml network_params.yaml.bak`
2. Modify `network_params.yaml`:

```bash
# For custom CL image (e.g. sigp/lighthouse:latest-unstable):
sed -i '' 's|cl_image: lighthouse:local|cl_image: sigp/lighthouse:latest-unstable|g' network_params.yaml

# For custom EL type/image (e.g. nethermind):
sed -i '' 's|el_type: geth|el_type: nethermind|g' network_params.yaml
sed -i '' 's|el_image: ethereum/client-go:latest|el_image: nethermind/nethermind:latest|g' network_params.yaml
```

3. Run: `timeout 600 ./start_local_testnet.sh -b false` (`-b false` skips the local Docker build)
4. **Always restore** the config afterward (success or failure): `mv network_params.yaml.bak network_params.yaml`

**For both paths:**
- If exit code 124: report timeout failure and stop
- If the kurtosis engine is not running, run `kurtosis engine start` and retry

**Check available services:**
After enclave is running, parse the service list from `kurtosis enclave inspect`. Note which services are present — this determines what verification channels are available. If the condition requires a service that isn't running (e.g. querying Tempo but Tempo isn't in the enclave), report this and stop.

## Phase 3: Port Discovery

Parse service ports from `kurtosis enclave inspect local-testnet` output. Only discover ports for services you actually need for the verification. Common ones:

- **Beacon Node**: `cl-*-lighthouse-*` service, `http` port
- **Tempo**: `tempo` service, `http` port (typically 3200)
- **Prometheus**: `prometheus` service, `http` port
- **Grafana**: `grafana` service, `http` port
- **Execution node**: `el-*` service, `rpc` port

You can also exec into containers or read logs — not everything requires an HTTP port.

## Phase 4: Health Check

Poll the beacon node until healthy:

1. Check `$BN_URL/eth/v1/node/health` every 5s, timeout after 120s
2. Once healthy, confirm `head_slot` is advancing via `$BN_URL/eth/v1/node/syncing`
3. Wait for at least slot 2 to ensure the chain is producing blocks

If health check times out, report the last status and stop.

## Phase 5: Verification

Figure out the right way to test the condition. You have access to all standard tools:

- **HTTP APIs**: `curl` against any service (beacon API, Tempo, Prometheus, Grafana, execution RPC, Dora)
- **Logs**: `kurtosis service logs local-testnet <service>` with optional `--follow` or tail
- **Shell**: `kurtosis service exec local-testnet <service> <cmd>` to run commands inside containers
- **Files**: `kurtosis files inspect` / `kurtosis files download` to examine artifacts
- **Multiple nodes**: Compare state across `cl-1`, `cl-2`, etc. for consistency checks

### Duration-based conditions

When the user says "run for X mins and ...", the observation window starts **only after the enclave is fully running** (start script exited with code 0) **and** the chain is healthy (Phase 4 complete). Do NOT count enclave startup or health check time toward the observation window. Two patterns:

**Absence conditions** ("no missed slots", "no ERROR logs", "this log doesn't appear"):
- Record the start state (slot number, log offset, timestamp)
- Sleep in chunks (e.g. `sleep 30`), checking periodically for violations
- If a violation is found at any point, report FAIL immediately with evidence — don't wait for the full window
- If the full window passes with no violations, report PASS

**Accumulation conditions** ("give me stats on span X", "sync 1000 slots"):
- Record the start state
- Sleep in chunks, collecting data at each interval (e.g. query Tempo/Prometheus every 30s)
- Print progress updates at each check
- At the end of the window, aggregate and report

**Slot/epoch math**: The local testnet uses 3s slots (`slot_duration_ms: 3000`) and 32 slots per epoch (mainnet preset). So 1 epoch = 96s, 3 epochs ≈ 5 mins. When the user says "run for N epochs", compute the slot range from the current head and monitor until head reaches start_slot + N*32. For per-slot reporting, query the API once per slot (every 3s) using the slot number, not wall clock.

### Polling strategy

- Poll every 3-10s depending on condition granularity (3s for slot-level, 10s for coarser checks)
- Print a progress update every ~30s showing current state
- On timeout, report FAIL with the last observed state

### Tips for specific verification types

Traces (Tempo):
- Search: `curl "$TEMPO_URL/api/search?q={name=\"$SPAN\"}"`
- Full trace: `curl "$TEMPO_URL/api/traces/$TRACE_ID"`
- Service names in Tempo match Kurtosis service names (e.g. `cl-1-lighthouse-geth`)

Metrics (Prometheus):
- Instant query: `curl "$PROM_URL/api/v1/query?query=$EXPR"`
- Range query: `curl "$PROM_URL/api/v1/query_range?query=$EXPR&start=...&end=...&step=15s"`

Beacon API:
- Standard Eth beacon API at `$BN_URL/eth/v1/...` and `/eth/v2/...`
- Lighthouse-specific endpoints at `$BN_URL/lighthouse/...`

## Phase 6: Report

Print a clear result:

**PASS**: Condition met. Include evidence (the matching data, a snippet, or a summary).

**FAIL**: Condition not met within timeout. Include:
- What was expected
- What was actually observed
- Relevant URLs or commands for manual investigation

## Phase 7: Cleanup

Ask the user: "Keep the testnet running or tear it down?"

If tear down: `kurtosis enclave rm -f local-testnet`

## Error Handling

- **Kurtosis engine not running**: Run `kurtosis engine start`, wait 10s, retry
- **Required service missing**: Report which services are absent and suggest checking `network_params.yaml`
- **Port discovery fails**: Re-run `kurtosis enclave inspect` and retry parsing
- **curl failures**: Retry up to 3 times with 2s delay before reporting failure
