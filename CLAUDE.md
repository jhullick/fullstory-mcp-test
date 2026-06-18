# Fullstory MCP — Analysis Playbook & Gotchas

Condensed, hard-won learnings for doing behavioral analysis against this
Fullstory org via the `fullstory` MCP server. Read this before building
metrics/funnels/segments — it will save you a lot of failed calls.

The event/property model itself lives in Notion ("MARBLE Events Taxonomy");
this doc is only about *how* to query it reliably in Fullstory.

---

## TL;DR rules

1. **NEVER use Fullstory "defined events."** Always query the raw **custom
   event** by name. Defined events are mis-attributed / mostly empty here.
2. **The NL builders mangle custom-variable names and pick wrong events.**
   Treat `build_metric` / `build_funnel` output as a *draft*; verify and fix
   the definition, then run via the `compute_metric` `metric_definition`
   escape hatch.
3. **`build_funnel` cannot carry an event-property filter** — it errors. Use
   **segments** (which can) for anything with a property condition.
4. **Authenticate first**: `mcp__fullstory__authenticate` → user opens URL →
   paste callback URL into `complete_authentication`. (Remote session: the
   browser will show a localhost connection error; that's expected.)

---

## 1. Never use "defined events" — use raw custom events

When you `discover_org_context` for an event you'll often see BOTH a custom
variable/raw event AND one or more **defined events** (e.g.
`booking_confirmed` `Ikz4MZ4TzFMa`, `milano:booking_confirmed`
`g8wL5BE3ML8S`). The NL builders love to pick a defined event by `withId`.

**Don't.** In this org the defined events return **0 / null** or are
mis-attributed, while the raw datalayer custom event holds the real data and
the event properties. Examples observed (last 30 days):

| Representation | Users | Notes |
|---|---|---|
| custom `booking_confirmed` | 5,114 | ✅ use this |
| defined `milano:booking_confirmed` | 5,442 | inconsistent; no `booking_lead_days` |
| defined `booking_confirmed` (`Ikz4MZ4TzFMa`) | ~0 in funnels | ❌ empty |

A funnel that mapped step 2 to the defined `booking_confirmed` returned **0
conversions** — purely an artifact of the wrong event. Event properties
(e.g. `booking_lead_days`) are attached to the **raw custom event**, not the
defined ones.

**Always reference events like this:**
```json
{ "primary": { "custom": { "named": { "name": "booking_confirmed" } } } }
```
Never `{ "primary": { "definedEvent": { "withId": { "value": "..." } } } }`.

---

## 2. The NL builders mangle custom-variable names (type suffixes)

`build_metric` rewrites property names by appending a **type suffix** that
does **not** exist in this org's schema:

- boolean → appends `_bool`  → `booking_availability_requested_slot_bool`
  → **validation error** "unknown custom variable ... in scope event".
- numeric → appends `_real`  → `booking_lead_days_real`.

Confusingly, the *correct* internal name depends on **how** the property is
used (see §4 and §5). The discover tool shows the bare name
(`booking_availability_requested_slot`, `booking_lead_days`).

**Workflow:** let a builder produce a draft definition, then hand-correct the
event (→ raw custom) and the property name, and run it via the escape hatch:
```
compute_metric(metric_definition = { ...corrected... })
```

---

## 3. Property filters: funnels CAN'T, segments CAN

- `build_funnel` with any event-property condition fails with
  `failed to read built funnel from session`. There is **no** funnel
  `definition` escape hatch (`compute_funnel` takes only `funnel_id`).
- `build_segment` encodes property conditions correctly. So:
  **build the property-filtered population as a segment**, then count it by
  attaching the segment to a unique-user metric (see §6).

---

## 4. Correct BOOLEAN property-filter schema (from build_segment)

The working shape for a boolean event-property condition (use the **bare**
name, no `_bool`):
```json
"deps": [
  { "customEventPropertyBool": { "equals": { "name": "booking_availability_requested_slot" } } }
]
```
- `value: false` is **omitted** (proto3 default) → the example above means
  the property **= false**.
- For **= true**, include it: `{ "equals": { "name": "...", "value": true } }`.
- ❌ `{ "fieldName": "...", "value": false }` and `equals: <bool>` do NOT work
  (`unspecified error` / proto parse error).

## 5. Correct NUMERIC aggregation schema (opposite suffix rule!)

For averaging/summing a numeric event property, the property string **does**
carry the `_real` suffix, and lives under `aggregation.numeric`:
```json
"aggregation": {
  "numeric": {
    "aggregation": "NUMERIC_AGGREGATION_METHOD_AVERAGE",
    "property": { "customEventProperty": "booking_lead_days_real" }
  }
}
```
(Yes — bool deps use the **bare** name, numeric aggregation uses the
**`_real`** name. Verify by computing; a structurally-wrong name → 0/null or
`unspecified error`.)

Unique-user count aggregation:
```json
"aggregation": { "unique": "UNIQUE_AGGREGATION_INDIVIDUAL" }
```

---

## 6. Counting a segment's population

Segments don't return a count directly. To get unique users in a segment:
1. `update_metric(metric_id = <a unique-user metric>, segment_id = <seg>)`
   → returns a new unnamed metric scoped to the segment.
2. `compute_metric(metric_id = <new>)`.

Pick a base metric whose event every segment member has (e.g. unique users of
`booking_confirmed`) so the scoped count equals the segment size.

---

## 7. Sequencing with negation ("A then B WITHOUT C in between")

This is the key modeling trick. You **cannot** express "no C between A and B"
directly:
- Funnels don't exclude intermediate events.
- Segment `excludeBehaviors` is **absolute** (no C anywhere in the window),
  which is wrong when C also legitimately occurs earlier in the journey.
  (e.g. everyone who viewed availability searched once already, so excluding
  `booking_searched` outright drops *everyone*.)

**Use ordered segments + subtraction:**
- `A` = segment `[event1 (with property)] → [target]`, `inOrder + inSameSession`.
- `B` = segment `[event1] → [C] → [target]`, `inOrder + inSameSession`
  (the middle C, being in-order, IS "C in between").
- **Numerator = |A| − |B|** (B ⊆ A). Caveat: unique-user dedup makes this a
  close approximation, not exact, when a user has multiple differing journeys.

`inSameSession: true` + `inOrder: true` are set via phrasing like
"in the same session and strictly in this order".

---

## 8. Always sanity-check the denominator and the unit

Two numbers can both be "correct" yet answer different questions:
- **Denominator scope.** "requested slot unavailable" (`requested_slot=false`,
  ~1,046 users) is very different from "...AND alternatives existed"
  (`requested_slot=false AND booking_availability=true`, ~330 users). Most
  users with an unavailable requested slot had **no** availability at all.
- **Unit:** `UNIQUE_AGGREGATION_INDIVIDUAL` (users) vs unique **sessions**
  differ. State which you used.
- **Gross vs net:** a plain `A → B` funnel counts conversions *including*
  people who re-searched in between; that is NOT "without another search."

---

## 9. Misc

- Default/implied time window: state it. Examples here use last 30 days
  (`startTime: "NOW/DAY-29DAY"`, `endTime: "NOW/DAY+1DAY"`).
- These booking events are implemented on **Milano** only — cross-platform
  analysis is currently constrained.
- `discover_org_context([...terms])` is the fast way to confirm the exact
  custom-variable name, data type, and scope before building anything.

---

## Worked example — "% of customers whose requested slot was unavailable who
booked WITHOUT searching again" (last 30 days)

1. Confirm vars: `discover_org_context(["booking_viewedAvailability",
   "booking_confirmed", "booking_searched", "booking_availability_requested_slot"])`.
2. Denominator (users, raw custom event + bool dep, bare name):
   `booking_viewedAvailability` where `booking_availability_requested_slot`
   = false → **1,046**.
3. Segment **A**: `viewedAvailability(false) → booking_confirmed`
   (`inOrder`, `inSameSession`). Count via §6 → **286**.
4. Segment **B**: `viewedAvailability(false) → booking_searched →
   booking_confirmed` (same constraints) → **212** (re-searchers).
5. Numerator = A − B = **74**. Answer = 74 / 1,046 = **~7%**.

(The Fullstory UI funnel `viewedAvailability(false & availability=true) →
booking_confirmed`, counting unique *sessions*, returns ~31% — a *gross
recovery rate* on a smaller denominator that does NOT exclude re-searchers.
Different question, not a contradiction.)
