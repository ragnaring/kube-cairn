# 01 — Architecture

This document describes the high-level structure of Cairn: its parts, where
they run, how they communicate and which external tools they rely on.
Data structures are in `02-data-model.md`; the UI in `03-ui-ux.md`; security
of Cairn itself in `04-security-model.md`.

## Overview

Cairn has two parts:

- **A. In-cluster components** — a Helm chart installed in each cluster.
- **B. The Cairn app** — a single binary on the operator's machine with an
  embedded web UI.

```
 Operator machine                         Cluster (×N)
┌──────────────────────────┐            ┌──────────────────────────────────────┐
│ Cairn app (Go binary)    │            │ kube-apiserver                       │
│ ┌──────────────────────┐ │  kubecfg   │   │                                  │
│ │ Cluster connectors ×N├─┼───────────►│   ├─ watch: nodes, pods, svc, netpol │
│ └──────────┬───────────┘ │  creds     │   ├─ watch: Trivy / Kubescape CRDs   │
│ ┌──────────▼───────────┐ │            │   └─ service proxy ──┐               │
│ │ Aggregator / graph   │ │            │                      ▼               │
│ └──────────┬───────────┘ │            │  ┌─────────────── cairn-gateway ──┐  │
│ ┌──────────▼───────────┐ │            │  │ aggregates metrics, Falco      │  │
│ │ HTTP + WS API        │ │            │  │ events, connection graph       │  │
│ └──────────┬───────────┘ │            │  └──┬───────────┬──────────┬──────┘  │
│ ┌──────────▼───────────┐ │            │     │           │          │         │
│ │ Embedded React UI    │ │            │  VictoriaMetrics  Falco ──►webhook   │
│ └──────────────────────┘ │            │  or external Prom                    │
│ Port scanner (external)──┼──► nodes   │     ▲                                │
└──────────────────────────┘            │  node-agent DaemonSet (eBPF)         │
        ▲ browser on localhost          └──────────────────────────────────────┘
```

## Why two parts

Some work must run inside the cluster, continuously:

- eBPF agents need to run on every node.
- Metrics must be collected even when the operator's laptop is off, otherwise
  history has gaps.
- Trivy, Kubescape and Falco operate in-cluster by design.

The UI, on the other hand, should use the operator's own credentials and
needs no extra exposure. Running it locally keeps the cluster's attack
surface unchanged.

## A. In-cluster components (Helm chart)

| Component | Role | Required |
|---|---|---|
| `cairn-gateway` | Small Go service. Aggregates the cluster's metrics, connection data and Falco events behind one API. | From milestone 2 |
| `node-agent` (DaemonSet) | eBPF agent (e.g. coroot-node-agent) measuring TCP connections between containers, exposed as Prometheus metrics. Works with any CNI. | From milestone 2 |
| Metrics store | `builtin`: VictoriaMetrics single-node. `external`: the user's own Prometheus. | From milestone 2 |
| Trivy Operator | Image vulnerability scanning; writes `VulnerabilityReport` and related CRDs. | Optional |
| Kubescape operator | Posture/misconfiguration findings as CRDs. | Optional |
| Falco | Runtime detection; sends events to the gateway via webhook. | Optional |
| Hubble (Cilium only) | Optional L7 flow data and policy drop events. | Optional |

Each optional security component has three modes: **install ours**,
**use existing** (point at an existing installation), or **off**.

### Deployment layout

- Everything Cairn installs lives in **one namespace, `cairn-system`**, as
  **one Helm release**. Uninstalling is one command.
- Components stay **separate pods**; we do not bundle them into one
  container. Reasons:
  - **Least privilege.** The eBPF agent and Falco need privileged access on
    every node; the gateway and metrics store need none. Bundling would give
    everything root on every node.
  - **Isolation.** A crash or memory leak in one tool does not take down the
    others, and each gets its own resource limits.
  - **Upgrades.** Each upstream tool keeps its own image and release cycle;
    we pin versions in the chart instead of rebuilding a combined image.
  - **"Use existing" mode.** A user who already runs Prometheus, Trivy or
    Falco simply turns ours off.
- Pod count is not the real cost; memory and privileges are. The chart
  documents the footprint of each component and offers a **minimal profile**
  (gateway + node-agent + VictoriaMetrics).

### The gateway

- Read-mostly service with a narrow, versioned HTTP API.
- Queries the metrics store (PromQL) and returns pre-shaped data: the
  connection graph for a time window, per-workload resource usage.
- Receives Falco webhooks and keeps a bounded in-memory/ring buffer of recent
  events (persistent storage is a later decision).
- Exposes a `ClusterIP` service only. No `NodePort`, `LoadBalancer` or
  Ingress.
- Minimal RBAC: read access to what it needs, nothing cluster-admin.

## B. The Cairn app

A single static Go binary that:

1. Reads the user's kubeconfig (all contexts, or a selected subset).
2. Opens one **cluster connector** per context, connecting to N clusters
   concurrently.
3. Serves the embedded React UI and an HTTP + WebSocket API on `localhost`.
4. Optionally runs the **external port scan** against node addresses, giving
   the outside-in view that in-cluster tools cannot provide.

