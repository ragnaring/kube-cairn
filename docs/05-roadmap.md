# 05 — Roadmap

The order in which Cairn is built. Each milestone ships something usable on
its own. Lens parity (see `00-vision.md`) is a long-term goal reached
incrementally: each milestone also brings a slice of it.

Dates are deliberately absent; scope is the commitment, not the calendar.

## Milestone 0 — Foundations

Goal: a skeleton that builds, runs and ships, so every later milestone is
only features.

- Repository layout as in `01-architecture.md` (`cmd/`, `internal/`, `web/`,
  `charts/`).
- Go module, React + TypeScript + Vite app embedded with `go:embed`.
- CI: lint, test and build for Go and the UI; release pipeline producing
  binaries for Linux, macOS and Windows (amd64/arm64).
- Local dev loop: UI hot reload against the Go API.
- **Map prototype:** evaluate the WebGL graph library candidates
  (`03-ui-ux.md`) with ~5,000 synthetic pods; record the choice as an ADR.
- First ADRs recorded (Go backend, two-part split, no inbound ports, sensors
  not UIs, graph library).

Done when: `cairn` starts, opens the browser, and shows an empty map served
from the binary; a release produces signed artifacts.

## Milestone 1 — Map from the Kubernetes API

Goal: replace the first thing people open Lens for, with a better picture.
Nothing installed in clusters.

- Read kubeconfig, connect to N contexts concurrently.
- Informers for nodes, pods, services, namespaces, workloads,
  NetworkPolicies; live updates over WebSocket.
- Physical layout: Fleet → Cluster → Node → Pod levels with semantic zoom.
- Health status; Declared edges from Services.
- Search (⌘K), selection, detail panel (Overview, YAML read-only, Events).
- **Lens parity slice:** resource browser (tables for core kinds and CRDs),
  logs in the bottom drawer, Kubernetes events.
- Optional: CPU/memory from the Kubernetes metrics API if metrics-server is
  present (k3s ships it).

Done when: an operator with three k3s clusters sees them all on one map
within a minute of starting Cairn, and can find any pod and read its logs
without leaving the app.

## Milestone 2 — Traffic and metrics

Goal: see who talks to whom, and how loaded things are.

- Helm chart: `cairn-system` namespace, gateway, node-agent DaemonSet,
  builtin VictoriaMetrics or external Prometheus; minimal profile.
- App detects the chart and connects to the gateway via the API service
  proxy.
- Observed edges with throughput, latency where available, animated flow.
- Roll-ups for node and cluster levels.
- Resources layer: heat colouring by CPU/memory; metrics tab with charts.
- Logical layout (namespace → workload).
- Time window selector.
- Optional Hubble integration on Cilium clusters (L7, drops).
- **Lens parity slice:** exec shell, port-forward, scale/restart/delete with
  confirmation, YAML edit and apply.

Done when: installing the chart with one `helm install` makes traffic lines
appear on the map, and the operator can spot an unexpected connection.

## Milestone 3 — Security layer

Goal: answer "where am I exposed and what should I fix first?" on the map.

- Built-in misconfiguration checks (no extra components needed).
- NetworkPolicy analysis: coverage, breadth, **allowed-but-unused** paths,
  "policies not enforced" detection, suggested minimal policies.
- Exposure analysis: declared exposure + opt-in external port scan.
- Exposure layout (Internet → edge → internal).
- Integrations, each optional (install / existing / off): Trivy Operator,
  Kubescape operator, Falco via webhook.
- Findings panel, context-based prioritisation, acknowledgements.
- Chart hardening per `04-security-model.md` (own NetworkPolicies, Pod
  Security labels, signed releases).
- **Lens parity slice:** Helm releases view; RBAC viewer (who can do what).

Done when: the "firewall too open" class of incident would have been visible
on the map before it happened.

## Milestone 4 — Request tracing

Goal: follow a single request through the cluster.

- OpenTelemetry eBPF auto-instrumentation (no code changes in workloads).
- Tempo (or an external trace backend) via the gateway.
- Requests layer: animated request paths on the map; click a path to see
  the trace with timings per hop.
- Link traces to findings (e.g. a request path that passes through a
  vulnerable or exposed pod).

Done when: an operator can pick a slow request and see on the map which
container added the latency.

## Beyond

- **Team-site mode:** same binary in-cluster, OIDC + impersonation.
- Remaining **Lens parity** items, tracked as a checklist in an issue
  (extensions-style plugin points, multi-cluster bulk actions, CRD-specific
  views, etc.).
- Cilium-specific policies; cross-cluster traffic (multi-cluster meshes).
- Desktop wrapper (decide later, `03-ui-ux.md`).
- History and replay: scrub the map back in time.

## Lens parity tracker

| Lens feature | Milestone |
|---|---|
| Multi-cluster from kubeconfig | 1 |
| Resource tables (all kinds, CRDs) | 1 |
| Resource detail and YAML view | 1 |
| Logs | 1 |
| Events | 1 |
| CPU/memory metrics | 1 (metrics API) / 2 (full) |
| Exec shell | 2 |
| Port-forward | 2 |
| Edit YAML / apply | 2 |
| Scale / restart / delete | 2 |
| Helm releases | 3 |
| RBAC view | 3 |
| Extensions / plugins | Beyond |

This table is kept up to date as Lens and Cairn evolve.
