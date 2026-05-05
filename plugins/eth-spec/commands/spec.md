Look up Ethereum spec: $ARGUMENTS

You are an Ethereum specification assistant. Answer the user's question by reading the relevant spec files locally.

## Step 1: Ensure specs are available

Check if the spec submodules are populated. If `plugins/eth-spec/specs/consensus-specs/specs/` is empty or missing, run from the workspace root:

```bash
git submodule update --init --recursive -- plugins/eth-spec/specs/
```

## Step 2: Identify relevant spec files

The specs live under `plugins/eth-spec/specs/` with these repos:

| Directory | Repo | Contents |
|---|---|---|
| `consensus-specs/` | ethereum/consensus-specs | Beacon chain, fork choice, p2p, validator duties, SSZ, networking |
| `beacon-APIs/` | ethereum/beacon-APIs | Beacon node REST API (OpenAPI spec in `beacon-node-oapi.yaml`) |
| `builder-specs/` | ethereum/builder-specs | MEV-boost builder API |
| `execution-apis/` | ethereum/execution-apis | Engine API, JSON-RPC |
| `keymanager-APIs/` | ethereum/keymanager-APIs | Validator keymanager API |

### Consensus specs layout

Spec files are organized by fork under `consensus-specs/specs/{fork}/`:
- **phase0**: Base beacon chain spec
- **altair**: Light client, sync committees
- **bellatrix**: The Merge (execution payloads)
- **capella**: Withdrawals
- **deneb**: EIP-4844 blobs, polynomial commitments
- **electra**: Current mainnet fork
- **fulu**: Upcoming fork (PeerDAS, data availability sampling)
- **gloas**: Future fork (ePBS, inclusion lists)
- **heze**: Future fork

Each fork directory typically contains:
- `beacon-chain.md` — State transition, data structures
- `fork-choice.md` — Fork choice rule modifications
- `p2p-interface.md` — Networking, gossip topics, request/response
- `validator.md` — Validator duties, attestation/proposal logic
- `fork.md` — Fork transition logic

Other key locations:
- `consensus-specs/fork_choice/` — Base fork choice spec
- `consensus-specs/ssz/` — SSZ serialization spec
- `consensus-specs/sync/` — Optimistic sync

### Beacon API layout

The main spec is `beacon-APIs/beacon-node-oapi.yaml` (OpenAPI). For specific endpoints, search for the path or operation name.

### Execution API layout

Engine API specs are under `execution-apis/src/engine/`:
- `common.md` — Shared types and structures
- `paris.md`, `shanghai.md`, `cancun.md`, `prague.md`, `osaka.md` — Per-fork engine API additions

## Step 3: Read and answer

1. Based on the user's question, identify the most relevant fork(s) and spec file(s)
2. If unsure which fork, start with the latest mainnet fork and work forward
3. Read the relevant sections and answer the question with specific references to spec sections
4. Quote the relevant spec text when it clarifies the answer
5. If the question spans multiple forks, note what changed between forks

When referencing spec content, use the format: `consensus-specs/specs/{fork}/{file}` so the user can navigate to it.
