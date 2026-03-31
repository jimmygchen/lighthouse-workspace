---
name: deploy-pr-test
description: Provision a Hetzner dev box, start a Kurtosis testnet with a PR branch, and optionally verify behavior
user_invocable: true
argument_description: "<pr_number> [verification condition], e.g. '1234 check head advances for 5 mins'"
---

Provision a dev box, deploy a Kurtosis testnet with a PR branch, and optionally hand off to `/remote-verify:verify` for verification.

Arguments: $ARGUMENTS

## Phase 1 — Parse arguments and provision dev box

### Step 1 — Parse arguments

Extract from `$ARGUMENTS`:
- **PR number**: first token (required). Must be numeric. If missing or non-numeric, stop and ask the user.
- **Verification condition**: everything after the PR number (optional).

### Step 2 — Trigger provisioning

1. Fetch current `dev_boxes.yml`:
   ```
   gh api repos/jimmygchen/dev-box-provisioner/contents/dev_boxes.yml
   ```
   Extract the file `content` (base64-decode it) and `sha`.

2. Compute tomorrow's date (box expiry):
   ```
   date -u -v+1d '+%Y-%m-%d' 2>/dev/null || date -u -d '+1 day' '+%Y-%m-%d'
   ```

3. Append a new entry to the YAML content:
   ```yaml
   - key: jimmygchen
     name: test-pr-<pr_number>
     until: <tomorrow>
   ```
   If an entry with `name: test-pr-<pr_number>` already exists, update its `until` date instead of appending.

4. Base64-encode the updated content (**strip newlines** — macOS `base64` adds line breaks by default):
   ```
   base64_content=$(echo -n "$updated_yaml" | base64 | tr -d '\n')
   ```
   Then commit via PUT:
   ```
   gh api -X PUT repos/jimmygchen/dev-box-provisioner/contents/dev_boxes.yml \
     -f message="Provision test-pr-<pr_number>" \
     -f content="$base64_content" \
     -f sha="<sha>"
   ```
   Capture the commit SHA from the response (`jq -r '.commit.sha'`) for workflow run matching in Step 3.

### Step 3 — Wait for provisioning workflow

1. Poll for the workflow run triggered by the commit from Step 2. Use the commit SHA to match the correct run:
   ```
   gh run list -R jimmygchen/dev-box-provisioner -w provision.yml -L 5 --json databaseId,status,conclusion,headSha
   ```
   Find the run whose `headSha` matches the commit SHA from Step 2. Poll every 15s until status is `completed`. Timeout after 5 minutes.

   **Important**: Do not assume the most recent run is the correct one — always match by `headSha`.

2. If the run failed, show the run URL (`gh run view <run_id> --web -R jimmygchen/dev-box-provisioner`) and stop.

3. Parse the server IP from workflow logs. **Important**: `gh run view --log` may return empty — use the API instead:
   ```
   # Get the job ID
   job_id=$(gh run view <run_id> -R jimmygchen/dev-box-provisioner --json jobs --jq '.jobs[0].databaseId')
   # Fetch logs via API and grep for IP
   gh api repos/jimmygchen/dev-box-provisioner/actions/jobs/$job_id/logs | grep "IP:" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'
   ```
   If multiple IPs found, use the one from the log line matching `test-pr-<pr_number>`.

   **If no IP is found**, show the last 20 lines of the job log and stop with an error.

## Phase 2 — Setup testnet on remote box

### Step 4 — Wait for SSH and cloud-init

First, poll SSH until the box is reachable:
```
ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no -o BatchMode=yes root@<ip> echo ok
```
Retry every 10s, timeout after 3 minutes.

Then set up a **ControlMaster** connection for all subsequent SSH commands (avoids repeated handshakes):
```
ssh -fN -o ControlMaster=yes -o "ControlPath=/tmp/deploy-pr-ssh-%h" -o StrictHostKeyChecking=no root@<ip>
```
All subsequent `ssh` commands should use `-o "ControlPath=/tmp/deploy-pr-ssh-%h"` instead of repeating `StrictHostKeyChecking`/`BatchMode`.

Then wait for cloud-init to finish. Cloud-init clones the lighthouse repo as its last step, so use that as the signal:
```
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "test -d ~/lighthouse"
```
Poll every 10s, timeout after 5 minutes. **Do not install any packages or clone repos manually** — cloud-init handles all dependencies (Docker, yq, kurtosis, lighthouse repo).

