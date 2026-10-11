# Recursive Patterns

**Excel dynamic-array catalog for recurrences, closed forms and classic fallbacks**

CFM, October 2026 knowledge base. Version of 11 October 2026.

---

## Contents

| | |
|:-:|:--|
|  | [Introduction: the recurrence problem and the five buckets](#recursive-pattern--cfm-october-2026-knowledge-base) |
| A | [multiplicative compounding grid: `SCAN` inside `MAKEARRAY`](#pattern-a--multiplicative-compounding-grid-scan-inside-makearray) |
| B | [additive period-counter + exponent: the actual "algebraic shortcut"](#pattern-b--additive-period-counter--exponent-the-actual-algebraic-shortcut) |
| C | [conditional-branch accumulator: reset, hold, or replenish](#pattern-c--conditional-branch-accumulator-reset-hold-or-replenish) |
| D | [rank-based cumulative allocation *without* SCAN](#pattern-d--rank-based-cumulative-allocation-without-scan) |
| E | [one-hot membership matrix + `MMULT`: groupby-sum as linear algebra](#pattern-e--one-hot-membership-matrix--mmult-groupby-sum-as-linear-algebra) |
| F | [`SCAN` as forward-fill: carrying a header label down to its detail rows](#pattern-f--scan-as-forward-fill-carrying-a-header-label-down-to-its-detail-rows) |
| G | [group subtotal via forward-filled key and `SUMPRODUCT` equality match](#pattern-g--group-subtotal-via-forward-filled-key-and-sumproduct-equality-match) |
| H | [payment-terms / aging amortization triangle (closed-form AR, AP, deferred balances)](#pattern-h--payment-terms--aging-amortization-triangle-closed-form-ar-ap-deferred-balances) |
| I | [period-lag opening balance (`HSTACK`+`DROP` shift), connecting flow solved as a residual](#pattern-i--period-lag-opening-balance-hstackdrop-shift-connecting-flow-solved-as-a-residual) |
| J | [plain unconditional roll-forward `SCAN`](#pattern-j--plain-unconditional-roll-forward-scan) |
| K | [coupled multi-row recursion: when a single spilled formula can't hold the whole recurrence](#pattern-k--coupled-multi-row-recursion-when-a-single-spilled-formula-cant-hold-the-whole-recurrence) |
| L | [FIFO/vintage layering via classic triangular `MIN`-with-expanding-range (deliberately *not* a dynamic array formula)](#pattern-l--fifovintage-layering-via-classic-triangular-min-with-expanding-range-deliberately-not-a-dynamic-array-formula) |
| M | [trailing-window slice via `TAKE`/`HSTACK`, replacing an `OFFSET`-anchored construction](#pattern-m--trailing-window-slice-via-takehstack-replacing-an-offset-anchored-construction) |
| N | [event-window flags and a per-series elapsed-month counter (2-D gated counter, no `SCAN`)](#pattern-n--event-window-flags-and-a-per-series-elapsed-month-counter-2-d-gated-counter-no-scan) |
| O | [piecewise-linear interpolation of a ramp, with a `MEDIAN` clamp](#pattern-o--piecewise-linear-interpolation-of-a-ramp-with-a-median-clamp) |
| P | [select first, compute second: array-keyed row gather from a master table](#pattern-p--select-first-compute-second-array-keyed-row-gather-from-a-master-table) |
| Q | [layered allocation into ordered bands: clipped differences of lagged ceilings (tax brackets)](#pattern-q--layered-allocation-into-ordered-bands-clipped-differences-of-lagged-ceilings-tax-brackets) |
| R | [vintage depreciation waterfall in closed form: cohort × period grid driven by asset age](#pattern-r--vintage-depreciation-waterfall-in-closed-form-cohort--period-grid-driven-by-asset-age) |
| · | [Flag conventions — progression from Fashion Boutique to Eagle Security](#flag-conventions--progression-from-fashion-boutique-to-eagle-security) |
| · | [Decision guide for the October exam](#decision-guide-for-the-october-exam) |
| · | [Bugs & gotchas addendum](#bugs--gotchas-addendum) |
| · | [Workbook coverage — all 34 solution workbooks, checked 11 October 2026](#workbook-coverage--all-34-solution-workbooks-checked-11-october-2026) |

---

# Recursive Pattern — CFM October 2026 knowledge base

Core problem this catalog addresses: dynamic array functions (`MAP`, `BYROW`, `BYCOL`) evaluate every element independently, so none of them can reference a value another call of the same function just produced. Any modeling situation with a genuine recurrence — `term[n] = f(term[n-1], driver[n])` — needs a different primitive.

Across the thirty-four workbooks solved so far (all checked against this catalog on 11 October 2026 — see "Workbook coverage" at the end), eighteen pattern families show up. Of the eighteen numbered A–R, fifteen (A, B, C, D, E, F, G, H, I, J, N, O, P, Q, R) are recipes for *how to write one formula*, split between two philosophies: five (A, B, C, F, J) thread state through `SCAN`; ten (D, E, G, H, I, N, O, P, Q, R) turn out to be closed-form, retrieval or filter-and-aggregate reformulations that avoid a fold entirely. The two latest workbooks, Eagle Security and Neverland, contain no `SCAN` at all — see Pattern Q and "Flag conventions" before the decision guide. Pattern K is different in kind — it's what to do when a recurrence can't be captured by one formula at all, because the state that has to carry forward is coupled across two or more named rows that reference each other; the fix is architectural, and the worked implementation lives in a companion file. Pattern L is also different in kind — it's what to do when a recurrence is genuinely order-dependent (FIFO layering) *and* competes across rows for a shared pool, so even Pattern K's tricks don't apply cleanly, and you fall all the way back to classic, non-array Excel. Pattern M is a *non-recursive* trailing-window slicing technique — included here because it replaces an OFFSET-based construction that looked recursive/stateful but wasn't.

Recognizing which of these buckets — SCAN-based fold, closed-form/filter reformulation, Pattern K's architectural fork, Pattern L's classic fallback, or Pattern M's static slice — a problem belongs to is most of the exam-time value of this catalog.

---

## Pattern A — multiplicative compounding grid: `SCAN` inside `MAKEARRAY`

**Where:** Fashion Boutique — `CaseData!K87`, `Price!J8/J18/J28`. Domestique Rides — `Executive Summary!J49`, `Subscription!J8/J18`, `Production!J8/J15/J24/J31` (scenario-select variant, below). Powerstill — `Calculations!J7`, `J57`, `J62`, `Assumptions!M10`. Granite Bay — `Debt!J96` (one-row degenerate case, below). United Landlords — `Calculations!J17`, `J27`, `J39`, `J54`, `J69`, `J84`, `J99`, `J100` (flag-gated variant, below).

**When it applies:** the rate varies period to period (not constant), and the recurrence needs to run independently across several parallel series (currencies, accounts, cost categories) stacked in a grid.

**Template:**
```
LET(
  clean_rates, IF(rates_range = "n.a.", 0, rates_range),
  compounded, MAKEARRAY( ROWS(clean_rates), COLUMNS(clean_rates),
                LAMBDA(i, j,
                  INDEX( SCAN(1, INDEX(clean_rates, i, 0), LAMBDA(s,c, s*(1+c))), j )
                ) ),
  compounded )
```

**Mechanism:** `MAKEARRAY` supplies the independent axis (which row); `SCAN` supplies the recurrence along the dependent axis (which column). `INDEX(clean_rates, i, 0)` slices out row `i` as a 1-D vector before handing it to `SCAN`, and the outer `INDEX(..., j)` pulls back out just the one cell `MAKEARRAY` is currently computing.

**Cost to be aware of:** the full row is re-scanned once per output cell. A 6×5 grid means 30 SCAN evaluations, not 6. Worth promoting the SCAN to a standalone spilling cell (referenced via `#`) once a grid gets large, rather than re-deriving it inline per output cell — confirmed as a live fix in Powerstill, where `Assumptions!M10` had already built the standalone diagnostic table but was disconnected from the calculation chain.

**Powerstill variant worth noting:** `Assumptions!M10` builds a standalone labeled output table (`VSTACK(HSTACK(title, columns), HSTACK(rows, MAKEARRAY(...)))`) — same recurrence, but exposed as a documented diagnostic table rather than consumed invisibly inside a LET chain. Good habit when you want to sanity-check the compounding grid independently of where it's used.

### Pattern A, scenario-select variant — compound every scenario row, then pick one cell per period (Domestique Rides)

**Where:** Domestique Rides — `Production!J8`, `J15`, `J24`, `J31`, `Subscription!J8`, `J18`, `Executive Summary!J49`. The rate grid has one row per scenario (e.g. `CaseData!$G$31:$K$33`) and one column per year.

**Template (confirmed, `Production!J8`):**
```
LET(
  operating_flag,      Time!$J$18#,
  current_year,        Time!$J$14#,
  current_scenario,    INDEX( production, production_selector ),

  growth_rates,        CaseData!$G$31:$K$33,
  compounded_rates,    MAKEARRAY( ROWS( growth_rates ), COLUMNS( growth_rates ),
                         LAMBDA( i, j, INDEX( SCAN( 1, INDEX( growth_rates, i, 0 ), LAMBDA( s, c, s * ( 1 + c ) ) ), j ) ) ),
  compounded_rows,     CaseData!$C$26:$C$28,
  compounded_columns,  CaseData!$F$23:$K$23,
  compounded_table,    VSTACK( compounded_columns, HSTACK( compounded_rows, compounded_rates ) ),
  compounded_growth,   INDEX( compounded_table,
                              XMATCH( current_scenario, compounded_rows, 0, 1 ) + 1,
                              XMATCH( current_year, compounded_columns, 0, 1 ) ),

  operating_flag * unit_sales_2020 * compounded_growth )
```

**Mechanism:** Pattern A builds the whole scenarios × years grid; `VSTACK`/`HSTACK` add a header row and a label column so `INDEX` can address it by name. `XMATCH( current_scenario, … )` picks one row, and `XMATCH( current_year, … )` with the 1 × T `current_year` spill picks one column per period, so the result is the 1 × T growth path of the selected scenario. This is the compute-first order that Pattern P argues against: every scenario is compounded and only one is kept. Selecting the scenario's rate row first and running one `SCAN` on it gives the same row and drops the `MAKEARRAY`.

**Gate before launch (`Production!J24`):** `growth_rates, IF( CaseData!$G$42:$K$42 <= start_of_production_date, 0, CaseData!$G$50:$K$52 )` zeroes the rates up to and including the launch year, so the index stays at 1 until the first full year after launch. The final `IF( start_of_production_date = current_year, 1, … )` forces the launch year itself to 1. It is the same idea as the flag-gated variant below (multiply the step by a flag), applied to the rate grid instead of to the step.

**Convention flags:** the label column (`$C$26:$C$28`) and the rate block (`$G$31:$K$33`) in `J8` come from different row ranges. That works only because both blocks list the scenarios in the same order (gotcha 2). The header row `$F$23:$K$23` is one column wider than the rates because it also covers the label column.

### Pattern A, one-row degenerate case (Granite Bay, `Debt!J96`)

`MAKEARRAY( ROWS( 1 ), COLUMNS( rates ), LAMBDA( i, j, INDEX( SCAN( 1, INDEX( rates, i, 0 ), … ), j ) ) )` on the single row `$J$95:$S$95` returns the same values as `SCAN( 1, rates, LAMBDA( s, c, s * ( 1 + c ) ) )` alone. `ROWS( 1 )` is 1, so the wrapper only adds the per-cell re-scan cost (10 scans instead of 1). Use the `MAKEARRAY` form only when there are several independent rows.

### Pattern A, flag-gated variant — annual step triggered by a calendar flag, flag and rate both derived inside the `LET`

**Where:** United Landlords — `Calculations!J17` (Rent Increase index, existing properties), `J27` (Cost Increase index, existing OPEX), `J39` (Land Appreciation index), `J54` (Cost Increase index, construction), `J69` / `J84` (new-property rent and OPEX), `J99` / `J100` (gross rent feeding the management fee). One workbook, but the identical recurrence appears eight times with only the rate column changing (column 2 = Rent Increase, 3 = Cost Increase, 4 = Land Appreciation of `CaseData!$J$15:$M$24`). The recurrence is Pattern A's; what is new is where the rate and the trigger come from. Provisional as a variant until a second case confirms it.

**Template (confirmed, `Calculations!J17`):**
```
LET(
  timeline,         $J$3#,
  inflation_flag,   IF( MONTH( timeline ) = 1, 1, 0 ),

  inflation_data,   CaseData!$J$15:$M$24,
  annual_rate,      XLOOKUP( YEAR( timeline ), INDEX( inflation_data,, 1 ), INDEX( inflation_data,, 2 ) ),
  effective_step,   inflation_flag * annual_rate,
  inflation_factor, SCAN( 1, effective_step, LAMBDA( prev, step, IF( step = 0, prev, prev * ( 1 + step ) ) ) ),

  ... driver * inflation_factor ... )
```

**Mechanism:** three moves, none of which needs a helper row.
1. **The flag is derived from the date spine.** `$J$3#` is a real date spine (`EOMONTH( start_date, SEQUENCE( 1, months, 0, 1 ) )`), so `MONTH( timeline ) = 1` gives the "new indexation year" trigger directly. Pattern C Variant 1 (the cyclic month counter, `Time!J11` in Fashion Boutique and Domestique Rides) existed to manufacture a calendar position when the timeline was only a run of period numbers; once the timeline carries dates, calendar arithmetic needs no fold.
2. **The rate row is derived, not laid out.** `XLOOKUP( YEAR( timeline ), years, rates )` retrieves the calendar year's rate for every month at once (Gather phase). Pattern A's `clean_rates` was a rate grid that already existed on the sheet.
3. **`effective_step = inflation_flag * annual_rate`** is zero in eleven months out of twelve, so the `SCAN` multiplies by `(1 + rate)` once a year and holds in between. The `IF( step = 0, prev, … )` hold branch is Pattern C Variant 2's idiom; it is arithmetically redundant here (`prev * ( 1 + 0 ) = prev`), so `LAMBDA( prev, step, prev * ( 1 + step ) )` — Pattern A's lambda verbatim — returns identical values.

**First-period convention:** the seed `1` sits *before* the first column, so if the first model month is a January the first column is already stepped (`Calculations!J17` uses `1 × ( 1 + 5% ) = 1.05` for January 2022). **Last-period convention:** none; the index simply holds after the final step.

**Progression across cases — same recurrence, flag and rate moving inside the formula:**

| Stage | Where | Calendar trigger | Rate source | Compounding |
|---|---|---|---|---|
| Classic corkscrew | United Landlords `CaseSolution!G209:G212` | helper row `IF( MONTH( G2 ) = 1, 1, 0 )` | helper rows `INDEX`/`MATCH` on year | per-column `= F210 * ( 1 + G206 * G$209 )` |
| April catalog entries | Fashion Boutique `Time!J11`, `CaseData!K87`, `Price!J8` | cyclic month counter built by `SCAN` (Pattern C V1) | rate grid on the sheet | `SCAN` inside `MAKEARRAY` (Pattern A) |
| October | United Landlords `Calculations!J17` | `MONTH( timeline ) = 1` inline | `XLOOKUP` inline | one `SCAN` on `flag * rate` |

**Promotion rule applies to the index, not to the flag.** The inline flag is stateless, so repeating it costs nothing. The `SCAN` is a recurrence, and the same one is rebuilt in eight consumers for only three distinct indices (rent, cost, land). Per `LET_Construction_Style_CFM.md`, a recurrence that more than one formula needs belongs in its own standalone spilling cell — one cell per index, each returning `inflation_factor` — with every consumer gathering it through `cell#` plus `INDEX`. The promoted form is not built in this workbook yet. Eagle Security has the same duplication in a different shape: the gather-and-flag preamble is rebuilt in `Calculations!J10`, `J20` and `J30` (`operation_flag` three times, `hours_available` twice), and the same fix applies — publish the 5 × 12 `operation_flag` and `hours_available` blocks once and gather them with `cell#`.

---

## Pattern B — additive period-counter + exponent: the actual "algebraic shortcut"

**Where:** Megabricks — `Calculations!J17, J31, J32, J54, J64, J72`. Thermin — quarterly demand growth, hourly-rate growth, markup escalation. Stanbridge — `Existing!J22`, `J50` (CAPEX-cost growth by vintage, sizing the asset before any depreciation method is applied).

**When it applies:** the rate is constant per period (one indexation rate, looked up once), and only the count of elapsed active periods varies. This is the case where you don't need SCAN's multiplicative fold at all.

**Template (calendar-difference variant — no SCAN needed, confirmed Thermin):**
```
LET(
  periodic_rate, IF(freq="Annual", 1+index, (1+index)^(1/4)),
  period,        YEAR(current_period) - YEAR(start_date),
  growth_factor, IF(period = 0, 1, periodic_rate ^ period),
  base_value * growth_factor )
```

**Template (gated-counter variant — additive SCAN needed, confirmed Megabricks):**
```
LET(
  periodic_rate, IF(freq="Annual", 1+index, (1+index)^(1/4)),
  indexation_count, SCAN(0, active_operating_flag, LAMBDA(s,c, s+c)),
  base_price * periodic_rate ^ indexation_count )
```

**The insight:** compounding a *constant* rate `r` for `n` periods is `r^n` — a closed form, no recursion needed for the compounding itself. Check first whether your period count is a simple calendar difference (Thermin — `YEAR(current) - YEAR(start)`, no SCAN anywhere) before reaching for the gated counter (Megabricks — periods can be switched on/off by an operating flag, so the *count itself* still needs an additive SCAN, even though the compounding step doesn't).

**Why this is the shortcut you were looking for:** it's the difference between "the recursion is unavoidable, use SCAN for the whole thing" (Pattern A) and "only part of the problem is sequential — isolate that part, solve the rest algebraically" (Pattern B).

**Per-series version:** when the counter has to run on many series at once (one start date, and possibly a duration, per property or city), see Pattern N — the same counter without `SCAN`, confirmed in United Landlords and Eagle Security. Eagle Security is Pattern B's closed form at full strength: `( 1 + g ) ^ period_count` with the count taken from a prefix sum of a flag, and no `SCAN` anywhere.

---

## Pattern C — conditional-branch accumulator: reset, hold, or replenish

**Where:** Fashion Boutique — `Time!J11` (calendar reset). Domestique Rides — `Time!J17` (calendar reset) in the monthly working file `24052026 v. 2 Revenues_Domestique_Rides_Worksheet.xlsx` only; the submitted solution `05242026_Revenues_v1_Domestique_Rides.xlsx` runs on an annual spine and has no cyclic counter. Wichita Warblers — `Calculations!J25` (hold-or-increment), `Calculations!J40` (replenish-on-boundary-change).

**When it applies:** you need a running total that isn't a pure fold — at each step it either advances normally, holds steady, or resets/replenishes, depending on a condition evaluated against the current driver, the running state, or both. This only works as a fold because the branch condition depends on the running total or a lookback comparison — exactly the "look at what I just computed" case `MAP`/`BYROW` can't do.

**Variant 1 — cyclic reset on a numeric threshold (calendar arithmetic; confirmed, `Time!J11`):**
```
SCAN(0, SEQUENCE(1, periods, 1, 0), LAMBDA(s, c, IF(s + c > 12, c, s + c)))
```
(Domestique Rides flips the boundary test slightly: `IF(s+c<=12, s+c, c)` — same idea, tested from the other side.)

**Variant 2 — hold-or-increment on an event flag (confirmed, `Calculations!J25`):**
```
SCAN(0, days_to_game, LAMBDA(acc, dtg, IF(dtg = 0, acc + 1/number_of_games, acc)))
```
**Mechanism:** the accumulator only advances (by a fixed fraction) on the period where the trigger condition fires (`days_to_game = 0`, i.e. it's game day); every other period it just holds its prior value. Same primitive as Variant 1, but the branch *withholds* progress instead of *resetting* it.

**Variant 3 — replenish on a boundary-change (group reset) rather than a numeric threshold (confirmed, `Calculations!J40`):**
```
SCAN(0, SEQUENCE(,COLUMNS(game_day)),
  LAMBDA(rem, i,
    IF(
      OR(i = 1, INDEX(game_day,1,i) <> INDEX(game_day,1,i-1)),
      INDEX(tickets_available,1,i),
      MAX(rem - INDEX(demand_prev,1,i), 0)
    ) ) )
```
**Mechanism, and a generalizable technique:** two things worth taking from this one. First, the *reset trigger* is a change-detection comparison (`INDEX(game_day,i) <> INDEX(game_day,i-1)`) rather than a magnitude threshold — "start a new group" instead of "overflowed a fixed cap." Second, the `SCAN` walks a plain `SEQUENCE` of positions, not the driver array itself — inside the `LAMBDA`, `INDEX(...,1,i)` reaches into *three different* parallel arrays (`game_day`, `tickets_available`, `demand_prev`) by position. **Scanning an index instead of a data array is the move whenever a single step of the recurrence needs more than one parallel series at once** — SCAN's own driver argument only gives you one series per step.

**Generalizable takeaway for all of Pattern C:** any accumulator with a conditional branch — reset, hold, or replenish — is a SCAN job, and the branch condition can test a numeric threshold, an explicit event flag, or a change-of-value between adjacent positions.

**Note — when the timeline is a date spine:** Variant 1's cyclic counter is unnecessary; `MONTH( timeline )` supplies the calendar position directly (United Landlords, see Pattern A's flag-gated variant).

---

## Pattern D — rank-based cumulative allocation *without* SCAN

**Where:** Big Fish — `Calculations!J103` (ranking) and `V166` (capacity allocation).

This is the most interesting one relative to what looks sequential, because it solves a genuinely sequential-feeling problem (allocate limited capacity to the highest-priority items first, in order) **without SCAN and without recursion at all.**

**Step 1 — rank each column independently** (this part is a plain `MAKEARRAY`, no fold needed, since ranking a column doesn't depend on other columns):
```
MAKEARRAY(ROWS(gross_profit), COLUMNS(gross_profit),
  LAMBDA(i,j, RANK.EQ(INDEX(gross_profit,i,j), INDEX(gross_profit,,j))))
```

**Step 2 — allocate capacity by priority, algebraically:**
```
MAKEARRAY(ROWS(demand), COLUMNS(demand),
  LAMBDA(i,j,
    LET(
      r,            INDEX(rank, i, j),
      rank_column,  INDEX(rank,  , j),
      demand_column,INDEX(demand,, j),
      higher,       SUMPRODUCT((rank_column < r) * demand_column),
      MIN(INDEX(demand,i,j), MAX(capacity - higher, 0))
    ) ) )
```

**The trick:** "how much capacity is left for item ranked `r`" is normally framed as sequential — process items in rank order, subtract each one's demand from the remaining pool. Instead, `SUMPRODUCT((rank_column < r) * demand_column)` computes "total demand of every item that outranks this one" directly, as a set condition — no iteration, no fold, no need to actually sort anything.

**Generalizable takeaway:** before reaching for SCAN, ask whether the "what happened before me" quantity you need can be expressed as a *filter-and-aggregate over a ranking or ordering key* instead of an actual walk through the sequence.

**Related:** when the ordered layers have known ceilings (tax brackets, loss buckets), the same allocation collapses further, to clipped differences of lagged ceilings with no `SUMPRODUCT` — see Pattern Q.

---

## Pattern E — one-hot membership matrix + `MMULT`: groupby-sum as linear algebra

**Where:** Sweet Dreams — `Purchasing Plan!J66` and `J99` (flat, by supplier), `J75` and `J108` (hierarchical, by product), `Inventory!J59`. Fashion Boutique — `Revenue!J30/J33/J40` (time-bucket variant, collapsing daily columns into monthly totals). Timing Flexibility (Four Mini Tasks) — `Task3 Data!I39/I40/I41` (interval-boundary variant via spilled `SUMIFS`, documented side by side with the time-bucket variant below).

**When it applies:** you need a groupby-sum — collapsing an n-row/column detail array down to an m-row/column total per category (supplier, product, retailer, calendar month) — and you want the *entire* aggregated result in one shot rather than one cell at a time.

**Template — flat labels already line up with detail rows (confirmed, `Inventory!J59` / `Purchasing Plan!J66`):**
```
LET(
  ingredients,      J53#,
  suppliers,        $F$86#,
  current_supplier, E66#,
  match_matrix,     --( suppliers = TRANSPOSE( current_supplier ) ),
  MMULT( TRANSPOSE( match_matrix ), ingredients ) )
```

**Template — hierarchical variant, categories live in header rows not per-row labels (confirmed, `Purchasing Plan!J108` cost, `J75` kg):** the raw label column mixes product headers, ingredient names, and blank/total rows — every ingredient row needs to inherit the product name of the header above it before MMULT will work. Three-stage build:
```
LET(
  ingredients,        $J$8:$U$46,
  labels,             $E$8:$E$46,
  ingredient_list,    $E$86#,
  list_price,         $H$86#,
  is_ingredient_row,  COUNTIF( ingredient_list, labels ) > 0,

  current_row,         FILTER( labels, is_ingredient_row ),
  current_ingredients, FILTER( ingredients, is_ingredient_row ),
  value,               current_ingredients * XLOOKUP( current_row, ingredient_list, list_price ),

  product_list,    $E$108#,
  product_tag_all, SCAN( "", labels, LAMBDA(carry,lbl, IF( COUNTIF( product_list, lbl ) > 0, lbl, carry ) ) ),
  product_tag,     FILTER( product_tag_all, is_ingredient_row ),

  match_matrix, --( product_tag = TRANSPOSE( product_list ) ),
  MMULT( TRANSPOSE( match_matrix ), value ) )
```
Stage 1 (`COUNTIF`+`FILTER`) compresses the messy label block down to just the real ingredient rows. Stage 2 (`SCAN` forward-fill — Pattern F) derives a product tag for every row, including header/blank rows. Stage 3 filters that tag array down to the same rows, builds the membership matrix, and runs the identical MMULT. Confirmed fully composable: `J75`'s kg version (which reads its ingredient list from `E53#`) drops the `list_price`/`value` step and feeds `current_ingredients` straight into the identical MMULT call — the tagging/matrix logic is agnostic to what's being summed (cost, volume, count).

**Template — time-bucket variant (confirmed, `Revenue!J30`):**
```
MMULT( ANCHORARRAY(J7), --( TRANSPOSE(ANCHORARRAY(Time!J12)) = ANCHORARRAY(Revenue!J28) ) )
```
`Time!J12#` is the daily date spine; `Revenue!J28#` is that same spine's `EOMONTH` bucket for each day. The one-hot matrix here isn't a category match at all — it's "does this day fall in this month" — confirming the mechanism collapses along the *time* axis exactly as well as a categorical one.

**Mechanism:** broadcasting an n×1 vector against a 1×m vector (`rows = TRANSPOSE(columns)`) produces an n×m boolean grid where entry `(i,j)` is `TRUE` exactly when detail row `i` belongs to group `j`. `--(...)` coerces that to a 1/0 **one-hot membership matrix**. `MMULT(TRANSPOSE(match_matrix), detail)` then sums `detail[i]` over every `i` where membership is 1, for every group `j`, simultaneously — a groupby-sum expressed as linear algebra, no `SUMIFS`, no `GROUPBY`, no per-cell `SUMPRODUCT`.

**Why it belongs next to Pattern D, not Pattern A:** like Pattern D, this has no recursion in the aggregation step itself — it's a *relational* many-to-one collapse. But where Pattern D's `SUMPRODUCT` computes one output cell at a time (needs a `MAKEARRAY` wrapper to fill a grid), MMULT produces the **whole aggregated result vector in a single array operation**.

**Uniqueness discipline:** the category list must contain each category exactly once across its own axis. Duplicates let a detail row double-count into two matching columns — same caution as the non-unique-label gotcha below.

### Pattern E, interval-boundary variant — spilled `SUMIFS` against a lagged timeline (side by side with the time-bucket variant)

**Where:** Timing Flexibility (Four Mini Tasks) — `Task3 Data!I38` (timeline), `I39` (revenues), `I40` (expenses), `I41` (operating profit), `G39:G41` (row totals). Single workbook so far — provisional until a second case confirms it.

**When it applies:** the same job as Fashion Boutique's `Revenue!J30` — collapse a fine-grained timeline (weekly) onto a coarser one (month / quarter / year) — but where the coarse timeline is itself a spilled array whose *length* changes with a frequency switch, and the detail columns carry no precomputed period key.

**The timeline feeding it (`Task3 Data!I38`):**
```
LET(
  years,    YEAR( end_date ) - YEAR( start_date ) + 1,
  months,   years * 12,
  quarters, years * 4,

  IF( period = 1,
      EOMONTH( start_date, SEQUENCE( 1, months, 0, 1 ) ),
      IF( period = 2, EOMONTH( start_date, SEQUENCE( 1, quarters, 2, 3 ) ),
          EOMONTH( start_date, SEQUENCE( 1, years, 11, 12 ) ) ) ) )
```
`period` is a workbook name, `VALUE( LEFT( 'Task3 Data'!$G$23, 1 ) )`, reading the leading digit of the dropdown label. Convention flags: `+ 1` because `YEAR(end) − YEAR(start)` counts elapsed years, not calendar years spanned; the `EOMONTH` offsets from a 1 January start are 0 (month-end), 2 (quarter-end), 11 (year-end). Both assume the timeline starts on 1 January of a calendar year — a mid-year start would misalign the offsets.

**Template (confirmed, `Task3 Data!I39`):**
```
LET(
  date_range, I38#,
  dates,      $I$30:$JI$30,
  data_range, $I$31:$JI$31,
  prev_range, HSTACK( 0, DROP( date_range, 0, -1 ) ),

  SUMIFS( data_range, dates, "<=" & date_range, dates, ">" & prev_range ) )
```
`I40` is identical with `data_range, $I$32:$JI$32` and the result multiplied by `-1` (expenses shown negative). Operating profit and row totals:
```
I41 = BYCOL( I39#:I40#, SUM )
G39 = BYROW( I39#, SUM )
```

**Mechanism:** three moves.
1. **Array criteria spill.** `SUMIFS` keeps its sum_range and criteria_range at detail length (261 weeks), but when the *criteria* argument is an array (`"<=" & date_range`, one string per period), it evaluates once per element and spills one sum per period. The output length is driven by `I38#`, so the row resizes to 60 / 20 / 5 columns with the switch — no drag, no excess columns to blank out.
2. **Pattern I's lag, applied to boundaries instead of balances.** `HSTACK( 0, DROP( date_range, 0, -1 ) )` is the one-period shift from Pattern I: each period's lower bound is the previous period's end, with a 0 seed for the first period. Each period is the half-open interval `(prev_end, end]`, so every week lands in exactly one period. The period-start row is the same lag plus one day.
3. **Range operator between two spills.** `I39#:I40#` returns the bounding range of two adjacent, equal-width spills, so `BYCOL(…, SUM)` adds revenue and expenses column by column and spills with them.

**Side by side with the time-bucket variant:**

| | Time-bucket `MMULT` (Fashion Boutique `Revenue!J30`) | Interval `SUMIFS` (Timing Flexibility `Task3 Data!I39`) |
|---|---|---|
| Membership test | Equality: detail key = period key | Interval: `prev_end < date ≤ end` |
| Needs a key row per detail column | Yes — `Revenue!J28#`, the `EOMONTH` bucket of each day | No — boundaries come from the timeline itself |
| Rows aggregated per formula | Many at once (k×n · n×m) | One data row per call |
| Accepts computed (in-`LET`) arrays as inputs | Yes | Only in the criteria argument; sum_range and criteria_range must be range references |
| Intermediate object | Full n×m one-hot matrix | None — native aggregation engine |
| Frequency switch | Key row must be rebuilt for the new frequency | Automatic, via `I38#` |

**The bridge between the two — what actually differs is the membership test, not the engine.** `MMULT` accepts the interval test directly, which delivers both data rows in one spill and works on computed arrays (not built in the workbook — untested; expenses would need their sign applied afterwards):
```
LET(
  date_range, I38#,
  dates,      $I$30:$JI$30,
  data_range, $I$31:$JI$32,
  prev_range, HSTACK( 0, DROP( date_range, 0, -1 ) ),
  membership, ( TRANSPOSE( dates ) <= date_range ) * ( TRANSPOSE( dates ) > prev_range ),

  MMULT( data_range, membership ) )
```
Choice rule: one data row from a sheet range, boundaries already on the sheet → spilled `SUMIFS`. Several rows at once, or inputs that are themselves computed arrays → `MMULT` with whichever membership test (equality or interval) the period definition calls for.

---

## Pattern F — `SCAN` as forward-fill: carrying a header label down to its detail rows

**Where:** Sweet Dreams — `Purchasing Plan!J75` and `J108` (feeding Pattern E's hierarchical variant), `Inventory!J26` and `J43` (`retailer_fill`).

**When it applies:** a label column has category headers sitting *above* the rows they govern (e.g. "Retailer 2" on its own row, followed by several product rows with no repeated label), and you need every detail row to carry its governing category as an ordinary value before you can group or match on it.

**Template (confirmed, `Inventory!J26`):**
```
SCAN( "", current_row, LAMBDA(carry, lbl, IF( COUNTIF(retailer_list, lbl) > 0, lbl, carry ) ) )
```

**Mechanism:** a third distinct job for `SCAN`, alongside compounding (A/B) and conditional reset/replenish/hold (C) — **carry-forward / last-observation-fill**. At each row, the lambda checks whether the current label *is itself* a recognized category name. If yes, that becomes the new carried value; if no (a product name, a blank row, a "Total" row), the accumulator just keeps whatever category it last saw. Genuinely reusable any time a sheet has the "header row, then several detail rows" layout — the default layout for every AR/AP/Inventory subledger block in the Working Capital cases.

---

## Pattern G — group subtotal via forward-filled key and `SUMPRODUCT` equality match

**Where:** Sweet Dreams — `Inventory!J43`.

**When it applies:** every row in a group — the header/subtotal row itself, not just the detail rows — needs to display that group's total, without physically collapsing the layout down to one row per group (that's Pattern E's job instead).

**Template (confirmed):**
```
MAKEARRAY( ROWS(current_row), COLUMNS(kg),
  LAMBDA(r, c,
    IF( INDEX(current_row,r,1) = "", "",
      IF( INDEX(is_retailer_row,r,1),
        SUMPRODUCT( ( retailer_fill = INDEX(retailer_fill,r,1) ) * INDEX(kg,0,c) ),
        INDEX(kg,r,c) ) ) ) )
```

**Mechanism:** `retailer_fill` (Pattern F's forward-fill) gives every row — header and detail alike — the name of the group it belongs to. The subtotal shown on a header row is `SUMPRODUCT` of every row whose `retailer_fill` matches *that row's own* `retailer_fill` value, times the per-row metric — an equality-based grouped sum, structurally the same move as Pattern D's `SUMPRODUCT((rank_column < r) * demand_column)`, except the condition is "same group" (`=`) rather than "outranks me" (`<`). Non-header rows just pass their own value through unchanged via the inner `IF`.

**Why it earns its own entry rather than living under F:** Pattern F only fills a *label* forward. Pattern G uses that filled label as a grouping key to aggregate a *second* array, broadcast back onto every row of the group rather than collapsed away.

---

## Pattern H — payment-terms / aging amortization triangle (closed-form AR, AP, deferred balances)

**Where:** Sweet Dreams — `Accounts Payable!J13/J14/J27` (and four mirrored supplier blocks), `Accounts Receivables!J13/J14/J29` (and mirrored retailer blocks).

**When it applies:** you need the closing balance of a receivable/payable as a function of *when* each transaction occurred and a fixed (possibly fractional) term — instead of rolling it forward period by period.

**Template (confirmed, generalized from `Accounts Payable!J14`/`J27`):**
```
LET(
  calendar_month, SEQUENCE(1, periods, 1, 1),
  txn_month,      txn_month_range,
  term,           payment_term_in_months,

  elapsed, calendar_month - txn_month,
  weight,  IF( elapsed < 0, 0,
             IF( elapsed < INT(term), 1,
               IF( elapsed < term, MOD(term, 1), 0 ) ) ),

  BYCOL( weight, LAMBDA(month_col, SUMPRODUCT(txn_amount, month_col)) ) )
```

**Mechanism:** `weight[i,j]` is the fraction of cohort `i`'s amount still outstanding as of month `j`: 0 before the transaction exists, fully outstanding for whole months the term covers, one fractional band for the month the term expires, then 0. `BYCOL` collapses the cohort × month triangle into one total-outstanding figure per month. No `SCAN`, no period walk — closing balance computed directly from the full history and a single elapsed-time comparison.

**Confirmed symmetric use:** identical structure drives AP (purchase orders → cash paid) and AR (sales → cash collected) — same template, different transaction array and term.

**When it stops applying:** the moment a flow's fate depends on something *other* than pure elapsed time against a fixed term — e.g. FIFO layering (which units were sold from which vintage), or a replenishment event — the closed form breaks down and you're back to Pattern C, J, or L.

---

## Pattern I — period-lag opening balance (`HSTACK`+`DROP` shift), connecting flow solved as a residual

**Where:** Sweet Dreams — `Accounts Receivables!J61/J63/J64`, `Accounts Payable!J138/J140/J141`, `Inventory!J67/J68/J70`. Thermin — `Calculations!J19` (inventory opening balance), `J98`, `J173`. Galway Inc — `Calculations!J32`, `J75`, `J94` (PP&E opening balances). Big Dog and Rock to Mineral — `Answers!J9/J16/J23` (opening balance per asset class). Stanbridge — `Existing!J9/J37/J65` (PP&E opening-balance shift). United Landlords — `Calculations!J124` (cash opening balance). Wichita Warblers — `Calculations!J28` (de-cumulating variant, below). Sweet Dreams — `Inventory!J70` (lead variant, below).

**When it applies:** you already have a closed-form (or otherwise independently computed) *closing* balance for every period, and you need (a) the matching opening balance, and (b) a connecting flow that reconciles the two, without introducing a second `SCAN`.

**Template — opening balance as a one-period lag of the closing-balance array (confirmed, `Stanbridge!Existing!J9`):**
```
DROP( HSTACK( seed_value, ANCHORARRAY(closing_balance_row) ),, -1 )
```

**Mechanism:** `HSTACK` prepends a single seed value (the true period-0 opening balance) onto the closing-balance array; `DROP(...,,-1)` removes the last element. The result is the same width as the original, where every entry is simply "the previous entry of the closing array."

**Template — the connecting flow, solved as an algebraic residual (confirmed, `Accounts Payable!J140`):**
```
( opening_balance + independently_known_flow - closing_balance ) * -1
```

**Mechanism:** once opening and closing are both known — one closed-form, the other from the shift above — and one of the two flows in the roll-forward is independently known, the *other* flow is just whatever balances the equation.

**Confirmed to generalize well beyond working capital:** Stanbridge reuses exactly this shift for three separate PP&E roll-forwards (existing assets, a straight-line block, and a declining-balance block), each paired with an unconditional `SCAN` (Pattern J) for the balance itself — confirming Pattern I is a general-purpose companion to *any* non-recursive or SCAN-derived closing-balance row, not something specific to working capital.

**De-cumulating variant — period flow as the difference of a closed-form cumulative (Wichita Warblers, `Calculations!J25:J28`):**
```
J26 = SCAN( 0, cash_from_sales, LAMBDA( a, cash, a + cash ) )             \\ cumulative cash (Pattern J)
J27 = cumulative_cash * pct_games_played                                  \\ cumulative revenue earned
J28 = LET(
        current_period_revenue, J27#,
        prior_period_revenue,   HSTACK( 0, DROP( current_period_revenue,, -1 ) ),
        current_period_revenue - prior_period_revenue )
```
Revenue recognition is defined on cumulatives — revenue earned to date equals cash collected to date × the share of games played (`J25`, Pattern C Variant 2). The period figure is then the cumulative minus its own one-period lag. Pattern I's shift is used here to turn a stock back into a flow, not to build an opening balance. Unearned revenue is the gap between the two cumulatives (`J30 = J26# − J27#`), so no separate roll-forward is needed. Whenever a rule is stated on totals to date (percentage of completion, cumulative caps), compute the cumulative in closed form and de-cumulate with the lag; don't fold the period flows.

**Lead variant — next period's value, the mirror of the lag (Sweet Dreams, `Inventory!J70`):**
```
LET(
  next_month_inventory, DROP( INDEX( J59#, MATCH( $B65, E59#, 0 ), 0 ),, 1 ),
  last_month_inventory, INDEX( next_month_inventory, 1, periods - 1 ),
  inventory_sold,       HSTACK( next_month_inventory, last_month_inventory ),
  inventory_sold * target_inventory )
```
Closing inventory is a target share of *next* month's sales. `DROP( x,, 1 )` removes the first column, so every position now holds the following period's value. `HSTACK( …, last )` pads the final period by repeating the last available month, which keeps the row at full width. The lag pads at the start with a seed, the lead pads at the end with a terminal value, and the case decides which terminal value to use; here it is the last month's sales held flat. Production is then the residual (`J68 = − sold + ( closing − opening )`), as in the template above.

---

## Pattern J — plain unconditional roll-forward `SCAN`

**Where:** Thermin — `Calculations!J22` (inventory rollforward), `J102`, `J177`. Wichita Warblers — `Calculations!J26` (cumulative cash), `J46` (unearned revenue). Galway Inc — `Calculations!J37`, `J80`, `J99` (PP&E rollforward). Big Dog — `Answers!J12/J19/J26` (fixed-asset net book value). Rock to Mineral — `Answers!J12/J19/J26` (same layout as Big Dog). Stanbridge — `Existing!J12/J41/J69`, `Expansion!J26`, `Overlay!J26` (net PP&E rollforward). United Landlords — `Calculations!J126` (closing cash).

**When it applies:** a balance must be built up period by period with no closed-form shortcut available, because the "movements" driving it (additions, depreciation, sales) are already independently known per period and just need to be accumulated — typically because flows are order-dependent (FIFO) or event-driven in a way not reducible to a fixed term.

**Template (confirmed, `Thermin Calculations!J22`):**
```
LET(
  movements, addition_row + reduction_row,
  SCAN(opening_balance, movements, LAMBDA(s,c, s+c)) )
```

**Mechanism:** the degenerate, unconditional case of the A/B/C family — no compounding, no reset, no counter, just a plain cumulative sum. It's the pattern to fall back on once you've ruled out Patterns D, E, G, H (closed-form/filter reformulations) and confirmed the flow really is cumulative with no shortcut. The single most-reused pattern in the catalog — confirmed across seven cases (Thermin, Wichita Warblers, Galway Inc, Big Dog, Rock to Mineral, Stanbridge, United Landlords) in four different domains (inventory units, cumulative cash, NBV, PP&E), always the same shape: `SCAN(seed, movements, SUM)`.

**Decision flag versus Pattern H:** if a balance's fate can be described purely as "how long has this transaction been outstanding relative to a fixed term," reach for Pattern H (no `SCAN`, closed form) and Pattern I for the opening-balance/residual scaffolding around it. Reach for Pattern J only when this period's flow genuinely depends on *which* prior units/cash it's drawing down (FIFO ordering) or on an event-triggered reset/floor (Pattern C) — i.e. the update rule itself must branch on running state, and a fold is unavoidable.

**Pairing with Pattern I, uncapped case (United Landlords, `Calculations!J124` / `J126`):** closing cash is `SCAN( 0, J125#, SUM )` and the opening balance is `DROP( HSTACK( 0, J126# ),, -1 )`. This is a single-row recurrence with no `MIN`/`MAX` cap, so each row stays one spill and Pattern K's circularity never arises: `J124` reads `J126#`, `J126` reads `J125#`, and `J125#` never reads `J124`. It is the boundary case that shows what *does* stay on the single-formula side of Pattern K — the opening balance is derived from the closing balance, never the other way round.

---

## Pattern K — coupled multi-row recursion: when a single spilled formula can't hold the whole recurrence

**Where:** Waterworks Corp — Demand waterfall / Inventory rollforward (Opening → Production → Available for sale → Units sold → Closing; Backlog bridge: Unmet demand → Units permanently lost).

**How this differs from everything above:** every pattern A–J is a recipe for one self-contained formula, entered once and spilled across all periods. Pattern K is the first case where the recurrence needs *two or more state variables* — e.g. an inventory balance and a demand backlog — that reference *each other's* previous value through a `MIN`/`MAX` cap. That's a structurally different problem, not a harder version of Pattern J.

**Why Excel refuses the naive version:** when each named row (Opening, Available, Units sold, Closing) is its own spilled formula, Excel's circularity checker sees the dependency graph between the *named ranges* — `Opening#` feeds `Available#`, which feeds `Units sold#`, which feeds `Closing#`, which feeds `Opening#` again. That's a cycle, even though the real data only ever reaches back one column.

**The fix — and only one of two options actually works:**
- ~~Pack the coupled state into one `SCAN` with a tupled accumulator~~ — tempting, but doesn't survive: `SCAN`'s output shape contract doesn't support returning a bundled multi-row state cleanly across a spill. Confirmed non-working by live test; don't spend exam time on this route. (Recognition-only entry — flagged clearly as "does not work.")
- **Drop the spill-formula style for this block and write classic per-column formulas**, each pointing only to the column on its left. No named-range dependency graph exists for Excel to check, so there's nothing to reject. You trade the "enter once" idiom for full row-by-row auditability — which, for a coupled-state block, is the correct trade anyway.

**Generalizable takeaway:** count how many *named rows* a recurrence actually touches before writing anything. One row referencing its own history → Patterns A–J apply cleanly. Two or more rows referencing *each other's* prior-period values (typically via a `MIN`/`MAX` cap) → stop trying to force each row into its own spilled formula, and don't waste time on a packed-tuple `SCAN` — go straight to classic per-column formulas.

**Implementation detail:** the worked in-cell formulas, the disproven tupled-SCAN attempt in full, and the gotcha checklist that came out of actually building one of these by hand live in the companion file `Working_Capital_Corkscrew_Formulas_CFM.md` — case-implementation mechanics, not a reusable dynamic-array recipe, so kept out of the main catalog.

---

## Pattern L — FIFO/vintage layering via classic triangular `MIN`-with-expanding-range (deliberately *not* a dynamic array formula)

**Where:** Thermin — `Calculations!J74:AG79` and `Calculations!J151:AG156` (two separate vintage-inventory FIFO allocation blocks).

**When it applies:** a flow is genuinely order-dependent — which vintage's inventory gets drawn down first to meet demand — which Pattern H explicitly rules out and which even Pattern C or J only handle awkwardly (a plain `SCAN` roll-forward can track a single balance's total, but not *which layer* it came from). Rather than force this into `SCAN`/`MAKEARRAY`, Thermin drops to classic, non-array Excel: each cell is its own ordinary formula, filled right and down, referencing an *expanding range* on its own row.

**Template (as filled across a row, vintage on rows / period on columns):**
```
' First column of each vintage row:
= MIN( <this vintage's available balance from the prior period's inventory-added row>,
       <period demand> - ( <this-vintage-row, prior columns> ) )

' e.g. concretely:
J74 = MIN( I85, J$53 - (J$73:J73) )
K74 = MIN( J85, K$53 - (K$73:K73) )
L74 = MIN( K85, L$53 - (L$73:L73) )
```

**Mechanism:** for vintage row 74 (units added in year 1) and period column K, the cell computes "how much of this vintage is left to draw on" (`J85`, the prior period's remaining balance for that vintage) capped by "how much demand is still unmet after every *other* vintage's allocation so far this period" (`K$53` total demand, minus the running total already allocated across other vintage rows for that same period, expanding as you move right: `J$73:J73` → `K$73:K74`). The expanding range is doing exactly what a fold would do — accumulating "what's already been claimed" — but via ordinary relative references instead of `SCAN`, because the recurrence here runs in *two* directions at once (down each vintage's own balance, and across each period's shared demand pool), which doesn't fit `SCAN`'s single-direction contract.

**Generalizable takeaway, paired with Pattern K:** Pattern K showed that when a recurrence couples multiple *rows*, the fix is to drop the single-spilled-array idiom for classic per-column formulas. Pattern L is the same lesson one level further: when a recurrence couples *both* an intra-row dependency (this vintage's remaining balance) *and* a cross-row dependency (competing for the same period's demand pool), don't force it into `MAKEARRAY`/`SCAN` just because every other pattern in this catalog is a dynamic array formula. A plain triangular grid, filled the old-fashioned way, is often the clearest and most auditable answer — and it's a legitimate exam-time choice, not a fallback of last resort.

### Pattern L — second domain (JAK Holdings, BCS Holdings): tax-loss carry-forwards by vintage, with expiry

**Where:** JAK Holdings — `Calculations!J36:N41` (utilisation: total in row 36, one row per loss vintage in 37–41), `J47:N51` (closing balance per vintage), row 42 and row 52 (usage and balance of the "expiring thereafter" pool, opening balance in `I52`). BCS Holdings — the same block at `Calculations!J43:N49` and `J54:N59`, pool opening balance in `I59`. Both blocks as shown here are the versions corrected on 11 October 2026 (see the two bugs below). Both are classic per-cell formulas; neither workbook has a dynamic-array version.

**Template (as filled, JAK, column K):**
```
K36 = MIN( K16, SUM( J47:J52 ) )                      ' total used: income, capped by every pool open at the end of last year
K37 = MIN( K36, J47 )                                 ' oldest vintage first
K38 = MIN( J48, K$36 - SUM( K$37:K37 ) )              ' each later vintage takes what is left
K42 = MIN( J52, K$36 - SUM( K$37:K41 ) )              ' the undated pool goes last
K47 = IF( K$45 < YEAR( $E47 ), J47 - K37, 0 )         ' carry the balance forward until the vintage's expiry year, then zero
K52 = J52 - K42                                       ' the undated pool never expires
```

**Mechanism:** the structure of the Thermin block, applied to taxes. Down each column, the expanding `SUM( K$37:K37 )` hands the year's utilisation to the vintages oldest first. Across each row, the balance carries forward from the previous column. The extra element is the expiry test `IF( year < expiry, …, 0 )`: a vintage drops out in its expiry year whether or not it was used.

**Relation to Pattern Q:** within one year, the vintage allocation is the clipped-difference identity of Pattern Q (see "Same identity, other places" there): `used_k = MAX( 0, MIN( total, ceiling_k ) − ceiling_{k−1} )`, with the cumulative opening balances as ceilings. The carry across years is still a recurrence, because each year's balances depend on the previous year's usage. That is why the whole block stays in Pattern L.

**Bug 1, fixed 11 October 2026 — the undated pool is a vintage too.** As first built, the "expiring thereafter" pool was a constant `SEQUENCE` spill (`I52#`, `I59#`), so it never got a reduced balance. The two workbooks went wrong in two different ways:
- **JAK:** the pool counts in the cap `SUM( J47:J52 )`, but no utilisation row draws from it. In 2022 the total `M36` is 5,106.91 while rows 37–41 allocate 706.91; the other 4,400 is the pool, used without being reduced. In 2023 `N36` uses the same 4,400 again (rows 37–41: 0). The key uses the pool once: 4,400 in 2022, then 0 (`CaseSolution` row 94; balance row 103 goes 4,400 → 0). 2023 utilisation is overstated by 4,400.
- **BCS:** row 49 does draw from the pool (`MIN( I59, … )`), but row 59 never falls. It uses 492.89 in 2026 and the full 6,600 again in 2027. The key uses 492.89, then 6,107.11 (`CaseSolution` rows 71 and 80). 2027 utilisation is overstated by 492.89.

Fix applied: `I52` / `I59` now return the opening balance as one number; JAK row 42 is the pool's usage row, last in FIFO order; rows 52 / 59 are `previous − used`, with no expiry test. Check: in every column, total utilisation = sum of the vintage rows, and each balance row = previous − used.

**Bug 2, fixed 11 October 2026 — the expiry test compared a year with a date.** The balance rows read `IF( K$45 < $E47, … )`. Row 45 holds year numbers (2019…2023), while `E47:E51` hold dates (31 December of the expiry year, displayed as "Expiring in Fiscal 2021"). A date is a serial number around 44,000, so `2021 < 44561` is always true and no vintage ever expired. The U.S. never shows it, because no U.S. vintage would have expired in time to matter. Other countries do: in JAK, Germany kept using its 2021 vintage (115,674 at the start) through 2023, so it showed no current tax in 2022 against the key's 7,249.43. Mexico, and Italy in BCS, were wrong in the same way. Fix applied: `YEAR( $E47 )` in all 25 balance cells of each workbook (JAK `J47:N51`, BCS `J54:N58`). After both fixes, the current, deferred and total tax tables in USD match `CaseSolution` for every country in both workbooks (JAK: U.S., Canada, Mexico, Germany; BCS: U.S., Canada, Italy). General rule: when a header row holds dates but displays years, compare `YEAR( date )` with the year, never the date itself — like the other silent failures in the gotchas list, it raises no error.

---

## Pattern M — trailing-window slice via `TAKE`/`HSTACK`, replacing an `OFFSET`-anchored construction

**Where:** Streamwise — `Debt!H157` (Revolver TTM interest) and `Debt!H158` (Term Loan TTM interest), confirmed tying out exactly to `CaseSolution!E171` (1,360.101375) and `CaseSolution!E172` (9,914.309625), which used `OFFSET`.

**Why it's in this catalog at all, despite having no recursion:** the original construction (`OFFSET` with a hardcoded anchor cell and a negative, computed column offset) *looked* like it belonged in the "backward-looking" family this catalog covers, and the instinct to reach for a dynamic-array replacement was correct — but the actual fix is a **static slice of a concatenated array**, not a fold. Worth documenting precisely so the difference doesn't get blurred with Patterns A–L.

**What it replaces:**
```
=SUM($E$149:E149)+SUM(OFFSET($T$35,0,-(12-COUNT($E$162:E162)),1,12-COUNT($E$162:E162)))
```

**Confirmed template (`Debt!H158`, Term Loan TTM interest):**
```
LET(
  historicals, CaseData!$E$36:$S$36,          \\ real history only — see the boundary note below
  forecast,    Debt!$H$119:$M$119,

  hist_count,  COLUMNS( historicals ),
  fcst_count,  COLUMNS( forecast ),
  full,        HSTACK( historicals, forecast ),

  MAKEARRAY( 1, fcst_count,
    LAMBDA(i, j,
      SUM( TAKE( TAKE( full, , hist_count + j ), , -12 ) )
    ) ) )
```

**Mechanism:** `HSTACK(historicals, forecast)` concatenates historical actuals and the forecast into one continuous timeline, eliminating the need for a hardcoded anchor cell. `j` is simply MAKEARRAY's column index — "how many forecast periods have elapsed, including this one" — playing the same role a `COUNT(...)` of already-filled forecast cells played in the OFFSET version, but delivered directly rather than re-derived by counting non-blank cells. `TAKE(full, , hist_count+j)` takes "all of history plus the first `j` forecast periods" — a single position marker into the concatenated array. The outer `TAKE(..., , -12)` clips that growing prefix to its last 12 entries — the trailing window — regardless of how far along `j` is, whether the 12 months end up all-historical, all-forecast, or a mix.

**The bug that made the first attempt fail, and the actual fix:** `TAKE`'s arguments are `TAKE(array, rows, [columns])` — the *first* numeric argument is rows, not columns. On a 1×n row vector, `TAKE(full_slice, -12)` requests **-12 rows** from an array that only has 1 row — a silent no-op that returns the whole slice unchanged, not the last 12 columns. It doesn't error; it just quietly sums every historical month instead of the trailing 12, and the result still looks like a plausible TTM figure until you tie it out. The fix is `TAKE(full_slice, , -12)` — leave the rows argument blank and pass `-12` as the columns argument. This is now gotcha #8 below; treat it as a standing rule any time `TAKE`/`DROP` are applied to a single-row or single-column array.

**A second, unrelated bug the same rebuild surfaced — know your true historical boundary.** `CaseData` in this case carries columns **T:Y** as its *own* separately-computed forecast (a different rate/balance path), not real actuals — even though they sit immediately to the right of the real history in **E:S**. The first version of this formula read `historicals` all the way through column Y, silently double-counting six forecast months as if they were history. Before building `full = HSTACK(hist, fcst)`, confirm exactly where genuine historical data ends — don't assume "everything before the model's own forecast starts" is history; a source tab can have its own forecast tacked on the end that looks like more history at a glance.

**Why this isn't a SCAN/fold pattern:** nothing here depends on a *prior cell's output* — `j` is a positional index, not carried state. This is a "growing window over a concatenated timeline," a pure slicing operation. `TAKE(TAKE(x, , n), , -12)` is the general idiom for "last 12 of the first n columns" and is reusable for any rolling trailing-window (TTM, trailing-quarter, rolling average) without OFFSET's anchor-cell scaffolding.

**`TAKE` vs `DROP` for the trailing clip:** `TAKE(array, , -12)` keeps the last 12 columns directly. `DROP(array, , -12)` does the opposite — it *removes* the last 12 and keeps the rest. To get "last 12" via `DROP` you'd need `DROP(slice, , COLUMNS(slice) - 12)`, which requires computing the discard size from the total length first. Rule of thumb: use `TAKE` when you know the size of the result you want; use `DROP` when you know the size of the piece to discard and don't care how much remains (e.g. "skip the header row" — `DROP(array, 1)`).

### Pattern M — second occurrence (United Landlords, `CaseSolution!F222`): trailing window ending at a looked-up date

**Where:** United Landlords — `CaseSolution!F222`, `=SUM( OFFSET( INDEX( G17:DV17, MATCH( D222, G3:DV3, 0 ) ), 0, 0, 3, -12 ) ) * C222` — trailing-twelve-month NOI at the sale date, times the valuation multiple. The same OFFSET-anchored construction as Streamwise, in a different domain.

**Replacement (not yet typed into this workbook — `Calculations` has no NOI row or sale-price cell):**
```
LET(
  NOI,         $J$22# + $J$79# - $J$32# - $J$94# - $J$103#,
  timeline,    $J$3#,
  sale_date,   CaseData!$N$7,
  multiple,    CaseData!$N$6,

  sale_index,  MATCH( sale_date, timeline, 0 ),
  multiple * SUM( TAKE( TAKE( NOI, , sale_index ), , -12 ) ) )
```
**What differs from Streamwise:** the window's right edge is a *date lookup* (`MATCH( sale_date, timeline, 0 )`) rather than MAKEARRAY's column index `j`, and the result is a single scalar rather than an array. The slicing idiom is unchanged, including the blank rows argument (gotcha 8). NOI here excludes land and construction (revenue less OPEX less management fee), matching `CaseSolution!G17`.

**Tie-out, computed outside Excel from the workbook's own numbers:** last-twelve-month NOI to December 2031 of 96,446,602.46 × 20 = 1,928,932,049.10, equal to `CaseSolution!F222`. Pattern M stays provisional until the formula above is entered and confirmed in Excel.

### Pattern M — third occurrence (YHF, `Debt!J159` / `J160`): the window that never moves

**Where:** YHF — `Debt!J159` (revolver TTM interest), `J160` (term-loan TTM interest). Same case family and the same `CaseSolution` OFFSET construction as Streamwise (`CaseSolution!D170:I171`), and the Streamwise template copied over, with one character changed:
```
MAKEARRAY( 1, fcst_count,
  LAMBDA( i, j, SUM( TAKE( TAKE( full, , hist_count + 1 ),, -12 ) ) ) )      \\ YHF: + 1
  LAMBDA( i, j, SUM( TAKE( TAKE( full, , hist_count + j ),, -12 ) ) ) )      \\ Streamwise: + j
```
**Effect (fixed 11 October 2026):** the lambda never read `j`, so every forecast column got the first month's trailing window (`J160` showed 9,914.31 in all six months). With `+ j` the window moves, and the formula is proven: fed the key's own monthly interest (`CaseSolution` rows 148–149), the same logic reproduces the key's trailing-twelve-month rows 170–171 exactly in all six months. The workbook still differs from the key after month 1 (term loan 9,816.28 vs 9,813.46 in month 2, rising to 9,438.78 vs 9,391.94), and the revolver differs by 1.88 already in month 1. Both come from the monthly interest *inputs*, not the slice. Where the workbook's debt schedule differs from the key: term-loan repayment is a share of the beginning balance (`K71 = K65 * $F$70`, 2,179.69 in month 1) instead of the key's fixed 2,812.50 a month; total interest accrued (`K154 = SUM( K151:K153 )`) leaves out the revolver row 150; the subordinated notes are retired on 31 August 2020 (`$F$92`) instead of the key's 31 July 2020 call at 102 into 510,000 of new senior notes. These change free cash flow, hence the revolver balance and its interest, hence the trailing window. Month 1 ties for the term loan because its first month's interest uses the opening balance only.

---

## Pattern N — event-window flags and a per-series elapsed-month counter (2-D gated counter, no `SCAN`)

**Where:** United Landlords — `Calculations!J54` (construction: `construction_duration` window flag → `elapsed_month` → bell-curve lookup), `J69` and `J84` (operation: `operation_flag` → `elapsed_month`). Eagle Security — `Calculations!J10`, `J20`, `J30` (`model_period_flag` → `period_count`, with a separate `operation_flag`). The prefix-sum counter is confirmed in two workbooks; the end-of-window flag and the profile lookup appear in United Landlords only and stay provisional.

**When it applies:** several parallel series (properties, projects) each have their own start date and duration on one shared date spine, and each needs (a) an on/off flag per month and (b) the count of active months so far — "month of construction", "month of operation" — to index a profile table.

**Template (confirmed, `Calculations!J54`):**
```
LET(
  timeline,                  $J$3#,
  property_data,             CaseData!$A$15:$H$24,
  purchase_date,             INDEX( property_data,, 6 ),
  preparation_months,        INDEX( property_data,, 7 ),
  months_of_construction,    INDEX( property_data,, 8 ),
  delay,                     IF( $F$9 = 0, 0, 12 ),

  construction_start,        MAKEARRAY( ROWS( property_data ), 1,
                               LAMBDA( r, c, EOMONTH( INDEX( purchase_date, r ), INDEX( preparation_months, r ) + delay ) ) ),
  construction_duration,     MAKEARRAY( ROWS( property_data ), COLUMNS( timeline ),
                               LAMBDA( r, c, IF( ( INDEX( timeline,, c ) >= INDEX( construction_start, r, 1 ) ) *
                                                 ( INDEX( timeline,, c ) < EOMONTH( INDEX( construction_start, r, 1 ), INDEX( months_of_construction, r ) ) ), 1, 0 ) ) ),

  construction_bell,         CaseData!$B$30:$N$53,
  construction_length,       CaseData!$B$29:$N$29,
  construction_month,        CaseData!$A$30:$A$53,
  elapsed_month,             MAKEARRAY( ROWS( property_data ), COLUMNS( timeline ),
                               LAMBDA( r, c, SUM( INDEX( construction_duration, r, SEQUENCE( 1, c ) ) ) ) ),
  POC,                       IFNA( INDEX( construction_bell, MATCH( elapsed_month, construction_month, 0 ),
                                                             MATCH( months_of_construction, construction_length, 0 ) ), 0 ) * construction_duration,

  locked_factor,             XLOOKUP( construction_start, timeline, inflation_factor ),

  ( construction_costs * locked_factor ) * POC )
```
The workbook as built (`Calculations!J54`) uses `construction_costs * inflation_factor` (live index); the line above is the case-compliant form — see the convention note below. `locked_factor` has not been run in Excel (fallback: `INDEX( inflation_factor, MATCH( construction_start, timeline, 0 ) )`).

**Mechanism:**
1. **Window flag, half-open.** `start <= date < EOMONTH( start, length )` switches on exactly `months_of_construction` months, the first being the start month. Operation then starts at `EOMONTH( start, length )`, the month after the last construction month (`J69`: `operation_start`).
2. **Elapsed-month counter without `SCAN`.** `SUM( INDEX( flag, r, SEQUENCE( 1, c ) ) )` sums the first `c` cells of row `r` — a prefix sum per row. This is Pattern B's gated counter (`SCAN( 0, flag, LAMBDA( s, c, s + c ) )`) generalised to N rows in one formula. Cost is rows × columns²/2 additions, trivial at 10 × 120.
3. **Two-key profile lookup with array keys.** `MATCH( elapsed_month, construction_month, 0 )` is a rows × periods array and `MATCH( months_of_construction, construction_length, 0 )` a rows × 1 array; `INDEX` broadcasts them to one lookup per cell. Each bell column sums to 100% (all thirteen columns checked), so the percentages of completion of each property sum to 1.

**Saturating counter, so the window flag must multiply the result.** The counter does not reset when the window closes — it stays at the project length. Before the window `elapsed_month = 0`, has no match, and `IFNA( …, 0 )` supplies the zero; after the window the lookup still *hits* a valid bell entry, so only the final `* construction_duration` returns it to zero. The classic corkscrew resets instead (`CaseSolution!G153:G162`: `IF( G$4 >= $C153 + $D153, 0, IF( $C153 <= G$4, 1, 0 ) + F153 )`, a Pattern C reset) and then needs only a zero guard on the lookup (`IF( G153 = 0, 0, … )`). Same output, two designs; with the saturating design never drop the window-flag multiplication.

**Cost-escalation convention — fixed by the case text.** The United Landlords case states that construction costs "will be fixed for the whole project at the first month of construction" and that the nominal price "will not be further inflated during the construction process" (case pp. 1–2). The locked factor is therefore required, matching the official solution (`CaseSolution!L127 = INDEX( $G$211:$DV$211, 1, J127 )`, `M127 = K127 * L127`). A live month-of-disbursement index (the as-built `J54`) contradicts the text. Computed outside Excel from the workbook's inputs, the live form ends at 69,959,232.64 of closing cash and the locked form at 82,843,606.52, equal to `CaseSolution!DV16`; the live form also moves the June 2023 deficit (−75,689,919 vs −74,219,830) and the maximum cash requirement (316,017,629 vs 305,021,570). General rule: before choosing an escalation convention for a cost line, search the case text for "fixed", "locked" or "determined at" — rent and OPEX may escalate live while construction does not, and uniformity across lines is not a reason to override the text.

### Pattern N — second occurrence (Eagle Security, `Calculations!J10` / `J20` / `J30`): onset flags and a global-clock counter driving a closed-form index

**Where:** Eagle Security — revenue `J10:U14`, salaries `J20:U24`, fixed costs `J30:U34`; operating profit `J40:U44 = J10# − J20# − J30#`, totals by `BYCOL` / `BYROW`. Each formula covers five selected cities over twelve months on a first-of-month spine (`J3 = EDATE( start_date, SEQUENCE( 1, months, 0, 1 ) )`).

**Template (confirmed, `Calculations!J10`; Gather block shown in Pattern P):**
```
  operation_start,      $F$10:$F$14,
  model_period_flag,    MAKEARRAY( ROWS( current_city ), COLUMNS( timeline ),
                          LAMBDA( r, c, IF( INDEX( timeline, 1, c ) >= start_date, 1, 0 ) ) ),
  operation_flag,       MAKEARRAY( ROWS( current_city ), COLUMNS( timeline ),
                          LAMBDA( r, c, IF( INDEX( timeline, 1, c ) >= INDEX( operation_start, r, 1 ), 1, 0 ) ) ),
  period_count,         MAKEARRAY( ROWS( current_city ), COLUMNS( timeline ),
                          LAMBDA( r, c, SUM( INDEX( model_period_flag, r, SEQUENCE( 1, c ) ) ) ) ),

  hours_monthly_fact,   IF( model_period_flag, ( 1 + current_hours_growth ) ^ period_count, 0 ),
  hours_available,      MAKEARRAY( ROWS( current_city ), COLUMNS( timeline ), LAMBDA( r, c, INDEX( current_hours, r, 1 ) ) ) * hours_monthly_fact * current_share,
  revenue_fact,         IF( model_period_flag, ( 1 + current_rev_growth ) ^ period_count, 0 ),
  revenue_realised,     MAKEARRAY( ROWS( current_city ), COLUMNS( timeline ), LAMBDA( r, c, INDEX( current_revenue, r, 1 ) ) ) * revenue_fact,

  operation_flag * hours_available * revenue_realised )
```
`J20` swaps the rate for salary per hour and `J30` drops `hours_available`; the preamble is otherwise identical.

**Mechanism: two flags, two jobs.**
1. **`model_period_flag` + `period_count` = the growth clock.** The counter is the prefix sum of the model-period flag, the same `SUM( INDEX( flag, r, SEQUENCE( 1, c ) ) )` as above. It counts months from the model start, not from each city's launch. Compounding is then Pattern B's closed form `( 1 + g ) ^ period_count`; no `SCAN`.
2. **`operation_flag` = the activation switch.** It is the onset flag `timeline >= launch date`, multiplied into the final expression only. A city that has not launched keeps growing its hours, price and cost indices and shows zero until its launch month.
3. **Why a global clock.** The case data is "Dec 2018 Base Data" and the official solution builds every index from that base for all ten cities (`CaseSolution!C19:O106`), then applies the On / Off Switch block (`CaseSolution!D110:O119`) afterwards. Pattern B's gated counter (Megabricks), where the active flag stops the count, would give a different answer here. Which clock applies is a reading of the case text, not a free choice.

**Convention flags.** First period: the base is December 2018, so January 2019 already carries one growth step (`period_count = 1`; `CaseSolution!D19 = C19 * ( 1 + $F5 )`). Monthly rate: `( 1 + annual / 100 ) ^ ( 1 / 12 ) − 1`, compounded rather than `annual / 12`; annual rates and market share are stored as whole percentages and divided by 100 in the Gather block. Onset test: `>=`, so a launch on the 1st switches on in the launch month; the spine and the launch dates are both first-of-month dates, and the test would shift by a month if one were month-end and the other not.

**Tie-out, computed outside Excel from the workbook's inputs.** As saved, the selection is Denver, Nashville, Rochester, San Antonio and Hartford, all launching 1 January 2019: 200,228,815.53 revenue, 78,849,907.55 salaries, 3,014,811.64 fixed costs and 118,364,096.34 operating profit, and an independent rebuild reproduces every cell (largest difference 5 × 10⁻⁹). That selection is not the official one. With the `CaseSolution` selection and launch dates (Nashville May, Denver June, St. Louis July, San Antonio August, Columbus April 2019) the same logic reproduces `CaseSolution` rows 130, 138, 146 and 154 month by month (largest difference 4 × 10⁻⁹): revenue 131,874,525.57, salaries 51,875,477.30, fixed costs 1,664,265.67, operating profit 78,334,782.60.

**Simplifications available, not yet entered in Excel.**
- `model_period_flag` is 1 in every column whenever the spine starts at `start_date`, so `period_count` equals the column position. It only does work if the spine starts before `start_date`; keep it as a guard, or use `SEQUENCE( 1, COLUMNS( timeline ) )`.
- The three flag and factor `MAKEARRAY` calls only broadcast. `--( timeline >= operation_start )` (1 × 12 against 5 × 1) gives the 5 × 12 flag directly, and `current_hours * hours_monthly_fact` broadcasts a 5 × 1 column against a 5 × 12 block, as the membership matrix of Pattern E already does.

---

## Pattern O — piecewise-linear interpolation of a ramp, with a `MEDIAN` clamp

**Where:** United Landlords — `Calculations!J69` (`vacancy_rate`: vacancy by month of operations, 75% at month 1, 25% at 6, 10% at 12, 5% at 24, linear between). The official solution lays the same ramp out as a helper row of hand-stepped values (`CaseSolution!G213:DV213`, e.g. `H213 = G213 - ( $G$213 - $L$213 ) / ( $L$4 - $G$4 )`) indexed by month of operations. One workbook; provisional.

**When it applies:** a rate or factor is given at a few breakpoints and must be interpolated linearly between them for every elapsed period, then held flat outside the breakpoint range.

**Template (confirmed shape, `Calculations!J69`; written with the breakpoint bounds read from the data rather than typed):**
```
vacancy_rate, MAKEARRAY( ROWS( property_data ), COLUMNS( timeline ),
  LAMBDA( r, c, LET(
    moo,        MEDIAN( MIN( moo_bp ), INDEX( elapsed_month, r, c ), MAX( moo_bp ) ),
    seg,        MIN( MATCH( moo, moo_bp, 1 ), ROWS( moo_bp ) - 1 ),
    lower_moo,  INDEX( moo_bp, seg ),
    upper_moo,  INDEX( moo_bp, seg + 1 ),
    lower_vac,  INDEX( vac_bp, seg ),
    upper_vac,  INDEX( vac_bp, seg + 1 ),
    IF( INDEX( operation_flag, r, c ) = 0, 0,
        lower_vac + ( upper_vac - lower_vac ) * ( moo - lower_moo ) / ( upper_moo - lower_moo ) ) ) ) )
```
(`moo_bp` = `CaseData!$F$6:$F$9`, `vac_bp` = `CaseData!$H$6:$H$9`. `J69` uses the literals `1`, `24` and `3`; for these breakpoints they equal `MIN( moo_bp )`, `MAX( moo_bp )` and `ROWS( moo_bp ) - 1`.)

**Mechanism:**
1. **`MEDIAN` of three is a clamp.** `MEDIAN( lo, x, hi )` equals `MIN( MAX( x, lo ), hi )`: elapsed months before the first breakpoint read as the first, and after the last breakpoint hold at the last, so the ramp is flat outside its range.
2. **Segment finder.** `MATCH( moo, moo_bp, 1 )` (approximate, ascending) returns the last breakpoint at or below `moo`; capping at `ROWS( moo_bp ) - 1` keeps the final breakpoint as the *upper* end of the last segment, so month 24 interpolates to exactly the last rate instead of indexing past the table.
3. **Linear interpolation** between the segment's two breakpoints.

**Convention flags:** first period — month of operations 1 returns the first breakpoint's rate (75%); last — flat at the last rate (5%) from month 24. The `operation_flag = 0` branch returns 0 vacancy, which is harmless only because the rent is also multiplied by `operation_flag`.

**Why this belongs with the closed-form group:** nothing depends on a prior cell's output — each cell interpolates from its own `moo`. Stateless, Gather-phase in the sense of `LET_Construction_Style_CFM.md`, and the replacement for a hand-built helper row.

---

## Pattern P — select first, compute second: array-keyed row gather from a master table

**Where:** Eagle Security — `Calculations!J10`, `J20`, `J30` (Gather block), plus the position helper `D10:D14`. One workbook; provisional. Its natural home is the Gather phase of `LET_Construction_Style_CFM.md`.

**When it applies:** a master table lists n candidates (ten cities) and a short typed list (`E10:E14`, five names, with launch dates in `F10:F14`) picks which to model. Changing a name must reroute the whole schedule with no formula edits.

**Template (confirmed, `Calculations!J10`):**
```
LET(
  data,                  CaseData!$B$7:$L$16,
  cities,                INDEX( data,, 1 ),
  hours,                 INDEX( data,, 3 ),
  hours_growth,          ( ( 1 + INDEX( data,, 4 ) / 100 ) ^ ( 1 / months ) ) - 1,
  share,                 INDEX( data,, 5 ) / 100,
  revenue,               INDEX( data,, 6 ),
  revenue_growth,        ( ( 1 + INDEX( data,, 7 ) / 100 ) ^ ( 1 / months ) ) - 1,

  current_selection,     MATCH( $E$10:$E$14, cities, 0 ),
  current_city,          INDEX( cities,  current_selection, 1 ),
  current_hours,         INDEX( hours,   current_selection, 1 ),
  current_hours_growth,  INDEX( hours_growth, current_selection, 1 ),
  ... )
```

**Mechanism:** `MATCH` with an array as its lookup value returns the whole m × 1 vector of positions in one call. `INDEX( column, positions, 1 )` with an array row argument returns the m × 1 column of attributes, in selection order. Every `current_*` binding is row-aligned with the typed selection, and everything after Gather works on m rows, broadcasting against the 1 × T timeline.

**Select first versus compute first.** The official solution does the reverse: it builds all ten cities for every month (`CaseSolution!C19:O106`) and then picks five rows with per-cell `INDEX` / `MATCH` (`CaseSolution!D125:O129`). Selecting first computes five rows, not ten, and the row order follows the selection, not the table.

**Convention flags:** every `current_*` must derive from the same `current_selection` (a second `MATCH` on a different range is the row-misalignment risk of gotcha 2). `MATCH` takes `0` (gotcha 5). A duplicated name gives a duplicated row and no error; a name missing from the table gives `#N/A` across that row. The selection is typed once (`E10:F14`) and mirrored (`E20 = $E10`, `F20 = $F10`), so the later blocks cannot drift from it.

---

## Pattern Q — layered allocation into ordered bands: clipped differences of lagged ceilings (tax brackets)

**Where:** Neverland — `Tax!Q10:U5885` (tax in each of five brackets for 5,876 employees, one spill), with `V10 = BYROW( Q10#, SUM )` for total tax. The official solution lays the same arithmetic out as five columns of per-cell formulas (`CaseSolution!T11:X5886` for the amount in each bracket, `Y11:AC5886` for the tax). One workbook; provisional. The same identity sits behind Pattern D's capacity allocation and the NOL FIFO waterfall in `Income_Tax_NOL_Schedule_CFM.md` §2.3.

**When it applies:** an amount (a salary, taxable income, a loss pool, a capacity) has to be spread over ordered bands with known ceilings, each band taking its share before the next one starts, and each band's share is then weighted (a marginal rate).

**Template (confirmed, `Tax!Q10`):**
```
LET(
  gross_salary, $P$10#,
  thresholds,   $Q$9:$U$9,
  tax_rate,     CaseData!$H$8:$H$12,

  MAKEARRAY( ROWS( gross_salary ), COLUMNS( thresholds ),
    LAMBDA( i, j,
      LET(
        salary, INDEX( gross_salary, i ),
        upper,  INDEX( thresholds, j ),
        lower,  IF( j = 1, 0, INDEX( thresholds, j - 1 ) ),
        rate,   INDEX( tax_rate, j ),

        MAX( 0, MIN( salary, upper ) - lower ) * rate ) ) ) )
```

**Mechanism:**
1. **The slice is an overlap.** `MAX( 0, MIN( salary, upper ) − lower )` is the length of the overlap between `[0, salary]` and the band `( lower, upper ]`: zero below the band, `salary − lower` inside it, the full width `upper − lower` above it. Multiplying by the band's rate gives the tax in that band; `BYROW( …, SUM )` gives the total.
2. **The lag is the only "recursion".** `lower` is the previous band's ceiling (`IF( j = 1, 0, INDEX( thresholds, j − 1 ) )`). That is Pattern I's one-period shift, `HSTACK( 0, DROP( x,, -1 ) )`, applied to ceilings instead of balances — the same `prev_range` as the half-open intervals of Pattern E's interval-boundary variant.
3. **Nothing folds.** A band's slice never depends on another band's result, so no `SCAN`. The waterfall "fill band 1, carry the remainder to band 2, …" has remainder `salary − Σ earlier slices`, and the sum of the earlier slices is `MIN( salary, lower )`, so the remainder is `salary − lower` and the clip returns the same number in one step.
4. **Why `MAKEARRAY`.** `MIN` and `MAX` collapse an array to a single value; they do not clip element by element. A 5,876 × 1 salary column set against a 1 × 5 ceilings row therefore cannot go through `MIN( salary, upper )` directly, and `MAKEARRAY` with `INDEX` per cell is the way round it (gotcha 15). The broadcast alternative uses `IF`, which does work element by element (untested in Excel; checked numerically against all 5,876 rows):
```
upper,   $Q$9:$U$9,
lower,   HSTACK( 0, DROP( upper,, -1 ) ),
rate,    TRANSPOSE( CaseData!$H$8:$H$12 ),
IF( gross_salary >= upper, upper - lower, IF( gross_salary > lower, gross_salary - lower, 0 ) ) * rate
```

**Convention flags:**
- The thresholds are cumulative *ceilings* (5,000; 10,000; 20,000; 45,000), so `lower` is the previous ceiling. Band widths would need a running sum first.
- Bands are half-open, `( lower, upper ]`. The `+0.01` in `CaseData!F9:F12` (`=G8+0.01`, the "From" column) is only a label; neither workbook uses it.
- **The open last band.** The case shows `∞` (text, `CaseData!G12`). The official solution gives the last bracket its own formula without a ceiling (`CaseSolution!X11 = MAX( $S11 − W$6, 0 )`). Neverland types `999999` in `Tax!U9`, which is fine for this data (largest adjusted salary 147,327.28) but silently truncates anything larger: a 2,000,000 salary would be taxed at 726,249 instead of 1,476,250. Prefer a very large ceiling that is obviously a sentinel, or the no-ceiling form below.
- `Q9:U9` are typed constants. The official header row is linked (`CaseSolution!T6 = TRANSPOSE( CaseData!G8:G12 )`), so it follows the case data.

**Alternative for the total only — the marginal-rate-difference form (untested in Excel; matches all 5,876 official totals to 1.5 × 10⁻¹¹).** Each bracket's rate is an increment over the one below it, so tax is the sum over floors of `( salary − floor ) × rate step` for the floors the salary has passed:
```
LET(
  gross_salary, $P$10#,
  floor,        HSTACK( 0, DROP( $Q$9:$U$9,, -1 ) ),
  rate,         TRANSPOSE( CaseData!$H$8:$H$12 ),
  rate_step,    rate - HSTACK( 0, DROP( rate,, -1 ) ),

  MMULT( ( gross_salary > floor ) * ( gross_salary - floor ), TRANSPOSE( rate_step ) ) )
```
It needs no ceiling for the last band, so no `∞` sentinel, and it uses no `MIN` or `MAX`. It cannot show the tax in each bracket, because `rate_step` terms are increments, not bracket rates; keep the layered form when the case asks for tax or revenue by bracket.

**Same identity, other places.** With layer capacities known before the allocation starts, a sequential fill and the clipped-difference form agree. Checked on 2,000 random cases each: (a) the NOL FIFO cascade `bucket_k_used = MIN( bucket_k_balance, total − SUM( prior used ) )`, with the cumulative balances as ceilings; and (b) Pattern D's `MIN( demand, MAX( capacity − higher, 0 ) )`, with the layers in rank order. If a layer's capacity depends on what an earlier layer did (rollovers across periods), the closed form no longer holds and Patterns C, K or L apply.

**Tie-out, computed outside Excel.** Fed the official adjusted salaries, the clip form reproduces `CaseSolution` Tax 1–5 and the amount in each bracket for all 5,876 employees (largest difference 0); total new tax is 42,297,497.70. The workbook as saved gives 42,297,484.18: the cleaned `Data` table rounds salaries to whole dollars, whereas the raw strings (and the official database) keep cents — 3,652 of 5,876 salaries carry cents, e.g. `$30,753.50`. Counts and identities tie exactly (4,876 winners, 991 losers, 9 neutral; maximum loser ID 2937638); money differs by single dollars (tax gain 4,102,333.08 against 4,102,340.24; maximum loser −46,195.45 against −46,195.55; gotcha 16). Separately, `Tax!H10` (2023 existing tax) is `gross × 29%` with no non-taxable minimum, 49,185,049, whereas `CaseSolution!L11:L5886` applies `MAX( ( gross − 7,500 ) × 29%, 0 )`, 36,496,777. No headline answer uses the 2023 columns, but the 2023 schedule is wrong as it stands.

---

## Pattern R — vintage depreciation waterfall in closed form: cohort × period grid driven by asset age

**Where:** Galway Inc — `Calculations!J16` (double-declining balance, half-year convention), `J45` and `J61` (straight line, half-year convention), totals `J26`, `J55`, `J71`; check column `H16 = G16# − BYROW( J16#, SUM )`. Big Dog — `Calculations!J38` (declining balance), `J68` (straight line), `J99` (MACRS-type class table), `J129` (sum-of-years'-digits table), repeated for each asset class from `J161`; totals `J63`, `J93`, `J124`, `J154`. Rock to Mineral — `Calculations!J40` (declining balance), `J69` (straight line), `J98` (units of production), `J127` (sum-of-years'-digits), repeated per class at `J159…J365`. All three feed a `CHOOSE( selector, … )` method switch (`Answers!J11` in Big Dog and Rock to Mineral) and then a Pattern J `SCAN` for net book value.

**When it applies:** assets are bought in several vintages (one per year, or grouped by year with `PIVOTBY`), and each vintage depreciates on a schedule that depends only on its own age. The textbook layout is a corkscrew per vintage, `NBV[t] = NBV[t−1] − dep[t]`; Pattern R computes every cell from the age alone.

**Template (confirmed shape, Galway `Calculations!J16`; vintages on rows, model years on columns):**
```
LET(
  acquisition_cost, $G$16#,                                   \\ n × 1
  acquisition_date, $F$16#,                                   \\ n × 1
  current_date,     J15#,                                     \\ 1 × T
  useful_life,      Assumptions!$F$19,
  period,           YEAR( current_date ) - YEAR( acquisition_date ),     \\ n × T age grid
  rate,             1 / useful_life * 2,
  book_value,       acquisition_cost * ( 1 - rate / 2 ) * ( 1 - rate ) ^ ( period - 1 ),

  IF( period < 0, 0,
      IF( period = 0, acquisition_cost * rate / 2,
          IF( period < useful_life, book_value * rate,
              IF( period = useful_life, book_value, 0 ) ) ) ) )
```

**Mechanism:**
1. **The age grid.** `YEAR( current_date ) − YEAR( acquisition_date )` broadcasts a 1 × T timeline against an n × 1 vintage column, giving n × T ages: negative before purchase, 0 in the purchase year. It plays the same role as the `elapsed` grid of Pattern H, with depreciation in place of payment terms.
2. **Declining balance is Pattern B's closed form.** The recurrence `NBV[p] = NBV[p−1] × ( 1 − r )` solves to `NBV[p] = cost × ( 1 − r ) ^ p`, so depreciation is `r × cost × ( 1 − r ) ^ p` with no fold (Big Dog `J38`, Rock to Mineral `J40`). With the half-year convention the first year takes `r / 2`, so the opening book value of year `p ≥ 1` is `cost × ( 1 − r / 2 ) × ( 1 − r ) ^ ( p − 1 )` (Galway `J16`). In the last year (`p = life`) the whole remaining book value is written off, so each row sums exactly to cost. The check column `H16` (cost less depreciation charged inside the horizon) is 0 for the 2018–2020 vintages, whose lives end inside the model, and equals the remaining book value for the later ones.
3. **Straight line is a step function of age.** Half a charge at age 0, a full charge while age < life, then the half-year residual `cost − charge × ( age − 0.5 )` (Galway `J45`).
4. **Table methods are a two-key lookup** — the same move as Pattern N's bell curve. `INDEX( table, MATCH( age + 1, year_column, 0 ), MATCH( life_or_class, header_row, 0 ) )` gives the year's percentage for MACRS, sum-of-years'-digits and units of production (Big Dog `J99`, `J129`; Rock to Mineral `J98`, `J127`).
5. **Collapse and roll forward.** `BYCOL( grid, SUM )` gives depreciation per year; `CHOOSE( selector, … )` picks the method; a single Pattern J `SCAN` builds net book value, with Pattern I for the opening balance.

**Convention flags:**
- **Age base.** Formula-driven methods count age from 0 in the purchase year; table methods use `age + 1`, because the tables start at year 1. Mixing the two shifts a method by one year without any error.
- **Declining balance never ends in Big Dog and Rock to Mineral.** `IF( period >= 0, depreciation, 0 )` keeps charging `r × ( 1 − r ) ^ p` forever, so the asset never reaches zero (or salvage) within the horizon. Galway's form ends at `period = useful_life` by writing off the remainder. Check which the case asks for: a switch to straight line, a write-off at end of life, or a pure geometric tail.
- **Salvage.** Rock to Mineral depreciates `cost × ( 1 − salvage )` under every method, including declining balance. The usual declining-balance rule applies the rate to the full cost and stops at salvage; read the case before choosing.
- **`IFERROR( …, 0 )` around the whole `LET`** (Big Dog, Rock to Mineral) hides typos: see gotcha 19. Big Dog `J68`/`J191` compute `accumulated_depr` as `acquisition_cost * period` instead of `depreciation * period`. It is harmless only because that branch is reached after the end of life, where both versions give a book value ≤ 0.

**Why this belongs with the closed-form group:** no cell reads another cell's result. The only true accumulation is the final net-book-value `SCAN`, and even that could be replaced by `cost − cumulative depreciation` taken from the same grid.

---

## Flag conventions — progression from Fashion Boutique to Eagle Security

The flags are the part of the catalog that moved most between April and October: from helper rows and `SCAN`-built counters to derived-in-the-formula flags and a closed-form index. The stages below use only the cells named in this catalog.

| Stage | Where | Calendar position / switch | Counter | Index |
|---|---|---|---|---|
| April | Fashion Boutique `Time!J11`, `Price!J8` | cyclic month counter built by `SCAN` on a run of period numbers (Pattern C V1) | `SCAN` | `SCAN` inside `MAKEARRAY` (Pattern A) |
| Gated counter | Megabricks `Calculations!J17…` | active-operating flag from the case | additive `SCAN` of the flag | `rate ^ count` (Pattern B) |
| Date spine | United Landlords `Calculations!J17` | `MONTH( timeline ) = 1` inline | none | one `SCAN` on `flag * rate` (Pattern A, flag-gated) |
| Event windows | United Landlords `Calculations!J54`, `J69` | start / end flags inline (`start <= t < end`) | `SUM( INDEX( flag, r, SEQUENCE( 1, c ) ) )` | live or locked factor from the monthly index |
| October | Eagle Security `Calculations!J10` | `timeline >= start` inline, two flags | the same prefix sum, on the global clock | `( 1 + g ) ^ count`, no `SCAN` |

**Flag shapes and the formulas that carry them:**

| Shape | Formula | Where |
|---|---|---|
| Calendar trigger | `IF( MONTH( timeline ) = 1, 1, 0 )` | United Landlords `J17…` |
| Onset (step) flag | `timeline >= start` | Eagle `operation_flag`, `model_period_flag`; United Landlords `operation_flag` |
| Window flag | `( t >= start ) * ( t < EOMONTH( start, n ) )` | United Landlords `construction_duration` |
| Elapsed counter | prefix sum of any flag | Eagle `period_count`; United Landlords `elapsed_month` |
| Output mask | the flag multiplies the final expression | every case above |

**Clock versus switch.** Decide first whether the index counts months from a fixed base date (global clock: the counter runs on the model-period flag, the onset flag only masks the output) or from each series' own start (gated clock: the onset flag gates the counter). Eagle Security and the rent, OPEX and land indices in United Landlords use the global clock; Megabricks uses the gated clock. The case text decides — a base-date header ("Dec 2018 Base Data") and annual growth rates point to the global clock.

**Promotion.** Flags are stateless and cost nothing to repeat inline. The counters and the indices built on them are what get repeated across consumers and belong in a standalone spill (see the promotion note under Pattern A, flag-gated).

---

## Decision guide for the October exam

| Situation | Pattern |
|---|---|
| Rate varies by period, need it replicated across several independent series (currencies, accounts) | A — `SCAN` inside `MAKEARRAY` |
| Rate is constant, only need count of elapsed/active periods | B — additive `SCAN` (counter, only if gated) + `rate^count` |
| Need a running total that resets, holds, or replenishes on a condition (calendar rollover, event flag, group-boundary change) | C — conditional-branch `SCAN` |
| Need "cumulative amount ahead of me in some priority order" (capacity/queue allocation) | D — `RANK.EQ` + `SUMPRODUCT` filter, no `SCAN` |
| Need a groupby-sum, whole result at once, category or time-bucket | E — one-hot membership matrix + `MMULT` |
| Need a time groupby-sum onto a spilled timeline whose length follows a frequency switch, one data row at a time | E, interval variant — spilled `SUMIFS` with array criteria and `HSTACK`+`DROP`-lagged lower bounds |
| Header-row labels need to reach the detail rows beneath them | F — `SCAN` as forward-fill |
| Every row in a group (including the header) needs to show that group's total, without collapsing the layout | G — forward-filled key + `SUMPRODUCT` equality match |
| Closing balance is purely a function of elapsed time against a fixed (possibly fractional) term | H — aging/amortization triangle, `BYCOL`+`SUMPRODUCT`, no `SCAN` |
| Closing balance already known some other way; need the matching opening balance + connecting flow | I — `HSTACK`+`DROP` one-period lag, flow solved as residual |
| Balance must build up period by period, no closed-form shortcut (order- or event-dependent, not reducible to a fixed term) | J — plain `SCAN(seed, movements, SUM)` |
| Two or more *named rows* reference each other's previous-period value (MIN/MAX-coupled state) | K — classic per-column formulas (see companion file); packed-tuple `SCAN` does not work |
| Flow is order-dependent (FIFO) *and* competes across rows for a shared period pool | L — classic triangular `MIN`-with-expanding-range, deliberately non-array |
| Need a rolling trailing-window sum/figure (TTM, trailing quarter) without an OFFSET anchor cell | M — `HSTACK` to concatenate, `TAKE(TAKE(x,,n),,-12)` to slice (mind the rows-vs-columns argument order on a 1-row array) |
| Rate or index steps once a year (or on any calendar trigger) and the timeline is a date spine | A, flag-gated variant — `SCAN( 1, flag * rate, … )` with `MONTH( timeline )` and `XLOOKUP( YEAR( timeline ), … )` inside the `LET`; promote the index to its own cell if more than one formula needs it |
| Several series on one date spine, each with its own start and duration; need an on/off flag and a month-of-activity counter to index a profile (S-curve, ramp) | N — window flag via `MAKEARRAY`, counter via `SUM( INDEX( flag, r, SEQUENCE( 1, c ) ) )`, window flag multiplied back in |
| Rate given at a few breakpoints, interpolate linearly per elapsed period, flat outside the range | O — `MEDIAN( lo, x, hi )` clamp + `MATCH( …, 1 )` segment + linear interpolation |
| Pick m named rows out of an n-row master table (typed list of cities, products) and compute only those | P — `MATCH( selection, key_column, 0 )` once, then `INDEX( column, current_selection, 1 )` for every attribute |
| Switch a series on from its own start date while its index counts months from a fixed base date | N, Eagle occurrence — onset flag `timeline >= start` masks the output; prefix-sum counter on the global clock; `( 1 + g ) ^ count` (Pattern B) |
| Spread an amount over ordered bands with ceilings (tax brackets, loss buckets, capacity tiers), each band weighted by its own rate | Q — `MAX( 0, MIN( amount, upper ) − lower ) * rate`, `lower` = previous ceiling; `MAKEARRAY` because `MIN` and `MAX` aggregate (or `IF` broadcast); total only: marginal-rate-difference `MMULT` |
| Several asset vintages, each depreciating on a schedule that depends only on its age (declining balance, straight line, MACRS / SYD / units-of-production tables) | R — n × T age grid `YEAR( t ) − YEAR( acquired )`; `cost × ( 1 − r ) ^ age` or a two-key table lookup; `BYCOL( …, SUM )`, then one Pattern J `SCAN` for net book value |
| Choose one scenario row of a compounded rate grid | A, scenario-select variant — but select the rate row first and run one `SCAN` (Pattern P's order) |
| A rule stated on totals to date (cumulative revenue earned, percentage of completion) | I, de-cumulating variant — closed-form cumulative, then `cum − HSTACK( 0, DROP( cum,, -1 ) )` |
| A balance set by *next* period's flow (target inventory as a share of next month's sales) | I, lead variant — `HSTACK( DROP( x,, 1 ), terminal_value )` |
| Tax losses (or any pool) used oldest first, with vintages that expire | L, NOL occurrence — classic per-cell; give the undated pool its own usage *and* balance rows; test expiry with `YEAR( date )` |

Rule of thumb before writing any fold: identify (1) is the rate/driver constant or varying, (2) is there an independent axis to replicate across, (3) does the "backward" dependency reduce to a count, a reset, a rank-based filter, an elapsed-time comparison, or a static slice, rather than a true multiplicative/additive chain, (4) how many *named rows* does the recurrence actually touch (one → A–J; two+ coupled → K; order-dependent and cross-row-competing → L). Only reach for `SCAN` once you've ruled out a closed-form, filter-based, or slice-based reformulation — Patterns D, E, G, H, M and R are proof that not every backward-looking problem actually needs a fold, and Eagle Security — flags, a prefix-sum counter and `( 1 + g ) ^ count` — and Neverland — bracket layering by clipped differences — are whole workbooks without one.

---

## Bugs & gotchas addendum

Silent-failure modes confirmed while building this catalog — none of them throw an error, which is exactly why they're worth having memorized rather than re-derived under exam time pressure.

1. **Non-unique label → `MATCH`/`XMATCH` silently resolves to the first match.** When a label repeats across a lookup range (e.g. the same product name appears once per retailer block), a lookup keyed on that label always returns the *first* occurrence — every other block's figures get silently overwritten with the first block's numbers. Surfaced three separate times in Sweet Dreams alone. Fix: match positionally (`SEQUENCE`+`INDEX`) when two arrays are built from the same range in the same row order, rather than re-matching a label that isn't guaranteed unique across the full range.

2. **Row-alignment mismatches across composed `FILTER` masks fail silently.** `FILTER` and `MMULT` don't error when dimensions still conform even if the underlying rows have drifted out of alignment (a one-row offset between two masks built from slightly different ranges) — they just compute a confident, wrong number. Checklist: confirm every source range in a composed formula starts and ends on the exact same rows, including blank/header rows.

3. **`IFERROR`'s fallback *shape*, not the primary expression's intended shape, determines what spills.** If the primary expression is meant to spill an array but the lookup errors before it ever runs, the whole expression collapses to a single error value — and `IFERROR`'s second argument then dictates the output shape. A bare `0` fallback silently produces one scalar instead of the intended spilled array, breaking anything downstream. Fix: build the fallback as an array of the correct shape from a dimension you know is stable (e.g. `TRANSPOSE(SEQUENCE(1, COLUMNS(known_stable_range), 0, 0))`), not a bare scalar.

4. **`BYROW`/`BYCOL` only ever pass their `LAMBDA` one argument.** A two-parameter lambda against a single-array `BYROW`/`BYCOL` call throws an unhelpful `#VALUE!` because the second parameter never receives anything. If a second array needs to participate, hold it fixed by reference inside the lambda instead of trying to pass it as a parameter. Also worth checking: `BYCOL` on a single-column range only ever calls its lambda once — a sign you're iterating the wrong axis.

5. **`MATCH` without an explicit third argument** defaults to approximate matching (`1`), not exact — always pass `0` unless approximate matching is actually intended.

6. **`INDEX` on a 2-D array needs both a row and a column argument**; omitting one silently returns the wrong slice rather than erroring.

7. **`ANCHORARRAY(...)` notation**, as stored internally by Excel for spill references, must always be normalized to `#` spill notation in this documentation, per style convention.

8. **`TAKE`/`DROP`'s first numeric argument is rows, the second is columns — on a single-row (or single-column) array, getting this backwards is a silent no-op, not an error.** `TAKE(row_vector, -12)` asks for -12 *rows* from something that only has 1 row, so it just returns the whole thing unchanged; it doesn't throw, and the result can still look like a plausible answer until it's tied out against an independent figure. Confirmed in Streamwise's TTM rebuild (Pattern M) — the fix is `TAKE(row_vector, , -12)`, leaving the rows argument blank and supplying `-12` as the columns argument.

9. **A source range that looks like more history often isn't.** A tab can carry columns immediately after the real historical data that are actually that tab's *own* separately-computed forecast, not further actuals — visually indistinguishable from real history at a glance. Before concatenating "all of history" into a formula (Pattern M, or any Pattern I lag), confirm the exact column where genuine historical data ends; don't infer the boundary from where the *consuming* model's own forecast happens to start.

10. **`SUMIFS` (and `COUNTIFS`, `AVERAGEIFS`) only spill through the *criteria* argument.** sum_range and criteria_range must be range references — a sheet range or a spill reference `X#` — not arrays computed inside a `LET`, and they must have the same dimensions as each other. Passing the period array as a criteria_range against a detail-length sum_range is a size mismatch and returns `#VALUE!`. The period array belongs in the criteria argument, concatenated with its operator (`"<=" & date_range`).

11. **A scenario switch that shifts dates must reach every formula that derives a date from the shifted quantity — and the base case will not reveal an omission.** In a timing-flexibility case one input (`delay`, `Calculations!$F$9`) moves every start date. Land acquisition (via `project_start`), construction (`J54`), new-property rent and OPEX (`J69`, `J84`) all add it; if any other formula that rebuilds `operation_start` from `purchase_date` does not (the new-property gross rent feeding the management fee, `J100`), the switched-off model still ties to the official solution to the cent and only the postponed scenario is wrong. Check: flip the switch and tie to the official scenario value (United Landlords: `CaseSolution!F221`, 284,511,208.65 for a 12-month postponement, reproduced exactly outside Excel, with the locked construction index of Pattern N, only when the management-fee rent is delayed too — omitting it overstates the peak cash requirement by 1,471,398.76). The delay input is also read as a flag, `IF( $F$9 = 0, 0, 12 )`, so an input of 2 years gives 12 months, not 24.

12. **`SEQUENCE( 1, months, … )` truncates a fractional period count silently.** `months = ( end_date - start_date ) / 365 * 12` gives 120.03 for 1 January 2022 – 31 December 2031 (3,651 days, three leap days in the window) and the spill is 120 columns. A window with fewer leap days comes out just *under* the true count — 1 January 2025 – 31 December 2026 is 729 days, 23.97, which truncates to 23 columns and drops the last month without any error. Count months by calendar instead: `( YEAR( end_date ) - YEAR( start_date ) ) * 12 + MONTH( end_date ) - MONTH( start_date ) + 1` (120 here).

13. **One workbook name doing two jobs.** In Eagle Security `months` (`Calculations!$F$55`, the horizon length) feeds `SEQUENCE( 1, months, 0, 1 )` for the timeline *and* the divisor in the annual-to-monthly rate `( 1 + annual / 100 ) ^ ( 1 / months ) − 1`. The two values coincide only because the horizon happens to be 12. With a 24-month horizon every monthly rate would be wrong and nothing would error; on the official selection, twelve-month revenue would read 129,365,563 instead of 131,874,526 (−1.9 %). Use a literal 12 or a separate `months_per_year` for the conversion, as the official solution does (`CaseSolution!F5 = ((1+E5/100)^(1/12))-1`).

14. **A workbook can be internally consistent and still sit in a different scenario from the answer key.** The Eagle Security file as saved holds a different city selection and launch dates from `CaseSolution`, so its totals do not match the key even though the logic does. Before comparing outputs, compare the inputs that drive the scenario (selection, start dates, switch values); then enter the key's inputs and tie out.

15. **`MIN`, `MAX` and `MEDIAN` aggregate arrays; they do not clip element by element.** `MIN( salary_column, ceiling_row )` returns one number for the whole block, not a grid. To clip a column against a row use `MAKEARRAY` with `INDEX` per cell (Neverland `Tax!Q10`, Pattern O's `MEDIAN` clamp), or an `IF` form, which does broadcast. The failure is silent when the aggregate happens to be a plausible figure.

16. **A rounding step inside data cleaning shows up as single dollars in the totals and nowhere else.** Neverland's cleaned `Data` table rounds salaries to whole dollars while the raw strings carry cents (`$142,345.20.-`). Every count, flag and ID ties out, so the model looks right, but money totals differ from the answer key by a few dollars (tax gain 4,102,333.08 against 4,102,340.24). Keep the cents when parsing text amounts and round only for display.

17. **A `MAKEARRAY` lambda that never reads its index returns the same value in every cell.** YHF `Debt!J159:J160` (fixed 11 October 2026) copied Streamwise's `TAKE( TAKE( full, , hist_count + j ),, -12 )` as `hist_count + 1`, so all six forecast months show month 1's trailing twelve months (term loan 9,914.31 throughout; key 9,914.31 → 9,391.94). The first column ties to the key, so checking one cell finds nothing. Check: every `MAKEARRAY` lambda uses `i` or `j` (or both); otherwise it is a broadcast, and a broadcast is better written without `MAKEARRAY`. Tie out the first *and* the last column.

18. **A spilled formula cannot read its own previous element.** Unicorn Construction `Debt!J14` is `IF( J12#, I14 + 1, 0 )`, meant as a period counter. `I14` is one fixed cell, not "the column to my left" for each element, so the spill is 1 in every period and `H14` (its total) shows 40. The row below survives by accident: `J18`'s classic `SUM( $J14:J14 )` cumulates the 1s back into the period number. Use `SEQUENCE( 1, periods )` for a counter, or `SCAN` when the step really depends on the previous value.

19. **`IFERROR( …, 0 )` around a whole `LET` hides a misspelt binding.** Rock to Mineral `Calculations!J188` defines `accumulated_depr` and then uses `accumulated_dep`. The misspelt name is `#NAME?` only in the branch that reads it (`book_value`, after the end of life), and `IFERROR` turns it into 0. With straight line the right answer there is also 0, so nothing shows. Put `IFERROR` around the one lookup that can legitimately fail, not around the whole formula, and check a `LET` once without it.

Circular-reference debugging technique (lagged-balance interest, lagged pricing-grid lookups, excluding a self-referencing plug from its own feeding total) is documented separately in `Circular_Reference_Avoidance_CFM.md` — general modeling discipline for debt/revolver schedules, not a dynamic-array recipe, so kept out of the main catalog per the usual curation rule. In the repository, iterative calculation is switched on in YHF, Granite Bay, Windsol and Greenrock Capital. Their lagged pricing-grid lookups read the previous column's ratio (YHF `Debt!J48`: `XLOOKUP( IF( J$11 = 1, $F$18, I$18 ), … ,, -1 )`; the Model Checking cases' `CaseData` grids, e.g. Stepstreet `K615`, Windsol `K493`, Greenrock `K593`), which keeps those lookups off the circular path; where the remaining loop sits in each model was not traced in this check.

---

## Workbook coverage — all 34 solution workbooks, checked 11 October 2026

Checked by reading every formula in the repository copies (`lucnicod/CFM`, file names `MMDDYYYY_Topic_vN_Company.xlsx`). "Classic" means per-cell formulas filled across, with no dynamic-array recurrence.

| Workbook | Patterns | Notes |
|---|---|---|
| Fashion Boutique | A, C V1, E (time bucket) | — |
| Domestique Rides | A (scenario-select) | C V1 counter only in the local v. 2 working file |
| Megabricks | B (gated counter) | — |
| Powerstill | A | — |
| Big Fish | D | `Summary!J7`: `GROUPBY` + `MMULT( matrix, ones )` row totals, used for sorting, not a recurrence |
| Galway Inc | R, I, J | — |
| Big Dog | R, I, J | `IFERROR` wraps every depreciation `LET` (gotcha 19 risk); `J68`/`J191` `accumulated_depr` slip (Pattern R) |
| Rock to Mineral | R, I, J | gotcha 19 (`J188`) |
| Stanbridge | B, I, J | — |
| Thermin | B, I, J, L | — |
| Wichita Warblers | C V2/V3, I (de-cumulating), J | named-`LAMBDA` library `MW.*` (rolling sum/avg via `SCAN` over an index + `TAKE`/`DROP`) is defined but unused |
| Sweet Dreams | E, F, G, H, I (lag and lead) | — |
| Waterworks Corp | K | classic |
| JAK Holdings | L (NOL) | undated pool used twice, and losses never expired; both fixed 11 Oct 2026 (Pattern L) |
| BCS Holdings | L (NOL) | undated pool balance never reduced, and losses never expired; both fixed 11 Oct 2026 (Pattern L) |
| Streamwise | M | — |
| Blackrat SPV #1, Blackrat SPV #2 | — | classic debt schedules, quarterly spine |
| YHF | M | gotcha 17, fixed 11 Oct 2026; debt schedule still differs from the key (Pattern M, YHF); iterative calculation on |
| Investment Fund Asset | — | classic roll-forward; interest and principal per quarter from `IPMT`/`PPMT` on the opening balance over the remaining term, refinancing target from `PV` |
| Unicorn Construction | — | classic; gotcha 18 |
| Granite Bay | A (one row) | iterative calculation on |
| Acme Inc, Alpha, WW Cars, BoTrainers | — | classic or non-recursive `LET` arithmetic (equity, consolidation, merger); no dynamic-array recurrence |
| Rock Climbing | — | scenario table lookups |
| Four Mini Tasks | E (interval) | — |
| United Landlords | A (flag-gated), I, J, M, N, O | — |
| Stepstreet Capital, Windsol, Greenrock Capital | — | classic, lagged grid lookups |
| Eagle Security | N, P | — |
| Neverland | Q | — |

Companion files cited in this catalog but not in `1. Patterns`: `Income_Tax_NOL_Schedule_CFM.md` (Pattern Q) and `Circular_Reference_Avoidance_CFM.md` (above).
