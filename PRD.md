# PRD — FlowLens

## Problem
Our product is made of many communicating entities. There is no shared, visual way to see how a request travels between them, or what each entity receives and returns. Knowledge lives in people's heads and scattered code. Onboarding, debugging, and design reviews are slow.

## Goal
A web app where anyone can pick a flow and immediately see: who calls whom, in what order, and the example request/response at every hop. Adding a new flow must be a data-only change that takes a developer under 15 minutes.

## Users
- **Engineers** debugging or extending an integration
- **New joiners** learning the system
- **QA / support / PMs** who need to understand behaviour without reading code

## Scope (v1)
1. Flow catalogue: list, search, filter by tag/status/owner
2. Sequence diagram generated from flow data; switching flow re-renders the diagram
3. Step detail panel: from → to, protocol, endpoint, headers, request body example, response example, error variants, notes
4. Flow data as versioned JSON files in git, validated against a schema, with a template
5. Authoring playground: paste/edit flow JSON, see live diagram and precise validation errors
6. Deep-linkable URLs (flow + step)
7. Export diagram as SVG

## Non-goals (v1)
- Live traffic, tracing, or any runtime integration (this documents designed flows, not observed ones)
- Editing flows through a form UI or persisting edits server-side
- Authentication / multi-tenant features
- Auto-generating flows from code or OpenAPI (possible later; see Roadmap)

## Success criteria
- A new flow is added by copying the template and editing JSON; CI blocks invalid flows with actionable errors
- A 60-step flow renders under 200 ms after load on a mid-range laptop
- A first-time user can find a flow and read a specific payload in under 30 seconds
- Zero flow-specific code in `src/`

## Key product decisions
- **Docs-as-data, git as the database.** Flows change at the pace of design decisions, need review, history, and ownership. Git + PR review gives all of it for free. A DB is deferred (see ARCHITECTURE ADR-1).
- **Sequence diagram as the primary view**, because the questions are about order and direction of messages. A topology overview (entities and which flows touch them) is a later addition.
