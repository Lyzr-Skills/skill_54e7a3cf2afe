---
name: anaplan-shipment-consolidation
description: Use for EVERY Anaplan read or write in the PRM shipment-consolidation model — partial loads, truck utilisation, MOQ exceptions, order multiples, Minimum Acceptable Load, To Review flags, Override Final Plan, Make Quantity Zero, Do Not Ship Partial Exception, SHC050, SHC080, SHC002, CTW004, Action/Slushy/Liquid windows, Product Logistics Groups. Any message naming a module, line item, load, lane, week or dimension member is a retrieval request — use this skill and follow it end to end. It carries verified module ids, thresholds, the real formulas, the sliced-query pattern, the cell_write schema and the verify-the-derived-twin rule, so almost all discovery can be skipped.
---

# Anaplan — PRM shipment consolidation

Everything below was read from the live model and cross-checked against Anaplan's own values.
**Do not re-discover any of it.** Query only what is genuinely not here.

**Tool-call budget: `4 + ceil(slices/10)`** — bind, status, `sql_schema`, one live threshold read,
then the query waves. Discovery (modules, line items, formats) is pre-solved below, so it costs nothing.
**Never loop per line item. Never one query per row. Never re-catalog what is listed below.**

---


## FAST PATH — obey this before anything else

These three rules exist because a measured run spent **52 MCP calls and 236 reasoning turns to perform
2 writes**. 29 of those calls re-learned facts that never change.

**1. BIND ONCE. NEVER RE-DISCOVER.**
Call `set_model_context` once and `get_model_status` once (poll only if not ready).
Then **do not call** `sql_schema`, `catalog_modules`, `catalog_line_items`, `get_model_context` or
`get_time_metadata` again in the same run. The module ids, line-item names and formulas are in this skill —
use them. Re-discovery is the single largest waste in the pipeline.

**2. ONE WIDE QUERY, NOT MANY NARROW ONES.**
Page the whole working set in (500 rows per call), then do all filtering, counting and
summing in your head — SQL will not aggregate for you.
The entire SHC080 working set is ~4,000 rows and returns in ~3.5 seconds:
```sql
SELECT "time","hlg010 - source #","hlg020 - destination #","hlp080 - product leaf #",
       "load id","final load","final plan"
FROM   "<model>.shc080 - final schedule by sup, loc and prod"
WHERE  <every dimension _is_leaf> AND "load id" IS NOT NULL
```
Never issue a second query for a slice you already hold. Never put `MOQ` or `Order Multiple` in the same
`included_objects` as time-phased items — different dimensionality, silent cross-join, wrong rows.

**3. READ ANAPLAN'S OWN VERDICT — DO NOT RECOMPUTE IT.**
The model already publishes its conclusions as line items. Read them instead of deriving them:
`To Review?` · `IsPartialLoad?` · `IsPartialLoadUnderMinimum?` · `PartialLoadCategory` ·
`Effective Do Not Ship Partial?`
Only compute what Anaplan does not expose: **sibling headroom** (`Final Plan − MOQ` per product in a PLG
group) and **MOQ-legal rounding** (nearest Order Multiple, never below MOQ).
Thresholds (MAL/MAPL) are still read live from SHC002 once per run — they are planner-editable and change.

**Budget for a well-run pass: about 6 tool calls.** status · one wide read · the writes · one verify read.
If you are past a dozen, you are re-discovering something you were already given.


## Step 0 — Bind, and wait properly
`set_model_context` once, then poll `get_model_status` until `ready`. If it returns `closed`, issue any
single `catalog_modules` call to wake it, then keep polling (≤5 polls, then back off gracefully).
**Never give up after one poll, and never report "no data" from an unready model.**

## Step 1 — Modules (ids already known — pass as `module_id`, never `module_name`)
| module | id | grain |
|---|---|---|
| SHC050 - Consolidation Summary | `102000000521` | Time × PLG × Source × Destination |
| SHC080 - Final Schedule by Sup, Loc and Prod | `102000000529` | Time × Source × Destination × Product Leaf |
| SHC002 - Shipment Consolidation Parameters Assigned | `102000000515` | PLG × Source × Destination |
| SHC085 - Loads Breaches by User | `102000000612` | + Users — **per-planner, see "what done means"** |
| CTW004 - Time Window Over Time | `102000000575` | Time × PLG × Source × Destination |
| CTW006 - Time Window Filters by User | `102000000601` | + Users — **NOT interchangeable with CTW004** |
| SHC060 - Consolidation Output | `102000000523` | Time × PLG × Source × Destination |
| SHC090 - Load Assignment | `102000000531` | Time × PLG × Source × Destination |
| SHC082 - Load Breaches by Week | `102000000586` | Time × PLG × Source × Destination |
**None have a Versions dimension — never pin a version.**

