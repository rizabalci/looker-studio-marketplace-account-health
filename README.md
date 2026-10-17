# Marketplace Account Health (Looker Studio template)

A Looker Studio report that tracks seller-fulfilled account health across several Amazon EU marketplaces plus Otto and bol, one page per marketplace, all fed from a single Google Sheet.

## What it shows and why

Amazon, Otto and bol each judge your delivery performance on their own portal, with their own metrics and their own thresholds. Checking five or seven portals by hand is slow, and the history disappears because the portals only show a rolling window. This report keeps every reading you take, shows the latest one as scorecards, and draws the trend so you can see a metric drifting toward a breach before the marketplace warns you.

Each Amazon page answers three questions:

1. Where do we stand right now? (scorecards from the latest reading)
2. How much room is left before the on-time target breaks? (headroom in units, not percent)
3. Which way is it moving? (trend charts and the full reading history)

The Otto and bol pages do the same for those marketplaces' own scorecards.

## Data source

One Google Sheet (`YOUR_SHEET_ID`) with these tabs. Each tab is connected to Looker Studio as its own data source.

| Tab | Written by | Purpose |
|---|---|---|
| `amazon_raw` | You or your ingest script | One row per marketplace per reading. Append only. |
| `looker` | Formulas only | Cleans `amazon_raw`, adds headroom and the latest-reading flag. Never type into it. |
| `amazon_orders` | You or your ingest script | Optional order-level detail for one marketplace you are watching closely. |
| `otto` | You or your ingest script | One row per Otto reading. |
| `bol` | You or your ingest script | One row per bol week reading. |

### `amazon_raw` / `looker` columns

`sample_data.csv` in this folder matches this schema.

| Column | Type | Meaning |
|---|---|---|
| read_at | Date and time (text in raw tab, converted in `looker`) | When the reading was taken. Format `YYYY-MM-DD HH:MM`. |
| marketplace | Text | Two-letter marketplace code: DE, FR, UK, IT, ES. |
| otd_num | Number | Orders delivered on time in the window. |
| otd_den | Number | Orders with a delivery result in the window. |
| otd_pct | Percent | On-time delivery rate as shown by Amazon. |
| ldr_10d_pct | Percent | Late dispatch rate, 10-day window. |
| ldr_30d_pct | Percent | Late dispatch rate, 30-day window. |
| vtr_pct | Percent | Valid tracking rate. |
| cancel_pct | Percent | Seller cancellation rate. |
| bhdr_pct | Percent | Buyer handling defect rate (or the nearest equivalent shown). |
| status | Text | Exactly `OK`, `Tight` or `Breach`. Drives the row colours. |
| note | Text | Free text: what changed, what you did. |
| otd_headroom | Number (`looker` tab only) | Late orders you can still absorb before the on-time rate drops under target. |
| is_latest | Number 0/1 (`looker` tab only) | 1 on the newest row per marketplace. |

Formulas in the `looker` tab (row 2, filled down):

```
A2:  =ARRAYFORMULA(IF(amazon_raw!A2:A="",,DATEVALUE(LEFT(amazon_raw!A2:A,10))+TIMEVALUE(MID(amazon_raw!A2:A,12,5))))
B2:  =ARRAYFORMULA(amazon_raw!B2:L)
M2:  =IF(C2="",,INT(C2 - D2*0.9))
N2:  =IF(B2="",,IF(COUNTIFS(B:B,B2,A:A,">"&A2)=0,1,0))
```

Change the `0.9` in the headroom formula if your on-time target is different. A negative headroom means you are already under target by that many orders.

### `amazon_orders` columns

