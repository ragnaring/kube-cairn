# 03 — UI and UX

How Cairn looks and behaves. The data it draws is defined in
`02-data-model.md`.

## Design goals

1. **The map is the home screen.** Lists and YAML are one click away, but
   the first thing you see is the shape of your clusters.
2. **Semantic zoom, not just scaling.** Each zoom level shows different
   information, not the same boxes smaller.
3. **Calm by default, loud when it matters.** A healthy cluster looks quiet.
   Problems and risks stand out without hunting.
4. **Fast on big clusters.** Thousands of pods and tens of thousands of edges
   stay smooth (60 fps pan/zoom on a normal laptop).
5. **Keyboard-friendly.** Operators live in terminals; search and navigation
   work without the mouse.
6. **Lens parity over time.** Everything Lens users rely on (resource
   browser, logs, exec, events, YAML edit, port-forward) has a home in the
   layout from day one, even if it ships later.

## Layout

```
┌───────────────────────────────────────────────────────────────────────┐
│ ⌘K search · cluster switcher · layers · time window       ● live     │
├─────────┬─────────────────────────────────────────────┬───────────────┤
│ Sidebar │                                             │ Detail panel  │
│         │                  MAP                        │               │
│ clusters│                                             │ selected      │
│ views   │                                             │ entity: info, │
│ resource│                                             │ findings,     │
│ browser │                                             │ metrics, logs,│
│         │                                             │ YAML, events  │
├─────────┴─────────────────────────────────────────────┴───────────────┤
│ Bottom drawer: logs / terminal / events (resizable, tabbed)           │
└───────────────────────────────────────────────────────────────────────┘
```

- **Sidebar:** clusters (with health dot), saved views, and the classic
  resource browser (tables by kind) for Lens-style work.
- **Map:** the main canvas.
- **Detail panel:** opens on selection; tabs for Overview, Findings,
  Metrics, YAML, Events.
- **Bottom drawer:** logs, exec shell and event stream, like an IDE terminal
  panel.

## Zoom levels

| Level | Shows | Edges |
|---|---|---|
| **Fleet** | One tile per cluster: size, health, worst severity, node count | Cross-cluster traffic, if any |
| **Cluster** | Nodes as blocks sized by capacity, filled by usage; server vs agent marked | Node-to-node rolled-up traffic |
| **Node** | Pods inside each node, grouped by namespace, coloured by status | Pod-to-pod |
| **Pod** | Containers, ports, images; Services in front of them | Container-to-container, with port/protocol |

Transition between levels is continuous: zooming into a node expands it in
place, neighbours stay visible at lower detail. Double-click zooms to fit an
entity; `Esc` zooms out one level.

### Alternative layouts

The physical layout (cluster → node → pod) answers "where does it run?".
Two more layouts answer other questions, toggled from the top bar:

- **Logical:** grouped by namespace → workload, ignoring nodes. Answers
  "what talks to what?".
- **Exposure:** entities arranged by distance from the outside world
  (Internet → Ingress/LoadBalancer/NodePort → services → internal). Answers
  "what is reachable from outside?".

## Visual language

- **Shapes:** clusters and nodes are rounded rectangles (containers of
  things); pods are circles; containers are small squares inside pods;
  External entities are hexagons at the map edge.
- **Status** is the fill/outline tint (neutral, amber, red).
- **Severity** is a separate badge in the corner (shield icon with colour),
  so a healthy-but-vulnerable pod is visibly both.
- **Edges:**
  - Observed: solid line, width by throughput, animated dots for live
    traffic (speed by rate).
  - Allowed but unused: dashed, faint; highlighted in red in the security
    layer.
  - Declared (Service, not yet seen): dotted.
  - Denied: red with an ✕ marker.
- **Dark and light themes**, both first-class.

## Layers

Toggles in the top bar, combinable:

- **Traffic** (milestone 2): edges and animated flow.
- **Resources** (milestone 2): heat colouring by CPU/memory.
- **Security** (milestone 3): severity badges, exposure, unused paths.
- **Requests** (milestone 4): animated request paths from traces.

With no layers on, the map shows topology and health only.

## Key interactions

- **Search (⌘K / Ctrl+K):** fuzzy search over all entities in all clusters;
  selecting a result flies the map to it.
- **Hover:** tooltip with name, status, key numbers.
- **Select:** opens the detail panel; highlights connected edges and dims the
  rest.
- **Follow a path:** from a selected pod, "show path to…" highlights the
  route to another entity through Services.
- **Time window:** traffic and metrics for the last 5 m / 1 h / 24 h, plus a
  scrubber to replay (milestone 2+).
- **Saved views:** a named combination of scope, layout, layers and filters,
  shareable as a URL.

## Lens-parity surfaces

Placed in the layout from the start, shipped incrementally:

| Feature | Where |
|---|---|
| Resource tables (all kinds, CRDs) | Sidebar → resource browser |
| YAML view / edit / apply | Detail panel → YAML |
| Logs (follow, multi-container, search) | Bottom drawer |
| Exec shell | Bottom drawer |
| Events | Detail panel and bottom drawer |
| Port-forward | Detail panel action |
| Scale / restart / delete | Detail panel actions, with confirmation |
| Helm releases | Sidebar section |

## Technology

- React + TypeScript, Vite build, embedded into the Go binary.
- Map rendering on **WebGL**. Candidates to evaluate in a prototype before
  choosing (record the decision as an ADR):
  - **Sigma.js + graphology**: proven for large graphs, good interaction.
  - **deck.gl**: very fast, flexible layers, less graph-specific.
  - **PixiJS custom renderer**: most control over nested/semantic zoom,
    most work.
- Layout computed with a hierarchical/packing algorithm for the physical
  view (nodes packed by capacity) and force-directed (e.g. ELK or d3-force
  in a Web Worker) for the logical view.
- State: a normalised client-side store fed by the snapshot + WebSocket
  diffs.

## Milestone 1 scope

- Fleet, Cluster, Node and Pod levels in the physical layout.
- Health status, Declared edges from Services.
- Search, selection, detail panel with Overview, YAML (read-only) and Events.
- Sidebar with clusters and a basic resource browser.

## Open questions

- Graph library: decide after a short prototype with ~5,000 pods of
  synthetic data.
- Do we want a desktop wrapper (e.g. Wails) later, or is `localhost` in the
  browser enough?
- Accessibility of colour coding: confirm palettes for colour-blind users.