🔴 **CTW004 and CTW006 are different modules, and only CTW004 has the windows.** Their PRD names them
together as "CTW004 / CTW006", which reads as interchangeable. **MEASURED 22 Sep:**

| | line items | grain |
|---|---|---|
| **CTW004** | `Action Window Filter`, `Slushy Window Filter`, `Liquid Window Filter`, `Frozen…` | lane × PLG |
| **CTW006** | `Active Window`, `Final Active Window by PLG`, `Final Active Start/End Date` | **+ Users** |

CTW006 has **no Action or Slushy line item at all**, and it is per-planner — the same shape as SHC085.
**MEASURED: `Active Window <> 0` on ZERO rows**, exactly like SHC085's empty `To Review?`, so scoping
from CTW006 scopes to nothing. **Take every window filter from CTW004.**

🔑 **The PLG modules vs SHC080.** SHC060 / SHC090 / SHC082 / CTW004 / SHC050 are all **lane × PLG**;
only SHC080 is **per product**. Their §5.1 sources `Initial Load ← SHC060.Total Loads` and
`Final Load ← SHC090.Final Loads` — PLG-level numbers — while SHC080's `Initial Load` / `Final Load`
are those same PLG values looked up onto every product row via `DAT080.EF PLG Assigned`. That is why
**every product on a truck shows the same Final Load**, and it is the mechanical reason the
PLG-sibling rule exists: you are reading one truck's number, not one product's.
These ids are stable across **copies of this model** (verified: sandbox and production match). They are
**not** portable to a different customer's model. **If an id returns nothing, fall back to
`catalog_modules` with `name_contains` and use the id you get back** — never assume the id is wrong data.

## Step 2 — Thresholds — ⚠️ BASELINE, NOT GOSPEL
These were read from SHC002 (all 120 lane-PLG rows) on 17 Sep 2026:

| PLG | Minimum Acceptable Load | Minimum Acceptable Partial Load |
|---|---|---|
| Primary Packaging | 0.90 | 0.50 |
| Solid Dose | 0.90 | 0.50 |
| Bulk Liquids | 0.80 | 0.40 |
| Flammable/Hazardous | 0.80 | 0.40 |
Capacity Buffer **0.05** · Max Period Pull Forward **5** — uniform across every lane at that time.

**Where these actually live (verified via `explain_cell`):**
```
SHC002.Minimum Acceptable Load
  = 'SHC000 - Shipment Consolidation Parameters'.EF Minimum Acceptable Load
    [LOOKUP: 'TRM020 - TRM Assignment by Source x Destination x PLG'.EF MoT Assigned]
```
- **`SHC000` (`102000000455`) is the EDIT POINT** — dimensioned **Transit Mode x PLG**, and it holds the
  raw/EF twin pair for all four parameters (`Minimum Acceptable Load` + `EF Minimum Acceptable Load`, etc).
- **`TRM020`** assigns a transit mode to each Source x Destination x PLG.
- **`SHC002` is the RESOLVED per-lane result** — read it for the value in force; it is the right read target.
🔴 **THE EF TRAP APPLIES TO READS, NOT JUST WRITES — THIS HAS ALREADY CAUSED A WRONG ANSWER.**
In `SHC000` the **raw input columns are largely 0 or blank** and the **EF twin carries the operative value**:
| Solid Dose, any transit mode | raw | **EF twin** |
|---|---|---|
| Max Period Pull Forward | **0.0** | **5.0** |
| Minimum Acceptable Load (Bulk Liquids / Flammable) | **0.0** | **0.8** |
| Capacity Buffer | **0.0** | *(EF carries it)* |
An agent that read `SHC000.Max Period Pull Forward` got **0.00** and concluded *"pull-forward is disabled on
this lane, which is why the shortfalls cannot self-heal."* **That conclusion is false** — the model uses
**5**. The cell was real; the column was wrong.
**RULE: on any parameter module, read the `EF …` column. Never the raw input. If you must cite a raw value,
say explicitly that it is an unused input, not the value in force.** Safest of all: read the resolved value
from **SHC002**, which is what the calculation consumes.

