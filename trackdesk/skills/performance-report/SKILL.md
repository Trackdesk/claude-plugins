---
name: performance-report
description: Build a performance report for a Trackdesk affiliate program — clicks, conversions, revenue, affiliate payouts, profit and fraud rates for a period, broken down by day, week, offer, affiliate, country, advertiser, tier or sub-ID, optionally compared with the previous period. Use when the user asks how the program, an offer or an affiliate is doing, for top or worst affiliates or offers, a trend, or a weekly or monthly summary.
argument-hint: '[period] [breakdown], e.g. "last month by offer"'
allowed-tools: mcp__plugin_trackdesk_trackdesk__search
---

# Performance report

Load the `trackdesk-api` skill first unless it is already loaded in this conversation; its
conventions (formats, time zones, presentation) apply here.

Request: $ARGUMENTS

## 1. Pin down the question

- **Period** — resolve it to calendar days and a time zone. With no period given, use the
  last 30 days and say so.
- **Breakdown** — one dimension per report:

  | Question about         | `groupingLevels`                                        | Row key                           |
  | ---------------------- | ------------------------------------------------------- | --------------------------------- |
  | trend                  | `REPORT_GROUPING_DAY`, `_WEEK`, `_MONTH`                | `day`, `week`, `month`            |
  | offers                 | `REPORT_GROUPING_OFFER`                                 | `offer` `{ id, name }`            |
  | affiliates             | `REPORT_GROUPING_ACCOUNT`                               | `account` `{ id, name }`          |
  | countries              | `REPORT_GROUPING_COUNTRY_CODE`                          | `countryCode`                     |
  | advertisers            | `REPORT_GROUPING_REVENUE_ORIGIN`                        | `revenueOrigin`                   |
  | tiers, managers        | `REPORT_GROUPING_TIER`, `_AFFILIATE_MANAGER_ACCOUNT`    | `tier`, `affiliateManagerAccount` |
  | landing pages, coupons | `REPORT_GROUPING_LANDING_PAGE`, `_COUPON`               | `landingPage`, `coupon`           |
  | sub-IDs                | `REPORT_GROUPING_AFF_S1`…`_AFF_S5`, `_ADV_S1`…`_ADV_S5` | `affS1`…, `advS1`…                |

  `_WEEK` is short for `REPORT_GROUPING_WEEK`, and so on; the search tool lists every
  grouping.

- **Metric** — rank by the metric the user named. When "best" or "worst" leaves it open,
  ask which one (revenue, conversions, profit…) instead of guessing.
- **Filters** — resolve names to IDs first: offers with `POST /api/node/offers/v1/list`,
  affiliates with `POST /api/node/affiliates/v1` (`filters.name` or `filters.emails`),
  advertisers with `POST /api/node/revenue-origins/v1/list`, tags with
  `GET /api/node/tags/v1`, tiers with `GET /api/node/tiers/v1`. Report filters take
  `offerIds`, `accountIds`, `sourceIds`, `revenueOriginIds`, `countryCodes`, `tierIds`,
  `affiliateTagIds`, `offerTagIds` and more.
- **Statuses** — count `CONVERSION_STATUS_APPROVED`, `_PENDING` and `_ON_HOLD` unless the user
  asks otherwise ("approved only" means `[CONVERSION_STATUS_APPROVED]`), and name the
  statuses counted in the answer.

## 2. Fetch, compare and condense in one call

`POST /api/reports/general-overview-report/v1` covers every breakdown; the separate
affiliate, offer and country overview reports are deprecated. Each row has its key field
plus `values`; `total.values` holds the totals. For a comparison, run the same request for
the previous period of equal length and join the full row sets by key. Comparing two top-N
lists would lose whichever entity moved most.

