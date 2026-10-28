# Task: Member health dashboard

The app you have been given is a skeleton. A member can sign in and see a flat
list of their latest blood results, and the wearable ingest endpoint accepts
deliveries without storing them. Turn it into a consumer product: a member signs
in and wants to know how their health is trending. Build the experience that
answers that from their blood results and wearable data, with routes into the
detail behind each summary.

This needs changes across the full stack: ingesting and modelling provider data,
serving it efficiently, and presenting it so a member can understand it. Many
decisions are left to you. Not everyone finishes. Prioritise, and note what you
left out and why.

## Requirements

- A home page that summarises the member's biomarkers and wearables, with
  drill-downs to the history behind each summary.
- Biomarker history: each marker's dated results, in chronological order, with
  dates and values the member can inspect and its reference ranges displayed
  alongside. Keep the latest value and unit visible. A table alone is not
  sufficient.
- Wearable history for the metrics below, charted with dates, values and units.
  The member chooses the time window, anywhere in their history up to all of it,
  and the granularity: at least day, week and month. Default to the latest 30
  calendar days, ending on the member's latest wearable date.
- Persistence in `POST /api/users/{user_id}/wearables` for every delivery shape
  below, modelled with the existing store API.
- Wearable data that arrives while the page is open appears without a reload.
- Summary text where it helps the member understand their data. Every figure it
  states must match the stored data.
- A layout that works on mobile and desktop.
- All seven members supported, each seeing only their own data.

| Metric | Daily value |
| --- | --- |
| Resting heart rate | Beats per minute |
| Steps | Total steps |
| Sleep efficiency | Percentage |

Choose the layout, charts, document structure, partition and sort keys,
endpoints and update mechanism. We look at how clearly the page shows what
matters, whether the numbers are right, how it performs, and how you reasoned
about the trade-offs.

## Biomarker data

Bootstrap loads blood draws into `results:{user_id}`, keyed by draw timestamp.
JSON copies are in [data/biomarkers/](data/biomarkers/), and the models are in
[app/models.py](app/models.py). Biomarker history does not require new ingest
code.

Draws cover different panels, so a marker's history can be ragged or a single
point. Each result records the status and ranges known at the time of its draw,
and some reference ranges were revised over time, so a marker's history can
carry different ranges from one point to the next. Decide how to display values
and ranges, and document what you assumed.

The client fetches `GET /api/home`. You can extend that response or add
endpoints. [Chart.js](https://www.chartjs.org/) is available in the client; the
performance monitor in [app/static/perf.js](app/static/perf.js) includes an
example.

## Wearable data

Wearable fixtures are in `data/wearables/{user_id}/`, one JSON file per delivery.
Each member uses one provider, and providers differ in field names, units and
which metrics they supply. Each request carries the provider, the member and a
`data` field in one of three shapes:

| Delivery | Fixture name | `data` |
| --- | --- | --- |
| Backfill | `{first}_{last}.json` | A list of days, with hourly samples |
| Daily | `{date}.json` | One day, with minute samples |
| Delta | `{date}T{hour}.json` | Part of one day, with the hour it covers in `window` |

A backfill arrives when a member connects a device, daily uploads follow, and
deltas stream in as the current day progresses. Histories range from 41 days to
five years, and the largest backfill file is about 7 MB.

Deliveries are partial in places. A member can have days with no delivery, a
delivery can carry a null or absent field, and a provider may not expose every
metric above. A delta carries only the samples and summaries available for its
hour, so a day assembles over several requests, and the latest day may still be
incomplete. Some days are delivered twice; the later file has an `_r2` suffix,
carries a corrected value and has a distinct `upload_id`.

Decide whether a repeated delivery for the same day replaces, merges with or is
rejected in favour of the earlier one, how deltas combine with what is already
stored, and how missing values and incomplete days appear on the charts.

## Import the deliveries

[app/routes/ingest.py](app/routes/ingest.py) contains the endpoint shell. It
accepts a single-day envelope, rejects backfill lists, and returns
`stored: false` without writing data.

With `scripts/dev` running, replay a member's files:

```sh
scripts/ingest-wearable-data.py --user user-1
```

Repeat for `user-2` to `user-7`. The script posts files in filename order:
backfill first, then daily files, then deltas. It stops at the first rejected
request.

Ingest is unauthenticated for this exercise. Leave authentication unchanged.
Bootstrap clears imported data, so run it before importing wearables.

## Performance and verification

Keep `GET /api/home` within the 35 ms p95 budget after importing all members:

```sh
scripts/bench.py --email marcus.hale@example.com
```

Measure the other members too, including `elena.ruiz@example.com`, who also has
five years of history. Measure any other endpoints the page depends on with
`--path`, including a five-year window, and record the results. The benchmark
requires a running server and exits with a non-zero status when it exceeds the
budget.

Follow the [README constraints](README.md#constraints). Add tests for ingest,
the histories returned to the client and any generated summary text.
[tests/test_wearables.py](tests/test_wearables.py) has fixture loading and
sign-in helpers to start from; its expected failure clears once ingest persists.

## Notes

Include a `NOTES.md` at the repository root covering:

- Assumptions you made where the brief was ambiguous, including how you display
  values and reference ranges.
- Storage layout, what it makes cheap or expensive to read, and how repeated
  deliveries, deltas and corrections combine.
- How weekly and monthly values aggregate and how missing or partial periods
  appear.
- How live updates reach the page, and how you tested them.
- Measurements, and what you would change at more members or longer histories.
- What you left out, what you would do next, and how you used AI tools.
