---
title: Energy Tracking
description: Per-print and lifetime electricity tracking via smart plugs
---

# Energy Tracking

Track electricity consumed by every print and across the whole lifetime of a printer, and cost it at the electricity price that was in effect while it was used. The numbers come straight from a smart plug's energy register — not from estimates.

---

## :material-information: What It Is

BamDude reads two values from each smart plug:

1. **Live wattage** — how many watts the printer is drawing right now.
2. **Lifetime energy counter** — total kWh the plug has measured since it was reset.

When a print starts, the lifetime counter is captured on the archive row as `energy_start_kwh`. When the print finishes, BamDude reads the counter again and stores the delta as `energy_kwh`. The starting value lives **on the archive row, not in memory** — so a backend restart mid-print does not lose the baseline, and the delta is still computed when the print completes. (If the print *ends* during the outage, see "If the print ends while BamDude is down" below.)

!!! warning "Figures recorded before 0.5.3 understate prints started outside BamDude"
    "At print start" has always been the intent, and for prints BamDude dispatched itself it was also the fact. For a print started from the printer's screen or sent straight from a slicer, the baseline used to be read once the archive row existed — and that row was only created after BamDude had fetched the print's 3MF back off the printer, which on a P1S measured 8m40s. Everything the printer drew in the meantime, bed heating included, was left out. The error was always in the same direction, so it never averaged out. Historical rows are not recalculated; there is nothing to recalculate them from.

For lifetime / date-range views, an hourly background loop snapshots each plug's lifetime counter into `smart_plug_energy_snapshots`. Date-range totals are then computed as `last_snapshot_in_range − last_snapshot_before_range` per plug.

!!! info "Permission Required"
    Reading energy data requires `stats:read`. Tracking starts automatically once a smart plug is bound to a printer — there is no separate "enable energy tracking" toggle.

---

## :material-power-plug: Requirements

Energy tracking needs a smart plug with **kWh metering** wired between the wall outlet and the printer. The plug type, power monitoring details, and configuration steps are covered in [Smart Plugs](smart-plugs.md).

| Plug Type | kWh Metering | Notes |
|-----------|:------------:|-------|
| Tasmota | :material-check: | Native HTTP energy endpoint |
| Home Assistant | :material-check: | Bind a HA energy sensor entity |
| REST / Webhook | :material-check: | Configure JSON path to extract kWh |
| MQTT | :material-check: | Configure MQTT topic + JSON path |

Plugs without an energy register (basic on/off plugs) still work for power control, but their archive rows will have `NULL` `energy_kwh` and won't contribute to lifetime totals.

---

## :material-meter-electric: Per-Print kWh

Captured on `PrintArchive`:

| Column | Captured When | Meaning |
|--------|---------------|---------|
| `energy_start_kwh` | At print start | Plug's lifetime counter at the moment the print began |
| `energy_kwh` | At print complete | `(end_counter − energy_start_kwh)`, the kWh this single print consumed |
| `energy_cost` | At print complete | The print's energy at the electricity price of its own time — see [Cost Calculation](#cost-calculation) (denominated in your configured currency) |

**Restart-resilient.** Because `energy_start_kwh` is persisted to the archive row inside the same transaction that records `started_at`, a backend crash or container restart mid-print does not lose the baseline — the next `on_print_complete` will compute the delta correctly.

**If the print ends while BamDude is down**, there is no `on_print_complete` to run: the printer finishes alone and BamDude only learns about it on the next start. The startup sweep closes that print and reads the counter then, so the archive still gets a figure — but the meter kept counting through however long the printer sat idle in between, so that idle draw is inside the number. Such archives carry `extra_data.energy_is_approximate = true`, and the log line says `(approximate — recovered print)`. The alternative was leaving the field empty, which reads as "this print used no power" — further from the truth than a modest over-count.

**Failed and cancelled prints still record energy.** A 6-hour print that fails at hour 4 still consumed 4 hours of electricity — the delta is still meaningful and is still written.

---

## :material-alert-decagram: When a Plug's Counter Stops

