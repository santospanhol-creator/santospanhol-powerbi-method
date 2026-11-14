# Editing by code and verification

Reference for the `powerbi-method` skill. It is valid for a `.pbip` project (report in PBIR, model in TMDL) on Windows with Power BI Desktop. Microsoft changes the schemas between Desktop versions: pin versions and test each update. What is marked "observed" is what was verified in practice, not a specification.

## 1. Formats

`.pbip` = a folder `<Name>.Report/definition/` (PBIR: `pages/pages.json`, `pages/<id>/page.json`, `pages/<id>/visuals/<id>/visual.json`) + `<Name>.SemanticModel/definition/tables/<table>.tmdl`. Everything is text: it is edited with small scripts, one per change, which are versioned in git. **The `.pbix` or `.pbip` lives in the client's working folder, not in git.**

## 2. Editing rules (each one cost hours)

1. **With the `.pbip` open in Desktop, its TMDL is not edited**: on closing, Desktop writes its in-memory model over the files and takes what you wrote with it (it happened with dozens of columns and measures). Close it first.
2. Columns and measures are inserted **before the `partition` block** (TMDL wants it at the end of the table; after it, the file will not open). A new `lineageTag` (UUID) per object.
3. **Idempotent**: check whether the object already exists before adding it; a `.bak_<reason>` copy before writing; and afterwards **count in the file** what was written. The script's "written" does not count.
4. When adding to a format list (`objects.values` of a table), **append, do not replace**: assigning the whole list erased the font formatting without raising an error.
5. **Do not click on the canvas in edit mode to automate**: it creates empty visuals or opens editors. The active page is changed in `pages.json` (`activePageName`) and "Apply external changes" is pressed.
6. **Pages with bookmarks are not cloned or rebuilt in place**: bookmarks point to visual ids, and the clone shows all layers at once. A new page is created beside it and the old one stays hidden until decided.
7. A **mold** (generator function) for sibling pages: same layout, so that what is learned on one applies to the other. But the figures belong to each business: each line measures what hurts it.
8. **A business-line filter at page level, never report level**: a global filter designed for the first line left the second line's page entirely blank, without an error.
9. Observed: a change made through TOM on the open model does not exist on disk until saved; if Desktop is closed without saving, it is lost. Check the `.tmdl` or the `visual.json` on disk.
10. Observed: a five-level matrix rendered only up to the first block; four levels at most and the detail on another page. Do not write expansion states (`expansionStates`) by hand: the matrix starts collapsed and the +/- works by itself.
11. Observed: if a PowerShell script file has no BOM, Windows PowerShell 5.1 reads it as ANSI and a character such as "·" reaches a measure garbled. Use a BOM, or pass the expression from a UTF-8 file; when reading PowerShell output from Python, force UTF-8.
12. Observed: with two Power BI instances open, "the first Power BI window" may be someone else's. Pick the window by its title and check which one is in front right before any action on it.

**TMDL scripts to paste** (when the model is a `.pbix` not saved as a `.pbip`): one `createOrReplace` script plus `ref table <table>` with the measures, and another with the relationships; they are pasted into Desktop's TMDL view.

## 3. The open model, without an MCP server

Desktop spins up a local Analysis Services instance: the port comes from the `msmdsrv` process (`-s <folder>` → `msmdsrv.port.txt`, UTF-16 with nulls). With the libraries that ship with SSMS (located on the machine; not redistributed):

- **ADOMD** to read: `EVALUATE SUMMARIZECOLUMNS(...)`, or evaluate all measures.
- **TOM** to write: replace a table's partition with M, recreate relationships, `RequestRefresh(DataOnly)` (not `Full`: it recalculates all calculated columns and one broken column fails the whole transaction). With `--ver` (dry run) first, a copy of the file before, and a DAX check after.

Third-party MCP servers and Microsoft's Power BI one also exist; this skill does not depend on any of them and is compatible with them: the method says which order to give, the tool executes it.

**Preview before building**: a local HTML generated with exactly the same rows that are loaded into the database, to agree on the page before assembling it in Power BI.

## 4. Verification: how a report is accepted as correct

- **Three sources, the same figure**: the deliverable (for example, an Excel file), the SQL view that Power BI consumes and the open model, category by category. Two matching is not enough (a real failure: the summary had cases for a new category and the database had none, because the load did not read the new sheet).
- **And the first leg**: what was loaded against the source system (every record with an id exists there and carries the same quantities). If all three are wrong in the same way, they reconcile and are wrong.
- **Coherence of all measures** against the model: none blank or in error, and internal rules (median ≤ P90 ≤ worst case; `>X` ≥ `>2X` ≥ `>4X`; % between 0 and 1; subset ≤ total).
- **Count the checks EXECUTED, not only the failed ones**: DAX comes back as text; a badly written type check meant none was executed and the summary said "0 failed". A verifier that concludes "I find nothing" must first prove it can find something (seed a known failure and check that it detects it).
- **Before calling a measure a "failure", read its DAX**: the population it uses may be the right one and the assumption, yours.
- **Distinguish "another model is open" from "this model is broken"** in the warning: a warning that scares without reason stops being read.
- **PBIR integrity** before opening: page order, hidden pages, number of visuals counting `visual.json` files (not folders: syncing a shared folder leaves empty folders), duplicate ids, internal id equal to the folder name, every cover-page button leads to a page that exists, all JSON valid.
- **Balances do not move**: splitting by activity or by another key only redistributes; the sum by key and period must equal that of the official source. It is checked on every regeneration, with an explicit tolerance.
- **No blanks in filters**: a blank hides amounts when filtering. Explicit labels ("(outside master data)", "(empty in master data)").
- **Compare like with like**: eight months of actuals against the budget for the same eight months; percentage points and not % for a rate.
- Verification goes into the **generation routine** (the same process that generates and loads the data), and the load into the database is done at the same time as the reports are delivered, so that what is seen in Power BI is the same as what was sent.
- **Never overwrite what the user edits by hand**: before regenerating a report they have open, save, make a copy, generate the previous version in another folder, compare, and pass to the generator whatever they changed.
- Each change leaves a short document (what was requested, what was found, why, what is learned) and its commit with the reason why.