| Column | Type | Meaning |
|---|---|---|
| metric | Text | Which metric the order hurt (OTD, LDR, VTR). |
| order_id | Text | Marketplace order ID. |
| order_day | Date | Order date. |
| due_utc | Date and time | Promised delivery or dispatch time. |
| confirmed_utc | Date and time | When shipment was confirmed. |
| delivered | Date and time | Carrier delivery time, blank if not delivered. |
| carrier | Text | DHL, DPD, UPS, and so on. |
| cause | Text | Your root-cause label. |
| owner | Text | Team or node responsible (never an individual). |
| read_at | Date and time | When the row was read. |

### `otto` columns

| Column | Type | Meaning |
|---|---|---|
| read_at | Date and time | When the reading was taken. This is the true Otto data date. |
| period | Text | Label as Otto shows it, for example `Month to date` or `Week 12`. |
| period_type | Text | month, week, rolling7. |
| period_start | Date | First day of the period. Used for sorting. |
| ontime_pct | Percent | On-time delivery rate. |
| early_pct | Percent | Early delivery share. |
| late_return_pct | Percent | Late return handling rate. |
| cancel_pct | Percent | Cancellation rate. |
| deliveries | Number | Deliveries in the period. |
| late_count | Number | Late deliveries in the period. |
| warehouse_processing_pct | Percent | Share of late deliveries caused by warehouse processing. |
| carrier_transport_pct | Percent | Share caused by carrier transport. |
| violations | Number | Open violations. |
| status | Text | OK, Tight or Breach. |
| note | Text | Free text. |

### `bol` columns

| Column | Type | Meaning |
|---|---|---|
| read_at | Date and time | When the reading was taken. |
| week | Text | Week label, for example `W12`. |
| week_start | Date | Monday of that week. |
| items_ontime | Number | Items delivered on time. |
| items_total | Number | Items delivered. |
| ontime_pct | Percent | items_ontime / items_total. |
| cancel_num | Number | Cancelled items. |
| cancel_den | Number | Items ordered. |
| cancel_pct | Percent | Cancellation rate. |
| response_pct | Percent | Customer response rate within the deadline. |
| tt_pct | Percent | Valid track and trace rate. |
| final | Text | yes once the week is closed, no for partial weeks. |
| strikes | Number | Strikes shown on the scorecard. |
| quality_score | Number | Quality score shown by bol. |
| status | Text | OK, Tight or Breach. |
| note | Text | Free text. |

## Calculated fields

| Field | Data source | Formula | Type |
|---|---|---|---|
| Period sort | otto | `MIN(DATE_DIFF(period_start, DATE(2020,1,1)))` | Number |
| Read at (dd.mm.yyyy) | any tab where read_at is text | `CONCAT(SUBSTR(read_at,9,2),".",SUBSTR(read_at,6,2),".",SUBSTR(read_at,1,4)," ",SUBSTR(read_at,12,5))` | Text |
| OTD target | looker | `0.9` | Percent |

The Period sort field exists because Otto period labels do not sort in time order as text.

## Page layout

Canvas 1200 wide. Every page starts with a one-line intro text box: data source, colour key and the targets. Do not write a fixed cadence like "read every Tuesday" into the text; if the cadence changes the text goes stale. Say "cards show the latest reading" instead.

### Amazon pages (one per marketplace: DE, FR, UK, IT, ES)

Page filter: `marketplace = XX` (create one filter per market, named "Market XX").

**Scorecard row** (chart filter "Latest reading": `is_latest = 1`):

| Scorecard | Metric | Aggregation |
|---|---|---|
| On-time delivery % | otd_pct | Max |
| OTD headroom (units) | otd_headroom | Max |
| Late dispatch 10d % | ldr_10d_pct | Max |
| Late dispatch 30d % | ldr_30d_pct | Max |
| Valid tracking % | vtr_pct | Max |
| Cancel rate % | cancel_pct | Max |
| BHDR % | bhdr_pct | Max |

Keep scorecard labels short ("BHDR %"); long labels truncate.

**Trend charts** (two time series, side by side):