Some plugs keep reporting their watts while their lifetime energy counter stands still. Read naively, every print on such a plug would be **0 kWh** — a frozen counter makes the start and end readings equal — so BamDude compares the counter with the plug's own power history.

**What counts as a suspected stall.** The counter held one value for at least **30 minutes** while the power history of the same plug covers that stretch (at least 95 %) and adds up to at least the plug's **missed-energy threshold** — 100 Wh unless you raise it on the plug. It is a suspicion, not a verdict about the device: a counter that only moves in coarse steps, or a stale cache, looks the same from here. A short or light print whose counter did not step (3 minutes at 120 W is 6 Wh) is not a stall and keeps its measured zero.

**What you see.**

- The plug's card says the counter is suspected stalled, since when, and roughly how much energy the history shows it missed — whether or not anyone subscribed to the warning.
- One **in-app** warning (*Plug energy counter stalled*, group *Sensors*) per episode — not per print or per check — with a link to the smart plugs. External channels (Telegram, email, …) are not offered for it.
- The episode closes when the counter moves up again. A drop is a counter reset: a new baseline, never a recovery. Changing what the plug measures closes it as *source changed*, not as repaired.

**What happens to a print's energy.**

| Archive shows | When | `energy_kwh` |
|---------------|------|--------------|
| `≈ $0.04`, tooltip *estimated from the plug's power history* with the coverage | The print's measured window meets a suspected stall and the power history covers it well | The whole window integrated **once** from the power history — never the counter's part plus an integral on top |
| *Energy unknown* with the reason | The history does not cover the window, the end of the print was not observed (a print closed after a restart), the plug was reconfigured during the print, the server clock went back before the plug's measurement began, or there was too much history to evaluate | empty — **not** zero |
| the figure as before | The counter moved normally | the counter's difference, as always |

Estimated and unknown prints are counted where print energy is shown: the Statistics page says how many prints are estimated from the power history and how many have no reliable figure (and are not counted as zero), and an order's cost line marks its electricity *≈* when it contains estimates. The plug totals, *Energy At The Plug* and date-range totals never contain an estimate — they are what the counters said. Archives recorded before this version are not recalculated. Once BamDude has written a print's energy — measured, estimated or unknown — that figure is final: a later callback reads a later counter (idle draw, the next job) and never replaces it, and a figure you reconstructed yourself is kept as it is.

**Measurement source.** When you change what a plug measures — its address, entity, topic, JSON path, multiplier, Zigbee device, printer, or the phantom-watts setting below — BamDude starts a new measurement source. Readings and counter snapshots from before the change never feed an estimate made after it; a print that spans the change gets *Energy unknown — source changed*. Renaming a plug changes nothing.

**What the power history must look like.** Readings further apart than the plug's gap limit are a gap, never a straight line between them: 3 minutes for Zigbee, three sampling intervals (at least 3 minutes) for Tasmota, REST and Home Assistant, and three publish intervals (at least 3 minutes) for MQTT — set the plug's *publish interval* if it is not Tasmota's default 300 s. A reading BamDude received but cannot trust stays in the history as a hole: a value that is no number, a burst it could not store, a database write whose outcome is unknown, and — on a plug that may keep repeating old watts after its relay opens (Zigbee by default) — any reading while the relay state is unknown. A known *off* is 0 W on every plug. The power chart shows these holes as gaps. An estimate is the area under the readings, not under the load: whatever changed between two readings — short peaks that happened to land on them, or a dip between them — is drawn as a straight line, so neither the 100 Wh threshold nor an estimate is a guarantee of what the printer drew.

**Plug settings** (*Settings → Smart plugs → Edit → Energy counter watch*):

| Setting | Default | Meaning |
|---------|---------|---------|
| Missed energy before a stall is suspected | 100 Wh | Raise it for a plug whose counter moves in coarse steps; it cannot go below 100 Wh |
| How often the plug publishes its power (MQTT only) | 300 s | Sets the MQTT gap limit (three intervals, at least 3 minutes) |
| Watts after the relay opens | the type's default | *May keep repeating old watts* makes readings with an unknown relay state unusable (the Zigbee default); *Reports what it measures* trusts them |

