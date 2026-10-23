---
name: review-payouts
description: Review what a Trackdesk program owes its affiliates — open balances up to a date per currency, who can be paid and who cannot yet (no billing details or payment method, under the minimum, held balances), past settlements and their status — and hand the payout itself over to Trackdesk, since Claude does not settle balances or pay affiliates. Use when the user asks how much is owed to affiliates, who should be paid this month, what a settlement contained, or wants to run a payout.
allowed-tools: mcp__plugin_trackdesk_trackdesk__search mcp__plugin_trackdesk_trackdesk__read_data
---

# Review payouts

Load the `trackdesk-api` skill first unless it is already loaded in this conversation; its
conventions apply here. Be exact about amounts and currencies.

Request: $ARGUMENTS

## 1. Review open balances

The open balance is approved payout not settled yet. Pick `until`, the last day whose
balances are included — usually the end of the previous month, at `23:59:59` in the
program's time zone. Add the thresholds the user names: `minAmount` (`{ value: '50' }`),
`minConversionCount`, `tierIds`, `offerIds`, `affiliateTagIds`,
`defaultPaymentMethodTypes`. With `read_data`:

```js
async () => {
	const filter = { until: '2026-09-30T23:59:59+02:00' }
	const balances = []
	let truncated = false
	for (let offset = 0, page = 0; page < 20; page++, offset += 500) {
		const res = await codemode.request({
			method: 'POST',
			path: '/api/node/billing/v1/open-balances/list',
			body: { filter, pagination: { limit: 500, offset } },
		})
		if (!res?.balances) return { error: res }
		balances.push(...res.balances)
		truncated = Boolean(res.pagination?.hasMore)
		if (!truncated) break
	}

	const affiliates = balances.map((b) => ({
		accountId: b.accountId,
		name: b.name,
		open: b.balancesByCurrency.map((c) => `${c.openBalance.value} ${c.currency.code}`),
		onHold: b.balancesByCurrency
			.filter((c) => Number(c.onHoldBalance?.value ?? 0) !== 0)
			.map((c) => `${c.onHoldBalance.value} ${c.currency.code}`),
		paymentMethod: b.defaultPaymentMethod?.paymentMethodType ?? null,
		hasBillingDetails: b.hasBillingDetails,
		conversions: b.openBalanceConversionCount,
	}))

	const totals = {}
	for (const b of balances)
		for (const c of b.balancesByCurrency)
			totals[c.currency.code] = (totals[c.currency.code] ?? 0) + Number(c.openBalance?.value ?? 0)

	const blocked = (a) =>
		!a.hasBillingDetails ||
		!a.paymentMethod ||
		a.paymentMethod === 'PAYMENT_METHOD_TYPE_UNSPECIFIED'
	return {
		filter,
		affiliateCount: affiliates.length,
		truncated,
		totals,
		ready: affiliates.filter((a) => !blocked(a)),
		blocked: affiliates.filter(blocked),
	}
}
```

When `truncated` is true, the totals cover only the first 10,000 affiliates: say so, and
narrow the filter instead of presenting them as the program's total. Present the totals per
currency, then the affiliates ready to be paid and, separately, those
missing billing details or a payment method. Mention held balances (`onHold`), which may become
payable later. Affiliates whose billing status is excluded do not appear in open balances at
all.

## 2. Hand the payout over to Trackdesk

Creating a settlement, marking affiliate settlements as paid, Tipalti payment batches,
bonuses, and affiliates' payment methods and billing details are not available through
Claude. When the user wants to pay, give them what to enter and send them to **Billing → Open
balance** in Trackdesk: the `until` date, the thresholds, the affiliates to include, and the
total per currency from step 1.

## Past settlements

- Settlements: `GET /api/node/billing/v1/settlements` and
  `GET /api/node/billing/v1/settlements/{id}`.
- Affiliate settlements and their status: `GET /api/node/billing/v1/account-settlements`
  (`query`: `timeRange.from`, `timeRange.to`, optionally `accountId`, `pagination.limit`).
- One affiliate's balance: `GET /api/node/billing/v1/account-balances/{accountId}`.
- Bonuses: `POST /api/node/affiliate-bonuses/v1/list`.

All through `read_data`. Check each endpoint's request schema with `search` before the
first call.
