# UI spec

## Layout (desktop ≥ 1024px)
```
┌──────────────────────────────────────────────────────────────┐
│ FlowLens        [search flows…]            [Playground] [☾]  │
├───────────────┬──────────────────────────────┬───────────────┤
│ Flow list     │  Flow title · status · owner │ Step detail   │
│ (filters:     │  summary                     │ (empty state: │
│  tag, status, │  ┌────────────────────────┐  │  "Select a    │
│  owner)       │  │  sequence diagram      │  │  step…")      │
│               │  │  (scrolls both axes,   │  │               │
│               │  │   lane headers sticky) │  │ from → to     │
│               │  └────────────────────────┘  │ Request ▾     │
│               │  [Export SVG] [Copy link]    │ Response ▾    │
│               │                              │ Errors ▾      │
└───────────────┴──────────────────────────────┴───────────────┘
```
Tablet/mobile: single column; list becomes a drawer; detail panel becomes a bottom sheet. Diagram scrolls horizontally inside its own container; the page body never scrolls sideways.

## Interactions
- **Pick flow:** click in list → diagram re-renders, URL becomes `#/flows/<id>`. First flow auto-selected if URL has none.
- **Select step:** click or focus a row, then Enter/Space. URL gets `?step=<id>`. Both arrows of a sync step highlight. Detail panel updates without re-layout.
- **Keyboard:** ↑/↓ moves between steps in time order; Esc clears selection; `/` focuses search.
- **Hover:** tooltip with full label and `METHOD endpoint`.
- **Lane header click:** shows entity info (name, kind, owner, description, docs link) and dims steps not involving that entity (toggle).
- **Search:** matches flow title, summary, tags, and entity names/ids used in the flow.
- **Filters:** tag (multi), status, owner. Deprecated flows are hidden unless the status filter includes them.

## Diagram visual language
- Lane = vertical dashed line under a header chip; chip shape/icon encodes `kind` (client, gateway, service, datastore, queue, external). Do not rely on colour alone.
- `sync` request: solid arrow, label above `METHOD endpoint`. Response: dashed return arrow with status.
- `async`: solid arrow with open (hollow) head and an envelope glyph.
- `internal`: small loop on the lane.
- Activation bars: thin rectangles on lanes, offset by depth.
- Step numbers appear at the arrow start (they ARE a sequence, so numbering is meaningful).
- Selected step: thicker stroke + accent colour + subtle row background.

## Detail panel
Order: title/label → `from → to` with kind badge → protocol/endpoint chip → Request (headers table, body JSON, optional bodyFields table) → Response (same) → Error variants (collapsed list) → Notes (rendered markdown subset).
Each JSON block: collapsible, line-numbered, **Copy** button (announces "Copied" via aria-live), horizontal scroll for long lines, monospace.

## Playground
Split view: JSON editor left (plain `<textarea>` with monospace and tab handling is enough for v1), live diagram right, issue list below the editor. Issues are clickable and jump the caret to the offending line when possible. Buttons: **Load template**, **Load example**, **Download JSON**. Edits live only in memory; show a notice that nothing is saved and that real flows are added via PR into `flows/`.

## States
- Loading: skeleton for list and diagram
- Empty list/search: "No flows match. Clear filters" button
- Flow failed validation: error panel listing issues with file and path (never a blank screen)
- Stale flow (`lastReviewed` > 180 days): neutral banner "Last reviewed N days ago by <owner>"
- Deprecated: banner at top of the diagram pane

## Visual direction
Technical and calm: this is a reference tool read for long periods. Neutral surfaces, one accent colour for selection, a clear semantic palette for entity kinds that remains distinguishable in dark mode. Use a readable UI sans and a monospace for payloads; set 14–16px body text, ≥ 4.5:1 contrast. Avoid decorative gradients and animation; only animate step selection (≤150 ms) and honour `prefers-reduced-motion`.

## Accessibility checklist
Landmarks (`header`, `nav`, `main`, `aside`); every SVG row has `aria-label` such as "Step 2: api-gateway to auth-service, Verify credentials, sync"; focus ring visible on rows and buttons; list and detail panel reachable by keyboard in logical order; diagram has a text alternative: a visually hidden ordered list of steps.