!!! warning "Restarting a hung plug"
    BamDude never sends a command because of a suspected stall. If the device has hung, **wait until prints on it have finished**, then unplug it from the mains and plug it back in — the relay button does not restart its electronics. Never cut a printer's power in the middle of a print.

Detection runs at most every five minutes, and only when a new counter reading has arrived — usually the next hourly snapshot or the end of a print, so it is not a promise to notice within half an hour. It reads the last day of history.

---

## :material-counter: Lifetime kWh and Date Ranges

The hourly snapshot loop (`SmartPlugManager._snapshot_loop`) records one row per plug into `smart_plug_energy_snapshots`. A snapshot is **also** written at the start of every print, from the same reading that seeds the archive's baseline — so a print's own start is a real boundary in the history rather than whichever hour the loop last happened to fire:

| Column | Meaning |
|--------|---------|
| `plug_id` | FK to the smart plug |
| `recorded_at` | UTC timestamp of the snapshot |
| `lifetime_kwh` | Plug's lifetime energy register at that moment |

For a date range `[date_from, date_to]`, BamDude computes per plug:

```
range_total = max(0, last_snapshot_in_range − last_snapshot_before_range)
```

The `max(0, …)` clamps to zero when the lifetime counter has been reset (e.g. after a plug factory-reset) so you never get negative energy.

**Which day is "today".** Dates you pick, and "energy used today", are resolved in **your** timezone -- the browser tells BamDude which one you are in. Snapshots are stored in UTC, as above, and the day boundary is converted before the query. Work BamDude does on its own schedule -- nightly backups, digests -- keeps using the server's timezone, which is what those should follow. Callers that cannot send a timezone (the Telegram bot, API keys, webhooks) are answered in the server's.

---

## :material-cash: Cost Calculation

Energy is costed at the price that was in effect when it was used. BamDude keeps a **history of the electricity price** — every change, with its time — and the price of any moment is the last change at or before it.

**A print.** Its cost is worked out step by step along the plug's counter: the reading at the start of the print, the hourly snapshots taken while it ran, and the reading at its end. Each step is costed at the price in effect when that step began. A print that runs through a price change is therefore costed partly at the old price and partly at the new one, not whole at the price of the moment it ended. A print whose energy is [estimated from the power history](#when-a-plugs-counter-stops) is costed at the time-weighted average price of its measured window.

**The plug totals.** *Cost At The Plug* on the Statistics page — for all time and for a date range — is worked out the same way, along each plug's hourly snapshots. Changing the price does not re-cost the past.

**Before the price history existed.** The history starts on the day BamDude was updated to a version that keeps it. Earlier prices were never recorded, so all energy from before that day is costed at the price that stood on the day of the update. Costs of prints that were already recorded are not touched.

Configure the price in **Settings → Printing → Cost Tracking**:

| Setting | Description |
|---------|-------------|
| **Electricity price source** (`energy_price_source`) | **Fixed price** (default) — you enter the price yourself. **Home Assistant sensor** — BamDude reads it from a sensor, see below. |
| **Electricity cost per kWh** (`energy_cost_per_kwh`) | The price now (default `0.15`). With a fixed price, every change you save is added to the history from that moment on. |

The numeric rate is dimensionless — display the cost in whatever currency matches your real tariff. BamDude doesn't convert currencies. A negative price counts as 0.

### Price from a Home Assistant sensor

For a tariff that changes during the day, BamDude can read the price itself. Home Assistant must be connected first (the Home Assistant card in **Settings → Network**).

1. In **Settings → Printing → Cost Tracking**, set **Electricity price source** to **Home Assistant sensor**.
2. In **Price sensor**, name the entity whose value is the price — for example `sensor.electricity_price`. A `sensor.*`, `input_number.*` or `number.*` entity is accepted.
3. Press **Read the price from this sensor**. BamDude reads the sensor before it saves: if the sensor cannot be read, nothing is saved and the message says why.

From then on:

