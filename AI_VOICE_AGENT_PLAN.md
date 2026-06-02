# AI Voice Agent — Implementation Plan

> **Goal**: Build a production AI voice agent that answers inbound calls, qualifies leads, updates HubSpot CRM, and performs outbound callbacks — all orchestrated by n8n Cloud.

> **Stack**: Aircall (phone) + Retell AI (voice agent) + n8n Cloud (orchestration) + Azure OpenAI (analysis) + HubSpot (CRM) + Pinecone/Supabase (knowledge base) + Telegram (alerts)

---

## Tools & Technologies Used

| Category | Tool | Purpose |
|----------|------|---------|
| **Telephony** | Aircall | Business phone number, call routing, SIP, fallback to humans |
| **Voice AI Platform** | Retell AI | Real-time STT, dialogue management, TTS, interruption handling, outbound calls |
| **Workflow Orchestration** | n8n Cloud | Webhook handling, workflow automation, retry logic, scheduling |
| **LLM / NLU** | Azure OpenAI (GPT-4o / GPT-4o-mini) | Post-call structured extraction, intent classification, sentiment analysis, summarization |
| **Speech-to-Text** | Retell AI (built-in) | Real-time transcription during live call |
| **Text-to-Speech** | Retell AI (built-in) | Real-time voice synthesis during live call |
| **CRM** | HubSpot | Contact management, notes, tasks, deals, lifecycle tracking |
| **Vector Database** | Pinecone / Supabase pgvector | Knowledge base storage, semantic search for RAG |
| **Embeddings** | Azure OpenAI (text-embedding-3-small) | Convert documents + queries to 1536-dim vectors |
| **Storage** | Supabase / AWS S3 | Raw transcripts, recordings, dead-letter queue, analytics data |
| **Notifications** | Telegram | Real-time alerts (new leads, errors, escalations, daily summaries) |
| **Auth & Security** | n8n Credential Store | Secure storage for all API keys and tokens |

## Category Breakdown

| Category | What It Covers |
|----------|----------------|
| 🔊 **Real-Time Voice** | Live conversation — STT, dialogue, TTS, interruption, latency (Retell AI) |
| 📞 **Telephony** | Phone numbers, call routing, SIP trunks, forwarding, business hours (Aircall) |
| ⚙️ **Orchestration** | Webhooks, workflow chaining, retries, dead-letter, scheduling (n8n Cloud) |
| 🧠 **AI / NLP** | Intent classification, entity extraction, sentiment, summarization (Azure OpenAI) |
| 🔍 **Knowledge Base / RAG** | Document chunking, embedding, vector search, confidence thresholds (Pinecone/Supabase) |
| 📋 **CRM** | Contact upsert, notes, tasks, deals, lifecycle stages, owner assignment (HubSpot) |
| 💾 **Storage** | Transcripts, recordings, analysis JSON, DLQ, metrics (Supabase/S3) |
| 📢 **Notifications** | Alerts, daily summaries, escalation pages, error alerts (Telegram) |
| 🔒 **Compliance** | Consent, DNC, quiet hours, redaction, retention, audit logs |
| 📊 **Monitoring** | Execution tracking, metric dashboards, weekly reports, anomaly alerts |

## Tools Per Phase

| Phase | Primary Tools Used |
|-------|-------------------|
| 1 — Accounts & Credentials | Retell AI, HubSpot, Azure OpenAI, Telegram, n8n, Aircall |
| 2 — Voice Agent Config | Retell AI |
| 3 — Inbound Context | n8n, HubSpot, Retell AI (webhooks) |
| 4 — Post-Call Processing | n8n, Azure OpenAI, HubSpot, Telegram |
| 5 — Outbound Callback | n8n, Retell AI, Telegram |
| 6 — Aircall Integration | Aircall, Retell AI |
| 7 — Knowledge Base (RAG) | Pinecone/Supabase, Azure OpenAI (embeddings), n8n, Retell AI |
| 8 — Human Escalation | Retell AI, Aircall, n8n, Telegram |
| 9 — Error Handling | n8n, Supabase/S3, Telegram |
| 10 — Outbound Triggers | n8n, HubSpot, Supabase |
| 11 — Monitoring | n8n, Telegram, Supabase/Google Sheets |
| 12 — Compliance | n8n, HubSpot, Supabase/S3 |
| 13 — Production Hardening | All tools |

---

## Phase 1: Accounts & Credentials Setup

