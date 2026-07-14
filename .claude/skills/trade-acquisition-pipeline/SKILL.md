---
name: trade-acquisition-pipeline
description: Build (or extend) a screened, owner-enriched acquisition-target list of independently-owned home-services contractors (HVAC, plumbing, electrical) for whichever US state the user names, for TJ's search-fund buy strategy. Use when asked to build/update/deepen a deal list or pipeline for a state ("build a list for <state>", "do <state> next", "deepen the <state> pool", "enrich the callable owners"), or to backfill DNC data on an existing state sheet. Not tied to any one state — always ask which state if it isn't specified.
---

# Trade Acquisition Pipeline

State-agnostic protocol for sourcing acquisition targets in HVAC, plumbing, and electrical
contracting. Every run is scoped to whichever state the user names — there is no default or
"home" state baked in anywhere in this skill; always ask if it isn't given.

Read `references/schema.md`, `references/rollup_blocklist.md`, and
`references/licensing_boards.md` before starting a new state — they contain detail that does
not fit here.

## 0. Get the parameters

Before doing any sourcing, confirm with the user (or use these defaults if they don't say
otherwise):

- **State.** Required. Everything below is scoped to it.
- **Buy-box.** Default: 10–100 employees, private, independently owned.
- **Trade priority.** Default: HVAC and plumbing are priority; electrical is a lower-priority
  tail (it's frequently bundled into HVAC businesses anyway, so pure-play electrical acquires
  worse). Multi-trade shops rank with their highest-priority trade.
- **Mode.** New state build, deepen an existing list (paginate further), enrich the
  already-identified owners, or backfill a DNC sweep on an old sheet. Each is a valid,
  narrower ask — don't assume full build-from-scratch every time.

## 1. Tooling — check this every time, don't assume

Apollo.io and ZoomInfo MCP tools may both be connected. **Check whether Apollo's API is
actually usable before relying on it** — on a free/locked plan its search/match endpoints
return `API_INACCESSIBLE` and burning time re-testing it is wasted effort. If Apollo works in
the current session, it's a fine substitute for the ZoomInfo calls below. If it doesn't,
don't build a plan around it — say so and proceed on ZoomInfo alone.

**ZoomInfo cost tiers (this is the whole workflow, memorize it):**

| Call | Cost | Returns |
|---|---|---|
| `search_companies` | FREE | Firmographics, employee count, revenue, ZI company id |
| `search_contacts` | FREE | Owner NAME, title, DNC flags, `hasMobilePhone` — but NOT the number |
| `enrich_contacts` | COSTS bulk credits | The actual mobile number, email, accuracy score |

**The rule this drives:** `search_contacts` is free and already tells you who the owner is,
whether they have a mobile on file, and whether it's DNC-flagged. Build the *complete* owner
roster at zero cost first. Only spend `enrich_contacts` credits on owners who are confirmed to
have a reachable, non-DNC mobile. Never enrich blind — typically ~a third of owners are
DNC-flagged or have no mobile on file, and every one of those is a wasted credit if enriched
speculatively.

Useful parameters:
- `search_contacts`: `companyId` (comma-separated, batch ~30 at a time), `managementLevel="C Level Exec,VP Level Exec"`, `requiredFields="mobilePhone"` (filters to only contacts with a mobile on file), `sort="-contactAccuracyScore"`.
- `enrich_contacts`: takes `personId` values from the search step, max 10 per call.

## 2. Source companies (free)

Run `search_companies` against the target state using the SIC codes in
`references/sic_codes.md` (1711 = Plumbing/Heating/AC, 1731 = Electrical Work — these are
national codes, the same in every state) and the employee-count buy-box. Paginate — a single
pass is usually a small fraction of the statewide universe in that headcount band; the user
may want the list to grow well past the first pull, so don't stop after one page and call it
done unless asked for a small sample.

**The SIC codes are dirty in every state.** SIC 1731 in particular sweeps in AV installers,
alarm/security firms, telecom, fire protection, and solar companies, along with occasional
total garbage (lamp distributors, research institutes, trade associations). Screen every
record's actual business description before keeping it — expect to discard a large fraction of
raw 1731 hits.

## 3. Screen out roll-ups and franchises — before spending any enrichment credit

Read `references/rollup_blocklist.md`. It has (a) a seed list of national PE-backed
consolidators and franchise systems that show up in every state, and (b) detection heuristics
for the ones you won't have seen before: parent-company mismatch, size implausible for the
brand, HQ city ≠ operating city, brand+city naming patterns (a sign of a roll-up stamping its
brand across acquired locals), and a C-suite that's too large for the reported headcount.