✅ **CONFIRMED BY THE MODEL'S AUTHOR (James Kelly, Anaplan, 17 Sep 2026)** — this is a deliberate,
model-wide design, not a quirk of one parameter:
> *"The EF is the one which the system takes down to the next planning level — the one which is 0 is the
> cell where a planner can overwrite the EF value."*

The parameters are built as a **default → override cascade**, each level adding dimensions:
1. a **hardcoded default** that cannot be changed (so a model copied to a customer tenant is not blank);
2. **global parameters** — the client's system-wide configuration, freely changeable;
3. **intermediate overrides** at progressively more dimensions;
4. the **lowest planning level**, where a planner may override for that specific calculation.
*Rationale (James): clients should not have to set a parameter at every level — with thousands of products
and customers that would be unworkable.*

⇒ **Therefore, for EVERY parameter in EVERY Anaplan model built this way:**
- the **`EF …` line item is the resolved value the system actually uses** — always read it;
- the **un-prefixed line item is an override SLOT**. A **0 or blank there means "no override at this level",
  NOT "the value is zero"**. Reading it as a value is a category error, and it has already produced one
  confident, wrong root-cause analysis.
- This generalises beyond shipment consolidation — expect the same pattern in Inventory Planning and any
  other app in the suite.

⇒ **MAL varies by MODE OF TRANSPORT, not only by product group.** Two lanes with the same PLG can carry
different thresholds if their assigned mode differs. Never generalise one lane's MAL to another.
If asked *why* a threshold is what it is, trace SHC000 + TRM020, not SHC002.

🔴 **THESE ARE PLANNER-EDITABLE AND THEY DO CHANGE.** They are tuning parameters, not constants —
the business has already moved Minimum Acceptable Load between 0.80 and 0.90 and got a different
answer set each time.

**Rule:** use the table above to plan the query. But **whenever the answer depends on a threshold**
— any classification, any breach call, any recommendation — **read SHC002 once in the same run and
use what you read.** It is a single cheap call against `102000000515`.
If a value differs from the table above, **use the live value and say so explicitly in your answer.**
Never classify a load against a threshold you did not read this run.

**Product → PLG** (from `DAT080`, effective assignment — read via the `ef plg assigned` twin, not the raw):
Blister Pack, Bottle, Gel Tube → **Primary Packaging** ·
Nurofen Tablets 12, Nurofen Tablets 24 → **Solid Dose** · Dettol Antiseptic 597ml → **Bulk Liquids** ·
Dettol Hand Sanitizer 100ml → **Flammable/Hazardous**
**If you meet a product that is not in this list, read `DAT080` — do not guess its group.** The group
determines the truck, so a wrong group is a wrong answer.

## Step 3 — The real formulas (via `explain_cell`; the PRD text is wrong in places)
```
IsPartialLoad?             = Consolidated Load < Minimum Acceptable PARTIAL Load AND Consolidated Load > 0
IsPartialLoadUnderMinimum? = Consolidated Load < Minimum Acceptable Load         AND Consolidated Load > 0
PartialLoadCategory        = Consolidated Load / Minimum Acceptable Load           (a RATIO, not a %)
AnyUnmetRequirementinPullForwardWindow? = MOVINGSUM(Unmet Requirement, 0, 5) < 0
To Review?  = IF Clear Review THEN FALSE
              ELSE Action Window = 1 AND AnyUnmetRequirementinPullForwardWindow?
                   AND 'Effective Do Not Ship Partial?'
SHC080 To Review? = Display? AND NOT ( MOD(Final Load,1)=0 OR MOD(Final Load,1) >= MAL )
```
**Two different thresholds:** "partial" tests against **MAPL**, "under minimum" against **MAL**.
The breach is on the **fractional remainder** of Final Load, never the absolute count.
A partial load with **no unmet requirement** in the pull-forward window is correctly **not** flagged —
Advance Build has already covered it.

