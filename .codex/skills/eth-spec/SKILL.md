---
name: eth-spec
description: Look up and explain Ethereum consensus, execution, beacon API, builder, and keymanager specifications from local spec repositories in the Lighthouse workspace. Use when the user asks about Ethereum spec behavior, fork differences, consensus rules, Beacon API or Engine API definitions, or wants Lighthouse code compared against the spec.
---

# Ethereum Spec Lookup

Use this skill from the Lighthouse workspace root.

Answer by reading local spec files under `plugins/eth-spec/specs/`, not by defaulting to web search.

## Ensure Specs Exist

If `plugins/eth-spec/specs/consensus-specs/specs/` is empty or missing, run:

```bash
git submodule update --init --recursive -- plugins/eth-spec/specs/
```

If the user asks for latest/current spec state, run:

```bash
git submodule update --remote -- plugins/eth-spec/specs/
```

## Spec Map

- `consensus-specs/`: beacon chain, fork choice, validator duties, p2p, networking, SSZ
- `beacon-APIs/`: beacon node REST API, especially `beacon-node-oapi.yaml`
- `builder-specs/`: builder API and MEV-boost related specs
- `execution-apis/`: Engine API and execution JSON-RPC
- `keymanager-APIs/`: validator keymanager REST API

Consensus fork directories:

- `phase0`
- `altair`
- `bellatrix`
- `capella`
- `deneb`
- `electra`
- `fulu`
- `gloas`
- `heze`

Common consensus files:

- `beacon-chain.md`: state transition, containers, processing
- `fork-choice.md`: fork choice changes
- `p2p-interface.md`: gossip and req/resp behavior
- `validator.md`: validator duties
- `fork.md`: fork transition

Other useful locations:

- `consensus-specs/fork_choice/`: base fork choice spec
- `consensus-specs/ssz/`: SSZ
- `consensus-specs/sync/`: optimistic sync
- `execution-apis/src/engine/`: per-fork Engine API specs

## Workflow

1. Identify which spec repo and fork apply.
2. If uncertain, start with the latest mainnet fork and work forward only when the question concerns future forks.
3. Search locally with `rg`; read the relevant sections.
4. If comparing Lighthouse code to spec, inspect analogous Lighthouse code with `rg`/`git grep`.
5. Answer with local file references such as `consensus-specs/specs/electra/beacon-chain.md`.
6. Quote only short excerpts when useful; otherwise paraphrase the rule and cite the file.
7. If behavior changed across forks, state the fork-specific difference.

Do not claim a spec requirement without tying it to a concrete local spec file or an explicit inference from code/spec context.
