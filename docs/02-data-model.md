# 02 — Data model

This document defines the in-memory model the Cairn app builds from all its
sources, and the shape in which it is sent to the UI. The architecture is in
`01-architecture.md`.

## Goals

- **One graph for everything.** Topology, traffic, metrics and security
  findings all attach to the same entities, so the UI can draw any of them
  on the same map.
- **Source-agnostic.** The model does not care whether a connection came
  from the eBPF agent or Hubble, or a finding from Trivy or Kubescape.
- **Streamable.** Every change is expressible as a small diff, so the UI
  stays live without refetching.
- **Level-of-detail friendly.** The UI can ask for one level (clusters,
  nodes, pods, containers) or one subtree without receiving the rest.

## Entities

The map is a tree of **entities** plus a set of **edges** between them.

```
Cluster
 ├─ Node            (server | agent; capacity, allocatable, usage)
 │   └─ Pod         (phase, owner workload, QoS, IPs)
 │       └─ Container (image, ports, restarts, usage)
 ├─ Namespace       (grouping, not a parent in the tree)
 ├─ Workload        (Deployment, StatefulSet, DaemonSet, Job, CronJob)
 ├─ Service         (type, ports, selector → pods)
 └─ NetworkPolicy   (selectors, ingress/egress rules)
External            (pseudo-entity: an IP/CIDR or DNS name outside the cluster)
```

The containment tree (cluster → node → pod → container) is what the map
zooms through. Namespaces, workloads and services are **grouping and
overlay** entities: the UI can group or colour by them, but they are not
levels of the zoom.

### Identity

Every entity has a stable, globally unique ID built from its location:

```
<cluster>/<kind>/<namespace>/<name>[/<sub>]

home-k3s/Node//rpi-01
home-k3s/Pod/media/jellyfin-6c9f7d-abcde
home-k3s/Container/media/jellyfin-6c9f7d-abcde/jellyfin
external/IP//203.0.113.10
```

- `cluster` is the kube context name, or a user-chosen alias.
- Cluster-scoped kinds leave `namespace` empty.
- Kubernetes UIDs are stored as attributes but not used as IDs, because IDs
  must be predictable from names (deep links, saved views).

### Common entity shape

```go
type Entity struct {
    ID       string            // as above
    Kind     Kind              // Cluster, Node, Pod, Container, ...
    Cluster  string
    Parent   string            // containment parent ID, "" for clusters
    Name     string
    Labels   map[string]string
    Attrs    map[string]any    // kind-specific fields, see below
    Status   Status            // Healthy | Warning | Error | Unknown
    Severity Severity          // worst open finding: None | Low | Medium | High | Critical
    Updated  time.Time
}
```

`Status` is operational health (pod not ready, node pressure). `Severity` is
the worst security finding attached. They are kept apart so the UI can show
both: a healthy pod can still be dangerous.

### Kind-specific attributes (first cut)

| Kind | Key attributes |
|---|---|
| Cluster | `version`, `distribution` (k3s, …), `cni`, `capabilities` (chart installed, CRDs found) |
| Node | `role` (server/agent), `addresses`, `capacity`, `allocatable`, `conditions`, `os`, `arch` |
| Pod | `namespace`, `phase`, `owner` (workload ID), `podIP`, `hostNetwork`, `qos`, `serviceAccount` |
| Container | `image`, `imageDigest`, `ports`, `restartCount`, `state`, `securityContext` summary |
| Workload | `kind`, `replicas` (desired/ready), `selector` |
| Service | `type`, `clusterIP`, `ports`, `selector`, `endpoints` (pod IDs) |
| NetworkPolicy | `podSelector`, `policyTypes`, parsed `ingress`/`egress` rules |

Metrics (CPU, memory) are **not** stored on entities. They are time series
fetched on demand (see *Metrics* below); entities carry only the latest
value for colouring.

## Edges

Edges connect two entities, usually containers or pods, or a pod and an
`External`.

```go
type Edge struct {
    ID       string    // "<from>-><to>:<port>/<proto>"
    From, To string    // entity IDs
    Port     int
    Protocol string    // tcp | udp | http | grpc | ...
    Kind     EdgeKind
    Stats    *EdgeStats // nil for edges with no observed traffic
    Severity Severity
}

type EdgeKind int
const (
    Observed  EdgeKind = iota // traffic seen (eBPF agent, Hubble)
    Allowed                   // permitted by NetworkPolicy, never seen
    Declared                  // implied by a Service selector, not yet seen
    Denied                    // traffic seen being dropped (Hubble, Cilium only)
)

type EdgeStats struct {
    Window     time.Duration
    Connections float64 // per second
    BytesTx, BytesRx float64
    Latency    *Quantiles // when available
    Errors     float64
    LastSeen   time.Time
}
```

