# 00 — Vision

## One sentence

Cairn is a tool that shows you what is running in your Kubernetes clusters,
how it talks to each other, and where it is exposed, on one beautiful,
zoomable map.

## The problem

People who run small and mid-sized clusters (k3s in particular) have three
recurring needs that today's tools serve poorly:

1. **Seeing the cluster.** Lens and similar tools are good at lists and YAML,
   but weak at showing the shape of a system: which workloads live where,
   what depends on what, and how much of the hardware is in use.
2. **Seeing the traffic.** It is hard to answer "who talks to whom?" and
   "which path does a request take through the cluster?" without assembling
   a service mesh, a tracing stack and several dashboards by hand.
3. **Seeing the risk.** Real incidents come from ordinary mistakes: a firewall
   that is too open, a missing NetworkPolicy, an image with known CVEs, a pod
   running as root. The information exists in separate scanners, but nothing
   puts it where the operator is already looking.

## What Cairn is

A standalone application that:

- Does what operators use Lens for today (browse and inspect resources across
  many clusters, logs, events, basic actions).
- Adds a **zoomable graphical map**: all clusters → nodes (server/agent,
  resources) → pods → containers, with smooth semantic zoom between levels.
- Draws **connections between containers** with live traffic on them.
- Shows **how requests travel** through the cluster, container to container.
- Overlays a **security layer** on the same map: open ports and overly
  permissive firewalls, missing or overly broad NetworkPolicies (including
  paths that are allowed but never used), image CVEs, misconfigurations and
  runtime alerts.

## Who it is for

- **Primary:** an operator or small team running one to a handful of k3s
  clusters, often on modest hardware, without a dedicated platform team.
- **Secondary:** anyone running conformant Kubernetes who wants a visual,
  security-aware view. Nothing in Cairn is k3s-only.

Cairn is built to be **distributed to other people**, not just for internal
use. That shapes everything: sensible defaults, everything configurable,
small footprint, no assumptions about the user's CNI or monitoring stack.

## Principles

1. **Our own UI throughout.** We never embed or skin another tool's UI.
   Other tools are headless *sensors*; we consume their data and APIs.
2. **Aggregate, don't reinvent.** Most of the intelligence (scanning,
   eBPF measurement, metrics storage, posture checks) comes from proven
   open-source tools. Our value is the aggregation layer and, above all, the
   visualisation.
3. **Talk to Kubernetes directly.** Use the Kubernetes API through client
   libraries and watches, never by shelling out to `kubectl`.
4. **No new open ports.** Cairn adds no inbound exposure to a cluster. The
   local app reaches in-cluster components through the Kubernetes API proxy,
   with the operator's own credentials.
5. **Configurable, with good defaults.** CNI, metrics backend and each
   security component can be ours, the user's existing one, or off.
6. **Useful from minute one.** The first milestone needs nothing installed in
   the cluster: point Cairn at a kubeconfig and see the map.
7. **Light.** The in-cluster footprint must suit a Raspberry Pi-class k3s
   node.

## Non-goals (for now)

- Replacing a full observability platform (long-term log storage, alerting
  rules, SLO management).
- Being a policy engine or admission controller. Cairn *shows* risk and may
  suggest fixes; it does not enforce.
- Managing cluster lifecycle (provisioning, upgrades).
- A hosted SaaS. Cairn runs on the operator's machine, and later optionally
  inside the cluster as a shared team site.

## What success looks like

- An operator opens Cairn and, within a minute, sees all their clusters on one
  map without installing anything.
- After installing the Helm chart, they can see which containers talk to each
  other and spot a connection they did not expect.
- They can answer "what in this cluster is reachable from the outside, and
  should it be?" by looking at the map, not by running five tools.
- They stop opening Lens.

## Name

A *cairn* is a pile of stones that marks the way. Cairn shows the paths
requests take, and, like a cairn on a mountain, warns where the ground is
unsafe.
