# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

`search-collector` runs as an addon on each managed cluster. It watches all Kubernetes resources via dynamic informers, transforms them into a flat property map (a `Node`), computes relationships between resources (edges), and syncs the resulting diff to the `search-indexer` over HTTPS.

## Commands

```bash
make run        # GOGC=25 go run -tags development main.go --v=2
make test       # go test ./... -failfast  (sets DEPLOYED_IN_HUB=true)
make coverage   # test + open HTML coverage report
make lint       # golangci-lint + gosec
make build      # CGO_ENABLED=1 go build -o output/search-collector
```

Run a single test package:
```bash
go test ./pkg/transforms/... -failfast -run TestFoo
```

## Architecture

The pipeline is linear: **Informer → Transformer → Reconciler → Sender**

```
K8s API watch → informer.GenericInformer
                    ↓ Event{Type, Node}
               transforms.Transformer   (fan-out to numCPU goroutines)
                    ↓ NodeEvent
               reconciler.Reconciler    (in-memory state: all nodes + edges)
                    ↓ Diff{add/update/delete nodes+edges}
               send.Sender              (POST JSON to search-indexer /aggregator/clusters/<name>/sync)
```

### Key packages

- **`pkg/informer`** — `GenericInformer` wraps the dynamic client with its own watch/re-list loop (does not use `client-go` SharedInformerFactory). `RunInformers` discovers all GVRs via the discovery API, starts one informer per resource type, and watches for new CRDs to start additional informers dynamically.

- **`pkg/transforms`** — One file per resource kind (e.g. `pod.go`, `deployment.go`). Each implements `BuildNode()` to extract searchable properties and `BuildEdges()` to compute relationships. `common.go` handles properties and edges that apply to every resource (owner references, hosting annotations). Properties prefixed with `_` are internal and not user-searchable.

- **`pkg/transforms/configurableCollection.go`** — Loads `merged-collector-config` CollectorConfig from cluster. Applies include rules (add fields/conditions/annotations to resources) and exclude rules (prevent collection entirely). **Last matching rule wins** for exclude/include ordering. `IsResourceExcluded(group, kind)` is called at informer event entry points.

- **`pkg/transforms/collectorConfigReload.go`** — Dynamic reload without pod restart (ACM-20047). The collector watches the `CollectorConfig` GVR itself. On change: `ReloadAndDiff()` snapshots current config, reloads from cluster, diffs old vs new. If `AffectedResources` changed → targeted re-list of those types. If `ExcludeRulesChanged` → full `syncInformers` pass. If nothing changed → no-op.

- **`pkg/reconciler`** — Holds the full in-memory state of all nodes and edges seen so far. On each cycle it computes the diff (add/update/delete) against the previous state. Uses an LRU cache to handle out-of-order delete/add sequences.

- **`pkg/send`** — Serializes the `Diff` into the `Payload` struct and POSTs it to the indexer. Implements exponential backoff on failure. On the first successful sync after a failure or startup, sends `ClearAll: true` to let the indexer reset state for this cluster.

### Data model

A `Node` is `map[string]interface{}` with a stable `UID`. Every resource gets these common properties: `kind`, `name`, `namespace`, `created`, `apigroup`, `apiversion`, `label`. Type-specific transforms add additional properties. The `NodeStore` passed to `BuildEdges()` provides lookup by UID and by (kind, namespace, name) triple.

### Configurable Collection (CollectorConfig)

The collector reads `merged-collector-config` from the cluster namespace. Rules are processed in order; **last matching rule wins** for exclude/include conflicts.

- **Include rules:** add extra fields (via JSONPath), conditions, annotations to matching resources
- **Exclude rules:** prevent collection of matching apiGroups/kinds at the informer level
- **Bare include (no fields):** still meaningful — re-includes a resource excluded by a broader wildcard
- **Dynamic reload:** no pod restart needed; changes propagate within ~1 minute via watch event

Key functions:
- `IsResourceExcluded(group, kind)` — walks ordered `excludeRules` slice; `ActionInclude` cancels prior `ActionExclude`
- `applyDefaultTransformConfig(node, additionalColumns)` — applies configured fields to a resource's properties
- `ReloadAndDiff(dynamicClient)` — snapshot → reload → diff → targeted re-list or full informer sync

### Dev config

For local development, set values in `config.json` (committed with safe defaults); env vars override the file. `DEPLOYED_IN_HUB=true` skips the lease reconciler (required for `make test`). The `-tags development` build tag is not currently used but is kept for compatibility with other search repos.

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| AGGREGATOR_URL | https://localhost:3010 | Indexer URL |
| CLUSTER_NAME | local-cluster | Name of cluster being collected |
| HEARTBEAT_MS | 300000 (5 min) | Keepalive interval |
| REDISCOVER_RATE_MS | 120000 (2 min) | Min interval between CRD re-syncs |
| REPORT_RATE_MS | 5000 (5 sec) | Send interval |
| RUNTIME_MODE | production | 'development' for local dev |
| DEPLOYED_IN_HUB | false | true = hub collector (sets `_hubClusterResource`) |
| FEATURE_CONFIGURABLE_COLLECTION | false | Enable CollectorConfig rules |

### Non-obvious gotchas

- **Update skip optimization:** if Properties unchanged, update is dropped. Exceptions: Application, Subscription, Policy, ValidatingAdmissionPolicyBinding (metadata/edges may change)
- **Application-first ordering:** `allEdges()` processes Application UIDs first so `_hostingApplication` is set before other edges compute
- **Purged nodes LRU (500):** prevents out-of-order add-after-delete race
- **First send is always complete** (`ClearAll=true`); subsequent are diffs
- **On diff send error:** automatically falls back to complete payload
- **Totals mismatch:** compares response `TotalResources`/`TotalEdges` → triggers resync on mismatch
- **Bare include rules were silently skipped** before PR #925 — now fixed; `appendExcludeRule(ActionInclude)` runs before the "has enrichment?" check

## Fleet Engineering Skills

All skills are available as slash commands. See the [Fleet Engineering skills catalog](https://github.com/OpenShift-Fleet/agentic-sdlc/blob/main/skills/README.md) for the full list with when-to-use guidance.

## Personal configuration

Read personal config at the start of any task that needs an assignee, email, or project key.
Use the tool-aware fallback chain: `~/.config/opencode/user.local.md` (OpenCode),
`.claude/user.local.md` (Claude Code), or `.cursor/rules/user.local.mdc` (Cursor, already in context).
If none exist, fall back to agent memory (`user-config`), then placeholders.