```js
async () => {
	const timezone = 'Europe/Prague'
	const grouping = 'REPORT_GROUPING_OFFER'
	const keyOf = (row) => row.offer
	const statuses = [
		'CONVERSION_STATUS_APPROVED',
		'CONVERSION_STATUS_PENDING',
		'CONVERSION_STATUS_ON_HOLD',
	]
	const periods = {
		current: { from: '2026-09-01T00:00:00+02:00', to: '2026-09-30T23:59:59+02:00' },
		previous: { from: '2026-08-01T00:00:00+02:00', to: '2026-08-31T23:59:59+02:00' },
	}

	const num = (x) => Number((typeof x === 'object' ? x?.value : x) ?? 0)
	const metrics = (v) => ({
		clicks: num(v.clickCount),
		conversions: num(v.conversionCount),
		revenue: num(v.totalRevenue),
		payout: num(v.totalCost),
		mlmCommissions: num(v.totalCommissionAmount),
		profit: num(v.profit),
		sales: num(v.totalConversionAmount),
		clickFraudRate: num(v.clickFraudRate),
		conversionFraudRate: num(v.conversionFraudRate),
	})

	const fetchPeriod = async (timeRange) => {
		const res = await codemode.request({
			method: 'POST',
			path: '/api/reports/general-overview-report/v1',
			body: {
				timezone,
				groupingLevels: [grouping],
				filters: { timeRange, conversionStatuses: statuses },
			},
		})
		if (!res?.rows) throw new Error(JSON.stringify(res))
		return res
	}

	const [current, previous] = await Promise.all([
		fetchPeriod(periods.current),
		fetchPeriod(periods.previous),
	])
	const id = (row) => JSON.stringify(keyOf(row))
	const before = new Map(previous.rows.map((r) => [id(r), metrics(r.values)]))
	const currentIds = new Set(current.rows.map(id))
	const rows = current.rows
		.map((r) => ({ key: keyOf(r), now: metrics(r.values), before: before.get(id(r)) ?? null }))
		.sort((a, b) => b.now.revenue - a.now.revenue)

	return {
		total: { now: metrics(current.total.values), before: metrics(previous.total.values) },
		rowCount: rows.length,
		top: rows.slice(0, 15),
		biggestDrops: rows
			.filter((r) => r.before && r.now.revenue < r.before.revenue)
			.sort((a, b) => a.now.revenue - a.before.revenue - (b.now.revenue - b.before.revenue))
			.slice(0, 5),
		goneSincePrevious: previous.rows
			.filter((r) => !currentIds.has(id(r)))
			.slice(0, 5)
			.map((r) => ({ key: keyOf(r), before: metrics(r.values) })),
	}
}
```

Adapt `grouping`, `keyOf`, the sort metric and the filters to the question; drop the
comparison when the user did not ask for one and it adds nothing.

For a trend, a `REPORT_GROUPING_DAY` row's `day` is the day's start in UTC; show it as a date
in the report's time zone.

## 3. Present

1. The headline: totals for the period with the change against the previous period (absolute
   and %), the statuses counted and the time zone.
2. A compact table of the leading rows by the chosen metric: name, clicks, conversions,
   conversion rate, revenue, payout, profit.
3. Two to four observations backed by the numbers: the largest movers, concentration (for
   example the top affiliate's share of revenue), rows with an unusual conversion rate or
   fraud rate, payout above revenue. Mark them as observations, not causes.
4. Offer the next step: a breakdown of the period's biggest mover, a different dimension, or
   the individual conversions behind a figure.

## Going deeper

- Individual conversions: `POST /api/reports/conversion-report/v1` with
  `pagination: { limit: 500, offset }` and `filters` (`timeRange`, `offerIds`, `sourceIds`,
  `conversionStatuses`, `externalIds`, `cids`…). Rows carry the affiliate's `source`
  (`id`, `publicId`), offer, status, amounts with `currencyCode`, `externalId` and
  `fraudSuspicion`.
- Individual clicks: `POST /api/reports/click-report/v1`.
- Daily totals with no breakdown: `POST /api/reports/daily-overview-report/v1` takes `from`
  and `to` as `YYYY-MM-DD` plus `timezone`.