These are **not acquirable from the operator** and outreach to them is wasted. Kill them at
sourcing, before any ZoomInfo credit is spent on them. Consolidation levels vary by state and
are spreading over time — don't assume any state is "clean" by default, and ask the user if
they know of regional roll-ups worth screening for harder in a given state.

Also watch for ownership flags that only surface once you have contact-level data: principals
sharing an email domain that belongs to a *different* company (e.g., two owners both on
`@someothercompany.com`), or an owner's email domain matching a known franchisor/dealer
network (e.g., a RainSoft dealer domain). Neither is catchable from company-level data alone —
read the enriched email domain on every target before finalizing.

## 4. Build the free owner roster

For every surviving company, run `search_contacts` to get: owner/executive name, title, DNC
flags (`mobilePhoneDoNotCall`, `directPhoneDoNotCall`), and whether a mobile is on file
(`hasMobilePhone`) — all free, all before spending a credit.

**Capture every principal, not just one contact per company.** You cannot buy a company
without knowing every person who has to sign. Where `search_contacts` surfaces multiple
owners/execs (co-owners, multiple generations, partners), record all of them in the
`Notes / Flags` column — this has repeatedly mattered (two-generation ownership, silent
co-owners who are DNC while a callable exec is the practical route in, etc.).

Classify each company by owner-contact status:
- **DIAL-READY**: owner cell already pulled via `enrich_contacts` and confirmed non-DNC.
- **ENRICH CELL (callable)**: owner name known, `hasMobilePhone=true`, not DNC-flagged, but the
  number itself hasn't been pulled yet. This is the highest-value queue — 100% expected
  conversion, no wasted credits.
- **Office line / email only**: DNC-flagged or no mobile on file. Do not call the personal cell.
- **NAME OWNER**: no C-level contact found in ZoomInfo at all — usually the smallest shops.
  Free next step is the state licensing-board lookup (see `references/licensing_boards.md`):
  the license's qualifying agent is public record and is usually the owner. The license number
  is also the strongest de-duplication key available, better than company name matching.

## 5. TCPA / Do-Not-Call — hard constraint, not optional guidance

These are personal mobile numbers from a data broker. Cold-dialing a cell on the Do Not Call
registry carries real TCPA liability. Never enrich or recommend dialing a DNC-flagged mobile.
Every row must carry its DNC status in the `Cell DNC` column, captured for free at
`search_contacts` time — check it before spending a credit, and check it again before anyone
picks up a phone. If a company's only contactable principal is DNC-flagged, its `Next Action`
is "Office line / email only", full stop.

## 6. Enrich, in priority order

1. `enrich_contacts` on the ENRICH CELL (callable) queue first — batches of 10 `personId`s.
   Expect a very high full-match rate; this is the highest-ROI move available and should
   usually run before any other enrichment.
2. Only after that queue is cleared, consider spending credits elsewhere.

## 7. Apply the revenue-per-employee (RPE) guard

ZoomInfo revenue is frequently modeled/imputed rather than reported, and the errors run in one
direction — always too high, never too low (watch for suspiciously identical revenue figures
across unrelated companies; that's a placeholder value, not a coincidence). Home-services
contractors run roughly $150K–350K revenue per employee.

- If `ZI Rev $` / employees falls within $100K–500K/employee: trust it. `Revenue Basis = ZI`.
- If it falls outside that band: discard it and re-estimate as `employees × $225,000`.
  `Revenue Basis = ESTIMATED`.
- `Est. EBITDA 12% $` = `Revenue (Best) × 0.12` in both cases — this is a flat placeholder
  margin, not a diligenced number. Always show `Revenue Basis` next to the EBITDA ranking so
  nobody mistakes a modeled estimate for a reported figure.

Track **Owner Confidence** (do we know who to call?) and **Size Confidence** (do we trust the
revenue?) as two independent columns. Collapsing them into one score hides the case where a
target has a fully verified owner and completely fictitious revenue, which happens often.

## 8. Build the spreadsheet

Use `scripts/build_sheet.py` (see `references/schema.md` for the full 24-column spec). It:

- Applies the RPE guard and computes `Revenue Basis` / `Est. EBITDA 12% $`.
- Assigns `Next Action` per the classification in step 4.
- Sorts DIAL-ready first, then ENRICH CELL (callable), then by `Est. EBITDA 12% $` descending.
- Zebra-stripes company rows in plain white/light-grey — **and nothing else**. The colour KEY
  row at the top is the *only* coloured region in the workbook; the user assigns row colours
  by hand afterward and their meaning is his, not ours. Never auto-colour company rows.
- When updating an existing workbook (deepen-the-pool / backfill runs), pass `--existing` so
  manually-applied row colours on previously-seen companies are preserved across the rebuild —
  only rows without a manual colour get re-striped.

