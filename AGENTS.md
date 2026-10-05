# AGENTS.md — FlowLens

FlowLens is a web app that visualises how requests flow between the entities (clients, gateways, services, datastores, queues, external systems) of our product. A user picks a **flow**, and the app renders a sequence diagram. Clicking any step shows the request and response example structures for that hop.

> "FlowLens" is a working name. Rename freely, but do it in one PR.

**Status:** greenfield. Read the docs below in order before writing code.

## Read first
1. `docs/PRD.md` – what and why, scope, non-goals
2. `docs/ARCHITECTURE.md` – decisions (with reasons), module boundaries, layout algorithm
3. `docs/FLOW_SCHEMA.md` – the data contract. This is the most important file.
4. `docs/UI_SPEC.md` – screens, interactions, states, accessibility
5. `docs/ROADMAP.md` – ordered tasks with acceptance criteria. Pick the next unchecked task.
6. `docs/CONTRIBUTING_FLOWS.md` – how to add or edit a flow
7. Examples: `flows/entities.json`, `flows/user-login.flow.json`, `templates/flow.template.json`

## Core principle
**Flows are data. The app is a renderer.** No flow, entity, or payload may be hard-coded in a component. If a new flow requires a code change, the design is wrong; fix the schema or renderer instead.

## Stack (fixed; do not swap without asking)
- TypeScript (`strict: true`), React 18, Vite
- Zod is the single source of truth for the schema. JSON Schema is generated from it.
- Custom SVG renderer for the diagram (no diagram library in v1; see ARCHITECTURE §Rendering)
- Vitest + Testing Library for unit/component tests, Playwright for one smoke e2e
- Plain CSS with CSS variables (no CSS framework). Light and dark themes.
- Package manager: pnpm

## Commands (created in roadmap task M0)
```
pnpm install
pnpm dev               # local app
pnpm validate:flows    # validates every file in /flows (also run in CI)
pnpm gen:schema        # regenerates schemas/*.schema.json from Zod
pnpm test              # unit + component
pnpm test:e2e          # Playwright smoke
pnpm lint && pnpm typecheck
pnpm build
```
Before declaring any task done, `pnpm validate:flows && pnpm lint && pnpm typecheck && pnpm test` must pass.

## Layout
```
flows/            data: entities.json + *.flow.json   (the product's real content)
templates/        flow.template.json (copy to start a new flow)
schemas/          GENERATED JSON Schemas. Never hand-edit.
src/
  domain/         Zod schemas, types, referential validation. No React, no DOM.
  data/           loaders (FlowRepository interface + FileFlowRepository using import.meta.glob)
  layout/         pure function layoutFlow(flow, entities) -> LayoutModel. No React, no DOM.
  render/         React components that draw a LayoutModel as SVG
  ui/             app shell, flow picker, detail panel, payload viewer, authoring playground
  state/          URL-synced selection state
scripts/          validate-flows.ts, gen-schema.ts
docs/
```
Dependency direction: `ui -> render -> layout -> domain`, and `ui -> data -> domain`. `domain` and `layout` must never import from React, `ui`, `render`, or `data`.

## Rules
- **Validation is two-layer.** (1) structural via Zod; (2) referential in `domain/validate.ts` (participants exist, step `from/to` are participants, `parent` rules, unique ids). Both run at load time AND in `pnpm validate:flows`. Errors must name the file, the JSON path, and the fix.
- **Layout is pure and tested.** All diagram geometry decisions live in `layout/`. Renderer components only map the `LayoutModel` to SVG. Add a unit test for every layout rule.
- **Payload examples are illustrative, never real.** No real tokens, emails, IDs, or customer data in `flows/`. Use obviously fake values (`"tok_example_123"`, `"user@example.com"`). Secrets in headers must be written as `"<redacted>"`.
- **Schema changes are breaking changes.** Bump `schemaVersion`, add a migration in `src/domain/migrate.ts`, update `docs/FLOW_SCHEMA.md`, templates, and examples in the same PR.
- **Repository abstraction.** UI talks only to `FlowRepository`. A future DB/API backend must be addable by writing a new implementation, without touching `ui/`.
- **State in the URL.** Selected flow, selected step, and view options are encoded in the URL hash so any view is linkable.
- **Accessibility floor:** keyboard navigable steps (arrow keys, Enter), visible focus, `aria-label` on SVG elements, colour is never the only signal (arrow style also encodes kind), respects `prefers-reduced-motion` and `prefers-color-scheme`.
- **Performance floor:** a flow with 60 steps and 10 participants renders and interacts smoothly; no re-layout on step selection.
- Keep dependencies minimal. Adding one needs a one-line justification in the PR description.

## Boundaries
**Always:** follow the roadmap order; write tests with the code; update docs when behaviour or schema changes; keep PRs to one roadmap task.
**Ask first:** adding a dependency; changing the stack; changing the schema; adding a backend; deleting or renaming existing flows.
**Never:** hard-code flow data in components; hand-edit `schemas/`; commit real secrets/PII; disable lint or type rules to pass checks; make network calls to third-party services at runtime.

## Definition of done (every task)
- Acceptance criteria in `docs/ROADMAP.md` for the task are met and demonstrated by tests or a described manual check
- All commands in "Commands" pass
- New/changed behaviour is documented
- The roadmap checkbox is ticked in the same PR

## Glossary
- **Entity:** a system participant (e.g. `web-app`, `auth-service`). Defined once in `flows/entities.json`, reused by all flows.
- **Flow:** one end-to-end scenario (e.g. user login) as an ordered list of steps.
- **Step:** one message from one entity to another. Sync steps have a request and a response; async steps are request-only.
- **Parent step:** the step whose handling caused this step (drives nesting and activation bars).
- **LayoutModel:** the pure data output of `layoutFlow`; coordinates for lanes, arrows, and activation bars.