## Step 4 — Query: the whole working set, PAGED (500 rows max per call)
`sql_schema` AND `sql_query` **both require `included_objects`** = `{module display name: [line items]}`.
Reuse the same dict on every call. Table names are `"<model name lowercased>.<module name lowercased>"` —
read `tableName` from the schema, never construct it. One table per dimensionality group: **pick by
dimension count, never index 0.**

**Slice rule (server-enforced):** every dimension must be constrained by `= '<member>'` or by its
`"<dim>_is_leaf"` flag (suffix is exactly `_is_leaf`, 8 chars), or the query is rejected.
`IN (...)` is a filter, valid only *in addition* to a leaf flag.

🔴 **NEVER MIX TIME-PHASED AND NON-TIME-PHASED LINE ITEMS IN ONE QUERY — IT SILENTLY CROSS-JOINS.**
`sql_schema` returns **one table per dimensionality group**. In SHC080, `MOQ` and `Order Multiple` are
**not time-phased** (3 dims, no time) while `Final Plan`, `Final Load`, `Load ID` etc. are (4 dims).
**Measured on Clorox -> California DC — reproduces on Week 49 FY26, Week 52 FY26 AND Week 3 FY27:**
| `included_objects` | rows returned |
|---|---|
| time-phased only (`Load ID`, `Final Load`, `Final Plan`) | **3** — correct: the 3 products on that truck |
| the same **+ `MOQ`** | **7** — WRONG: every product in the model, incl. ones not on that lane |
The extra rows look completely plausible and carry real-looking values. **Nothing errors.**
**RULE: run two queries — one for the time-phased items, one for `MOQ`/`Order Multiple` — and join them
yourself on (source, destination, product).** If you must sanity-check, count the products: a truck has the
products actually consolidated onto it, not the whole catalogue.

**SELECT only the line items you need** — dimensions belong in WHERE, not SELECT. Sole exception: when the
output format itself needs a dimension echoed (e.g. a week × value table).

🔴 **EVERY QUERY NEEDS `LIMIT`, THE CAP IS 500, AND AGGREGATES ARE BANNED.** Measured 23 Sep 2026 — this
changed overnight and it is the single most important rule here:

| query | result |
|---|---|
| no `LIMIT` | `SQL_ADMISSION_REJECTED` — *"requires an explicit LIMIT ... no greater than 500"* |
| `COUNT(*)` / `SUM` / `AVG` / `GROUP BY` / window fn | `SQL_ADMISSION_REJECTED` — *"not admitted over a leaf scan"* |
| `SELECT * ... LIMIT 500` | ✅ 500 rows |
| `LIMIT 501` | rejected — *"the maximum is 500"* |
| `LIMIT 500 OFFSET 500` | ✅ **pagination works** |

So: **page with `LIMIT 500 OFFSET n` until a short page comes back, and do every count, sum and
group-by yourself on the rows.** SHC080's working set is roughly 4,000 rows, i.e. ~8 pages.

### 🔴 PAGING DOES NOT RESCUE `SELECT DISTINCT` — that one needs a DIFFERENT CALL
Asking a module table which members exist is the single most natural way to explore a model, and it
is refused outright. MEASURED 24 Sep, on the exact shape:

| query | verdict |
|---|---|
| `SELECT DISTINCT <member> ... ` no LIMIT | rejected |
| `SELECT DISTINCT <member> ... LIMIT 500` | **still rejected — a LIMIT does not help** |
| `SELECT COUNT(DISTINCT ...) ... LIMIT 1` | rejected |

The MCP's own words: *"SELECT DISTINCT over a module table scans and reduces every matching cell
before LIMIT applies, so it is not admitted interactively. **To find which periods, versions or
items exist, read the dimension itself: `get_time_metadata`, `catalog_lists`, or the list table.**
To check whether a slice has data, query it with a tight slice and `LIMIT 1`."*

⇒ **It is a different CALL, not a different query.** Re-running it, adding a LIMIT, or resetting the
session will not clear it — the shape is categorically refused, so it comes back the next time that
step runs. Read the dimension; do not interrogate the fact table for its own member list.

#### The replacement, and it is FASTER than the DISTINCT ever was
**Every list is queryable as its own SQL table.** Name the list in `included_objects` with an empty
line-item array, take the table name back from `sql_schema`, and select `label`:

