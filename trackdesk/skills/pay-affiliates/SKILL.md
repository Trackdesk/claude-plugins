---
name: pay-affiliates
description: Prepare and run affiliate payouts in a Trackdesk program — review open balances up to a date, flag affiliates who cannot be paid yet (no billing details or payment method, under the minimum, held balances), create the settlement after the user confirms, and mark affiliate settlements as paid once the money has gone out. Use when the user asks how much is owed to affiliates, who should be paid this month, wants to run a payout or create a settlement, or to mark payouts as paid.
allowed-tools: mcp__plugin_trackdesk_trackdesk__search
---

# Pay affiliates

Load the `trackdesk-api` skill first unless it is already loaded in this conversation; its
conventions, and above all its confirmation rule, apply here. Payouts move money: be exact
about amounts and currencies, and never round them before the user confirms.

Request: $ARGUMENTS

## 1. Review open balances

The open balance is approved payout not settled yet. Pick `until`, the last day whose
balances are included — usually the end of the previous month, at `23:59:59` in the
program's time zone. Add the thresholds the user names: `minAmount` (`{ value: '50' }`),
`minConversionCount`, `tierIds`, `offerIds`, `affiliateTagIds`,
`defaultPaymentMethodTypes`.

```js
async () => {
	const filter = { until: '2026-09-30T23:59:59+02:00' }
	const balances = []
	for (let offset = 0, page = 0; page < 20; page++, offset += 500) {
		const res = await codemode.request({
			method: 'POST',
			path: '/api/node/billing/v1/open-balances/list',
			body: { filter, pagination: { limit: 500, offset } },
		})
		if (!res?.balances) return { error: res }
		balances.push(...res.balances)
		if (!res.pagination?.hasMore) break
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
		totals,
		ready: affiliates.filter((a) => !blocked(a)),
		blocked: affiliates.filter(blocked),
	}
}
```

Present the totals per currency, then the affiliates ready to be paid and, separately, those
missing billing details or a payment method. Mention held balances (`onHold`), which may become
payable later. Affiliates whose billing status is excluded do not appear in open balances at
all.

## 2. Create the settlement after confirmation

Before creating it, state the `until` date, the thresholds, the affiliates included (names),
and the total per currency. Ask whether to leave out the blocked affiliates. Wait for the
user's yes.

```js
async () => {
	const res = await codemode.request({
		method: 'POST',
		path: '/api/node/billing/v1/settle-balances',
		body: {
			filter: { until: '2026-09-30T23:59:59+02:00' },
			accountIds: ['…'],
			note: 'September 2026 payout',
		},
	})
	if (!res?.id) return { error: res }
	return {
		settlementId: res.id,
		total: `${res.amount?.value} ${res.currency?.code}`,
		affiliateSettlements: res.accountSettlements?.map((s) => ({
			id: s.id,
			accountId: s.accountId,
			amount: `${s.amount?.value} ${s.currency?.code}`,
		})),
		errors: res.errors ?? [],
	}
}
```

Send the same `filter` you reviewed, so the settlement covers exactly the balances the user
saw. Report the settlement ID, the total, each affiliate's amount and every per-affiliate
error. A settlement created by mistake can be deleted with
`DELETE /api/node/billing/v1/settlements/{id}` (the whole run) or
`DELETE /api/node/billing/v1/account-settlements/{id}` (one affiliate), which returns the
balances to open — again only after the user confirms.

## 3. Pay and mark as paid

Trackdesk records payouts; the money itself goes out through the program's bank, PayPal or
other provider. Once the user says the payments were sent, list the affiliate settlements
with `GET /api/node/billing/v1/account-settlements` (`query`: `timeRange.from`,
`timeRange.to`, optionally `accountId`, `pagination.limit`) and, after confirming which ones,
set each to paid:

```js
async () => {
	const ids = ['…']
	const results = []
	for (const id of ids) {
		const res = await codemode.request({
			method: 'PUT',
			path: `/api/node/billing/v1/account-settlements/${id}/status`,
			body: { status: 'ACCOUNT_SETTLEMENT_STATUS_PAID' },
		})
		results.push({
			id,
			ok: !(res?.message && res?.code !== undefined),
			error: res?.message ?? null,
		})
	}
	return results
}
```

Affiliate settlements paid through Tipalti cannot be changed by hand, and
`ACCOUNT_SETTLEMENT_STATUS_ERROR` cannot be set manually.

Programs paying through Tipalti create payment batches with the `Tipalti Payments V1`
endpoints. A payment batch sends real money: create one only when the user asks for it
explicitly, after showing the exact affiliates, amounts and currencies it will pay, and with
their yes to that list.

## Related

- One-off bonus or deduction: `POST /api/node/affiliate-bonuses/v1`; it is paid with the
  first settlement on or after its billable date.
- Leave an affiliate out of payouts: `PUT /api/node/affiliates/v1/{publicId}/billing-status`
  with `AFFILIATE_BILLING_STATUS_EXCLUDED`.
- One affiliate's balance: `GET /api/node/billing/v1/account-balances/{accountId}`.

Check each endpoint's request schema with the search tool before the first call.
