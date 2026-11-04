# Spoke — Decisions and limits

The client's brief: *"If something in this brief is ambiguous, decide, and write down what
you decided and why."* This is that record. Dates are when each decision was taken.

## Station register

| Topic | Decision | Why |
|---|---|---|
| Station recorded twice | One row kept per `station_id`, the most recently delivered | The duplicate comes from the merge of two asset systems, as the client announced |
| Missing dock capacity | Stays `NULL` in every layer | Zero would be false information, and capacity is not part of the gold contract |
| Missing values in general | A missing number stays `NULL`, never a sentinel value | Silver describes the data; presentation choices belong to the consumer |
| Hand-typed district names | Whitespace normalised, then title case | Marketing quotes districts by name: each one needs a single, stable spelling |
| Station names | Whitespace normalised, case left as delivered | Title-casing a proper name can damage it |
| Latitude and longitude | `DOUBLE` | `FLOAT` loses precision; `DECIMAL` is meant for money |
| Cleaning scope | Only defects actually observed are fixed | Generic cleaning invents values |
| Rows that cannot be typed | Sent to `silver.station_quarantine` with their original values, an array of reasons and a timestamp | A rejected row must stay explainable and replayable |
| Load strategy | Full refresh: bronze, silver and quarantine are all overwritten on each run (4 Nov 2026) | The register is a complete snapshot delivered once. An incremental load would add complexity for nothing, and appending the quarantine would repeat the same rejects on every run |

## Ride feed

| Topic | Decision | Why |
|---|---|---|
| Corrupt records | Quarantined first, before deduplication | Everything downstream only sees readable rows |
| Rescued data (`_rescued_data`) | Kept in bronze, not carried to silver. Each bronze notebook raises an error if the current run produced any rescued value (4 Nov 2026) | It signals that the source changed shape. A human decides whether the schema must evolve, then silver is rebuilt from bronze, which lost nothing. A print would go unnoticed in a scheduled run |
| Identical copies | One copy kept, the most recently delivered | These are the app's upload retries, as the client announced |
| Same `ride_id`, different content | Every version quarantined (`duplicate_ride_with_different_data`) | Nothing says which version is right |
| `device` struct | Flattened into `device_app_version` and `device_platform` | Easier to read and filter as columns |
| Ride duration | Recomputed from `started_at` and `ended_at` | The client warned the phone's figure is unreliable |
| Quality flags | Computed in silver: copies received, ride never closed, unreliable duration (gap of **more than** 60 seconds) | Gold only has to count them |
| Silver write mode | `overwrite`, for the table and its quarantine (28 Oct 2026) | Silver is recomputed from all of bronze each run; appending doubled every ride on the second run |

## Gold

| Topic | Decision | Why |
|---|---|---|
| `missing_end_station` | A ride never closed when `end_station_id` **or** `ended_at` is missing (27 Oct 2026) | Either one means we do not know where or when the ride ended |
| `duplicate_ride` | Number of **distinct** rides that arrived more than once, not the number of extra copies | Reading of *"records that arrived more than once"* |
| `unknown_station` | Computed in gold against the register. A ride counts once if its start **or** its end station is unknown. A missing end station is not an unknown one (27 Oct 2026) | The register can change, so the check runs as late as possible. The contract counts rides, not references |
| `unreliable_duration` | Only rides with both a start and an end time can be judged | A ride that never closed has no recorded duration to compare |
| Quarantined rows | Not in `data_quality_report` | None of the four issue categories covers them; they are counted in the quarantine tables |
| Maintenance trips | Excluded from both usage tables, kept in the quality report | They are not customer demand, but their data can still be faulty |
| `daily_station_usage` rows | One row per station and day with at least one departure or arrival, only for registered stations (28 Oct 2026) | The client's tools read activity, not an empty calendar |
| Day of an arrival | The day of `ended_at`; a departure counts on the day of `started_at` (28 Oct 2026) | A bike that leaves at 23:50 is at its end station the next morning |
| `hourly_demand` population | Every customer ride, including those from the two unregistered stations; hours with no ride have no row (28 Oct 2026) | The client says those rides are *"perfectly real"* |
| `avg_ride_minutes` | Average of the recomputed duration, no rounding (28 Oct 2026) | Rounding to the minute would erase the morning/evening difference the client wants to see; rounding belongs to the BI tool |
| Gold write mode | Delta tables, fully overwritten on each run, rather than materialized views (28 Oct 2026) | Each table is recomputed from all of history; rerunning gives the same result |

## Known limits

- **Silver rides are not incremental.** They re-read all of bronze on each run and are
  overwritten, quarantine included. Correct and repeatable, but the cost grows with
  history. (The station register is fully reloaded by design: see above.) In production: stream from bronze, then
  `MERGE` on `ride_id` inside `foreachBatch`, handling copies and conflicts that arrive on
  different days and accumulating the copy count.
- **Gold is fully recomputed.** Beyond a few months of data, only affected days would be
  rewritten (`MERGE` or `replaceWhere`).
- **No automatic reconciliation.** Consistency was checked by the acceptance tests and by
  hand, but a row lost in a future run would not raise an error. The check to add:
  bronze rows = silver rows + removed copies + quarantined rows.
- **No `missing_ride_id` rule.** The delivered files contain no ride without an id. If
  one arrives, rides without an id would be grouped under a single null key and mistaken
  for a version conflict.
- **Manual run.** Notebooks are run in order by hand; there is no scheduler yet.