```
sql_schema  {"included_objects": {"HLP080 - Product Leaf #": []}}
   -> tableName  "<model>.hlp080 - product leaf #"
      columns    _id · _parent_id · label · parent · code · displayname
sql_query   SELECT "label" FROM "<that table>" ORDER BY "label" LIMIT 500 OFFSET 0
```

MEASURED 24 Sep: Product Leaf **18 members in 1.2s** · Product Logistic Groups 5 · Source 7 ·
Destination 6 · **Load IDs 2,584 in 8.7s over 6 pages** with `ORDER BY "label" LIMIT 500 OFFSET n`.
A `DISTINCT` over the module would have scanned ~4,000 fact rows to recover 7 product names; this
reads them from the dimension itself — **and it returns the PARENTS too**, so you get the hierarchy
a leaf-sliced `DISTINCT` could never show.

| you want | the call |
|---|---|
| which lists exist | `catalog_lists {limit, skip}` — paged; the key is **`items`**, not `results` |
| the MEMBERS of a list | the list table, as above |
| time periods | **`get_time_metadata`** — no SQL at all |
| line items of a module | `catalog_line_items {module_id}` — id is the `entity_long_id` STRING; key is **`items`** |
| does this slice have data | the slice itself + `LIMIT 1` |

⚠️ **`catalog_lists` returns names and ids only — NOT members**, and `catalog_properties(list_id)`
comes back **empty** on real business lists. Both look like the answer and neither is. The list
TABLE is the one that works.

🔑 **But before you page 8 times to add something up, read the total from the PARENT item instead.**
The rejection says so itself:

> *"Return the rows without a total, or **read the total from the parent item: find it via
> `catalog_lists top_level` or the list table's parent column, then slice the module to that one
> item.** Use a bulk/export path for wider reads."*

Anaplan has already aggregated up its own hierarchies. A total at a parent list item is **one row**,
not eight pages — so for "how many / how much in total", slice to the parent rather than scanning
leaves and summing. Page the leaves only when you need the individual rows to reason about.
⚠️ This does **not** work on Time: `time = 'All Periods'` / `'FY27'` return empty (measured 18 Sep,
`sql_query` serves leaf time members only). Parent-slicing is for list dimensions.

⚠️ **A REJECTED QUERY IS NOT AN EMPTY TABLE.** The rejection arrives as an `error` object, and code
that reads `results` with a default of `[]` turns it into "no rows". MEASURED 23 Sep: that is exactly
what happened to our own reader — it reported every override and both reviews missing and looked like
a wiped model. **Check for `error` on every response before you believe a count of zero.**

The module's "cell count" in the Anaplan UI is a dimensional cross-product, **not** data volume — on
17 Sep 2026 SHC080 reported ~11 million cells while holding **43,890** populated rows. That is a dated
illustration of the *ratio*, not a constant. The permanent rule — **size your query by the slice you
need, never by the advertised cell count** — still holds; it is just that the slice now arrives in
pages. If you need several independent slices, batch up to **10 `sql_query` calls in parallel**, never
one per line item.

### Tools added 23 Sep — the surface grew from 16 to 20
| tool | use |
|---|---|
| **`cell_change_history`** | *who* changed a cell, *when*, and from what to what — and whether it was typed or imported. Same `module` / `line_item` / `slice` shape as `explain_cell`. **Reach for this first when a value is not what you expected**; it answers "did someone override this" without guesswork. |
| `get_version_metadata` | the model's versions plus a `write_target` (ok / avoid) and `write_notes` per version. **Call before `cell_write` on a versioned module** — PRM's SHC modules have no Versions dimension, so it does not apply here, but other models do. |
| `list_time_anchors` | every period the model offers as "now", so relative time resolves against the model's clock rather than today's date. Reports a `conflict` when candidates disagree. |
| `set_time_anchor` | fix what this conversation treats as "now". Use after `list_time_anchors` when a request says "current" / "last" / "recent". |

**Result checks:** `CORE_TIMEOUT` / `MODEL_BUSY` → transient, retry that query once.
**All-null across a whole result set → call `get_model_status` before reporting anything.** A model that
went back to sleep returns nulls under HTTP 200 that look exactly like real empty data.
Empty result → report which constraint found nothing **and** the observed members of every pinned
dimension. Never a bare "no rows".

---

## Writing