The edge kinds are what makes the security view possible: comparing
**Allowed** with **Observed** shows paths that are permitted but never used,
which are candidates for tightening (see `04-security-model.md`).

An edge between containers is **rolled up** for higher zoom levels: pod →
pod, node → node, cluster → cluster. Roll-ups are computed in the backend so
the UI never has to sum large edge sets.

## Findings

Security and configuration issues from any source become **findings**
attached to one or more entities or edges.

```go
type Finding struct {
    ID        string     // source-specific stable ID
    Source    string     // trivy | kubescape | falco | netpol | portscan
    Category  Category   // Vulnerability | Misconfiguration | Exposure | Runtime | Policy
    Severity  Severity
    Title     string
    Detail    string     // markdown
    Targets   []string   // entity or edge IDs
    Refs      []string   // CVE IDs, control IDs, URLs
    Remediation string   // optional, markdown
    FirstSeen, LastSeen time.Time
}
```

Mapping from sources (first cut):

| Source | Becomes |
|---|---|
| Trivy `VulnerabilityReport` | one finding per CVE, aggregated per container image in the UI |
| Kubescape posture results | one finding per failed control per resource |
| Falco event | `Runtime` finding on the pod/container, time-stamped |
| NetworkPolicy analysis | `Policy` finding: no policy, allow-all, or allowed-but-unused edge |
| Port scan | `Exposure` finding on the node: open port, with what listens on it if known |

## Metrics

Time series are not part of the graph. The UI asks for them explicitly:

```
GET /api/v1/metrics?entity=<id>&metric=cpu|memory|net_rx|net_tx&range=1h&step=30s
```

The app translates this to PromQL via the gateway. Entities keep only the
latest sample so the map can colour nodes and pods by load without extra
requests.

## API shape for the UI

### Snapshot

```
GET /api/v1/graph?level=pod&scope=<entity-id>
```

Returns the entities and rolled-up edges at the requested level under the
given scope (a cluster, a node, a namespace). `scope` omitted = all clusters.

### Live updates

A WebSocket at `/api/v1/stream`. The client subscribes with the same
`level` + `scope`; the server sends diffs:

```json
{ "op": "upsert", "type": "entity", "data": { ... } }
{ "op": "delete", "type": "edge",   "id": "..." }
{ "op": "upsert", "type": "finding", "data": { ... } }
```

Each message carries a monotonically increasing `rev`. On reconnect the
client sends its last `rev`; if the server can't replay from it, it sends a
fresh snapshot.

### Details

```
GET /api/v1/entities/<id>            full entity, findings, related edges
GET /api/v1/entities/<id>/yaml       raw Kubernetes object
GET /api/v1/entities/<id>/logs       container logs (stream)
GET /api/v1/entities/<id>/events     Kubernetes events
```

These cover the Lens-style inspection features.

## Scale assumptions

- Target: 10 clusters, 100 nodes, 10,000 pods, 50,000 edges in memory
  without strain on a laptop.
- The UI never receives the whole edge set at container level for all
  clusters at once; level-of-detail plus roll-ups keep payloads bounded.
- Edges with no traffic in the last window (configurable, default 15 min)
  fade, and are dropped after a longer TTL (default 24 h) unless they are
  `Allowed` or `Declared`.

## Milestone 1 subset

Milestone 1 needs only: Cluster, Node, Pod, Container, Namespace, Workload,
Service, NetworkPolicy entities; `Declared` edges from Services; no metrics
beyond the Kubernetes metrics API if present; no findings. The types above
are designed so later milestones only add data, not reshape it.

## Open questions

- Should NetworkPolicy evaluation (computing `Allowed` edges) happen in the
  app or the gateway? App is simpler; gateway avoids per-laptop recompute
  for the future team-site mode.
- Do we persist anything locally (saved views, layout positions, finding
  acknowledgements)? Probably a small SQLite or bbolt file; decide in an ADR.
- Edge identity when pods are replaced: keep history at workload level so a
  rollout does not look like all-new traffic.
