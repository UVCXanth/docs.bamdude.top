---
title: Stats & Energy
description: Cumulative print stats, per-print energy capture from your smart plug, and date-range farm-wide totals
---

# Stats & Energy

The Stats page is BamDude's dashboard for "what did the farm actually do?" — print counts, filament consumed, energy used, time spent. It's driven entirely by `print_archives` (no separate stats table to drift), so the numbers always match the archive list under the same filter.

## :material-chart-bar: Top-level KPIs

The header bar shows four lifetime counters:

| Metric | Source |
|---|---|
| **Prints completed** | `print_archives` rows with `status='completed'`. |
| **Filament consumed** | Sum of `filament_used_grams` across all archives in range (not just completed), grouped by material/colour. Failed / cancelled prints contribute the filament **actually** extruded, not the full slicer estimate, so the total matches what inventory was deducted. |
| **Print time** | Measured, not estimated: for each archive with both a start and an end timestamp it's the elapsed time between them, and only archives missing one falls back to the slicer's `print_time_seconds`. A print BamDude had to recover after a disconnect is credited with its *expected* duration rather than the whole offline gap — otherwise a weekend outage would add days of phantom print time. |
| **Energy** | Two pairs of tiles, always both. **Energy While Printing** / **Cost While Printing** — the sum of `energy_kwh` and `energy_cost` (the per-print figures recorded at completion) over the prints in range: what the jobs themselves drew. **Energy At The Plug** / **Cost At The Plug** — everything the smart plugs counted, idle included: the plugs' lifetime counters for all time, hourly snapshots for a date range. The gap between the pairs is what standing still costs. See [Energy → Two Energy Figures](energy.md#two-energy-figures-printing-and-at-the-plug). |

The cost figures use `default_filament_cost` and the electricity price, both set in **Settings → Printing → Cost Tracking**.

## :material-calendar-range: Time-range filter

A range picker above the KPIs scopes everything below — **Today**, **This Week**, **This Month**, **Last 7 Days**, **Last 30 Days**, **Last 90 Days**, **This Year**, **All Time** or a **Custom Range**. The presets follow your calendar day: **Today** is today in your time zone from its first minute, a week starts on Monday, and the last day of a range is included. The KPIs become the same metrics over the chosen range; the per-printer breakdown re-renders to match.

## :material-chart-line: Time-series charts

Below the KPIs, two stacked line charts:

- **Prints per day** — bar chart of completed archives bucketed by date, colour-coded by printer.
- **Filament per day** — same bucketing, stacked by material so you can see "we shifted from PLA-heavy to PETG-heavy in March".

Hover any bar to see the breakdown for that day.

## :material-printer-3d-nozzle: Per-printer breakdown

**Prints by Printer** is a bar chart with one bar per printer and a switch for what the bar measures — **Weight** (filament), **Prints** or **Time** (print hours). Energy and cost are not broken down per printer — the energy tiles are farm-wide figures.

**Defects by printer** — printed, defective and the rate per machine over the selected timeframe, worst rate first; completed prints only.

## :material-flash: Per-print energy capture

Energy tracking is opt-in. To capture it on each print:

1. Add a smart plug under **Settings → Smart Plugs** (Tasmota, Home Assistant, REST/webhook, or MQTT — see [Smart Plugs](smart-plugs.md)).
2. Bind the plug to a specific printer.
3. The plug must report cumulative kWh — Tasmota's `Total` field, HA's `sensor.<plug>_energy_total`, etc.

On each print:

- At `print_start`, BamDude reads the plug's current kWh into `print_archives.energy_start_kwh`.
- At `print_complete` it reads the plug again, computes `current - energy_start_kwh`, and stores **the delta directly** in `print_archives.energy_kwh`. There is no separate `energy_end_kwh` column — the end reading is consumed during the subtraction and discarded.
- The reads are restart-resilient — values come from a fresh DB session each time, never an in-memory dict, so a backend restart between start and complete doesn't break the capture.

If a plug isn't bound, or the plug is offline at one of the two boundaries, `energy_kwh` stays null and that print is left out of the printing figures.

### Hourly snapshots

Independently of the prints, BamDude takes an **hourly snapshot** of every plug's cumulative kWh into `smart_plug_energy_snapshots`. They are what **Energy At The Plug** is read from for a date range: per plug `max(0, last_in_range - baseline)`, summed across plugs. They do not fill in a print whose own capture is missing.

Snapshots are kept: there is no retention setting for them and nothing prunes them, so a date range of any age can still be answered. The table grows by one row per plug per hour, plus one at each print start.

## :material-trophy: Records