```json
{"module":"<display name>",
 "writes":[{"line_item":"<name>","value":<v>,
            "dimensions":{"Time":"Week 13 FY27","HLG010 - Source #":"Mondi", ...}}]}
```
Max **1000 cells per call, all in ONE module** — a batch spanning SHC050 and SHC080 needs two calls.
**Write sequentially, never in parallel** — concurrent access wedges the model.

### Verify the derived twin, never the cell you wrote
| you write | you verify |
|---|---|
| `Override Final Plan` | `EF Override Final Plan` |
| `Make Quantity Zero?` | `Final Make Quantity Zero?` |
| `Do Not Ship Partial Exception?` | `Effective Do Not Ship Partial?` |
The write response returns `success: true` but **`updated_count: null`** — it carries no count and is
**not evidence**. Only a changed twin proves the write landed.

🔴 **`success: true` can sit on top of a write that failed.** The real outcome is in
`result.failures[]`, which the top-level flag does not reflect:

```json
{"result": {"success": true, "message": "OK", "updated_count": null,
            "failures": [{"requestIndex": 0, "failureType": "cellIsCalculated",
                          "failureMessageDetails": "the cell is a calculated/aggregate cell and is not writeable"}]}}
```

**Measured 22 Sep** on `SHC095.ISQuantityRejected?`. **Always read `failures[]`**, and map each entry
back to the cell by `requestIndex` — in a 1,000-cell batch that index is the only way to know *which*
write was refused. Empty `failures` plus a moved twin is the only proof of a successful write.

⚠️ **`explain_cell` returning no formula does NOT mean a cell is an input.** The same line item
returned an empty formula on one slice and `Final Quantity Rejected from MOQ Exception <> 0` on
another. Before treating anything as writable, confirm it with a write and read `failures[]` — a
missing formula is a gap in the answer, not evidence of an input.

### ⚠️ The inversion
On a partial load, `Do Not Ship Partial Exception? = TRUE` makes it **SHIP**.
FALSE leaves the model default, which **suppresses** it.
Setting a field named "Do **Not** Ship" to TRUE is what **releases** the load.
(On a load that is *not* partial, TRUE suppresses it — check `IsPartialLoad?` first.)

### ⚠️ The PLG-sibling rule
A truck is shared by **every product in the same Product Logistics Group**. Size any correction against the
**combined group total**, never one product alone. A product sitting **exactly on its MOQ** cannot absorb a
reduction — move it to a sibling with headroom. If no sibling has headroom, the load is MOQ-locked and can
only move **up**.

### Truck capacity — read `TRM Capacity`, and know its unit
`SHC050` carries **`TRM Capacity`** — **it is NOT in product units.** Observed: Clorox -> California DC
Primary Packaging reads **39**, while the true capacity is **31,200 units**; China DC reads the equivalent of
**12** against **9,600 units**. In both cases `units = TRM Capacity x 800`, where 800 is the standard
order multiple on those products.
**Read `TRM Capacity`, then convert — do not back-solve from `Final Plan / Final Load` as if no source
existed.** If you do derive it, say so explicitly and flag that every quantity recommendation depends on
the derivation. (The x800 link is observed on two lanes, not proven across the model — verify before
relying on it on a new lane.)

### ⚠️ Before trimming, CHECK FOR AN OPEN UNMET REQUIREMENT
Trimming a load to reach a whole truck **sheds units**. If that product already carries a negative
`Unmet Requirement`, trimming makes a real shortfall worse.
**Measured case:** Clorox -> California DC Bottle carries **-38,400** unmet in Weeks 48-52 FY26
(110 negative rows on that lane, -1,570,400 total). An agent that trimmed 8,800 units there to land on
"exactly 1.000 load" was shedding stock into a documented shortfall **without checking**.
**RULE: never recommend a trim without first reading `Unmet Requirement` for the affected product across
the pull-forward window. If it is negative, top up instead - or state explicitly why shedding is acceptable.**

### Two valid fixes for a spillover breach — state which you chose and why
A load whose remainder is below MAL can be corrected **either way**:
- **TRIM DOWN** to the next whole truck (remainder → 0). Cheapest on freight, but **sheds units** — check
  whether there is an open `Unmet Requirement` before recommending it.
- **TOP UP** so the remainder clears MAL (≥ the threshold). Keeps the units, costs an emptier truck.
**Both are legitimate. Name the one you picked, give the other as the alternative, and state the trade-off**
(freight cost vs unmet requirement). Never present one as the only option when both are arithmetically open.

