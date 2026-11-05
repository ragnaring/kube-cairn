# 04 — Security model

This document covers two things:

1. **Security of Cairn itself**: what it can access, what it exposes, and how
   we keep it from becoming the weakest link.
2. **The security layer Cairn shows**: what findings it produces, from which
   sources, and how they are analysed and prioritised.

The data shapes (entities, edges, findings) are in `02-data-model.md`.

---

## Part 1: Security of Cairn itself

### Threat model

Cairn sees everything in a cluster and, through its in-cluster agents, runs
privileged code on every node. That makes it a valuable target. We assume:

- The **operator's machine** is trusted, but may run other software and other
  users' browsers.
- The **cluster network** may contain compromised workloads.
- **Upstream tools** we ship (VictoriaMetrics, coroot-node-agent, Trivy,
  Kubescape, Falco) can have vulnerabilities of their own.
- The **Cairn app** may be run by people with very different RBAC rights.

Primary risks:

| Risk | Example |
|---|---|
| Privilege escalation through Cairn | A user with read-only kubeconfig uses Cairn to do something they could not do with `kubectl`. |
| Exposure of the local UI | Another site in the browser, or another user on the machine, reaches the `localhost` API. |
| New attack surface in the cluster | A compromised pod talks to the gateway or the metrics store and reads or poisons data. |
| Data leakage | Secrets, env vars or logs shown or cached where they should not be. |
| Supply chain | A tampered Cairn binary, image or chart. |

### Principles

1. **No privilege beyond the operator's own.** The local app uses the
   user's kubeconfig credentials for every Kubernetes request. It never
   holds a more powerful service account. If the user cannot `get secrets`,
   neither can Cairn on their behalf.
2. **No new inbound ports.** In-cluster components are `ClusterIP` only.
   The app reaches them through the Kubernetes API service proxy, which
   requires the operator to have `services/proxy` permission on that
   service. Access to Cairn's in-cluster data is therefore governed by
   Kubernetes RBAC.
3. **Least privilege per component.** Each in-cluster component has its own
   service account and RBAC, scoped to what it needs.
4. **Read by default, write on purpose.** Mutating actions (delete, scale,
   edit YAML, exec) are explicit, confirmed in the UI, and always performed
   with the operator's credentials.
5. **Every optional component is off-able.** Users who do not want privileged
   agents in a cluster can run the topology view with no chart at all.

### The local app

- Binds to `127.0.0.1` by default. Binding to other interfaces requires an
  explicit flag and enables authentication.
- **Session token:** on start the app generates a random token and opens the
  browser with it. The UI exchanges it for an `HttpOnly`, `SameSite=Strict`
  cookie. API calls without it are rejected. This stops other local users
  and other browser tabs from using the API.
- **Origin / Host checks** on every HTTP and WebSocket request, to block
  DNS rebinding and cross-site requests.
- Strict **Content Security Policy**; the UI loads nothing from the network.
- Kubeconfig credentials stay in memory in the Go process and are never sent
  to the browser.
- Secrets are never displayed by default. Revealing a Secret value is a
  per-click action, and only if the operator's RBAC allows reading it.

### In-cluster components

| Component | Privileges | Notes |
|---|---|---|
| `cairn-gateway` | Unprivileged, non-root, read-only root FS. RBAC: read pods/services/nodes, read Cairn CRDs if any. | No access to Secrets. |
| `node-agent` | Privileged (eBPF, host PID/network). No Kubernetes write rights. | Largest risk surface; pinned upstream version, own namespace, documented. |
| VictoriaMetrics | Unprivileged, non-root. No Kubernetes RBAC. | Reached only by the gateway and node-agent. |
| Falco | Privileged (kernel probes / eBPF). | Upstream chart defaults, webhook to gateway only. |
| Trivy / Kubescape | Upstream defaults, which include broad read access. | Documented clearly; off by default. |

- The chart ships **NetworkPolicies for its own namespace**: only the
  node-agents may write to the metrics store, only the gateway may read it,
  and only the Kubernetes API server (via the service proxy) and Falco may
  reach the gateway. Cairn should pass its own security checks.
- The Falco webhook is authenticated with a shared secret generated at
  install time.
- Pod Security: `cairn-system` is labelled for the strictest Pod Security
  level each component allows; privileged components are isolated and
  listed.

### Team-site mode (later)

When the binary runs in-cluster as a shared site:

- Login via **OIDC**. The user's identity is forwarded to the Kubernetes API
  with **impersonation**, so each user still sees only what their own RBAC
  allows. The site's service account has only `impersonate` rights for the
  configured groups.
- Exposed via the user's own Ingress; Cairn adds none by default.

### Supply chain

- Release binaries and images are **signed** (cosign) and ship with an
  **SBOM**.
