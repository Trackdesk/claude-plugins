# Trackdesk plugins for Claude

Manage your [Trackdesk](https://trackdesk.com) affiliate, referral or partner program from
Claude in plain language: ask how the program performs, review new affiliates and pending conversions, and
see what you owe your affiliates.

| Plugin                     | What it does                                                                      |
| -------------------------- | --------------------------------------------------------------------------------- |
| [trackdesk](trackdesk/)    | Connects your Trackdesk program and adds skills for reports, reviews and payouts. |

## Claude Code

```
/plugin marketplace add Trackdesk/claude-plugins
/plugin install trackdesk@trackdesk
/plugin configure trackdesk@trackdesk
```

Enter your **Tenant ID** (`acme` for `https://acme.trackdesk.com`), restart Claude Code and
sign in with `/mcp`. The [plugin's README](trackdesk/README.md) covers setup, what you can
ask, what the plugin connects to, and troubleshooting.

## claude.ai, Claude Desktop and Cowork

Add Trackdesk as a connector in Claude under **Settings → Connectors → Add custom
connector**, with the URL `https://<tenant>.trackdesk.com/api/mcp`, and sign in to Trackdesk.

## Support

Questions and suggestions: [support@trackdesk.com](mailto:support@trackdesk.com). Report
security issues to [integrations@trackdesk.com](mailto:integrations@trackdesk.com).

Licensed under the [Apache License 2.0](LICENSE).
