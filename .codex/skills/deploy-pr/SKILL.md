---
name: deploy-pr
description: Provision a Hetzner dev box, deploy a Lighthouse GitHub PR branch into the Kurtosis local testnet, expose Grafana and Beacon API access, and optionally hand off to a node verification workflow. Use when the user asks to deploy, test, run, or verify a Lighthouse PR on a temporary remote dev box or testnet.
---

# Deploy Lighthouse PR

Use this skill from `/home/jimmy/workspace/lighthouse-workspace`.

This workflow provisions a temporary `test-pr-<number>` dev box through
`jimmygchen/dev-box-provisioner`, checks out the PR branch on the remote box,
starts `scripts/local_testnet/start_local_testnet.sh`, and prints connection
details or a verification handoff command.

## Required Inputs

Parse the user request as:

- PR number: first numeric token, required.
- Verification condition: all text after the PR number, optional.

If the PR number is missing or non-numeric, stop and ask for it.

## Provision Dev Box

1. Fetch `dev_boxes.yml` content and SHA:

```bash
gh api repos/jimmygchen/dev-box-provisioner/contents/dev_boxes.yml
```

2. Decode `content`, compute tomorrow's UTC date, and add or update:

```yaml
- key: jimmygchen
  name: test-pr-<pr_number>
  until: <tomorrow>
```

Use a portable date command:

```bash
date -u -v+1d '+%Y-%m-%d' 2>/dev/null || date -u -d '+1 day' '+%Y-%m-%d'
```

3. Base64-encode the updated YAML with no newlines:

```bash
base64_content=$(printf '%s' "$updated_yaml" | base64 | tr -d '\n')
```

4. Commit the update through the GitHub API and capture `.commit.sha`:

```bash
gh api -X PUT repos/jimmygchen/dev-box-provisioner/contents/dev_boxes.yml \
  -f message="Provision test-pr-<pr_number>" \
  -f content="$base64_content" \
  -f sha="<sha>"
```

## Wait For Provisioning

Poll the provisioning workflow by matching the commit SHA. Do not assume the
latest run is the matching run.

```bash
gh run list -R jimmygchen/dev-box-provisioner -w provision.yml -L 5 \
  --json databaseId,status,conclusion,headSha
```

Poll every 15 seconds until the matching run completes. Timeout after 5
minutes. If it fails, show the run URL and stop.

Fetch logs through the API, because `gh run view --log` may be empty:

```bash
job_id=$(gh run view <run_id> -R jimmygchen/dev-box-provisioner \
  --json jobs --jq '.jobs[0].databaseId')
gh api repos/jimmygchen/dev-box-provisioner/actions/jobs/$job_id/logs |
  grep "IP:" |
  grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'
```

If multiple IPs are present, use the one from the log line matching
`test-pr-<pr_number>`. If no IP is found, show the last 20 log lines and stop.

## Prepare Remote Box

Poll SSH until reachable, retrying every 10 seconds for up to 3 minutes:

```bash
ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no -o BatchMode=yes root@<ip> echo ok
```

Set up a ControlMaster connection for subsequent commands:

```bash
ssh -fN -o ControlMaster=yes -o "ControlPath=/tmp/deploy-pr-ssh-%h" \
  -o StrictHostKeyChecking=no root@<ip>
```

Then wait for cloud-init with a 5-minute timeout:

```bash
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> "cloud-init status --wait"
```

Do not install packages or clone repos manually. Cloud-init handles Docker,
`yq`, Kurtosis, and the Lighthouse clone.

## Start Testnet

Fetch and check out the PR branch in the existing remote clone:

```bash
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> \
  "cd ~/lighthouse && git fetch origin pull/<pr_number>/head:pr-<pr_number> && git checkout pr-<pr_number>"
```

Start the local testnet with a 10-minute timeout:

```bash
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> \
  "cd ~/lighthouse/scripts/local_testnet && ./start_local_testnet.sh"
```

If the script fails, show the last 50 output lines and stop.

## Discover Ports And Health

Inspect the enclave:

```bash
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> \
  "kurtosis enclave inspect local-testnet"
```

Parse:

- Beacon API port: host port mapped to `5052` for `cl-1-lighthouse-geth`.
- Grafana port: host port mapped to `3000`.
- Metrics port: host port mapped to `5054` for `cl-1-lighthouse-geth`, if available.

Check beacon node health:

```bash
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> \
  "curl -s http://localhost:<beacon_port>/eth/v1/node/syncing"
```

Confirm `head_slot` is non-zero and `is_syncing` is false. If syncing, wait and
retry for up to 60 seconds.

Start background Lighthouse logs:

```bash
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> \
  "nohup kurtosis service logs -f local-testnet cl-1-lighthouse-geth > /tmp/lighthouse-cl-1.log 2>&1 & echo \$!"
```

Capture the PID and verify it:

```bash
ssh -o "ControlPath=/tmp/deploy-pr-ssh-%h" root@<ip> \
  "kill -0 <pid> && echo running"
```

## Create Grafana Tunnel

Find a free local port:

```bash
python3 -c "import socket; s=socket.socket(); s.bind(('',0)); print(s.getsockname()[1]); s.close()"
```

Create the tunnel:

```bash
ssh -fNL <local_grafana_port>:localhost:<remote_grafana_port> \
  -o "ControlMaster=auto" \
  -o "ControlPath=/tmp/deploy-pr-%h" \
  -o "StrictHostKeyChecking=no" \
  root@<ip>
```

Verify:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:<local_grafana_port>/api/health
```

If the tunnel fails, continue without Grafana and note how to access it
manually over SSH.

## Final Output

If a verification condition was provided, hand off to the local node-check
workflow if available. Include Grafana params only when the tunnel health check
returned `200`:

```text
/node-check:remote-ssh --host root@<ip> --bn-port <beacon_port> --metrics-port <metrics_port> --grafana-url http://localhost:<local_grafana_port> --grafana-user admin --grafana-pass changeme --log-source /tmp/lighthouse-cl-1.log -- <condition>
```

Without Grafana:

```text
/node-check:remote-ssh --host root@<ip> --bn-port <beacon_port> --metrics-port <metrics_port> --log-source /tmp/lighthouse-cl-1.log -- <condition>
```

If no verification condition was provided, print:

```text
Testnet running for PR #<pr_number>

Connection details:
  SSH:        ssh root@<ip>
  Beacon API: localhost:<beacon_port> via SSH, or ssh root@<ip> curl localhost:<beacon_port>
  Grafana:    http://localhost:<local_grafana_port> (tunneled, admin/changeme)
  Logs:       ssh root@<ip> "tail -f /tmp/lighthouse-cl-1.log"

To verify:
  /node-check:remote-ssh --host root@<ip> --bn-port <beacon_port> --metrics-port <metrics_port> --grafana-url http://localhost:<local_grafana_port> --grafana-user admin --grafana-pass changeme --log-source /tmp/lighthouse-cl-1.log -- <condition>

Cleanup:
  - Box auto-expires tomorrow (<expiry_date>)
  - Delete sooner by removing the entry from dev_boxes.yml through gh api
  - Close Grafana tunnel: ssh -O exit -o ControlPath=/tmp/deploy-pr-%h root@<ip>
```

## Error Handling

- On SSH timeout or connection refusal, report the failing command and suggest
  checking whether the box is still provisioned.
- On testnet startup failure, show the last 50 lines.
- On provisioning failure, show the workflow run URL.
- On Grafana tunnel failure, continue and print a note.