- Upstream images in the chart are pinned by digest.
- Cairn's own images are scanned with Trivy in CI before release.

---

## Part 2: The security layer

### Categories

| Category | Question it answers | Sources |
|---|---|---|
| **Exposure** | What is reachable from outside, and should it be? | Port scan, Services (NodePort/LoadBalancer), Ingress, hostNetwork/hostPort pods |
| **Network policy** | Is traffic inside the cluster restricted as much as it could be? | NetworkPolicies + observed traffic |
| **Vulnerability** | Do running images contain known CVEs? | Trivy Operator |
| **Misconfiguration** | Are workloads and the cluster configured safely? | Kubescape; built-in checks |
| **Runtime** | Is something suspicious happening right now? | Falco |

### Exposure analysis

The question "what can be reached from outside?" combines two views:

- **Inside-out (declared):** Services of type NodePort / LoadBalancer,
  Ingress rules, pods with `hostNetwork` or `hostPort`, and k3s specifics
  (e.g. ServiceLB/Klipper, Traefik). This tells us what the cluster *intends*
  to expose.
- **Outside-in (observed):** the port scan from the operator's machine (or a
  chosen vantage point) against node addresses. This tells us what is
  *actually* open, including things outside Kubernetes (SSH, the k3s API on
  6443, etcd, kubelet 10250, node exporters).

Findings:

- Port open from outside but not declared by any Service/Ingress → **High**
  (something outside Kubernetes, or a firewall gap).
- Sensitive ports reachable (kubelet, etcd, API server from untrusted
  networks) → **Critical**.
- Declared exposure that is reachable → informational, shown on the map so
  the operator can confirm it is intended.

The scan is **opt-in per cluster**, rate-limited, and runs only against the
cluster's own node addresses, never against arbitrary targets.

### NetworkPolicy analysis

This is Cairn's most distinctive analysis.

1. **Coverage:** for each pod, is it selected by any ingress / egress policy?
   Pods with no policy are effectively allow-all → finding.
2. **Breadth:** policies that allow from all namespaces, all pods, or
   `0.0.0.0/0` → finding, severity by what the target is.
3. **Allowed vs. observed:** evaluate policies to compute the set of
   **Allowed** edges between workloads, then compare with **Observed**
   traffic over a window (default 7 days):
   - Allowed and observed → fine.
   - **Allowed but never observed** → candidate for tightening; shown as a
     dashed red edge in the security layer.
   - Observed but not allowed → only possible if the CNI does not enforce
     policies (e.g. plain flannel without a policy controller) → **High**
     finding on the cluster: "policies are not enforced".
4. **Suggestions:** generate a minimal NetworkPolicy that permits exactly the
   observed traffic, for the operator to review and apply. Cairn never
   applies it automatically.

Evaluation follows Kubernetes NetworkPolicy semantics (additive allow,
default allow when no policy selects a pod). Cilium-specific policies
(`CiliumNetworkPolicy`) are a later addition.

k3s note: k3s ships an embedded network policy controller with flannel, so
policies are usually enforced; Cairn detects and reports when they are not.

### Vulnerabilities

- Read Trivy `VulnerabilityReport` CRDs per container image.
- Aggregate per image in the UI (one image used by many pods is one problem,
  shown on all of them).
- Prioritise by severity **and context**: a Critical CVE in a pod that is
  exposed to the Internet ranks above the same CVE in an isolated batch job.

### Misconfiguration

- Kubescape control results, mapped to the affected resources.
- A small set of **built-in checks** so something useful shows even without
  Kubescape: privileged containers, running as root, `hostPath` mounts,
  missing resource limits, `latest` tag, default service account with
  token automount, Secrets as env vars.

### Runtime

- Falco events arrive at the gateway via webhook, are mapped to
  pod/container, and appear as time-stamped findings that pulse on the map.
- Runtime findings expire from the "active" view after a configurable time
  but remain in history.

### Prioritisation

Each finding has a base severity from its source. Cairn adjusts the
displayed priority with context from the graph:

- **Exposure:** reachable from outside → raise.
- **Blast radius:** many observed connections or privileged pod → raise.
- **Isolation:** covered by a tight NetworkPolicy and not exposed → lower.

The rules are simple and explained in the UI ("raised because this pod is
reachable from the Internet"), never a black-box score.

### Acknowledging findings

Operators can mark a finding as **accepted risk** or **false positive** with
a note. Acknowledgements are stored locally (and in team-site mode, shared)
and hide the finding from the default view without deleting it.

## Open questions

- Where acknowledgements live in team-site mode: a CRD in the cluster is
  attractive (GitOps-friendly, RBAC-controlled).
- Whether the port scan should also be able to run from inside the cluster
  (a Job) to compare inside vs. outside reachability.
- How far to go with Cilium-specific policies and Hubble policy verdicts.