A **Records** panel picks out the extremes of the selected range: **Longest Print**, **Heaviest Print**, **Most Expensive**, **Busiest Day** and the current **Success Streak**.

**Most Expensive counts the electricity, not just the filament.** It ranks prints on filament **plus** the power the print actually drew **plus** the [printer wear](#printer-wear) charged to it, and shows the split (`filament` / `power` / `wear`) so the figure can be checked against the print's own page. A long print that ran a printer for ten hours does not lose the podium to a short one on pricier filament while having cost more to run.

!!! info "A print with no smart plug is ranked on its filament alone"
    Its power was never measured, and putting a made-up number on the podium would
    be worse than leaving it out. Prints measured at **zero** and prints **never
    measured** stay distinguishable.

---

## :material-bullseye-arrow: Cost calculations

| Cost | Formula |
|---|---|
| **Per-print filament cost** | Sum of each tracked spool's share (`grams_from_spool × spool.cost / spool.weight`). Any grams **not** covered by an assigned spool are topped up at `default_filament_cost / 1000` per gram, **when a default is set**. So a multi-colour print with only some AMS slots mapped to inventory still reflects the whole print, not just the tracked slots. |
| **Per-print energy cost** | The print's energy costed along the plug's counter, each step at the electricity price in effect then — not the whole print at one price. Zero when no plug capture (`energy_kwh IS NULL`). See [Energy → Cost Calculation](energy.md#cost-calculation). |
| **Cost At The Plug** | What the plugs counted — for all time or for the selected range — costed the same way, along each plug's hourly snapshots. Changing the electricity price does not re-cost the past. |
| **Per-print wear** | The print's measured duration × the printer's **Wear cost per printing hour**, stamped when the print ends. Empty when the printer has no rate or the duration was not measured. See [Printer wear](#printer-wear). |
| **Total** | Filament + energy + wear. |

!!! warning "A rate you have not set is no rate"
    With no `default_filament_cost` configured, untracked filament is **not**
    costed — a print with no rate carries **no cost** at all, rather than a cost of
    zero, which reads as free.

    The same rule holds in the initial archive estimate, in the cost
    recalculation and in project print plans — no path prices a print with a
    number nobody entered, so one print never carries two different totals.

    Nothing changes if you *have* set a price: a spool's own cost per kg still wins
    over the default, and the default still covers filament from spools that are
    not in your inventory.

These feed the per-archive cost line in the archive detail card and the project / print-plan totals.

### Printer wear

A printer can carry an hourly rate for its own wear — nozzles, belts, the machine paying itself off. Set it in the printer's settings (**Edit Printer** → **Wear cost per printing hour**); an empty field or `0` turns it off for that printer.

- **Which prints get a figure.** Every print that ends on a printer with a rate — completed, failed or cancelled — is charged for the time it actually ran: the measured time from its start to its end, at the rate the printer had at that moment.
- **Which do not.** A print whose duration was not measured gets no figure, rather than a guessed one: a print closed after a restart of BamDude or closed by an operator, and a print BamDude joined when it was already running. A printer without a rate charges nothing.
- **A changed rate applies to new prints only.** The figure is stored with the print when it ends. Changing or clearing the rate later does not touch prints that have already ended, and prints that ended before a rate was set are not charged afterwards.
- **Where it shows.** On the archive card, next to the print's other costs (the tooltip names the rate); on this page as the **Printer Wear** tile, shown when the selected period has any wear charged, and as `wear` in the **Most Expensive** split; and in an order's cost in the [Workshop](projects.md#the-figures), as its own `wear` component — so the order's margin counts it too.

## :material-database-export: Exporting

The header has an "Export CSV" button that dumps the current view (filtered range) as a CSV with one row per archive — useful for invoicing print-as-a-service runs or feeding the data into another tool.

The Maintenance page has a similar Excel export for service intervals — see [Maintenance](maintenance.md).

### Export options

| Format | Best for | Contents |
|---|---|---|
| **CSV** | Spreadsheets, ad-hoc analysis, scripts | One row per archive: printer, file, status, start time, duration, filament grams, filament details, energy kWh, costs |
| **Excel** | Reports with formatting, sharing with non-technical stakeholders | Same columns as CSV plus formatting, frozen header, per-column types |

Both exports respect the **currently active filters** — date range and per-user filter. Reset filters first to export the full dataset.

---

## :material-view-dashboard: Widget-based dashboard

The Stats page is a customisable grid of widgets, not a fixed report. You can:

- **Drag** any widget by the handle in its header to reorder
- **Resize** with the size button in the header — it cycles through a quarter, a half and the full width
- **Hide** widgets you don't need — the **N Hidden** button in the page header lists them and brings them back
- **Reset Layout** — the button in the page header restores the original arrangement

The layout is kept **in the browser**: another browser or device starts from the default arrangement.

### Available widgets

| Widget | What it shows |
|---|---|
| **Quick Stats** | Four tiles for the active range: **Total Prints**, **Print Time**, **Filament Used**, **Filament Cost**. |
| **Energy** | The four energy tiles — **Energy While Printing**, **Energy At The Plug**, **Cost While Printing**, **Cost At The Plug** — and **Printer Wear** when any wear was charged in the range. |
| **Success Rate** | A ring gauge with the **Successful**, **Failed** and **Cancelled** counts beside it, and dispatch errors on their own line. At half width or wider it also lists the prints per printer. See the note below on how cancelled prints are counted. |
| **Time Accuracy** | A gauge of how close actual print times were to the estimates (100 % is a perfect estimate), with the per-printer averages at half width or wider. Prints that BamDude had to recover after a disconnect are excluded: their end time is reconstructed from the estimate, so they'd score a perfect 100% nobody measured. |
| **Failure Analysis** | The failure rate for the range and the top failure reasons — see [Failure Analysis](failure-analysis.md). |
| **Print Activity** | A calendar heatmap of prints per day; for a range of a week or less, an hour-by-hour heatmap instead. |
| **Records** | The extremes of the range — see [Records](#records). |
| **Printer Stats** | **Prints by Printer** (a bar per printer, switchable between **Weight**, **Prints** and **Time**), **Print Duration**, **Print Habits**, **Print Time of Day** and **Defects by Printer**. |
| **Filament Trends** | **Usage Over Time**, **By Material** and **Color Distribution**, with the [By material number](#by-material-number) table below them. |

### By material number {#by-material-number}

The **Filament Trends** widget ends with a **By material number** table: the filament drawn from the spools that carry each [material number](inventory.md#material-number) in the selected range, and what it cost. Its columns are **Material number**, **Spools** (how many different spools were printed from), **Consumed** and **Cost**; the heaviest consumer is listed first.

The table is read from the built-in inventory's usage records, so it counts prints only — AMS weight corrections and runout close-outs are left out — and spools without a material number are not listed. It appears once at least one numbered spool was printed from in the range, and stays empty in Spoolman mode.

### Cancelled prints have their own bucket

Prints you (or the queue) stopped are **not** counted as failures. `stopped`, `cancelled`, and `skipped` archives land in a dedicated **Cancelled** bucket, separate from genuine quality failures (`failed` / `aborted`), so aborting a job doesn't punish your numbers:

- **Success Rate** divides by *completed + failed only* — cancelled prints are excluded from both the numerator and the denominator, so stopping a print never drags the gauge down.
- The **failure-rate report** (and its weekly trend) uses the same rule — cancelled / stopped / skipped prints are excluded from both sides of the ratio.
- They **still count toward Total Prints**, and stay visible on their own **Cancelled** line in the Success Rate breakdown, so nothing silently vanishes.

### Dispatch errors are not failed prints

An attempt to start a print that ended before anything was printed — the upload or the start was refused, the printer never acknowledged it, or it took the file but did not start — is a **dispatch error**, not a failed print. It is left out of the Success Rate, the failure-rate report and its weekly trend, the daily charts and the Success Streak, and shows on its own **Dispatch errors** line. Like cancelled prints, dispatch errors still count toward **Total Prints** and appear in the export. Failed prints from before this was recorded, with no cause stored, still count as failed. The Telegram `/stats` command counts the same way: failed means a failure the printer detected, and stopped or cancelled prints and dispatch errors have lines of their own.

### Per-User filtering

When you hold the `stats:filter_by_user` permission (Administrators only by default), a **user dropdown** appears in the stats header next to the timeframe selector. Selecting a user filters every widget, the failure-analysis report, and CSV/Excel exports to that user's prints — useful for universities, makerspaces, or any environment that needs per-person accountability or cost tracking.

| Filter value | Effect |
|---|---|
| **All Users** | Default — global statistics |
| `<specific user>` | Only that user's prints |
| **No User (System)** | Prints without user attribution (slicer-initiated, pre-auth, virtual-printer uploads) |

!!! info "Granting the permission"
    To give the dropdown to non-admins, create a custom group in **Settings → Users** and add `stats:filter_by_user`. See [Authentication](authentication.md).

### Energy "warming-up" indicator

For a date range, the plug figures are computed from hourly snapshots of each smart plug's lifetime counter (see [Smart Plugs](smart-plugs.md)). On a brand-new install, or for a plug added after the range began, a snapshot from before your selected range may not exist yet. The **Energy At The Plug** and **Cost At The Plug** tiles then show a small yellow warning icon with a tooltip explaining the situation. The printing pair is not affected.

The indicator disappears for any range that starts after every plug's first snapshot. While it shows, a plug without a baseline is measured from its earliest snapshot, so the figure leaves out the beginning of the range and may undercount.

---

## :material-cash: Cost configuration

Cost tiles only show numbers when the underlying inputs are configured.

1. **Settings → Printing → Cost Tracking** — set **Currency** (`USD`, `EUR`, `UAH`, etc.) and the electricity price: a fixed **Electricity cost per kWh** (`energy_cost_per_kwh`), or a Home Assistant sensor as the **Electricity price source** — see [Energy → Cost Calculation](energy.md#cost-calculation)
2. **Settings → Filaments / Spoolman** — set per-spool `cost` + `weight` (or a global `default_filament_cost` per kg)
3. Optionally, a **Wear cost per printing hour** on each printer — see [Printer wear](#printer-wear)
4. New prints store their costs on the archive at completion time. A changed electricity price applies to energy used from that moment on; energy used earlier keeps the price of its own time

### Recalculate Costs

Existing archives keep the prices that were active when they completed — historical data isn't auto-rewritten when you bump rates. **Recalculate Costs** rebuilds every print's filament cost from its recorded usage:

1. Click **Recalculate Costs** in the dashboard header
2. Every archive's filament cost is summed again from its usage records. The prices recorded with those records are kept; filament no spool covered — or recorded without a price — is priced at today's default rate
3. The dashboard re-renders against the new totals

To change what past prints cost because a spool's price was wrong, use **Re-price usage** below.

### Re-price usage {#re-price-usage}

A print keeps the cost it was booked with. When a spool's price was entered wrong — 200 per kg typed for 20 — every print from that spool carries the wrong cost, and recalculating does not fix it, because the recorded prices are kept. **Re-price usage** in the dashboard header is the correction for that case:

1. Correct the spool's **Cost per kg** in the [inventory](inventory.md#cost-tracking).
2. Click **Re-price usage**. In the **Re-price recorded usage** dialog choose a **Spool number**, a period (**From** / **To**), or both. Leaving the spool number empty takes every spool in the period; a period needs both its first and its last day.
3. Click **Preview**. Nothing is written yet: the dialog shows how many usage records would change, how many prints would get a new cost, how many orders would move, and the filament cost of the selection before and after.
4. Click **Re-price** to apply. The button stays unavailable until a preview has been made and has something to change.

The recorded usage in the selection is priced again at each spool's current price, and each print's cost moves by exactly the difference — another spool's share of the same print stays as it was recorded. Filament weights and energy are never touched. Usage records whose spool has been deleted keep their cost.

Re-pricing needs the same permission as **Recalculate Costs** (`archives:update_all`) and works with the built-in inventory only; it is not available in Spoolman mode.

!!! info "Reprint cost behaviour"
    Reprints are **additive** — a reprint cost adds to the original archive's total instead of overwriting it, so the per-archive total reflects the cumulative spend across every run of that file. This means stats numbers track real money spent, not "what the original print would cost at today's rate."

---

## :material-refresh: Auto-refresh

The Stats page polls every **60 s** so dashboards left open during a print session stay fresh without a manual reload. The refresh icon in the header forces an immediate refetch — useful right after a long print finishes if you don't want to wait the next tick.

Mutations from elsewhere in the app (deleting an archive, recalculating costs, editing a filament price) invalidate the underlying queries automatically — you don't need to click refresh after them.

---

## :material-api: API

Everything this page reads lives under `/api/v1/statistics/`:

| Endpoint | Returns |
|---|---|
| `GET /statistics/overview` | The KPI block: counts, hours, grams, cost, energy, per-printer accuracy and defects (`date_from`, `date_to`, `created_by_id`) |
| `GET /statistics/aggregate` | The time series, breakdowns and records the charts and the archive calendar fold |
| `GET /statistics/failures` | Failure analysis — see [Failure analysis](failure-analysis.md) |
| `GET /statistics/export` | The failure-analysis summary as CSV, or `?format=xlsx` — see [Export](export.md) |
| `POST /statistics/recalculate-costs` | Rebuild every archive's filament cost from its recorded usage |
| `POST /statistics/reprice-usage` | Re-price the recorded usage of one spool and / or one period at the spools' current prices; a preview unless asked to apply |

These endpoints are not served under `/api/v1/archives/…`: `/archives/stats`, `/archives/aggregate`, `/archives/analysis/failures`, `/archives/stats/export` and `/archives/recalculate-costs` do not exist and are not redirected. `/api/v1/statistics/reports` is reserved for the reports that will be built here.
