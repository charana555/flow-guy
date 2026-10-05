# Adding or editing a flow

Audience: engineers and agents. No application code changes are required.

## Steps
1. **Check entities.** Open `flows/entities.json`. Reuse existing ids. If a participant is missing, add it there first (id, name, kind, owner).
2. **Copy the template:** `cp templates/flow.template.json flows/<your-flow-id>.flow.json`. The file name must be `<id>.flow.json`.
3. **Fill the header:** id, title, summary, owner, status (`draft` while in review), lastReviewed (today).
4. **List participants left to right** in the order you want lanes drawn (usually: initiator first, datastores last).
5. **Write steps in time order.** For each hop add `from`, `to`, `kind`, `label`, `request`, and `response` (for `sync`).
   - If the call happens *because of* another step, set `parent` to that step's id. Typical: gateway → service calls are children of the client → gateway call.
   - Use `async` for events/queue messages, `internal` for in-service work worth showing.
6. **Add realistic fake payloads.** Copy the shape from real traffic/contracts, then replace values. Redact secrets as `"<redacted>"`.
7. **Validate:** `pnpm validate:flows` (fix every error; read warnings). Optionally paste the file into the in-app Playground to view it.
8. **Open a PR** with the rendered preview link. Reviewer checks accuracy against the code or contract, not the JSON syntax.
9. Set `status` to `stable` when the owning team approves.

## Checklist before PR
- [ ] One trigger, one outcome
- [ ] Every sync step has a response; errors listed where callers must handle them
- [ ] `parent` links reflect causality
- [ ] No real secrets, tokens, emails, or customer data
- [ ] `lastReviewed` and `owner` set
- [ ] `pnpm validate:flows` passes with no new warnings

## Editing an existing flow
Keep step ids stable (deep links depend on them). Update `lastReviewed`. If you change an entity id, update every flow that references it in the same PR; the validator will list them.

## Instructions for agents generating flows from code
- Derive hops from real call sites and contracts (OpenAPI, proto, event schemas); never invent endpoints.
- If something is unknown, omit the field rather than guessing, and add a `notes` line "Unverified: <what>".
- Always emit `status: "draft"`.