Later, the same binary can run **in-cluster as a shared team site**, with
OIDC login and its own service account in place of a kubeconfig.

### Internal structure

```
cmd/cairn            entry point, flags, config
internal/cluster     one connector per kube context: client-go informers,
                     CRD watches, gateway client via API proxy
internal/graph       the aggregated model: clusters → nodes → pods →
                     containers, edges (connections), findings
internal/sources     adapters: kube, metrics, trivy, kubescape, falco,
                     hubble, portscan — each implements a common interface
internal/api         HTTP + WebSocket API for the UI
internal/scan        external port scanner
web/                 React + TypeScript UI, embedded with go:embed
charts/cairn         Helm chart for in-cluster components
```

### Cluster connector

- Uses `client-go` **informers** (list + watch) for nodes, pods, services,
  endpoints/EndpointSlices, NetworkPolicies and namespaces. Changes stream
  into the graph in real time.
- Watches Trivy and Kubescape CRDs when present, using their upstream Go types.
- Reaches `cairn-gateway` through the **Kubernetes API service proxy**
  (`/api/v1/namespaces/<ns>/services/cairn-gateway:http/proxy/...`), the
  same mechanism `kubectl proxy` uses. Port-forward is the fallback if a
  feature needs streaming that the proxy handles poorly.
- Detects capabilities on connect (is the chart installed? which CNI? which
  CRDs exist?) and degrades gracefully: the map always works from the
  Kubernetes API alone.

### Source adapters

Every data source implements one small interface (sketch):

```go
type Source interface {
    Name() string
    Available(ctx context.Context) bool
    Run(ctx context.Context, sink graph.Sink) error // push updates until ctx is done
}
```

This keeps each integration isolated, testable, and individually optional.

### Aggregator / graph

- Holds the in-memory model for all connected clusters.
- Merges updates from all sources into nodes, edges and findings.
- Publishes **diffs** to subscribed UI clients over WebSocket, so the UI does
  not refetch the world on every change.
- The UI requests the level of detail it is showing (cluster, node, pod,
  container), and the API returns or streams only that slice.

## Data sources

| Source | Data | Via |
|---|---|---|
| Kubernetes API (×N) | nodes, pods, containers, services, NetworkPolicies | client-go watches |
| Trivy Operator | `VulnerabilityReport` etc. | CRD watches |
| Kubescape operator | posture findings | CRD watches |
| Falco | runtime events | webhook → gateway |
| VictoriaMetrics / Prometheus | CPU, memory, network, connection metrics | gateway (PromQL) |
| Hubble (optional) | L7 flows, policy drops | gateway |
| Port scan | open ports on nodes from outside | local app |
| Later: OTel eBPF + Tempo | per-request traces | gateway |

## Configuration

Configured per cluster (Helm values) and per app (config file / flags).
Key switches:

| Setting | Values | Default |
|---|---|---|
| `cni` | `flannel` \| `cilium` \| `auto` | `auto` |
| `metrics.mode` | `builtin` \| `external` | `builtin` |
| `metrics.external.url` | Prometheus URL | — |
| `trivy.mode` / `kubescape.mode` / `falco.mode` | `install` \| `existing` \| `off` | `off` |
| `hubble.enabled` | bool (Cilium only) | `false` |

## Technology choices

- **Backend: Go.** First-class Kubernetes ecosystem (`client-go`, Helm SDK,
  upstream Go types for Trivy, Kubescape, Falco and VictoriaMetrics), single
  static binary, small in-cluster footprint.
- **Frontend: React + TypeScript**, with a WebGL-capable graph library so the
  map stays fast on large clusters. The specific library is decided in
  `03-ui-ux.md` / an ADR.
- Considered: Rust (kube-rs, Tauri) as a viable alternative. Rejected: Java
  (JVM footprint, distribution) and Node (weaker Kubernetes client).

## Security of the architecture (summary)

Detailed in `04-security-model.md`. In short:

- No new inbound ports into any cluster.
- The local app acts with exactly the operator's own kubeconfig permissions.
- The UI binds to `localhost` only by default.
- In-cluster components run with least-privilege RBAC and are individually
  optional.

## Mapping to the roadmap

| Milestone | Parts involved |
|---|---|
| 1. Map from Kubernetes API only | App: cluster connector (kube source), graph, API, UI. No chart. |
| 2. Metrics + eBPF | Chart: gateway, node-agent, metrics store. App: metrics source, traffic edges. |
| 3. Security layer | Trivy, Kubescape, Falco, NetworkPolicy analysis, port scan. |
| 4. Request tracing | OTel eBPF auto-instrumentation + Tempo via gateway; animated flows. |

## Open questions

- Graph library for the UI (Sigma.js, Cosmograph, deck.gl, or a custom
  PixiJS layer).
- Whether the gateway needs persistent storage for Falco events and
  findings history, or if the metrics store and CRDs are enough.
- Service proxy vs. port-forward for streaming endpoints.
- How NetworkPolicy "allowed but never used" analysis combines policy
  evaluation with observed connections (belongs in `04-security-model.md`).