### Order of operations
Apply the **MOQ floor first**, then round **up** to the Order Multiple. An MOQ that is not a clean multiple
pushes the true minimum **above** the MOQ (e.g. MOQ 7,000 + multiple 5,000 + requirement 6,000 → **10,000**).

### Never
Write without explicit approval in the conversation — analysis is not approval.
Claim a number you did not read back. Call a write successful because the call succeeded.

---

## Business meaning — from Anaplan's own PRDs, and not derivable from the model

### ⚠️ Unmet Requirement and Advance Build are NOT physical stock
Their PRD calls this out under the heading *"Crucial Conceptual Clarification"*:

> *"Unmet Requirement and Advance Build are strictly planning consolidation line items. They indicate
> the quantities of stock **virtually** moved around in the planning schedule to accommodate
> consolidation rules. They **do not represent physical warehouse inventory, actual physical
> stockouts, or physical inventory carrying events.** They are purely metrics of planning /
> consolidation schedule alignment."*

So: an Unmet Requirement of −38,400 is a **scheduling offset**, not a shortage anyone can walk into a
warehouse and see. Treat it as a plan-alignment cost, never as an inventory emergency — and say so
in your answer. Their NFR requires it: *"Raw black-box scores are unacceptable… the agent must
explicitly note that Unmet Requirements / Advance Builds represent planning shifts rather than
physical inventory stockouts/holdings."*

### 🔑 Choosing the lever: TOP UP or ZERO OUT — the criterion is PULL-FORWARD VIABILITY
When a load breaches, there are exactly two legal moves, and their PRD says which to pick:

| condition | move | field |
|---|---|---|
| pull-forward **viable** — demand exists to pull within `SHC002.Max Period Pull Forward` | **top up** so the load clears MAL, rounded to a legal quantity | `Override Final Plan` |
| pull-forward **unviable** — *"pulling demand forward is restricted or inventory/capacity is unavailable"* | **zero it out** | `Make Quantity Zero?` |

`Max Period Pull Forward` is the same window the `To Review?` MOVINGSUM uses. Read it live from
SHC002 (it is **5** on the sandbox today) and check whether there is demand inside it to pull.

This is the positive rule. The shortfall guard — *never trim a product carrying a negative Unmet
Requirement inside the pull-forward window* — is the negative one. Both apply.

Where both moves are legal, **present both** and say what each costs. Their own worked example does
exactly that rather than choosing for the planner — MOQ PRD section 8 gives every load two buttons:

```
[ Accept Suggested Override (2,000) ]  [ Set Quantity Zero ]
[ Apply Override (4,000) ]             [ Accept Suggested Zero-Out ]
```

So a recommendation without its alternative is half a row. Price the other lever whenever it is
legal on that load, and return it as an object the planner can act on, not as prose:

```json
"alternative": {"item": "Make Quantity Zero?", "to": true, "why": "sheds 7,200u into a 3-week shortfall"}
```

`null` only when the other lever genuinely is not available. **Both options are write targets**, so
the alternative has to be legal too — MOQ floor, order multiple, and never the same action as the
recommendation.

#### 🔑 When the two rules disagree: MEETING THE REQUIREMENT WINS

A load can satisfy both conditions at once — pull-forward unviable (their matrix says zero out)
*and* carrying a negative Unmet Requirement (the shortfall guard says do not shed). The two PRDs
prescribe opposite actions and neither document resolves it. **Measured on `L0010_32-27`:** zeroing
it left **~7,200 units of requirement uncovered for two to three weeks**.

The client settled it on 21 Sep. Gautam:

> *"Meeting the requirement is priority over meeting MOQ first… **pull forward is better than
> pushing back to meet moq.**"*

James, on the same call, framing what the agent is for:

> *"You need a human to make a decision on that. Whereas we would want an agent to try and make a
> recommendation like a human would."*

So on a conflicted load:

1. **Propose the top-up** if a legal one exists.
2. Say the conflict exists — do not resolve it invisibly.
3. **Name the zero-out you did not take, and what it would have cost.** The recommendation is
   yours; the choice stays the planner's.
4. If no legal top-up exists, the zero-out is the only move left. Propose it, and say plainly that
   it sheds into a known shortfall.

