# Roadmap (ordered; pick the first unchecked task)

Each task = one PR. Tick the box in the PR. Acceptance criteria (AC) must be demonstrated.

## M0 — Scaffold
- [ ] **M0.1** Vite + React + TS strict + pnpm; ESLint, Prettier, Vitest, Testing Library; folder layout from AGENTS.md; scripts listed in AGENTS.md exist (stubs allowed for those later tasks implement). CI workflow runs the pipeline.
  - AC: `pnpm dev` shows an empty shell; `pnpm lint && pnpm typecheck && pnpm test && pnpm build` pass in CI.

## M1 — Data contract
- [ ] **M1.1** `domain/schema.ts` (per FLOW_SCHEMA.md), `gen:schema` script emitting `schemas/*.schema.json`.
  - AC: schema rejects unknown fields; example files in `flows/` and `templates/` (structurally) pass.
- [ ] **M1.2** `domain/validate.ts` implementing every referential rule and warning, returning `Issue[]` with JSON paths.
  - AC: one failing fixture + test per rule; messages name the offending id and suggest a fix.
- [ ] **M1.3** `scripts/validate-flows.ts` and `data/FileFlowRepository`; invalid files surface as issues instead of crashing.
  - AC: breaking an example flow makes `pnpm validate:flows` exit 1 with file + path + fix; a test loads every real flow.

## M2 — Layout engine
- [ ] **M2.1** `layout/layoutFlow.ts` per ARCHITECTURE (tree build, DFS, activations, lanes, rows).
  - AC: golden tests: flat flow, 3-level nesting, async, self-call, siblings under one parent; `y` strictly increasing; activation depth correct; response of a parent appears after its last descendant; deterministic output.

## M3 — Diagram renderer
- [ ] **M3.1** `render/SequenceDiagram` draws a LayoutModel: lanes, headers by kind, arrows, markers, activation bars, step numbers, labels with wrapping.
  - AC: renders `user-login` correctly (visual check recorded as a screenshot in the PR); each row focusable with aria-label; hidden ordered-list text alternative present.
- [ ] **M3.2** Selection: click/keyboard selects a step, highlights request + response + activation; ↑/↓ navigation.
  - AC: component test for click, Enter, arrows, Esc; no re-layout on selection (layout memoised, test asserts call count).

## M4 — App shell
- [ ] **M4.1** `FlowPicker` with search, tag/status/owner filters; `DiagramPane` header with title/status/owner/summary; stale/deprecated banners.
  - AC: choosing a flow swaps the diagram; search matches entity names; empty/loading/error states per UI_SPEC.
- [ ] **M4.2** URL-hash state (`#/flows/<id>?step=<id>`), back/forward, deep link on load.
  - AC: round-trip test; unknown flow id shows a helpful not-found state.

## M5 — Step detail
- [ ] **M5.1** `StepDetailPanel` + `PayloadViewer` (collapsible JSON, copy, headers table, bodyFields table, error variants, notes renderer limited to the markdown subset, sanitised).
  - AC: all fields in FLOW_SCHEMA render; copy button works and announces; no raw HTML injection possible (test with `<script>` in notes).
- [ ] **M5.2** Lane header click → entity info popover and optional "focus this entity" dimming.

## M6 — Authoring
- [ ] **M6.1** `Playground`: JSON textarea, live diagram, issue list with jump-to-line, Load template/example, Download JSON.
  - AC: invalid JSON shows parse error with line/column; valid-but-wrong shows validator issues; fixing updates diagram live (debounced 200 ms).
- [ ] **M6.2** `docs/CONTRIBUTING_FLOWS.md` verified end-to-end by adding a second real example flow using only the documented steps (e.g. `user-login-locked-account`) with zero code changes.
  - AC: that PR touches only `flows/`.

## M7 — Polish & ship
- [ ] **M7.1** Export SVG (inline styles so it renders standalone) and Copy link.
- [ ] **M7.2** Dark/light themes, responsive layouts, reduced motion, accessibility pass (axe has no serious issues).
- [ ] **M7.3** Playwright smoke; performance check with a generated 60-step/10-lane flow (< 200 ms layout+render); README with screenshots; preview deploys per PR.

## Later (do not start without sign-off)
- Entity topology overview (graph of entities, edges = flows touching them; click edge → flows list)
- Cross-flow links ("continues in flow X")
- `alt`/`opt`/`loop` fragments (schemaVersion 2)
- Importers: OpenAPI / proto / AsyncAPI → draft flows
- Mermaid/PlantUML export
- DB/API backed repository (`HttpFlowRepository`) when non-engineer editing or scale demands it
- Drift check: contract tests that compare flow examples with real OpenAPI schemas in CI
