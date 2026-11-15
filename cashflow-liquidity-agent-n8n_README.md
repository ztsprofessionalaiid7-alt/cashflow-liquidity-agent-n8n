# AI Cash-Flow Forecasting & Liquidity Alert Agent (n8n)

An n8n workflow that projects 8 weeks of cash balance per entity with plain arithmetic, and when the balance falls below the safe buffer an AI Agent (Gemini) checks overdue receivables with a tool and recommends actions. Liquidity alerts go to Slack.

▶ **Demo video:** https://drive.google.com/file/d/1zL8fjDNVWdkhFdRpWUWXV-J9d2wN7wqe/view

## How it works

1. Mock cash data arrives for two entities (opening balance, minimum safe buffer, 8 weeks of inflows and outflows)
2. A Code node projects the weekly balances, finds the lowest balance and week, and counts weeks below the buffer (rules, not AI)
3. The AI Agent reviews the projection, calls the `receivables_aging` tool to find overdue invoices when the buffer is breached, and returns risk level, summary and actions as JSON
4. A Code node validates the answer. Entities with status `alert` go to Slack
5. Every forecast is saved to Postgres

```
Manual Trigger -> Mock Cash Data -> Project Cash -> AI Agent (Gemini + receivables_aging tool)
-> Parse Forecast -> Liquidity Alert?
   yes -> Slack Alert -> Save Forecast
   no  -> Save Forecast
```

## AI Agent

AI Agent node with one tool (`receivables_aging`). The numbers are computed in code, so the forecast is exact and auditable. The agent only reads it and suggests actions.

## Tech stack

n8n, AI Agent, Gemini, Slack, Postgres (Supabase)

## Setup

1. Copy the JSON from `workflow.md` into a file named `cashflow-forecast-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `cashflow-alerts`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: the mock inputs run through all branches

No other API keys are needed, the tool nodes are self-contained mock tools.

## Database

```sql
create table if not exists public.cashflow_alerts (
  id bigint generated always as identity primary key,
  entity text,
  opening_balance numeric,
  lowest_balance numeric,
  lowest_week numeric,
  weeks_below_buffer numeric,
  status text,
  risk_level text,
  summary text,
  actions text,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| Subsidiary A (opening 120,000, buffer 25,000) | OK: lowest balance 110,000, no weeks below the buffer |
| Subsidiary B (opening 40,000, buffer 20,000) | Alert: below the buffer for 7 of 8 weeks, lowest balance -46,000 in week 8, receivables tool used |

## Notes

- The projection uses the mock inflows and outflows exactly as given, no seasonality or growth is modelled
- Replace Mock Cash Data with your bank or accounting export, and `receivables_aging` with a real invoice lookup for client use
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