| Step | Task |
|------|------|
| 1.1 | Create Retell AI account — get API key, note dashboard URL |
| 1.2 | Create Retell AI phone agent — choose voice, set language to English |
| 1.3 | Get Retell AI phone number (or connect Aircall via SIP/forwarding) |
| 1.4 | Create HubSpot private app — scopes: contacts.read/write, notes.write, tasks.write, deals.read |
| 1.5 | Get Azure OpenAI endpoint + deployment name (GPT-4o or GPT-4o-mini) |
| 1.6 | Create Telegram bot via BotFather — get bot token + chat ID |
| 1.7 | Set up n8n Cloud workspace — note webhook base URL |
| 1.8 | Add all credentials to n8n credential store (Retell HTTP Header, HubSpot App Token, Azure OpenAI Header, Telegram) |
| 1.9 | Fill out `config/environment.template.md` with all values (don't commit secrets) |

## Phase 2: Voice Agent Configuration (Retell AI)

| Step | Task |
|------|------|
| 2.1 | Create inbound agent in Retell dashboard |
| 2.2 | Paste system prompt from `n8n/prompts/voice-agent-system-prompt.md` into agent config |
| 2.3 | Configure agent voice — pick natural-sounding voice, set speed/pitch |
| 2.4 | Set agent greeting: "Hi, thanks for calling [Company]. How can I help you today?" |
| 2.5 | Enable transcript output in Retell settings |
| 2.6 | Enable recording URL output |
| 2.7 | Configure webhook URL for `call.started` → `https://<n8n>/webhook/ai-voice/inbound-context` |
| 2.8 | Configure webhook URL for `call.ended` → `https://<n8n>/webhook/ai-voice/post-call` |
| 2.9 | Set up dynamic variables in Retell — `first_name`, `caller_type`, `lifecycle_stage` (fed from n8n) |
| 2.10 | Test agent in Retell playground — verify it responds naturally |
| 2.11 | Configure interruption handling — allow caller to interrupt mid-sentence |
| 2.12 | Set max call duration (e.g., 10 minutes) |
| 2.13 | Configure end-of-speech detection sensitivity |

## Phase 3: Inbound Context Workflow (n8n)

| Step | Task |
|------|------|
| 3.1 | Import `inbound-call-context.workflow.json` into n8n |
| 3.2 | Replace credential placeholders with real n8n credential IDs |
| 3.3 | Verify webhook path: `POST /webhook/ai-voice/inbound-context` |
| 3.4 | Test: Normalize Payload node — extract `call_id` and `phone` from Retell webhook body |
| 3.5 | Test: Search HubSpot Contact by phone number |
| 3.6 | Test: Build Agent Context — returns `caller_type`, `first_name`, `lifecycle_stage` |
| 3.7 | Test: Respond With Context — returns JSON to Retell within <2 seconds |
| 3.8 | Test end-to-end: call the Retell number → verify n8n receives webhook → returns context |
| 3.9 | Verify Retell agent uses dynamic variables (greets existing customer by name) |
| 3.10 | Add error handling — if HubSpot search fails, return `caller_type: "unknown"` gracefully |

## Phase 4: Post-Call Processing Workflow (n8n)

| Step | Task |
|------|------|
| 4.1 | Import `post-call-processing.workflow.json` into n8n |
| 4.2 | Replace credential placeholders |
| 4.3 | Verify webhook path: `POST /webhook/ai-voice/post-call` |
| 4.4 | Test: Normalize Call Payload — extract transcript, call_id, direction, duration, recording_url |
| 4.5 | Test: Azure OpenAI Analysis — send transcript → get structured JSON (intent, sentiment, entities, summary) |
| 4.6 | Validate response against `n8n/schemas/call-analysis.schema.json` |
| 4.7 | Test: Upsert HubSpot Contact — find or create contact from extracted entities |
| 4.8 | Test: Build CRM Note — format note using HubSpot note template |
| 4.9 | Test: Create HubSpot Note — verify note appears on contact record |
| 4.10 | Test: Task Required? — IF node checks `analysis.task_required === true` |
| 4.11 | Test: Create HubSpot Task — verify task with correct priority, subject, due date |
| 4.12 | Test: Telegram Alert — verify notification with intent, sentiment, lead quality |
| 4.13 | End-to-end test: make a real call → hang up → verify HubSpot note + task + Telegram within 30s |

## Phase 5: Outbound Callback Workflow (n8n)

| Step | Task |
|------|------|
| 5.1 | Create outbound agent in Retell AI (different prompt — "Hi {first_name}, this is [Company] calling about...") |
| 5.2 | Import `outbound-callback.workflow.json` into n8n |
| 5.3 | Replace credential placeholders |
| 5.4 | Test: Prepare Callback — extract contact_id, to_number, reason, consent, DNC, attempts |
| 5.5 | Test: Eligibility Check — consent=true AND dnc=false AND attempts<3 |
| 5.6 | Test: Create Outbound AI Call — POST to Retell `/v2/create-phone-call` with dynamic variables |
| 5.7 | Test: Telegram Skipped Alert — fires when eligibility fails |
| 5.8 | Verify outbound call triggers post-call webhook → reuses Phase 4 workflow |
| 5.9 | Test with your own phone number — verify AI calls you and conversation works |
| 5.10 | Add business hours check (don't call outside 9am-6pm in contact's timezone) |

## Phase 6: Aircall Integration

| Step | Task |
|------|------|
| 6.1 | Decide routing: Aircall forwards to Retell number OR Retell owns the number directly |
| 6.2 | If forwarding: configure Aircall call forwarding rules to Retell number |
| 6.3 | If SIP: configure Aircall SIP trunk to Retell |
| 6.4 | Test: call Aircall number → verify it reaches Retell AI agent |
| 6.5 | Configure Aircall fallback — if Retell is down, ring human agents |
| 6.6 | Set up Aircall business hours — after-hours calls go to AI, business hours go to humans (or vice versa) |
| 6.7 | Test call routing for different scenarios (business hours, after hours, busy) |

## Phase 7: Knowledge Base (RAG)

| Step | Task |
|------|------|
| 7.1 | Choose provider: Pinecone, Supabase pgvector, or AWS Bedrock Knowledge Bases |
| 7.2 | Prepare knowledge documents — FAQs, pricing, services, policies, objection handling |
| 7.3 | Chunk documents (500-1000 tokens per chunk) |
| 7.4 | Generate embeddings (text-embedding-3-small) and upsert to vector store |
| 7.5 | Configure Retell AI to query knowledge base during conversation (via function calling or webhook) |
| 7.6 | Create n8n webhook for knowledge lookup: receive query → embed → vector search → return top 3 results |
| 7.7 | Test: ask agent a product/pricing question → verify it answers from knowledge base |
| 7.8 | Test: ask agent something NOT in knowledge base → verify it says "I'll have someone follow up" |
| 7.9 | Add confidence threshold — if similarity score < 0.75, don't use the result |
| 7.10 | Add metadata filters — filter by product, region, customer type |

## Phase 8: Human Escalation & Handoff

| Step | Task |
|------|------|
| 8.1 | Define escalation triggers in Retell agent prompt (angry caller, legal question, human request) |
| 8.2 | Configure Retell to transfer call to Aircall queue when escalation triggered |
| 8.3 | Add escalation webhook to n8n — log escalation reason, notify Telegram immediately |
| 8.4 | Test: say "I want to speak to a human" → verify transfer happens |
| 8.5 | Test: express anger/frustration → verify AI escalates |
| 8.6 | Add fallback: if no human available, AI takes message and creates urgent task |
| 8.7 | Telegram alert for escalation: include caller name, reason, sentiment, contact URL |

## Phase 9: Error Handling & Reliability

| Step | Task |
|------|------|
| 9.1 | Add validation node at start of each workflow — check required fields exist |
| 9.2 | Add try/catch around Azure OpenAI call — if analysis fails, create basic note with raw transcript |
| 9.3 | Add retry logic to HTTP requests (3 retries with exponential backoff) |
| 9.4 | Create dead-letter workflow — failed executions saved to Supabase/S3 |
| 9.5 | Add Telegram alert for all workflow failures |
| 9.6 | Implement idempotency — store processed `call_id`s, skip duplicates |
| 9.7 | Add duplicate task prevention — check if task already exists for this call_id |
| 9.8 | Test: simulate Azure OpenAI timeout → verify graceful degradation |
| 9.9 | Test: simulate HubSpot API error → verify error is logged and alerted |
| 9.10 | Add webhook signature verification (if Retell supports it) |

## Phase 10: Outbound Triggers & Automation

| Step | Task |
|------|------|
| 10.1 | Create HubSpot workflow: when `ai_callback_requested = true` → trigger n8n outbound webhook |
| 10.2 | Create scheduled trigger: every 30 min, check for missed calls → trigger callback |
| 10.3 | Create web enquiry trigger: new form submission → qualify → callback if hot lead |
| 10.4 | Add attempt tracking — store attempts in HubSpot custom property or Supabase |
| 10.5 | Add max attempts (3 per contact per campaign) |
| 10.6 | Add cooldown period — don't call same contact within 4 hours |
| 10.7 | Add timezone-aware scheduling — only call during business hours in contact's timezone |
| 10.8 | Test full loop: missed call → n8n detects → waits 15 min → AI calls back → post-call updates CRM |

## Phase 11: Monitoring & Analytics

| Step | Task |
|------|------|
| 11.1 | Create Telegram daily summary: calls handled, leads captured, tasks created, escalations |
| 11.2 | Track key metrics in Supabase/Google Sheets: call volume, avg duration, intent distribution |
| 11.3 | Track sentiment distribution — alert if negative sentiment spikes |
| 11.4 | Track lead quality distribution — hot/warm/cold/not_qualified |
| 11.5 | Track escalation rate — alert if >20% of calls escalate |
| 11.6 | Track AI confidence scores — alert if average drops below 0.7 |
| 11.7 | Add n8n execution monitoring — track success/failure rates |
| 11.8 | Create weekly report workflow — summarize all metrics, send to Telegram/email |

## Phase 12: Compliance & Security

| Step | Task |
|------|------|
| 12.1 | Implement consent tracking — record consent status per contact |
| 12.2 | Implement DNC (Do Not Call) list — check before every outbound call |
| 12.3 | Configure quiet hours — no calls before 8am or after 8pm local time |
| 12.4 | Add call recording disclosure — agent says "This call may be recorded" at start |
| 12.5 | Implement transcript redaction — remove credit card numbers, SSNs before storing |
| 12.6 | Store raw transcripts outside HubSpot (Supabase/S3) — HubSpot gets summaries only |
| 12.7 | Set recording retention policy — auto-delete after 90 days |
| 12.8 | Audit log — log all automated CRM writes with timestamp and source |
| 12.9 | Separate dev/staging/production credentials |
| 12.10 | Review TCPA (US), ACMA (AU), GDPR (EU) compliance for your target market |

## Phase 13: Production Hardening & Go-Live

| Step | Task |
|------|------|
| 13.1 | QA sampling — randomly review 10% of AI call transcripts for quality |
| 13.2 | Prompt versioning — store prompt versions, track which version produced which results |
| 13.3 | A/B test agent prompts — compare greeting styles, qualification approaches |
| 13.4 | Load test — simulate 10 concurrent calls, verify no timeouts or dropped webhooks |
| 13.5 | Set up Retell AI fallback — if Retell is down, Aircall rings humans directly |
| 13.6 | Document runbook — how to replay failed calls, how to update prompts, how to add to KB |
| 13.7 | Create manual replay workflow — re-process a call_id through post-call pipeline |
| 13.8 | Final end-to-end test with real external number |
| 13.9 | Activate all n8n workflows in production |
| 13.10 | Route first real calls to AI agent |
| 13.11 | Monitor first 24 hours — watch Telegram alerts, check HubSpot notes quality |
| 13.12 | Iterate on prompt based on real call patterns |

---

## Quick Reference: Workflow → Webhook Mapping

| Workflow | Webhook Path | Trigger |
|----------|-------------|---------|
| Inbound Context | `POST /webhook/ai-voice/inbound-context` | Retell `call.started` |
| Post-Call Processing | `POST /webhook/ai-voice/post-call` | Retell `call.ended` |
| Outbound Callback | Manual / HubSpot trigger / Schedule | n8n internal |

## Quick Reference: What Happens Per Call

```
INBOUND:
  Phone rings → Aircall → Retell AI
  → Retell asks n8n "who is this?" (inbound context webhook)
  → n8n searches HubSpot → returns context
  → Retell greets caller with context → has conversation
  → Call ends → Retell sends transcript to n8n (post-call webhook)
  → n8n → Azure OpenAI analysis → HubSpot note + task → Telegram alert

OUTBOUND:
  Trigger (missed call / schedule / manual)
  → n8n checks eligibility (consent, DNC, hours, attempts)
  → n8n calls Retell API → AI makes the call
  → Call ends → same post-call pipeline as inbound
```
