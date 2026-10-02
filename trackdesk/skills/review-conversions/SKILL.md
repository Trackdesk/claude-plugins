---
name: review-conversions
description: Review Trackdesk conversions waiting for a decision (pending, on hold or in quality control), spot suspicious ones — fraud flags, duplicate orders or customers, outlier amounts, payout above revenue, bursts from one affiliate — and approve, reject or refund them after the user confirms, including bulk updates from a list of order IDs such as a shop's refunds export. Use when the user asks to review, approve, reject, refund or clean up conversions, or asks what happened to an order, click or customer.
allowed-tools: mcp__plugin_trackdesk_trackdesk__search
---

# Review conversions

Load the `trackdesk-api` skill first unless it is already loaded in this conversation; its
conventions, and above all its confirmation rule, apply here.

Request: $ARGUMENTS

## 1. Fetch the conversions

`POST /api/reports/conversion-report/v1` returns individual conversions. `pagination` is
required (`limit` up to 500). Narrow `filters` to what the user asked: `timeRange`,
`conversionStatuses`, `offerIds`, `sourceIds` (affiliates), `externalIds` (order IDs),
`customerIds`, `cids`, `conversionIds`, `couponCodes`, `countryCodes`. Without a status in
the request, review `CONVERSION_STATUS_PENDING`, `_ON_HOLD` and `_QUALITY_CONTROL` from the
last 30 days.

Rows name the affiliate only by `source` (`id`, `publicId`); look names up with
`POST /api/node/affiliates/v1` and `filters.publicIds`.

```js
async () => {
	const filters = {
		timeRange: { from: '2026-09-01T00:00:00+02:00', to: '2026-09-30T23:59:59+02:00' },
		conversionStatuses: ['CONVERSION_STATUS_PENDING'],
	}
	const rows = []
	for (let offset = 0, page = 0; page < 20; page++, offset += 500) {
		const res = await codemode.request({
			method: 'POST',
			path: '/api/reports/conversion-report/v1',
			body: { filters, pagination: { limit: 500, offset } },
		})
		if (!res?.pagination) return { error: res }
		rows.push(...(res.rows ?? []))
		if (!res.pagination.hasMore) break
	}

	const num = (d) => Number(d?.value ?? 0)
	const countBy = (key) =>
		rows.reduce((seen, r) => {
			const k = key(r)
			if (k) seen[k] = (seen[k] ?? 0) + 1
			return seen
		}, {})
	const byExternalId = countBy((r) => r.externalId)
	const byCustomer = countBy((r) => r.customerId)
	const amountsByOffer = {}
	for (const r of rows) (amountsByOffer[r.offer?.id] ??= []).push(num(r.conversionAmount))
	const medianAmount = Object.fromEntries(
		Object.entries(amountsByOffer).map(([offer, amounts]) => [
			offer,
			amounts.sort((a, b) => a - b)[Math.floor(amounts.length / 2)],
		]),
	)

	const conversions = rows.map((r) => ({
		id: r.conversionId,
		createdAt: r.createdAt,
		status: r.conversionStatus,
		affiliate: r.source?.publicId,
		offer: r.offer?.name,
		type: r.conversionType?.name,
		amount: num(r.conversionAmount),
		revenue: num(r.revenue),
		payout: num(r.cost),
		currency: r.currencyCode,
		externalId: r.externalId || null,
		country: r.countryCode,
		flags: [
			r.fraudSuspicion?.isSuspected && `fraud: ${r.fraudSuspicion.causes.join(', ')}`,
			r.externalId && byExternalId[r.externalId] > 1 && 'duplicate order ID',
			r.customerId && byCustomer[r.customerId] > 1 && 'repeat customer',
			medianAmount[r.offer?.id] > 0 &&
				num(r.conversionAmount) > 5 * medianAmount[r.offer?.id] &&
				"amount over 5× the offer's median",
			num(r.cost) > num(r.revenue) && 'payout above revenue',
			r.accountSettlementId && 'settled, status locked',
		].filter(Boolean),
	}))

	const totals = {}
	for (const c of conversions) {
		const t = (totals[`${c.status} ${c.currency}`] ??= { count: 0, revenue: 0, payout: 0 })
		t.count++
		t.revenue += c.revenue
		t.payout += c.payout
	}

	return {
		count: conversions.length,
		totals,
		flagged: conversions.filter((c) => c.flags.length > 0),
		byAffiliate: Object.entries(countBy((r) => r.source?.publicId))
			.sort((a, b) => b[1] - a[1])
			.slice(0, 10),
		sample: conversions.filter((c) => c.flags.length === 0).slice(0, 20),
	}
}
```

## 2. Summarize before acting

Lead with the count and the total revenue and payout per status and currency. Then show the
flagged conversions in a table with their reasons, and the affiliates with an unusual share
or burst of conversions. Treat flags as reasons to look closer, not as proof of fraud. Offer
the full list when it is long rather than pasting it.

To answer "what happened to order X", filter by `externalIds` (or `cids` or `customerIds`)
and give the status, amounts, affiliate, offer and dates of each match.

## 3. Change statuses after confirmation

Tell the user how many conversions change, from which status to which, and their total
revenue and payout. Approving makes the payout part of the affiliate's balance; rejecting,
refunding or marking as fraud removes it. Then wait for the user's yes.

Leave out conversions that already have the target status, and those included in an
affiliate settlement (`accountSettlementId` set): the API refuses to change a settled
conversion. Name the settled ones; changing them would first need that affiliate settlement
deleted, which the `pay-affiliates` skill covers.

`PUT /api/node/conversions/v1/{id}/status` changes one conversion. Run a confirmed batch in
one `execute` call:

```js
async () => {
	const ids = ['…']
	const status = 'CONVERSION_STATUS_APPROVED'
	const results = { updated: [], failed: [] }
	for (const id of ids) {
		const res = await codemode.request({
			method: 'PUT',
			path: `/api/node/conversions/v1/${id}/status`,
			body: { status },
		})
		if (res?.message && res?.code !== undefined) results.failed.push({ id, error: res.message })
		else results.updated.push(id)
	}
	return { updated: results.updated.length, failed: results.failed }
}
```

Report how many were updated and list the failures with the API's message.

## Refunds and other lists of order IDs

When the user brings order IDs (a refunds export, a list of cancelled orders):

1. Look them up with the conversion report, `filters.externalIds` in chunks of up to 100.
2. Report which order IDs matched one conversion, which matched several, and which matched
   none.
3. After confirmation, set the matched conversions to `CONVERSION_STATUS_REFUNDED` (or the
   status the user asked for) with the loop above.

`POST /api/node/conversions/v1/update-status-by-external-id` changes a status by order ID
directly, but needs the advertiser's `revenueOriginId` and the conversion type's `code`
(from `GET /api/node/conversion-types/v1/for-cid/{cid}`) and shows nothing before it
writes; prefer the lookup above when talking to a user.
