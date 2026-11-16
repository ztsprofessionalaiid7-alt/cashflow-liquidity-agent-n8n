# AI Cash-Flow Forecasting & Liquidity Alert Agent (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `cashflow-forecast-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI cash-flow forecasting & predictive liquidity alert agent (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create table cashflow_alerts in Supabase SQL Editor\n3. Slack channel: cashflow-alerts (change in the Slack node)\n4. Click Execute workflow: 2 mock entities, an 8-week projection each (one healthy, one short of cash)\nReal client use: replace Mock Cash Data with HTTP calls to your accounting API on a Schedule Trigger",
        "height": 260,
        "width": 580
      },
      "id": "b3108cd6-2007-41c5-a1f6-293647f89794",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "bb80e8f3-257c-43b3-9c52-dc16264a329a",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { entity: 'Subsidiary A', opening_balance: 120000, min_buffer: 25000, inflows: [30000, 28000, 35000, 30000, 32000, 30000, 33000, 31000], outflows: [40000, 25000, 22000, 45000, 24000, 23000, 41000, 25000] } },\n  { json: { entity: 'Subsidiary B', opening_balance: 40000, min_buffer: 20000, inflows: [10000, 12000, 8000, 15000, 9000, 11000, 10000, 12000], outflows: [30000, 15000, 28000, 12000, 32000, 14000, 29000, 13000] } }\n];"
      },
      "id": "190730bf-02e9-4d31-8c5a-15eb73736912",
      "name": "Mock Cash Data",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return $input.all().map((it) => {\n  const e = it.json;\n  let bal = e.opening_balance;\n  const balances = [];\n  e.inflows.forEach((inflow, i) => { bal += inflow - e.outflows[i]; balances.push(bal); });\n  const lowest = Math.min.apply(null, balances);\n  return { json: {\n    entity: e.entity,\n    opening_balance: e.opening_balance,\n    min_buffer: e.min_buffer,\n    projected_balances: balances.join(', '),\n    lowest_balance: lowest,\n    lowest_week: balances.indexOf(lowest) + 1,\n    weeks_below_buffer: balances.filter((b) => b < e.min_buffer).length,\n    status: lowest < e.min_buffer ? 'alert' : 'ok'\n  }};\n});"
      },
      "id": "368000e8-b6c9-4f97-b60a-a02b906d27ba",
      "name": "Project Cash",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        440,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Review this 8-week cash projection.\nEntity: {{ $json.entity }}\nOpening balance: {{ $json.opening_balance }}\nMinimum safe buffer: {{ $json.min_buffer }}\nProjected weekly balances: {{ $json.projected_balances }}\nLowest balance: {{ $json.lowest_balance }} in week {{ $json.lowest_week }}\nWeeks below buffer: {{ $json.weeks_below_buffer }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a treasury assistant. If the projection falls below the buffer, call the receivables_aging tool with the entity name and use the overdue invoices in your actions. Respond with ONLY a JSON object, no markdown, no extra text: {\"risk_level\":\"low|medium|high\",\"summary\":\"one or two sentences\",\"actions\":\"2-3 short actions in one string\"}"
        }
      },
      "id": "6c199608-79ba-4470-a984-2d16043c090e",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        680,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "09212ac9-eed5-42cc-ae1c-e4361bedec9e",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        660,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "name": "receivables_aging",
        "description": "Returns overdue customer invoices for an entity. Input: the entity name only, for example 'Subsidiary B'. Returns a JSON list of customer, amount and days overdue.",
        "jsCode": "const aging = {\n  'subsidiary a': [],\n  'subsidiary b': [\n    { customer: 'Northwind', amount: 12000, days_overdue: 45 },\n    { customer: 'Contoso', amount: 9500, days_overdue: 30 },\n    { customer: 'Fabrikam', amount: 7000, days_overdue: 60 }\n  ]\n};\nreturn JSON.stringify(aging[String(query).trim().toLowerCase()] || []);"
      },
      "id": "cd477dac-6724-4970-87cd-b1bf09c55831",
      "name": "Receivables Aging Tool",
      "type": "@n8n/n8n-nodes-langchain.toolCode",
      "typeVersion": 1.3,
      "position": [
        880,
        420
      ]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Project Cash').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst risk = ['low', 'medium', 'high'].includes(String(r.risk_level).toLowerCase()) ? String(r.risk_level).toLowerCase() : 'medium';\nreturn { json: Object.assign({}, ctx, {\n  risk_level: risk,\n  summary: String(r.summary || '').slice(0, 400),\n  actions: String(r.actions || '').slice(0, 500)\n}) };"
      },
      "id": "2a9878c5-26cf-4e53-834d-398e92b654b3",
      "name": "Parse Forecast",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1040,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "620fc6df-a3b7-465b-9404-96682b1a4a90",
              "leftValue": "={{ $json.status }}",
              "rightValue": "alert",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "7bd2d44b-e2d3-4b99-91c8-3c6db374ed56",
      "name": "Liquidity Alert?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1260,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "cashflow-alerts",
          "mode": "name"
        },
        "text": "=:warning: *Liquidity alert: {{ $json.entity }}* ({{ $json.risk_level }} risk)\nLowest balance: {{ $json.lowest_balance }} in week {{ $json.lowest_week }} (buffer {{ $json.min_buffer }})\n{{ $json.summary }}\nActions: {{ $json.actions }}",
        "otherOptions": {}
      },
      "id": "25de979d-c0d0-4561-a684-60bf64a69a72",
      "name": "Slack Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1520,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "cashflow_alerts",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "entity": "={{ $json.entity }}",
            "opening_balance": "={{ $json.opening_balance }}",
            "lowest_balance": "={{ $json.lowest_balance }}",
            "lowest_week": "={{ $json.lowest_week }}",
            "weeks_below_buffer": "={{ $json.weeks_below_buffer }}",
            "status": "={{ $json.status }}",
            "risk_level": "={{ $json.risk_level }}",
            "summary": "={{ $json.summary }}",
            "actions": "={{ $json.actions }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "entity",
              "displayName": "entity",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "opening_balance",
              "displayName": "opening_balance",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "lowest_balance",
              "displayName": "lowest_balance",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "lowest_week",
              "displayName": "lowest_week",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "weeks_below_buffer",
              "displayName": "weeks_below_buffer",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "risk_level",
              "displayName": "risk_level",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "summary",
              "displayName": "summary",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "actions",
              "displayName": "actions",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "44c7a8ad-2716-42ed-8d90-97fedec12283",
      "name": "Save Forecast",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1520,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Cash Data",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Cash Data": {
      "main": [
        [
          {
            "node": "Project Cash",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Project Cash": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Forecast",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Forecast": {
      "main": [
        [
          {
            "node": "Liquidity Alert?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Liquidity Alert?": {
      "main": [
        [
          {
            "node": "Slack Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Forecast",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Forecast",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Receivables Aging Tool": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
