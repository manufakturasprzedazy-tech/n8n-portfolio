# n8n Workflow Portfolio — Nicolas Bednarz

Production automations built between 2024 and 2026 for clients in e-commerce, manufacturing, real estate, services and healthcare, plus internal tooling.

**46 workflows · 965 nodes · 20+ integrated services**

| | |
|---|---|
| Contact | nicolas.bednarz2209@gmail.com · [LinkedIn](https://www.linkedin.com/in/nicolasbednarz) |
| Site | [opiekunautomatyzacji.pl](https://opiekunautomatyzacji.pl) |
| Also | [github.com/manufakturasprzedazy-tech/ekstraktor](https://github.com/manufakturasprzedazy-tech/ekstraktor) — FastAPI service, live on Railway |

---

## How to view these

Every file is a complete, importable n8n workflow.

1. Open any n8n canvas (cloud or self-hosted)
2. Copy the raw JSON
3. Paste directly onto the canvas — `Ctrl+V`

The full graph renders immediately: nodes, connections, parameters and branching logic. No import dialog needed.

**Credentials are not included.** n8n stores them separately from workflow definitions, so these files carry references only — never secrets. Any API key visible in an HTTP node has been replaced with a placeholder.

---

## Where the volume is

| Group | Workflows | Nodes | What it does |
|---|---:|---:|---|
| [Content & social publishing](#content--social-publishing) | 9 | 293 | AI content generation → video render → publish to 4 platforms, driven from Telegram |
| [Marketplace orders & invoicing](#marketplace-orders--invoicing) | 6 | 214 | Order sync across Allegro, Erli, Empik, eMAG · FX conversion · invoice issuing |
| [Voice AI](#voice-ai) | 6 | 132 | Phone booking bots — LLM-agent and deterministic variants, compared in production |
| [AI email triage](#ai-email-triage) | 7 | 46 | Classification, PII anonymization, jailbreak filtering, LLM judge, Slack approval |
| [Invoicing & tax compliance](#invoicing--tax-compliance) | 6 | 69 | Advance/final invoices, KSeF submission, payment-failure recovery |
| [Scraping & enrichment](#scraping--enrichment) | 2 | 43 | Apify-based scraping into Sheets and Airtable |
| [Personal productivity system](#personal-productivity-system) | 4 | 50 | Task system with AI daily planning over Telegram |
| [Sales agents & other](#sales-agents--other) | 6 | 118 | Email sales agents with memory, LLM evaluation pipeline |

---

## Content & social publishing

Nine connected workflows. The client runs the entire system from a Telegram chat and never touches n8n.

| Workflow | Nodes | Description |
|---|---:|---|
| `MB_Telegram_Bot` | **152** | State machine driving the whole system. Session handling, type menus, callback acknowledgement, sequential approval, per-element edit and regeneration |
| `MB_Regenerator` | 38 | Regenerates a single element by scope — hashtags, script, per-platform copy, SEO, graphics or render — without rerunning the pipeline |
| `MB_Content_Generator` | 29 | Routes by content type, pulls active guidelines and property data, generates the piece |
| `MB_Publish` | 25 | Per-platform publishing: Facebook Reels three-step upload, Instagram container + publish, TikTok, WordPress |
| `MB_Video_Render` | 20 | json2video render with status polling and scenario validation |
| `MB_TokenMonitor` | 12 | Six-hourly check of quota and token health across six external APIs |
| `MB_Metadata_Generator` | 8 | Per-platform captions and SEO metadata from a rules block |
| `MB_Publish_Scheduler` | 5 | 15-minute cron dispatching anything scheduled |
| `MB_ErrorHandler` | 4 | Error trigger mapping failures to plain-language messages sent to the operator |

**Engineering notes.** Publishing is failure-isolated — one platform going down does not block the others, verified on a live run. An out-of-memory crash was traced to unforced image output format returning ~9 MB PNGs; setting JPEG at quality 80 cut payload size by 20–30×.

---

## Marketplace orders & invoicing

| Workflow | Nodes | Description |
|---|---:|---|
| `MDZ_Invest_Pobieranie_Zamowien` | 52 | Order pull across three Allegro accounts, Erli, Empik and eMAG, with NBP currency conversion |
| `MDZ_Test_1` | 56 | Full multi-marketplace sync variant |
| `MDZ_Zamowienia` | 48 | Production order sync |
| `MDZ_INVEST_WYSTAWIANIE_FV` | 26 | Invoice issuing with country mapping for six markets |
| `MDZ_FV` | 16 | Accounting sheet population with batching and rate limiting |
| `MDZ_INVEST_Test_FV` | 16 | Staging variant of the above |

Six markets (PL, CZ, HU, BG, RO, SK), two invoice types, SHA1-HMAC authentication against the accounting API.

---

## Voice AI

Two architectures for the same problem, built and compared in production: an LLM agent with tools, and a fully deterministic flow.

| Workflow | Nodes | Description |
|---|---:|---|
| `NoAgentVoiceBotv3` | 55 | Deterministic booking flow — slot computation, availability ranges, response payload assembly. No LLM in the decision path |
| `NoAgentVoiceBot_-_Complete` | 23 | Webhook validation, request-type routing, availability logic |
| `TheBestOfVoiceBot` | 23 | Refined variant of the deterministic architecture |
| `Voice_AI_BS` | 12 | LLM agent with availability tool and conversation memory |
| `Sekretrarka_AI` | 10 | ElevenLabs voice agent with appointment tools — check, create, edit, delete |
| `VAPI_Klinika` | 9 | VAPI webhook → agent with Postgres memory → Airtable booking records |

**Takeaway from running both:** the deterministic flow is larger but predictable, and for booking it turned out to be the safer default. The agent variant is smaller and more flexible but harder to guarantee.

---

## AI email triage

Classifies incoming mail, drafts a reply, and routes it for human approval. It never sends autonomously.

| Workflow | Nodes | Description |
|---|---:|---|
| `ET-A_Email_Triage_Skrzynka` | 19 | Gmail trigger → normalization → **PII anonymization** → **jailbreak classification** → model via OpenRouter → per-thread memory → structured output parser |
| `ET-A_Slack_Akcje_Odbiornik` | 9 | Receives Slack button actions, sends the approved draft or hands the thread to a human |
| `ET-A_Agentic_Variant` | 6 | Agent-based variant of the same task, using tool calling and memory |
| `ET-A_Dzienne_Podsumowanie` | 4 | Daily 08:00 rollup of the last 24 hours by category |
| `ET-A_Setup_Slack_Channels` | 3 | Idempotent provisioning of ten Slack channels |
| `ET-A_Setup_Labels` | 3 | Idempotent provisioning of ten Gmail labels |
| `ET-A_Error_Handler` | 2 | Error trigger → Slack alert with execution context |

**Engineering notes.** PII detection uses custom rules for Polish NIP, REGON, IBAN and phone formats, because the standard library does not recognise them. An LLM judge decides in **fail-closed** mode whether a reply may go out without a human. Measured cost: **$0.008–0.011 per email**, taken from API usage rather than estimated.

---

## Invoicing & tax compliance

| Workflow | Nodes | Description |
|---|---:|---|
| `SH-A_Shoper_Platnosc_Odrzucona` | 16 | 5-minute cron catching unpaid online orders → BaseLinker cross-check → state re-verification → proforma with payment link |
| `FZ-A_Zaliczki_EOM_v2` | 12 | End-of-month cron issuing advance invoices, two-layer idempotency |
| `DZ-A_Doplata_Zwrot` | 12 | Webhook → surcharge proforma → PDF → written back to the order |
| `FZ-A_Zaliczki_EOM` | 11 | First production version |
| `FZ-B_Faktura_Koncowa` | 11 | Webhook → final invoice derived from the advance, PDF back to the order |
| `KSEF-Wysylka-Dzienna` | 7 | Daily submission to the Polish national e-invoicing system, with a separate verification line and a single aggregated error report |

Every one of these is idempotent — reruns cannot double-issue an invoice.

---

## Scraping & enrichment

| Workflow | Nodes | Description |
|---|---:|---|
| `Extract__Enrich_LinkedIn_Comments_to_Leads_with_Apify__Google_Sheets_CSV` | 28 | Form or manual trigger → paginated Apify runs → lead extraction and enrichment → Google Sheets / CSV |
| `Tiktok_scraper_content` | 15 | Apify keyword and account scraping → transcript check → content ideas into Airtable |

---

## Personal productivity system

Internal tooling — a gamified task system I built for myself and use daily.

| Workflow | Nodes | Description |
|---|---:|---|
| `SystemSAM_-_Telegram_Handler_v2` | 23 | Context-routed Telegram handler for task updates |
| `SystemSAM_-_Daily_Plan_Generator_v2` | 15 | Subtasks + habits + player stats → OpenAI → generated daily plan |
| `SystemSAM_-_Daily_Summary_v2` | 6 | Evening summary request |
| `SystemSAM_-_Daily_Reminder_v2` | 6 | Inactivity-triggered nudge |

---

## Sales agents & other

| Workflow | Nodes | Description |
|---|---:|---|
| `Telegram_10` | 65 | Telegram bot with Airtable-backed session store and router dispatch |
| `Agent_AI_-_Centrum_Dowodzenia` | 17 | Slack-triggered assistant, branches on text vs audio and transcribes voice input |
| `_Ogrodzenia_-_Agent_Sprzedaowy_Gmail` | 10 | Email sales agent — data extraction → GPT-4.1 agent → Postgres conversation memory → pricing pulled from Sheets |
| `Email_Automation_Agent_v13` | 10 | Gmail → validation → CRM lookup → agent with conversation memory |
| `Uzgadnianie_faktur_-_pipeline_3-etapowy_demo` | 9 | Three-stage LLM pipeline — normalize → match → verify. Built for a benchmark of 14 models on payment reconciliation |
| `Konwersacja_Mailowa` | 7 | Email conversation agent with a pricing tool, Postgres memory and CRM lead creation |

---

## A note on what is here

These are working files, not polished demos. Some carry version suffixes, some are staging variants of a production flow, and a few are earlier iterations kept deliberately so the evolution is visible.

Two of these started from publicly available community templates and were rebuilt for a client's process rather than written from scratch: `Email_Automation_Agent_v13` and `_Ogrodzenia_-_Agent_Sprzedaowy_Gmail`.

Happy to walk through any of them on a screen share.
