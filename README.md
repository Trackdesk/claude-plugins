# Trackdesk plugins for Claude

Manage your [Trackdesk](https://trackdesk.com) affiliate program from Claude Code in plain
language: ask how the program performs, review new affiliates and pending conversions, and
prepare affiliate payouts.

## Install

In Claude Code:

```
/plugin marketplace add Trackdesk/claude-plugins
/plugin install trackdesk@trackdesk
/plugin configure trackdesk@trackdesk
```

Enter your **Tenant ID**, the subdomain your program runs on (`acme` for
`https://acme.trackdesk.com`), and restart Claude Code.

## Sign in

Run `/mcp`, choose `plugin:trackdesk:trackdesk` and authenticate. Your browser opens the
Trackdesk sign-in and consent page; once you approve, Claude works as you, with your role's
permissions. You can disconnect it at any time under **Settings → Integrations → OAuth2 →
Connected applications**.

Where a browser sign-in is not possible, create a token under **Settings → Integrations →
Personal access tokens** and enter it as the plugin's **API key** with
`/plugin configure trackdesk@trackdesk`.

## What you can ask

- "How did the program do last month compared with the month before? Break it down by
  offer."
- "Who were our ten most profitable affiliates this quarter?"
- "Go through the pending affiliate applications and point out anything suspicious."
- "Review last week's pending conversions and approve the ones that look clean."
- "Here is our refunds export — mark these orders as refunded."
- "Who should we pay for September, and who is missing payout details?"

Claude picks the right skill on its own. You can also start one directly:

| Command                          | Does                                                                        |
| -------------------------------- | --------------------------------------------------------------------------- |
| `/trackdesk:performance-report`  | Clicks, conversions, revenue, payouts and profit by any breakdown           |
| `/trackdesk:review-affiliates`   | Pending sign-ups with their answers and warning signs; approve or reject    |
| `/trackdesk:review-conversions`  | Pending conversions with fraud and duplicate checks; approve, reject, refund |
| `/trackdesk:pay-affiliates`      | Open balances, settlements, marking payouts as paid                         |

## Safety

- Claude reads your data freely but asks for your explicit confirmation before it changes
  anything, and tells you exactly what will change first.
- Every request runs with your own Trackdesk permissions. What your role cannot do, Claude
  cannot do either.
- Claude Code also asks before each tool call unless you have allowed it.

## Troubleshooting

- **No Trackdesk tools in Claude Code** — check that the Tenant ID is set
  (`/plugin configure trackdesk@trackdesk`), restart Claude Code and sign in with `/mcp`.
- **"Missing required X-Api-Key header" when signing in** — your program does not offer
  the browser sign-in yet; use an API key as described above.
- **"Permission denied"** — your Trackdesk role or plan does not include that action.

## Support

Questions and suggestions: [support@trackdesk.com](mailto:support@trackdesk.com).