### Step 5 — Checkout PR branch

```
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "cd ~/lighthouse && git fetch origin pull/<pr_number>/head:pr-<pr_number> && git checkout pr-<pr_number>"
```

### Step 6 — Start local testnet

```
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "cd ~/lighthouse/scripts/local_testnet && ./start_local_testnet.sh"
```

Use a **10-minute timeout** for this command (timeout=600000). Treat non-zero exit code as failure.

If the script fails, show the last 50 lines of output and stop.

### Step 7 — Discover infrastructure

1. Inspect the enclave to find port mappings:
   ```
   ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "kurtosis enclave inspect local-testnet"
   ```

2. Parse from the output:
   - **Beacon API port**: the host port mapped to `5052` for `cl-1-lighthouse-geth`
   - **Grafana port**: the host port mapped to `3000` for the Grafana service
   - **Metrics port**: the host port mapped to `5054` for `cl-1-lighthouse-geth` (if available)

3. Confirm chain health — query beacon API via SSH to verify head is advancing:
   ```
   ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "curl -s http://localhost:<beacon_port>/eth/v1/node/syncing"
   ```
   Check that `head_slot` is non-zero and `is_syncing` is false. If syncing, wait and retry up to 60s.

### Step 8 — Start background log redirect

```
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "nohup kurtosis service logs -f local-testnet cl-1-lighthouse-geth > /tmp/lighthouse-cl-1.log 2>&1 & echo \$!"
```

Capture the PID and verify the process is running:
```
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "kill -0 <pid> && echo running"
```

## Phase 3 — Grafana tunnel and handoff

### Step 9 — Set up Grafana SSH tunnel

Find a free local port:
```
python3 -c "import socket; s=socket.socket(); s.bind(('',0)); print(s.getsockname()[1]); s.close()"
```

Create the tunnel:
```
ssh -fNL <local_grafana_port>:localhost:<remote_grafana_port> \
    -o "ControlMaster=auto" \
    -o "ControlPath=/tmp/deploy-pr-%h" \
    -o "StrictHostKeyChecking=no" \
    root@<ip>
```

Verify tunnel is up:
```
curl -s -o /dev/null -w "%{http_code}" http://localhost:<local_grafana_port>/api/health
```

### Step 10 — Hand off or print details

**If a verification condition was provided**, invoke the remote-verify skill.

If the Grafana tunnel was successfully established (Step 9 health check returned 200), include Grafana params:
```
/remote-verify:verify --host root@<ip> --bn-port <beacon_port> --grafana-url http://localhost:<local_grafana_port> --grafana-user admin --grafana-pass changeme --log-source /tmp/lighthouse-cl-1.log -- <condition>
```

If the Grafana tunnel failed, omit Grafana params (remote-verify will use beacon API and logs only):
```
/remote-verify:verify --host root@<ip> --bn-port <beacon_port> --log-source /tmp/lighthouse-cl-1.log -- <condition>
```

**If no condition was provided**, print connection details:

```
Testnet running for PR #<pr_number>

Connection details:
  SSH:        ssh root@<ip>
  Beacon API: localhost:<beacon_port> (via SSH tunnel, or ssh root@<ip> curl localhost:<beacon_port>)
  Grafana:    http://localhost:<local_grafana_port> (tunneled, admin/changeme)
  Logs:       ssh root@<ip> "tail -f /tmp/lighthouse-cl-1.log"

To verify:
  /remote-verify:verify --host root@<ip> --bn-port <beacon_port> --grafana-url http://localhost:<local_grafana_port> --grafana-user admin --grafana-pass changeme --log-source /tmp/lighthouse-cl-1.log -- <your condition>

Cleanup:
  - Box auto-expires tomorrow (<expiry_date>)
  - Delete sooner: remove entry from dev_boxes.yml via gh api
  - Close Grafana tunnel: ssh -O exit -o ControlPath=/tmp/deploy-pr-%h root@<ip>
```

## Error handling

- If any SSH command fails with connection refused/timeout, inform the user and suggest checking if the box is still provisioned.
- If `start_local_testnet.sh` fails, show the last 50 lines of output.
- If Grafana tunnel fails, proceed without it — print a note that Grafana is unavailable and the user can access it manually via SSH.
- If the provisioning workflow fails, show the run URL for manual inspection.
