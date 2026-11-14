---
name: powerbi-method
description: Method for building and reviewing Power BI reports - thin reports over one shared semantic model, star schema over the data warehouse, DAX measure conventions, report design, verification against SQL, publishing and gateway refresh. Use when creating, extending or auditing a Power BI report or semantic model (dataset), or when editing a .pbip project (TMDL/PBIR) by script.
---

# Power BI Method

A generic method for building a Power BI report on top of a relational data warehouse (SQL Server or similar), with one shared semantic model and verification against the source. It does not depend on any client or industry. If the user writes in another language, answer in that language: the method's terms (star schema, thin report, bookmark, gateway) are standard Power BI terms.

Detail and rationale live in two reference files, read when the step calls for them:

- `references/model-measures-and-columns.md`: model shape, relationships, DAX measures, columns and Power Query.
- `references/editing-and-verification-by-code.md`: editing a `.pbip` project (TMDL/PBIR) by script, and how a report is accepted as correct.

## Principles

1. **One semantic model, many reports.** The model is built and published once; reports are thin files connected live to it.
2. **The warehouse rules.** The model starts from warehouse views; whatever can be solved in SQL is not transformed in Power Query.
3. **Star schema, not a flat table.** Facts and dimensions, one-to-many relationships in a single direction.
4. **Measures, not calculated columns.** Business logic lives in model measures.
5. **Read-only.** Power BI is for SEEING. Writing or triggering actions is done in another tool.
6. **No reconciliation against SQL, no publishing.**

## Step 1. Business questions and source

- With whoever consumes the report: a closed list of questions it must answer (about 10 at most) and the device it is read on (mobile or desktop).
- With whoever owns the warehouse: which views feed each question, the grain of each one, and the SQL control query for each key total.
- If a metric is missing from the warehouse, request the view; do not patch it in Power Query.

## Step 2. Star schema over the warehouse

Detail in `references/model-measures-and-columns.md`.