- Chart 1: dimension read_at (Date), metrics otd_pct and vtr_pct, average, linear interpolation for missing days. Reference lines at the OTD and VTR targets.
- Chart 2: dimension read_at (Date), metrics ldr_10d_pct and ldr_30d_pct, average, linear interpolation. Reference line at the LDR limit.
- Axis format `dd.mm.`.

**History table**: all columns from the `looker` tab, sorted read_at descending. Read at shown as dd.mm.yyyy hh:mm.

**Order detail table** (only on the page you watch closely): the `amazon_orders` source, sorted read_at descending, wrap text on. Raise the page height (Page settings, height 1400) so the table does not run off the bottom.

### Otto page

- Intro text: say that the source strip (if you have one) shows the newest read of all sources, and that the Read at column is the true Otto data date.
- Column chart: dimension period, metric ontime_pct (Average), sorted by Period sort ascending, reference line at your Otto on-time target.
- Table: all `otto` columns, sorted read_at descending.

### bol page

- Column chart: dimension week, metric ontime_pct (Min), sorted ascending, up to 30 bars, reference line at your bol on-time target.
- Weekly table: week, items_ontime, items_total, ontime_pct, cancel_pct, tt_pct, final, strikes, quality_score, status, note.
- Page height 900.

### Formatting on every history table

- Conditional formatting on `status`, apply to entire row: `Breach` = light red, `Tight` = light yellow. `OK` rows stay plain. The values must be spelled exactly Breach, Tight, OK.
- Header and body text wrap on. Widen `note`, narrow `read_at` and `status`.
- Style tab, Missing data: show `-` instead of `null`.
- Dates shown as dd.mm.yyyy (or dd.mm.yyyy hh:mm for read_at).

## Controls

- Date range control on each page (default: last 90 days), applied to read_at.
- No marketplace drop-down on the Amazon pages; the page filter already fixes it.
- Optional: a status filter control (OK, Tight, Breach) above the history table.

## Targets

Use the public thresholds from each marketplace's seller help pages and put them in the intro text and reference lines. For Amazon seller-fulfilled accounts the commonly published targets are on-time delivery at least 90%, late dispatch under 4%, valid tracking at least 95% and cancellation under 2.5%. Check the current values for your marketplaces before you rely on them; Amazon changes them.

## Build steps

1. Copy the Sheet structure: create the five tabs and paste the headers above. Paste `sample_data.csv` into `amazon_raw` to start.
2. Add the `looker` tab formulas. Check that `is_latest` shows 1 once per marketplace.
3. In Looker Studio, create a blank report. Theme of your choice, canvas 1200 wide.
4. Add data sources: Google Sheets connector, Sheet `YOUR_SHEET_ID`, one source each for `looker`, `amazon_orders`, `otto`, `bol`. Tick "Use first row as headers". Set data freshness to 15 minutes.
5. In each source, set types: read_at as Date and Hour, percent columns as Percent, counts as Number. If a percent column was stored as 4 for 4%, divide in the Sheet rather than relying on a display format; Looker's Percent display on a whole number shows 400%.
6. Add the calculated fields listed above.
7. Build the DE page: intro text, the "Market DE" page filter, the scorecard row with the "Latest reading" filter, two trend charts, history table, conditional formatting.
8. Duplicate the page for FR, UK, IT, ES and change only the page filter and the intro text.
9. Build the Otto and bol pages.
10. Set Missing data to `-` on every table, check page heights so nothing runs past the bottom edge, then open in View mode and check every page.

## Tips

- Field names containing the word "date" can confuse the calculated-field parser. Name columns like `order_day` or `week_start` instead.
- Primary and secondary sort on the same field gives a chart error. Use a different secondary sort.
- Turn cross-filtering off on tables if clicking a row should not filter the rest of the page.

## Design rules

Never estimate or backfill a missing reading; a gap stays a gap and shows as `-`. Status is set from the marketplace's own figures, not from a forecast. Scorecards stay at account and marketplace level, never per person.
