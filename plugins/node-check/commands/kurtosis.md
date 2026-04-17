---
name: kurtosis
description: Spin up a Kurtosis local testnet and verify any user-defined condition against it
user_invocable: true
argument_description: "Any verifiable condition, e.g. 'run for 5 mins and make sure no missed slots', 'give me stats on fork_choice_write_lock span', 'using sigp/lighthouse:latest check head advances'"
---

Verify a user-defined condition against a Kurtosis local testnet. The condition can be anything observable — traces, metrics, logs, beacon API, container state, cross-node comparison, etc.

Condition: $ARGUMENTS

## Key domain knowledge

**Slot/epoch math**: Local testnet uses 3s slots, 32 slots/epoch (mainnet preset). 1 epoch = 96s. 5 mins ≈ 100 slots ≈ 3 epochs.

**Tempo service names**: Match Kurtosis service names (e.g. `cl-1-lighthouse-geth`, not `lighthouse`).

**Observation window**: When the user says "run for X mins/epochs", timing starts ONLY after the enclave is fully running (start script exited 0) AND the chain is healthy (head slot advancing). Never count startup time.

**Condition patterns**:
- *Absence* ("no missed slots", "this log doesn't appear"): fail-fast on first violation, PASS only if the full window elapses clean.
- *Accumulation* ("give me stats on span X", "sync 1000 slots"): collect data in intervals, aggregate and report at the end.
- *Per-slot/epoch reporting* ("print values every slot for 3 epochs"): query at each slot boundary (every 3s), not wall-clock intervals.

## Testing a specific PR

If the condition references a PR (URL or `#NNNN`), first resolve its branch and set up a worktree **before** the enclave lifecycle below:

1. `gh pr view <N> --json headRefName,headRepositoryOwner` — get branch name and fork owner (login).
2. Check `git worktree list` — if an existing worktree already has the PR branch checked out, ask the user which to use. Otherwise:
3. From `<workspace>/lighthouse`: `git fetch <fork-owner> <branch>` then create a worktree:
   ```bash
   git worktree add -b "pr-<N>-<slug>" "../worktrees/lighthouse-pr-<N>-<slug>" FETCH_HEAD && \
   echo "gitdir: ../../lighthouse/.git/worktrees/lighthouse-pr-<N>-<slug>" > "../worktrees/lighthouse-pr-<N>-<slug>/.git"
   ```
4. `cd` into the new worktree and run the enclave lifecycle from there (fresh build). Always rebuild fresh for a PR test — do not reuse an existing enclave unless the user says so.

## Enclave lifecycle

1. `kurtosis enclave inspect local-testnet` — if running, detect LH version via `/eth/v1/node/version` and ask user to reuse or rebuild. (Skip the reuse prompt when testing a PR — always rebuild.)
2. If not running, find `start_local_testnet.sh` in `scripts/local_testnet/` (check worktree path or main `lighthouse/` dir).

**Two startup paths:**

- **No custom image (default):** `timeout 600 ./start_local_testnet.sh` — builds a local Docker image from the worktree.
- **Custom image specified** (e.g. "using sigp/lighthouse:special" or "using nethermind as EL"):
  1. `cp network_params.yaml network_params.yaml.bak`
  2. `sed` the `cl_image`, `el_type`, `el_image` fields as needed
  3. `timeout 600 ./start_local_testnet.sh -b false` (skip local Docker build)
  4. **Always restore**: `mv network_params.yaml.bak network_params.yaml` (on success or failure)

If kurtosis engine is down: `kurtosis engine start`, wait 10s, retry.

## Verification

After enclave is up, discover ports from `kurtosis enclave inspect` output, confirm chain health (head slot advancing past slot 2), then verify the condition.

**Quick reference** (saves tool calls):
- Tempo search: `curl "$TEMPO/api/search?q={name=\"$SPAN\"}&limit=10"`
- Tempo trace: `curl "$TEMPO/api/traces/$ID"`
- Prometheus: `curl "$PROM/api/v1/query?query=$EXPR"`
- Beacon API: `$BN/eth/v1/node/syncing`, `$BN/eth/v2/beacon/blocks/{slot}`
- Logs: `kurtosis service logs local-testnet $SVC` (pipe to grep)
- Exec: `kurtosis service exec local-testnet $SVC "$CMD"`

Print progress every ~30s. On completion, report PASS with evidence or FAIL with what was expected vs observed. Ask user whether to keep or tear down the enclave.
