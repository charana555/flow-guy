# Flow data contract (schemaVersion 1)

Two kinds of files live in `flows/`:
- `entities.json` — the catalogue of participants (one file)
- `<id>.flow.json` — one flow per file. File name must equal `<flow.id>.flow.json`.

Add `"$schema": "../schemas/flow.schema.json"` (or `entities.schema.json`) at the top for editor autocomplete.

## entities.json
```json
{
  "schemaVersion": 1,
  "entities": [
    { "id": "web-app", "name": "Web App", "kind": "client", "owner": "frontend-team",
      "description": "Browser SPA used by customers." }
  ]
}
```
| field | type | required | notes |
|---|---|---|---|
| id | string, kebab-case | yes | unique, stable; renaming breaks flows |
| name | string | yes | display name |
| kind | `client` `gateway` `service` `datastore` `queue` `external` | yes | drives lane icon/shape |
| owner | string | no | team name |
| description | string | no | one or two sentences |
| docsUrl | string (URL) | no | link to runbook/repo |

## Flow file
```json
{
  "$schema": "../schemas/flow.schema.json",
  "schemaVersion": 1,
  "id": "user-login",
  "title": "User login",
  "summary": "Email/password login returning a session token.",
  "tags": ["auth"],
  "owner": "identity-team",
  "status": "stable",
  "lastReviewed": "2026-10-05",
  "participants": ["web-app", "api-gateway", "auth-service", "user-db"],
  "steps": [ /* see below */ ]
}
```
| field | type | required | notes |
|---|---|---|---|
| id | kebab-case string | yes | unique across flows; used in URLs |
| title | string | yes | ≤ 60 chars |
| summary | string | yes | what triggers it and what the outcome is |
| tags | string[] | no | lowercase; used for filtering |
| owner | string | yes | team accountable for accuracy |
| status | `draft` `stable` `deprecated` | yes | |
| lastReviewed | `YYYY-MM-DD` | yes | UI warns if older than 180 days |
| participants | entity ids[] | yes | **order = lane order, left to right**; 2–12 items; each must exist in entities.json; every participant must be used by at least one step (warning otherwise) |
| steps | Step[] | yes | 1–200; array order = time order |

## Step
```json
{
  "id": "s2",
  "parent": "s1",
  "from": "api-gateway",
  "to": "auth-service",
  "kind": "sync",
  "label": "Verify credentials",
  "request":  { "protocol": "grpc", "endpoint": "auth.v1.AuthService/Verify",
                "headers": { "x-request-id": "req_example_1" },
                "body": { "email": "user@example.com", "password": "<redacted>" } },
  "response": { "status": "OK", "body": { "userId": "usr_123", "valid": true } },
  "errors": [
    { "when": "Wrong password", "status": "UNAUTHENTICATED", "body": { "valid": false } }
  ],
  "notes": "Passwords are compared with argon2id; see auth runbook."
}
```
| field | type | required | notes |
|---|---|---|---|
| id | string | yes | unique within the flow; stable (used in deep links) |
| parent | step id | no | the step whose handling causes this one. Must appear earlier in the array and `parent.to === this.from`. Omit for top-level steps |
| from, to | entity ids | yes | must be in `participants`. `from === to` is only allowed with `kind: "internal"` |
| kind | `sync` `async` `internal` | yes | `sync`: request + response. `async`: fire-and-forget message/event, no response. `internal`: work inside one entity (e.g. "hash password") |
| label | string | yes | human sentence, ≤ 50 chars, verb first |
| request | Message | required for sync/async | |
| response | Message | required for sync; forbidden for async/internal | |
| errors | ErrorVariant[] | no | alternative responses; shown in detail panel, not drawn |
| notes | string (markdown subset: paragraphs, lists, inline code, links) | no | |

### Message
| field | type | notes |
|---|---|---|
| protocol | string | free text with conventions: `https`, `grpc`, `kafka`, `sqs`, `sql`, `redis`, `amqp`, `websocket` |
| endpoint | string | HTTP path, gRPC method, topic/queue name, or statement summary |
| method | string | HTTP verb, optional for non-HTTP |
| status | string or number | response only: `200`, `404`, `"OK"` |
| headers | object<string,string> | only meaningful headers; redact secrets |
| body | any JSON | **an example**, not a schema |
| bodyFields | `{ name, type, required?, description? }[]` | optional field-level docs for the body, shown as a table beside the example |

### ErrorVariant
`{ "when": string, "status": string|number, "body"?: any, "headers"?: object }`

## Referential rules (enforced in `domain/validate.ts`)
Errors (block CI):
1. unique flow ids; unique step ids per flow; file name matches `id`
2. every participant and every `from`/`to` exists in entities and in `participants`
3. `parent` exists, appears earlier, and `parent.to === step.from`; no cycles
4. `response` present iff `kind === "sync"`; `request` present unless `kind === "internal"`
5. `from === to` only for `internal`

Warnings (do not block, shown in playground and CI log):
- unused participant; `lastReviewed` older than 180 days; sync step without `errors` on a step whose `to` is `service` or `external`; body contains something that looks like a real secret (`/Bearer\s+[A-Za-z0-9._-]{20,}/`, JWT-shaped strings, AWS key prefixes)

## Conventions for good flows
- One flow = one trigger and one outcome. Split happy path and failure scenarios into separate flows (`user-login`, `user-login-locked-account`) rather than branching.
- 5–25 steps is the sweet spot. More than 40 means split it.
- Labels describe intent ("Verify credentials"), endpoint fields describe mechanism.
- Realistic but fake data; keep bodies under ~25 lines.

## Zod reference (implement in `src/domain/schema.ts`)
```ts
const Id = z.string().regex(/^[a-z0-9]+(-[a-z0-9]+)*$/);
const Message = z.object({
  protocol: z.string().optional(), endpoint: z.string().optional(), method: z.string().optional(),
  status: z.union([z.string(), z.number()]).optional(),
  headers: z.record(z.string()).optional(), body: z.unknown().optional(),
  bodyFields: z.array(z.object({ name: z.string(), type: z.string(),
    required: z.boolean().optional(), description: z.string().optional() })).optional(),
}).strict();
const Step = z.object({
  id: z.string().min(1), parent: z.string().optional(),
  from: Id, to: Id, kind: z.enum(['sync','async','internal']),
  label: z.string().max(50), request: Message.optional(), response: Message.optional(),
  errors: z.array(z.object({ when: z.string(), status: z.union([z.string(), z.number()]),
    body: z.unknown().optional(), headers: z.record(z.string()).optional() })).optional(),
  notes: z.string().optional(),
}).strict();
export const FlowSchema = z.object({
  $schema: z.string().optional(), schemaVersion: z.literal(1), id: Id,
  title: z.string().max(60), summary: z.string(), tags: z.array(z.string()).optional(),
  owner: z.string(), status: z.enum(['draft','stable','deprecated']),
  lastReviewed: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  participants: z.array(Id).min(2).max(12), steps: z.array(Step).min(1).max(200),
}).strict();
```
`.strict()` is deliberate: typos in field names must fail loudly.

## Evolution
Additive optional fields do not need a version bump. Anything renamed/removed/required bumps `schemaVersion` with a migration. Candidates for v2 (do not build now): `alt`/`opt`/`loop` fragments, `links` between flows, per-step `latencyBudgetMs`, OpenAPI `$ref` for bodies.