- One model table = one warehouse view (1:1). Prefixes carried over from SQL: `D_` dimensions, `F_` facts, already-aggregated business views, `Dim_Fecha` calendar (use `Dim_Date` if your warehouse is in English).
- One central business dimension and one date dimension shared by all facts. Own calendar marked as a date table; Power BI's auto date/time turned off.
- Grain documented per table in one sentence ("one row per X and per Y"). One-to-many relationships, from the dimension to the fact, single direction. Bidirectional only with written justification. Avoid many-to-many: if it appears, a bridge dimension is missing.
- **Period versioning from SQL**: if the warehouse clones views per year in the name, the model points to the view for the period; a new period is added, it does not overwrite the previous one (that would break the history).
- **Columns are the contract with SQL**: warehouse names are kept. No renaming in Power Query; the visible format is handled in the report.
- Import by default. DirectQuery only if the data must be real-time and the source can take it.
- Performance: only the columns that are used, correct data types, keys and technical columns hidden, no long text in facts.
- **Composite key resolved in the view** (a single column), never relationships on two columns nor keys manufactured in Power Query.
- **Data loaded in snapshots → "snapshot" pattern**: an insert-only SQL table with `FechaCorte` (cut-off date), and in the model one `..._Actual` table (today's state) and one `..._Hist` table (evolution).
- **Catalogs and small dimensions come from a SQL view, never typed into the model** (`DATATABLE`): they silently drift out of sync.
- **Single-step M source** (`Sql.Database` and the view), with no transformations; server and database from configuration, never literals in the repository.
- **Business-line filter at page level, never report level.** When a second business line is added, review everything the first one assumed: view, master data, global filters and relationships.
- Disconnected table `P_UltimaActualizacion` holding the last-refresh stamp, visible on the cover page. Other disconnected support tables also use the `P_` prefix.
- Deliverable: a model document with tables, grain, relationships and a table of measures.

## Step 3. DAX measures

- They are hosted in the fact table they belong to, or in a `_Medidas` table if they are cross-cutting.
- **Domain names**, readable, with the unit in the name in new models (`'Waiting time (min)'`, `'% Records with data'`). Percentages with `%` in the name; net and gross with a clear, documented marker.
- **Versions and variance**: when the business revolves around compliance (target, budget, actual), one measure per version and one variance measure per version.
- One base measure per concept (sum, count); the others **are built on measures and do not repeat logic**.
- Always `DIVIDE` instead of `/`. `BLANK()` when there is no data, not a misleading zero.
- Filter with a Boolean predicate inside `CALCULATE`, not `FILTER` over the whole table. `VAR` for readability and performance.
- Format (decimals, unit, thousands) set on the measure, not on the visual.
- **Populations and thresholds defined in SQL** (flags such as `ES_...`, `EXCESO_...`, `BANDA_...`), a single definition shared with any other report that counts the same thing; DAX only counts and filters.
- **Long-tail quantities (times, waits, amounts)**: median, mean, P90 and worst case together, and the **extremes counted**, not averaged, alongside the quantity they carry.
- **Coverage next to each average** (`'% X with data'`): an average computed over 30% of the records is the average of those records.
- **Same quantity, same population base** across the whole page.
- Evolution by snapshot with the "previous snapshot" pattern (`VAR` + `REMOVEFILTERS` + `<`) when the business lives by cut-offs and not by calendar.
- Document each measure: what it calculates, grain, source and SQL control query. Without that pair, it is not finished.

## Step 4. Report pages and design

- **Report as an app**: a single visible page, `Inicio` (Home); the rest hidden in view mode, reached through navigation. Menus per area.
- **Navigation through bookmarks and action buttons**, not native page-to-page navigation. Naming convention `<Page>_<Action>` so you do not get lost when there are hundreds.
- **One idea per page.** Reading order: KPIs and traffic lights on top, comparisons in the middle, detail (table or matrix) below.
- Base visuals: card, KPI or indicator for compliance against target, slicer in a side panel, table or matrix for detail. Maps only if the data is geographic.
- Custom visuals only in an operational report that needs them and after a security review; executive reports are read-only with native visuals.
- **Phone layout** on every page consumed from the mobile app: KPIs and traffic lights first, one column, no wide tables.
- **Progressive disclosure**: filters → five macro cards → context → "what to look at" → drillthrough detail, on a hidden page with a back button. **Extremes first**, clickable bands, worst cases already sorted; a card with a footer stating the population ("over N of M"); no total row under medians.
- Sibling pages from **one mold**, but each line measures what hurts it.
- A page with bookmarks is not cloned or rebuilt in place: the new one is created beside it and the old one is left hidden.
- Common template: same theme, same logo (as a registered resource), same `Inicio` page. A new report is cloned from the template, not started blank.
- Visual style: few colors, a consistent traffic light (same threshold throughout the report), typography and sizes from the theme, no decoration that does not inform.

## Step 5. Validate

Before publishing, another person (or a verifier other than whoever built it) checks, against the model open in Desktop through its local Analysis Services port (see `references/editing-and-verification-by-code.md`):

1. Every key total = SQL control query (same filters) = the deliverable that is sent, if any, category by category. Record measure, Power BI value, SQL value and difference.
2. **All** measures evaluated: none blank or in error, and coherence rules (median ≤ P90 ≤ maximum; `>X` ≥ `>2X`; percentages in [0, 1]; subset ≤ total). Report how many checks **were executed**, not only how many failed. Before calling a measure a failure, read its DAX.
3. Slicers: select, clear and combine; totals reconcile on every selection.
4. Navigation: every button or bookmark goes where it says; no visible orphan pages.
5. Mobile: open the phone layout.
6. First rows of each table and matrix compared with SQL (this is where aggregation failures show up first).
7. If the report was edited by code: project integrity (visuals counted by `visual.json`, unique ids, buttons pointing to pages that exist, valid JSON).

## Step 6. Publishing

- One workspace per client or environment; the semantic model is published once and reports are published pointing to it (live connection).
- Model owner: a named person, never "nobody". Minimum access; the consumer joins as a viewer.
- If there is row-level security, it is defined in the model and tested with "View as".
- No credentials embedded in the report file.

## Step 7. Refresh and gateway

- On-premises source: **On-premises data gateway** installed; the data source is assigned in the service with credentials stored in the gateway, never in the report or the repository.
- Schedule in line with when the data changes in the warehouse (once or twice a day is usually enough, after the loads), with a failure notification to the owner.
- After the first scheduled refresh, check that the `P_UltimaActualizacion` stamp advances and that the key total still reconciles. A "green" refresh is not proof: look at the data.
- Document: gateway, data source, schedule, owner and who is notified on failure.

## Final checklist

- [ ] Business questions agreed with whoever consumes the report
- [ ] Source views delivered, with a control query per key total
- [ ] Star schema; grain and relationships documented
- [ ] Own calendar and auto date/time disabled
- [ ] Columns keep the warehouse contract, not renamed
- [ ] Measures documented; `DIVIDE`; no repeated logic
- [ ] Thin report connected to a shared model (not one model per report)
- [ ] `Inicio` visible, the rest hidden, navigation by bookmarks
- [ ] KPIs and traffic lights on top, detail below, one idea per page
- [ ] Phone layout done
- [ ] Totals reconcile with SQL (evidence recorded) and with the deliverable, if any
- [ ] All measures evaluated with no blanks or errors; executed checks counted
- [ ] Populations and thresholds defined in SQL; coverage next to each average
- [ ] Catalogs from a SQL view (no `DATATABLE`); line filters at page level
- [ ] No credentials in the report file; source authenticated in the gateway
- [ ] Model owner and refresh schedule defined
- [ ] Last-update stamp visible
- [ ] Nothing sent to the final recipient without the approval of the report owner

Spanish version: references/es/
