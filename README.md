# AI Lead Capture, Qualification & CRM Routing (n8n)

An n8n workflow that sits behind a website form. Every inbound lead gets cleaned up, scored by Claude plus a set of fixed rules, and routed to the right place in HubSpot, Slack and Gmail within seconds of the form being submitted.

## How it works

```
POST /webhook/lead-capture
  -> normalize + validate (handles different field names from different forms)
       bad email, missing name or honeypot filled -> 400 response
  -> Claude qualifies the lead into a fixed JSON schema
       fit score, intent score, spam probability, intent type, seniority,
       company size, budget signal, timeline, use case, reasoning, next step
  -> blend AI scores with deterministic rules -> tier
  -> route:
       hot          -> HubSpot contact + deal -> Slack alert to sales (1h SLA)
       warm         -> HubSpot contact -> email to SDR team (24h SLA)
       cold         -> nurture email to the lead -> HubSpot contact
       disqualified -> logged only
  -> every lead logged to Google Sheets -> 200 response
```

**Why the score isn't just the model's number.** Claude is good at reading the message (is this a real buying question or a student asking for an internship?) but its scores drift between runs. So the final score is `0.45 x fit + 0.55 x intent`, then adjusted by rules you can read and audit:

| Signal | Points |
|---|---|
| personal email domain (gmail, yahoo...) | -12 |
| role address (info@, sales@...) | -6 |
| phone given / company given | +4 each |
| detailed message (180+ chars) | +5 |
| very short message (under 20 chars) | -8 |
| buying-process words (RFP, pilot, procurement...) | +6 |
| job-seeker words (internship, resume, hiring...) | -25 |
| attributed UTM campaign | +3 |

Tiers: hot 75+, warm 50+, cold 25+, below 25 or spam probability 0.7+ is disqualified. Every adjustment is saved with the lead so a rep can see why it scored the way it did.

## Setup

1. Import `ai-lead-capture-qualification-crm.json`.
2. Add credentials on the Anthropic, HubSpot, Slack, Gmail and Google Sheets nodes.
3. In **Score & Route**, replace the `@yourcompany.com` owner emails with your teams.
4. Set the Slack channel, the Google Sheet ID (`YOUR_GOOGLE_SHEET_ID`) and the Gmail sender.
5. Point your form at `POST /webhook/lead-capture`. Add a hidden `website_url_hp` field as a honeypot.

## Stack

n8n (Webhook, Code, LLM chain with structured output parser, Switch), Anthropic Claude, HubSpot, Slack, Gmail, Google Sheets.