- **The sensor is read once an hour**, just before the hourly energy snapshots, so each hour of energy is costed at the price of that hour. A price that changes more often than hourly is sampled, not averaged.
- **The unit is converted.** A sensor that reports a price per MWh or per Wh (its unit of measurement ends in `/MWh` or `/Wh`) is converted to a price per kWh. Any other unit is taken as per kWh.
- **A negative price counts as 0.** No print gets a cost below zero.
- **An unavailable sensor keeps the last price.** If Home Assistant does not answer, or the sensor's value is not a number (`unavailable`, `unknown`), the last price that was read stays in effect until the next successful read. The server log says the price was not read.
- **The price field is read-only.** It is labelled **Last price read from Home Assistant (per kWh)** and cannot be typed into while the sensor is the source. Switch the source back to **Fixed price** to enter a price by hand.
- **The push endpoint keeps working.** `POST /api/v1/settings/electricity-price` (below) is still accepted while the sensor is the source; the pushed price takes effect at once and is recorded in the history, and the next hourly read replaces it with the sensor's value.

---

## :material-toggle-switch: Two Energy Figures: Printing and At the Plug

The Statistics page answers "how much energy" twice, as four tiles. There is no setting that chooses between them — both pairs are always shown, for all time or for the selected date range.

| Tiles | What they count |
|-------|-----------------|
| **Energy While Printing** · **Cost While Printing** | Measured between the start and end of each print — what the jobs themselves drew. It is the sum of the per-archive `energy_kwh` and `energy_cost` of the prints in the period, and leaves out everything between prints: idle, standby, heating with no print running. |
| **Energy At The Plug** · **Cost At The Plug** | Everything the smart plugs counted: printing, idling, warming up, and anything else on the same socket. For all time it is read from the plugs' lifetime counters; for a date range it comes from the hourly snapshots — `last_in_range − baseline_before_range` per plug. |

The gap between the two pairs is what standing still costs, and that is why both are shown: one figure alone cannot say whether a high bill comes from printing or from printers left on.

Read the printing pair if you bill customers per job. Read the plug pair if you want to know what your printer farm actually costs to keep running.

---

## :material-clock-alert: "Warming-Up" Indicator

For a date range, the plug figures need **at least one snapshot from before the start of the range** to measure from. On a fresh install, or for a plug that was added after the range began, that baseline doesn't exist yet; BamDude then measures from the plug's earliest snapshot, so the figure leaves out the beginning of the range.

When this happens, the Statistics page shows a yellow warning icon on **Energy At The Plug** and **Cost At The Plug**, with the tooltip:

> :material-alert-outline: *Energy tracking is still collecting hourly snapshots. Date-range totals will become accurate once at least one snapshot exists before the selected range. Early values may undercount.*

The printing pair is not affected — it does not come from snapshots. The icon disappears as soon as enough snapshots exist. No configuration needed; the system is just collecting data. Backend flag: `energy_data_warming_up=True` on the stats response.

---

## :material-home-automation: Tibber / Octopus / Dynamic Tariff Integration

If you're on a dynamic electricity tariff (Tibber, Octopus, Nordpool, …) and the price is a sensor in Home Assistant, the simplest setup is to let BamDude read it: see [Price from a Home Assistant sensor](#price-from-a-home-assistant-sensor) above. It needs no API key and no automation.

The alternative is to push the rate into BamDude from Home Assistant, as described here — useful when the price needs a template, or when you want it to arrive the moment it changes instead of once an hour. Every pushed price is recorded in the price history with its time, so energy used after the push is costed at the new price and energy used before it keeps the old one.

### 1. Create an API key

**Settings → API Keys → Create**, and flip the **Update electricity price** toggle on the key's row. This is a narrowly-scoped opt-in for `POST /settings/electricity-price` — it does **not** grant general settings-write capability (the wider `PATCH /settings` route is unchanged). Copy the key.