⚠️ This is a **tie-break, not a general preference.** Where pull-forward is unviable and there is
no unmet requirement, the matrix stands unchanged: zero it out. Topping up with no demand behind it
builds stock nobody asked for.

### Horizon discipline — identical in both PRDs
- **Action Window** — primary scope. *"Immediate operational horizon… exceptions must be resolved immediately."*
- **Slushy Window** — primary scope. *"Near-term… overrides are evaluated to optimise pull-forwards prior to entering the Action Window."*
- **Liquid Window** — **out of active scope**. *"Requirements remain flexible and exceptions are deferred."* Report as context, never propose a fix there.

### 🔑 Anaplan already counts the review population — do not count it yourself
`SHC082 - Load Breaches by Week` (`102000000586`, lane x PLG) carries native counters, and they are
**the same population** our own row-counting produces:

```
# To Review                = IF SHC080.To Review?[ANY: DAT080.EF PLG Assigned] THEN 1 ELSE 0
# To Review - Action Window = IF CTW004.Action Window Filter THEN '# To Review' ELSE 0
# To Review - Beyond Action Window
```

**MEASURED 22 Sep on the sandbox:** `SUM(# To Review)` = **75**, and counting the displayed breaching
loads by hand also gives **75**. `SUM(# To Review - Action Window)` = **0**, and the hand count in the
Action Window is also **0**. Exact agreement.

So when you need "how many are still in review, in scope", **read SHC082's counter column and add it
up yourself** — SQL will not sum it for you (aggregates are rejected, see Step 4):

| scope | column to page and total |
|---|---|
| Action Window | `SHC082."# To Review - Action Window"` — Anaplan has already scoped it for you |
| all windows | `SHC082."# To Review"` |
| Action + Slushy | `SHC082."# To Review"`, kept where CTW004's Action **or** Slushy filter is set |

⚠️ **`SUM(...)` in the query is rejected.** Either slice SHC082 to the **parent** PLG/lane item and
read the aggregate in one row, or page the column with `LIMIT 500 OFFSET n` and total it yourself.
SHC082 is lane × PLG, so even paging it is roughly one page, not eight.

This still matters even with the summing on you, because **the scoping decision is the unreliable
part, not the addition.** The same question asked of the agent from equivalent states has returned
**0, 2, 27, 49, 51 and 71**. Anaplan's column has already decided what is in the Action Window —
**take its gating, not your own.**

### What "done" means
> *"Final Load >= MAL (or zeroed out)"* **and** *"To Review? flags cleared (= FALSE)."*

Their validation step (MOQ PRD §7.1) names **SHC085** — `To Review?` *per USER* — not SHC050.

🔴 **Do not use SHC085 as the acceptance test. MEASURED 22 Sep on the sandbox:**

| module | `To Review?` TRUE |
|---|---|
| SHC050 (lane x PLG) | **2** |
| SHC085 (+ Users) | **0** |

Same model, same moment. A run checking SHC085 would have reported **"nothing left in review"** on a
model with two open reviews — a false PASS, the one failure mode that looks exactly like success.

The 2-vs-0 is measured. The *reason* is not fully traced: SHC085 adds a `Users` dimension, and the
model carries per-user configuration (`SHC003 - Consolidation User Filter Selections`,
`CTW007 - Time Window Selections by User`), so it most likely populates only for a planner whose
selections cover the lane. Either way the conclusion holds — **SHC085 did not reflect an open
review that SHC050 did.**

**Check SHC050.** SHC085 answers *"is THAT planner's queue clear"*, which is a different question
from *"is the lane clear"* and only agrees once the planner's filters are set up.

### The two modules are coupled — that is why the stages feed back
SHC050 is lane/PLG level; SHC080 is product level. Suppressing a partial at lane level strips product
quantities in SHC080, which changes MOQ outcomes, which changes the lane KPIs again. A value read
before the other stage's write is **stale**. Re-read after every recalculation.

### Never invent a cost
Their FR-2 defines a freight-vs-plan-deviation trade-off, but **the symbols did not survive into the
document** — it reads *"the Plan Deviation Penalty ()"*. There is no cost function and no freight
rate anywhere. Reason from what you can read — load against MAL, Unmet Requirement in the
pull-forward window, whether an Advance Build already covers it — and say which drove the call.
**Do not produce a currency figure or a confidence percentage.** There is nothing to derive one from.
