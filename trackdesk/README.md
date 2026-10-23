# Trackdesk

Manage your [Trackdesk](https://trackdesk.com) affiliate, referral or partner program from
Claude in plain language: ask how the program performs, review new affiliates and pending conversions, and
see what you owe your affiliates. The plugin connects Claude to your program's Trackdesk MCP server
and adds skills that teach Claude your program's reports, review workflows and payouts.

## Set up

In Claude Code:

```
/plugin marketplace add Trackdesk/claude-plugins
/plugin install trackdesk@trackdesk
/plugin configure trackdesk@trackdesk
```

Enter your **Tenant ID**, the subdomain your program runs on (`acme` for
`https://acme.trackdesk.com`), and restart Claude Code. Then run `/mcp`, choose
`plugin:trackdesk:trackdesk` and authenticate: your browser opens the Trackdesk sign-in and
consent page, and once you approve, Claude works as you.

In claude.ai, Claude Desktop and Cowork:

1. In Trackdesk, navigate to **Settings → Integrations → OAuth2** and click **Add to Claude**.
2. Add the prefilled connector in Claude, then connect it.
3. Sign in to Trackdesk and click **Allow access**. Each team member signs in with their
   own Trackdesk account.

You can also add a custom connector in Claude under **Customize → Connectors**, using
`https://<tenant>.trackdesk.com/api/mcp`. Choose automatic OAuth client registration when
Claude asks how to identify itself to the authorization server.

## What you can ask

- "How did the program do last month compared with the month before? Break it down by
  offer."
- "Who were our ten most profitable affiliates this quarter?"
- "Go through the pending affiliate applications and point out anything suspicious."
- "Review last week's pending conversions and approve the ones that look clean."
- "Here is our refunds export — mark these orders as refunded."
- "Who should we pay for September, and who is missing payout details?"

Claude picks the right skill on its own. You can also start one directly:

| Command                         | Does                                                                         |
| ------------------------------- | ---------------------------------------------------------------------------- |
| `/trackdesk:performance-report` | Clicks, conversions, revenue, payouts and profit by any breakdown            |
| `/trackdesk:review-affiliates`  | Pending sign-ups with their answers and warning signs; approve or reject     |
| `/trackdesk:review-conversions` | Pending conversions with fraud and duplicate checks; approve, reject, refund |
| `/trackdesk:review-payouts`     | Open balances per currency, who can be paid, past settlements                |
| `/trackdesk:trackdesk-api`      | The API conventions the other skills build on                                |

## What the plugin connects to

- **One remote MCP server:** `https://<tenant>.trackdesk.com/api/mcp`, your own program's
  Trackdesk server. The plugin talks to nothing else, runs no local commands and installs no
  hooks.
- **What it sends there:** the requests Claude makes to read or change your program's data,
  with the access token of your Trackdesk sign-in.
- **What the server reaches:** only your program's Trackdesk API, with your role's
  permissions.

## Safety

- Reading and changing go through separate tools — `read_data` reads, while `create_data`,
  `update_data` and `delete_data` change things — so Claude can read freely, and your Claude
  app asks for your approval before each change unless you have allowed that tool to run
  without asking.
- The skills have Claude tell you exactly what will change and wait for your yes.
- What your Trackdesk role cannot do, Claude cannot do either.
- Claude cannot settle balances, pay affiliates, grant bonuses or change billing details:
  creating settlements, marking them paid, bonuses and payment details stay in Trackdesk.
  Claude can leave an affiliate out of payouts or bring them back by changing their billing
  status. Recording, approving or changing conversions changes the commissions affiliates
  earn, so the skills have Claude ask you first like for any other change.
- To disconnect Claude, navigate to **Settings → Integrations → OAuth2**, find its OAuth2
  credentials, click **Show details**, then **Disconnect** next to your account.

## Troubleshooting

- **No Trackdesk tools** — check that the Tenant ID is set
  (`/plugin configure trackdesk@trackdesk`), restart Claude Code and sign in with `/mcp`.
- **Updated from version 1.0.0** — the plugin no longer uses an API key: run `/mcp`, choose
  `plugin:trackdesk:trackdesk` and sign in.
- **"Permission denied", or Claude says it cannot find or call an action** — your Trackdesk
  role or plan does not include that action.

## Support

Questions and suggestions: [support@trackdesk.com](mailto:support@trackdesk.com). Report
security issues to [integrations@trackdesk.com](mailto:integrations@trackdesk.com).

[Privacy policy](https://trackdesk.com/privacy-policy) ·
[Terms and conditions](https://trackdesk.com/terms-conditions) ·
[API reference](https://developers.trackdesk.com)

Licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
