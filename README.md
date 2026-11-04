# Spoke — Seeing where a bike-sharing fleet drifts

> **A bike-sharing operator was sending its rebalancing vans out every morning on instinct, because nobody could say where bikes actually went.**
>
> I built a Databricks pipeline that turns its station register and nightly ride files into three tables the BI team can trust: daily flows per station, hourly demand, and a plain count of what could not be used.
>
> `Databricks` · `PySpark` · `Delta Lake` · `Auto Loader` · `Unity Catalog`

*Realistic client brief, fictional company.*

---

## The client

Spoke runs a docked bike-sharing network: sixty stations, around nine hundred bikes, and
a mobile app riders use to unlock and return them. Every morning, vans move bikes from
stations that filled up overnight to stations that ran dry.

## The problem

Those van routes were decided by two experienced people working from instinct. The
company wanted to stop relying on them, but before automating anything it needed to
*see* what happens on the network, every day, in numbers it could defend in a meeting.

## What I was given

| Source | Shape | Arrives | Known issues |
|---|---|---|---|
| Station register | CSV, 61 rows, one per station | Once | A station recorded twice after two asset systems were merged; missing dock capacities; district names typed by hand, inconsistently |
| Ride feed | JSON Lines, 4,809 records over 7 days | One file per night | Rides re-sent by the app after a dropped connection; rides that never closed; a duration computed on the phone; two new stations missing from the register; maintenance staff using the same app |

## What I delivered

Three Delta tables in `spoke.gold`, the only objects the client's BI team reads.

| Output | Answers |
|---|---|
| `daily_station_usage` | *Where does the fleet drift?* Rides started and ended per station per day, and the net flow |
| `hourly_demand` | *When does demand happen?* Rides started per hour of the day, and their average duration |
| `data_quality_report` | *How much of this can we not use?* One row per kind of problem, with the number of records affected |

## How it is built

```
stations.csv ──batch──►  bronze.station  ──►  silver.station  ─────────────┐
                                              silver.station_quarantine    │
                                                                           ├──►  gold (3 tables)
rides/*.jsonl ──Auto Loader──►  bronze.rides  ──►  silver.rides  ──────────┘
                                                   silver.rides_quarantine
```

- **Bronze** keeps the files as delivered: every column as text, an explicit schema for
  the JSON, corrupt and unexpected fields captured instead of lost, and lineage columns
  (source file, ingestion time, batch id). The station register is a complete snapshot
  delivered once, so it is fully reloaded on each run. The ride feed is ingested
  incrementally with Auto Loader, so a new nightly file is the only thing read on the
  next run.
- **Silver** is where the data becomes trustworthy: duplicates removed, values typed, and
  every row that cannot be typed sent to a quarantine table with the reasons, next to its
  original values. Quality flags are computed here, once.
- **Gold** only aggregates. It applies the business rules — maintenance trips are not
  demand — and counts the flags.

Notebooks, in run order: [`notebooks/`](notebooks/).

## Decisions the brief left open

The brief asked for assumptions to be written down rather than asked about. The main
ones are below; the full list, with the limits, is in [DECISIONS.md](DECISIONS.md).

| Question | Decision | Why |
|---|---|---|
| A ride that never closed: what counts? | Missing end station **or** missing end time | Either one means we do not know where or when the bike was returned |
| Which duration to trust? | Recomputed from start and end times; the phone's figure is only compared against it | The client warned that the phone's figure can be wildly wrong |
| Same ride id, different content | Every version goes to quarantine | Nothing says which version is right; picking one would hide the problem |
| Missing dock capacity | Left empty, never replaced by 0 | Zero is a false statement about a real dock |
| Rides at the two unregistered stations | Counted in hourly demand and in the quality report, absent from the per-station table | They are real demand, but there is no station row to attach them to |
| A ride that leaves at 23:50 and arrives after midnight | The arrival counts on the day the bike arrives | That is where the bike is the next morning; the table spans one day more than the feed |

## Data quality, in numbers

| Issue | Records affected |
|---|---|
| `duplicate_ride` — rides received more than once | 55 |
| `missing_end_station` — rides that never closed | 78 |
| `unknown_station` — rides referencing a station absent from the register | 9 |
| `unreliable_duration` — phone duration off by more than a minute | 205 |

The output passes all 36 acceptance checks of the client's analyst, including after a
second run on the same files.

## What resisted

- **The second run doubled everything.** The silver rides table was rebuilt from the
  whole bronze history, then *appended*. The first run was correct; the second stacked a
  full copy on top. Fix: a step that recomputes everything must overwrite, and only a
  step that reads new data alone may append. Testing a rerun is now part of how I check
  a pipeline.
- **A `dropna` that dropped real arrivals.** After building a station × day grid, empty
  rows were removed with `dropna`, whose default removes a row when *any* listed column
  is null. On the last day, stations had arrivals but no departures, and those rows
  vanished. `how="all"` removes only rows where both are empty.
- **Joining on a missing station.** An anti-join to find unknown end stations also
  caught every ride with *no* end station, because a null never matches anything. Rides
  that never closed were being reported as unknown stations until they were filtered
  out first.

## Limits and next steps

- **Silver is recomputed in full, and overwritten.** Every silver table and its
  quarantine re-read all of bronze on each run, so they are written in `overwrite` mode:
  correct and repeatable. For the station register this is the right design, not a
  shortcut — it is a complete snapshot of 61 rows. For rides it is a limit: the cost
  grows with every nightly file. Next step: stream from bronze and `MERGE` on `ride_id`,
  handling copies and conflicts that arrive on different days and accumulating the copy
  count.
- **Gold is fully recomputed.** Beyond a few months, only the affected days would be
  updated.
- **No automatic reconciliation.** The pipeline does not yet raise an error if
  bronze ≠ silver + removed copies + quarantine.
- **Notebooks are run by hand.** Orchestration is the subject of a later project.

## Stack

| | |
|---|---|
| Platform | Databricks Free Edition, serverless compute |
| Languages | Python (PySpark) |
| Storage | Delta Lake, Unity Catalog (catalog `spoke`, schemas `bronze` / `silver` / `gold`) |
| Ingestion | Batch read for the CSV register, Auto Loader (`availableNow`) for the JSON ride feed |
| Orchestration | Manual, notebooks run in order |

---

*Completed November 2026 · 18 hours · [Source](https://github.com/AntoineCFR/de-portfolio-01-spoke)*