!!! note "Older docs referenced `PATCH /settings`"
    The general `PATCH /settings` endpoint still works for API keys, but it exposes the entire settings payload (SMTP / LDAP / MQTT credentials, HA token, every UI rendering knob) — a much wider surface than the dynamic-tariff use case needs. The new `POST /settings/electricity-price` endpoint accepts only `{energy_cost_per_kwh}`, returns the full settings response so HA can confirm the new value applied, and is gated by the per-key `Update electricity price` toggle so admins explicitly opt in. Configs pointing at the legacy URL keep working; switch to the narrow URL on your next config refresh.

### 2. Add a REST command in HA

Add to your `configuration.yaml`:

```yaml
rest_command:
  bamdude_electricity_price:
    url: "http://YOUR_BAMDUDE_IP:8000/api/v1/settings/electricity-price"
    method: POST
    headers:
      X-API-Key: "YOUR_API_KEY"
    content_type: "application/json"
    payload: '{"energy_cost_per_kwh": {{ states("sensor.electricity_price") }}}'
```

### 3. Trigger the REST command on price change

```yaml
automation:
  - id: bamdude_push_electricity_price
    alias: "Update BamDude electricity price"
    mode: restart
    trigger:
      - platform: state
        entity_id: sensor.electricity_price
        for: "00:00:05"
    condition:
      - condition: template
        value_template: >
          {{ states('sensor.electricity_price')|float(none) is not none }}
    action:
      - service: rest_command.bamdude_electricity_price
```

| Provider | Typical sensor entity |
|----------|-----------------------|
| Tibber | `sensor.tibber_prices` (current price attribute) |
| Octopus Energy | `sensor.octopus_energy_electricity_current_rate` |
| Nordpool | `sensor.nordpool_kwh_*` |

!!! tip "Verify the sensor returns a number"
    BamDude expects a numeric value of 0 or more for `energy_cost_per_kwh`; a negative number is refused. If your sensor returns a string with a currency symbol, adjust the template (`{{ states('sensor.x')|float }}`) before pushing. The pushed value is taken as a price per kWh — unlike the sensor source, the push does no unit conversion.

For more on the Home Assistant integration architecture see [Smart Plugs → Home Assistant](smart-plugs.md).

---

## :material-chart-line: Stats-Page Widgets

The Statistics page shows energy as four tiles, for all time or for the selected date range — see [Two Energy Figures](#two-energy-figures-printing-and-at-the-plug):

- **Energy While Printing** and **Cost While Printing** — what the prints themselves drew
- **Energy At The Plug** and **Cost At The Plug** — everything the plugs counted, idle included
- **Per-printer breakdown** so you can see which machine is drawing most

When some prints' energy is estimated from the power history, or unknown, the printing figures say how many — see [When a Plug's Counter Stops](#when-a-plugs-counter-stops).

All four are computed by the server. See [Statistics](stats.md) for the full widget tour.

---

## :material-link: Related

- [Smart Plugs](smart-plugs.md) — plug types, configuration, HA / Tasmota / REST / MQTT setup
- [Archiving](archiving.md) — `energy_kwh` / `energy_cost` fields on each archive row
- [Statistics](stats.md) — energy widget, cost charts, date-range filters
- [Print Queue](print-queue.md) — auto-power-off after print + smart-plug-driven queue automation
- [Export](export.md) — CSV/XLSX with per-print energy + cost columns

---

## :material-lightbulb: Tips

!!! tip "Use your real tariff"
    Pull the all-in rate from your last electricity bill (energy + delivery + taxes + fees). A "headline" rate from your supplier's website usually understates the true cost.

!!! tip "Failed prints still cost"
    `energy_kwh` is recorded for `failed` and `cancelled` archives too — that's another data point arguing for [Failure Analysis](failure-analysis.md) and [Obico](obico.md). Every fail is real money on the meter.

!!! tip "Snapshot baselines need uptime"
    The hourly snapshot loop only runs while BamDude is running. If you stop the container for two days and ask for a 7-day total, the missing 48h shows up as a flat baseline — the warming-up icon will surface this.

!!! tip "Printing figures for invoicing, plug figures for ROI"
    Use **Cost While Printing** when invoicing customers — they shouldn't pay for your standby. Use **Cost At The Plug** when calculating whether the farm is paying for itself.
