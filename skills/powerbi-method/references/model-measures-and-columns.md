# Model, relationships, measures and columns

Reference for the `powerbi-method` skill. Table, column and measure names are generic examples (`sales`, `production`, `incidents`); replace them with those of the model you are working on. The `D_`, `F_`, `P_` prefixes and the Spanish-language names such as `Dim_Fecha`, `FechaCorte` and `Inicio` are the ones used by the method; use your own equivalents if your warehouse is in another language.

## 1. Architecture: thin reports over a shared model

- Published report files are report only, connected live to a semantic model published in the service. The model is built once and several reports reuse it. A model embedded in the file is only a cache.
- **Report as an app**: one visible `Inicio` (Home) page; the rest hidden in view mode. Navigation through bookmarks and action buttons, plus menu pages per area. Native page-to-page navigation is not used.
- **Common template**: shared theme and logos as registered resources; new reports are cloned from the template.
- **`.pbip` projects**: new reports are saved as a project (report in PBIR, one JSON per visual; model in TMDL, one `.tmdl` per table) and built and fixed by code (see `editing-and-verification-by-code.md`).

## 2. Model tables: names and shapes

Each model table is a warehouse view:

- `D_...` dimensions, `F_...` facts, already-aggregated business views, `Dim_Fecha` calendar, `P_...` disconnected tables (`P_UltimaActualizacion` = refresh stamp; `P_ValoresMedida` = measure selector).
- **The model table is named for its role even if the SQL view has another name**: the view `vw_IncidenciasActual` enters the model as `F_Incidencias`. Columns, on the other hand, are not renamed.
- **"Snapshot" pattern** for data loaded in cut-offs: an insert-only SQL table (each load is a snapshot with `FechaCorte`; reloading the same snapshot does not duplicate because the views keep the latest). In the model, two tables: `..._Actual` (latest snapshot, today's state) and `..._Hist` (all snapshots, evolution).
- **Period versioning** from SQL: views with the year in the name (`...2025`, `...2026`); a new period clones the view, it does not overwrite.
- **Catalogs from SQL, never typed into the model.** A small dimension written as `DATATABLE` missed the new element the day it was added at the source, silently: it had no relationships, so the totals reconciled, but the slicer did not offer it. It is replaced by a table that reads from the catalog view, keeping the table name and columns so as not to break visuals.

## 3. Relationships (TMDL)

```
relationship <Fact>_<Dim>
    fromColumn: <Fact>.<KEY>
    toColumn:   <Dim>.<KEY>
```

- **Many-to-one, single direction, active** (these are the TMDL defaults, which is why they are not declared).
- **Composite key resolved in SQL**: if the grain of the dimension is entity + period, the view exposes a single concatenated text column and the relationship goes through it.
- The current-snapshot table and the history table are both related to the same dimension; the fact's business date goes to `Dim_Fecha[Date]`.
- A comment saying what it joins and on which key. Careful: in TMDL, a `///` comment before `relationship` broke model loading in the Desktop versions tested; check that your version accepts it before generalizing.
- When replacing a table by code, **relationships are noted before and recreated after**, preserving cardinality, filter direction, security filter and `IsActive`: removing the table deletes them, and the server rejects the whole transaction if any is left pointing at nothing.

## 4. DAX measures

Shape of each measure in TMDL:

```
/// What it counts, over which population, and where the threshold comes from, if any.
measure 'Readable name (unit)' = <DAX>
    formatString: #,##0
    displayFolder: <Area> | <Area> evolution | KPI Times | KPI Extremes
```

- A `displayFolder` per area, with separate folders for evolution, times and extremes. A `///` description on every measure that is not obvious.
- `formatString` always on the measure: `#,##0` counts, `0.0` minutes or hours, `0.0%;-0.0%;0.0%` percentages, `+#,##0;-#,##0;0` signed variations, `dd/mm/yyyy hh:mm` date stamps.
- Base measures: `COUNTROWS(fact)`, `DISTINCTCOUNT(fact[Entity])`, `SUM`.
- **Filter with a Boolean predicate**, not `FILTER` over the table: `CALCULATE(SUM(F[Amount]), F[Type] = 1)`. `FILTER(table, flag = 1)` only inside the `...X` iterators.
- **Populations are defined by SQL**: the view exposes flags (`ES_<POPULATION>`, `CANCELADO`, `ANOMALIA_<X>`, `EXCESO_<THRESHOLD>`, `BANDA_<X>`) and the measures only count or filter them. **Thresholds live in a single definition** (the view, or the script that also generates any report in another format), never duplicated in DAX: two reports counting different things under the same name is the failure this avoids.
- **Long-tail distributions**: median (`MEDIANX`), mean, P90 (`PERCENTILEX.INC(..., 0.9)`) and worst case (`MAX`) **together**; the distance between median and mean IS the finding. And the **extremes counted, not averaged**: `'Entities over <Threshold>'`, `'... over 2x <Threshold>'`, their `%` of the population and **the quantity they carry**, not just how many there are.
- **Coverage next to each figure**: `'X with data'` and `'% X with data'`.
- **Human units**: `DIVIDE(minutes, 60)` when the quantity is thought of in hours.
- Always `DIVIDE`. `BLANK` when there is no data. **Documented exception**: a measure `'X (color)' = COALESCE(COUNTROWS(...), 0)` only for conditional formatting.
- **Data stamp as text** for a card: `"Data as of " & FORMAT(MAX(F[FechaCorte]), "dd/mm/yyyy hh:mm")`.
- **Same quantity, same base across the page**: a review found an amount card that did not exclude cancelled items and a percentage of the same amount that did; whoever divided the two cards did not get the displayed percentage. Every measure that shares a quantity shares the population filter.
- A measure cannot have the same name as a column of any table in the model; rename the column or give the measure a different name.

**Time intelligence.** When the axis of analysis is the business period or the snapshot, and not the calendar, use the "previous snapshot" pattern instead of `TOTALYTD` or `SAMEPERIODLASTYEAR`:

```
measure 'Change vs previous snapshot' =
    VAR _cutoff   = MAX ( F[FechaCorte] )
    VAR _previous = CALCULATE ( MAX ( F[FechaCorte] ), REMOVEFILTERS ( F[FechaCorte] ), F[FechaCorte] < _cutoff )
    VAR _prev     = IF ( NOT ISBLANK ( _previous ),
                         CALCULATE ( COUNTROWS ( F ), REMOVEFILTERS ( F[FechaCorte] ), F[FechaCorte] = _previous ) )
    RETURN IF ( NOT ISBLANK ( _prev ), COUNTROWS ( F ) - _prev )
```

With `FechaCorte` on the axis it gives one figure per snapshot and its change. **Warning**: in a script of Desktop's TMDL view, a measure cannot call another one created in the same script; repeat the logic with a comment explaining it, or create them in two passes.

## 5. Columns

- **They keep the SQL contract** (language, casing, not renamed).
- `summarizeBy: none` on text, dates and text flags; `sum` on numeric columns and on 0/1 flags (`int64`), which are thus counted by summing.
- **Import model**: a new column in the view does not appear on its own. It must be declared in the `.tmdl` (or the schema refreshed) and then refreshed; otherwise the measure that uses it errors and the page comes out blank.
- **Every column in the view has an alias.** An unnamed expression (for example, an `ISNULL(...)` without an alias) prevents the model from loading.
- **Exception to not renaming**: a link column is renamed in the model to the text that should appear as the header (with `sourceColumn` intact). Visuals and DAX reference it by its model name, never by `sourceColumn`: that breaks the visual with "problem with one or more fields".
- **Clickable links**: a column with `dataCategory: WebUrl` and `urlIcon: true` (icon instead of the full URL). Invented formatting properties break the visual without a clear error: use the documented route.
- **Calculated columns**: they exist in old models and are fragile (three were left broken pointing to a table that had been removed and forced refreshing with `DataOnly` instead of `Full`). They reinforce the rule: measures, or a column in the SQL view.

## 6. Power Query / M

A single step, no transformations:

```
let
    Source = Sql.Database("<server>", "<db>"),
    dbo_<view> = Source{[Schema="dbo", Item="<view>"]}[Data]
in
    dbo_<view>
```

All cleaning, joining and population calculation goes in the view. Server and database come from configuration, never literals in the repository; credentials are supplied by Desktop or the gateway. Changing the source of a table means changing the `Item` and the step name, nothing else.

## 7. Design and visual types

- Usual mix: slicer (filter panels), action buttons (navigation), card, KPI and indicator (compliance against target), table and matrix (detail). Geospatial with shape map or Azure map over embedded GeoJSON. Custom visuals (editable matrix with write-back, Deneb or Vega, advanced slicers, embedded Power Apps) only in the operational report; the rest, read-only.
- **Progressive disclosure on three levels**: header and filters → five macro cards → two context charts → "what to look at" chart and table → drillthrough to a hidden detail page (`pageBinding: Drillthrough`, drill filters on the dimensions, "< BACK" button).
- **Extremes first**: on top, the counted ones ("N cases over X"); in the middle, the clickable **bands** (cuts at the business thresholds, not at round numbers; the worst band on top); below, **the worst cases already sorted** from highest to lowest. The order is part of the information.
- **Card = figure + label + footer with the population** ("over N of M"). Without the footer, it cannot be read.
- **No total row under medians**: it reads as a sum and it is not.
- **Detail of a process by phases**: stacked bar, one record per row, one phase per segment; waiting time in warm tones and active time in greens, to see where the time went without reading numbers. In the detail table, the actual dates and times of each phase, not just the durations.
- **Colors taken from the existing report, not invented.** With a single series, the `dataPoint` goes without a selector (with a `metadata` selector it falls back to the theme's blue). Theme colors that do not resolve in a code-generated visual are written as `'#RRGGBB'` literals.
- Palette and typography are those of each client's template.
