# nodegraph.lib

A zero-dependency visual node-graph editor in a single ES module. No build step,
no framework — drop `nodegraph.js` next to your page and import it. The stylesheet
is embedded and injected automatically.

Orthogonal auto-routed edges, draggable nodes with input/output param rows,
geometric groups with boundary ports, undo/redo, copy/paste, JSON persistence,
and vector PDF export. Themes are CSS custom properties.

## Quick start

```html
<div id="app" style="position:absolute; inset:0"></div>
<script type="module">
  import { createGraph } from './nodegraph.js';
  const graph = createGraph(document.getElementById('app'));

  const a = graph.addNode({ x: 80, y: 120, label: 'Source', params: [{ name: 'value' }] });
  const b = graph.addNode({ x: 380, y: 120, label: 'Sink', params: [{ name: 'input' }] });
  graph.connect({ node: a.id, param: a.params[0].id }, { node: b.id, param: b.params[0].id });
</script>
```

The container needs a size (the demo uses `position:absolute; inset:0`). Try the
bundled demo with `node server.js [port]`.

## API — `createGraph(container)`

Adds the graph UI to `container` (the `ng-graph` class is applied for theming).
Every call re-renders and records one undo step.

| Call | Description |
|---|---|
| `addNode({ x, y, label, params: [{ name }], width })` | add a node; returns it with generated ids |
| `removeNode(id)` / `setNodeLabel(id, label)` / `moveNode(id, x, y)` | delete / rename / move |
| `setNodeWidth(id, width)` / `setNodeColor(id, color)` | resize (min 100) / accent color (`null` = default) |
| `addGroup({ x, y, w, h, label, color })` | add a group box (defaults 320×220) |
| `removeGroup(id)` | remove the box only — contained nodes stay |
| `setGroupLabel` / `setGroupColor` / `setGroupFill` / `setGroupRect` | rename / tint / outline-only / reposition |
| `moveGroup(id, x, y)` / `groupMembers(id)` | move + carry members and nested groups / ids inside |
| `addGroupPort(groupId, { side, t })` / `removeGroupPort` / `getGroupPorts` | boundary ports (`side`: top/right/bottom/left, `t`: 0..1) |
| `addParam(nodeId, { name })` / `removeParam(nodeId, paramId)` / `moveParam(nodeId, paramId, toIndex)` | manage param rows (edges follow reorders) |
| `connect(source, target)` | refs `{ node, param }` or `{ group, port }`; throws on unknown port, self-connect, or duplicate |
| `removeEdge(id)` / `setEdgeLabel` / `setEdgeColor` / `setEdgeLabelT(id, t)` | edit an edge (`t` slides its label, 0..1) |
| `undo()` / `redo()` | last 50 steps; `false` when nothing to do |
| `exportJSON()` / `importJSON(text)` | pretty `{ version: 1, nodes, edges, groups }` string; import validates before touching state |
| `exportPDF(nodeIds?)` | one-page vector PDF as a `Uint8Array` (defaults to all nodes) |
| `setDark(on)` | toggle the dark palette |
| `view` / `panBy(dx, dy)` / `zoomTo(scale)` | live `{ x, y, scale }`; pan by pixels; zoom centred (0.2–4) |
| `fitView()` / `resetView()` | frame everything / back to origin at 100% |
| `getNodes()` / `getEdges()` / `getGroups()` | live model arrays (read-only by convention) |
| `addContextMenuItem(label, action)` | append an item to the context menu |
| `destroy()` | remove listeners and SVG |

## Gestures

- **Select**: click; shift-click toggles; marquee-drag empty canvas. Repeat clicks
  at one spot cycle through overlapping edges.
- **Move**: drag nodes (grid-snapped) or group bodies (carries members + nested
  groups). Auto-routed edges recompute on drop.
- **Connect**: drag port → port (either direction), or port → group border to
  create a boundary port there. Invalid connections show a toast, never a dialog.
- **Edit**: double-click a node / param / edge / group label for an inline editor
  (Enter commits, Esc cancels). Drag a param row to reorder; drag the right edge
  of a selected node to resize.
- **Bend edges**: drag a segment grip; double-click a grip to collapse a run;
  drag an edge label to slide it.
- **Pan / zoom**: middle-drag to pan, wheel to zoom at the cursor. Everything
  snaps to a 20 px grid.
- **Keyboard**: `Ctrl/Cmd + A` select all, `C`/`V` copy/paste, `Z`/`Shift+Z`/`Y`
  undo/redo, `Delete` removes the selection, `Esc` cancels.
- **Context menu**: right-click or double-click empty canvas.

## The graph model

Plain JSON-serialisable objects; the pure functions over this shape are exported
and usable without any UI:

```js
{
  nodes: [{ id, label, x, y, width?, color?, params: [{ id, name }] }],
  edges: [{ id, source, target,   // each: { node, param } or { group, port }
            label, labelT, points: [{ x, y }], color? }],
  groups: [{ id, label, x, y, w, h, color?, fill?, ports: [{ id, side, t }] }],
}
```

An empty `points` array means the router picks the route; any waypoint list is
re-bridged into a 90° polyline, so diagonals can never appear.

## Theming

Every color is a CSS custom property on `.ng-graph`; add the `dark` class (or
call `setDark(true)`) to swap the palette. Override any of them in your own
stylesheet:

`--ng-bg`, `--ng-grid`, `--ng-node-bg`, `--ng-node-border`, `--ng-header-bg`,
`--ng-text`, `--ng-muted`, `--ng-port-in`, `--ng-port-out`, `--ng-edge`,
`--ng-edge-selected`, `--ng-group-fill`, `--ng-group-border`, `--ng-group-port`.

A node's or group's own `color` overrides the theme for that element.

## Advanced exports

Besides `createGraph`, the module exports its internals — model CRUD, geometry
helpers (`GRID`, `edgePoints`, `nodeRects`, `snapToGrid`, …), the headless
`Renderer`, `buildPDF`, and `NG_CSS` (the stylesheet source) — so you can build
custom tooling without the UI. If you mutate the model directly instead of using
the API, call `invalidateRects()` afterwards so cached rectangles and routes are
refreshed.

## Files

| File | Purpose |
|---|---|
| `nodegraph.js` | the library (single ES module) |
| `index.html` | the demo |
| `server.js` | minimal static server: `node server.js [port]` |
