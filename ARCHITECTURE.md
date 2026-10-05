# Architecture

## Overview
```
flows/*.json ──(build-time glob)──> FileFlowRepository ──> FlowRepository interface
                                                                  │
                                    Zod + referential validation ◄┘
                                                                  │
                                 UI state (URL hash) ──> layoutFlow() ──> LayoutModel ──> SVG renderer
                                                                                         └─> Detail panel (selected step)
```
Static site, no backend. Everything is built from files in the repo.

## ADRs (decisions and why)

**ADR-1: JSON files in git, not a database (v1).**
Reasons: changes need PR review and history; flows are small and few (tens to low hundreds); static hosting is trivial; CI can validate. Revisit when: non-engineers must edit flows without git, flows exceed ~500, or we need cross-flow queries at scale. The `FlowRepository` interface is the seam: a future `HttpFlowRepository` backed by a DB swaps in without UI changes. JSON over YAML: native to the toolchain, no parser dependency, strict.

**ADR-2: Separate entity catalogue from flows.**
`flows/entities.json` defines each entity once (name, kind, owner, description). Flows reference entities by id. This prevents drift (same service named three ways), allows cross-flow views ("which flows touch `auth-service`?"), and lets us validate references.

**ADR-3: Zod is the schema source of truth; JSON Schema is generated.**
One definition yields TS types, runtime validation, and `schemas/flow.schema.json` (editors use it via `"$schema"` for autocomplete in flow files). Never maintain them separately.

**ADR-4: Sequence diagram with a parent/child step model.**
A flat list of messages loses causality ("gateway calls auth because of the login request"). Each step may have `parent`; a sync step's response is drawn after all of its descendants, producing correct nesting and activation bars. Array order is time order among siblings.

**ADR-5: Custom SVG renderer, pure layout engine.**
Diagram libraries (Mermaid, React Flow) either do not support clickable per-step payload panels with our nesting semantics or fight us on layout. Sequence layout is simple, deterministic geometry, so we own it. `layoutFlow` is pure and unit-tested; React only paints the result. A Mermaid text export can be added later as a serializer.

**ADR-6: URL-hash state.** `#/flows/user-login?step=s3` makes every view shareable and gives back/forward for free. No router dependency; a small hash parser in `state/`.

## Module contracts

### domain
- `schema.ts`: `EntitySchema`, `FlowSchema`, `StepSchema`, inferred types
- `validate.ts`: `validateFlow(flow, entities): Issue[]` where `Issue = { path: string; message: string; severity: 'error'|'warning' }`
- `migrate.ts`: `migrate(raw)` returns data at the current `schemaVersion`

### data
```ts
interface FlowRepository {
  listFlows(): Promise<FlowSummary[]>;
  getFlow(id: string): Promise<Flow>;
  listEntities(): Promise<Entity[]>;
}
```
`FileFlowRepository` uses `import.meta.glob('/flows/*.flow.json')`. Invalid files must not crash the app: the catalogue lists them under "invalid flows" with their issues (dev), and CI fails.

### layout
`layoutFlow(flow, entities, opts?) -> LayoutModel`

```ts
type LayoutModel = {
  width: number; height: number;
  lanes: { entityId: string; x: number; label: string; kind: EntityKind }[];
  rows: Row[];           // top-to-bottom drawable messages
  activations: { entityId: string; x: number; y1: number; y2: number; depth: number }[];
};
type Row = {
  stepId: string; part: 'request' | 'response';
  fromX: number; toX: number; y: number;
  label: string; sublabel?: string;   // e.g. "POST /v1/login", "200 OK"
  style: 'solid' | 'dashed' | 'async' | 'self';
};
```
Algorithm (depth-first):
1. Build a tree from `parent` links; roots ordered by array order.
2. `visit(step)`: emit request row; open activation on `step.to` (depth = open activations on that lane); `visit` children in order; emit response row (sync only); close activation.
3. `async` steps emit one `async` row and open no activation. `internal` steps (`from === to`) emit a `self` row looping on the lane.
4. Lanes follow `flow.participants` order, evenly spaced (min gap 180px). Labels wrap to 2 lines then truncate with a tooltip.
5. Row spacing constant (e.g. 56px); `y` strictly increases.

### render
`<SequenceDiagram model selectedStepId onSelect />`. Arrowheads via SVG `<marker>`. Each row is a focusable `<g role="button" tabindex=0>`; request and response rows of a step both select that step. The selected step highlights both arrows and its activation bar.

### ui
`App` → `FlowPicker` (search/filter) · `DiagramPane` · `StepDetailPanel` · `PayloadViewer` (collapsible JSON, copy button, simple tokenizer, no heavy dependency) · `Playground` (JSON editor + live diagram + issue list).

## Testing strategy
- `domain`: table-driven tests per referential rule, one failing fixture per rule
- `layout`: golden tests on small flows (flat, nested 3 deep, async, self-call, siblings) asserting row order, activation depth, y monotonicity
- `data`: every real file in `flows/` is loaded and validated in a test (the CI gate)
- `ui`: picking a flow changes the diagram; selecting a step shows its payload; URL round-trip
- e2e: one Playwright smoke: open app, choose a flow, click a step, assert payload visible

## Deployment
Static build served from any static host. CI: validate flows → lint → typecheck → test → build. A preview deploy per PR is recommended so reviewers see the rendered diagram for new flows.
