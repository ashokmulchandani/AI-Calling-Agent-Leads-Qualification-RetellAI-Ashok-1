# AI Voice Agent Design

## Objective

Build a production-quality AI voice agent that can answer inbound enquiries from new leads and existing customers, gather useful information during the first conversation, analyze intent and sentiment, update HubSpot CRM notes, create follow-up tasks, and perform outbound callbacks in an engaging conversational manner.

The system should use n8n Cloud as the orchestration layer, Aircall as the phone/business calling layer, HubSpot as the CRM, Azure OpenAI for NLP/NLU and structured extraction, and a knowledge base such as Pinecone, Supabase pgvector, or AWS Bedrock Knowledge Bases for grounded answers.

## Key Capabilities

- Answer inbound calls.
- Identify new leads versus existing customers.
- Conduct natural multi-turn conversations.
- Transcribe calls.
- Classify caller intent.
- Extract entities such as name, phone, email, company, service interest, budget, urgency, location, and preferred callback time.
- Analyze sentiment.
- Summarize the call.
- Create segmented HubSpot notes.
- Create HubSpot tasks with due date, priority, owner, and associated contact/deal/ticket.
- Notify the team on Telegram.
- Trigger outbound AI callbacks.
- Escalate urgent or sensitive calls to humans.

## Confirmed Architecture

Real-time voice is handled by Retell AI. n8n is NOT in the live audio path — it orchestrates events before and after the call only.

**Locked-in stack:**

- Aircall: business phone number, SIP trunk to Retell AI, HubSpot native integration, human fallback.
- Retell AI: inbound + outbound AI voice agent, STT, TTS, dynamic variables, webhooks.
- n8n Cloud: workflow orchestration — context lookup, CRM updates, notifications, retries, outbound triggers.
- HubSpot: CRM source of truth — contacts, notes, tasks, deals.
- Azure OpenAI (GPT-4o-mini): post-call extraction, intent, sentiment, summarization.
- Pinecone or Supabase pgvector: vector knowledge base for grounded answers.
- Telegram: operational alerts — leads, errors, escalations.
- Supabase or AWS S3: raw transcripts, analysis JSON, dead-letter queue.

## Conversation Intelligence

The voice agent must support the following NLU concepts.

### Intent Classification

Example intents:

- new_sales_enquiry
- pricing_question
- booking_request
- existing_customer_support
- complaint
- cancellation
- reschedule
- billing_question
- callback_request
- spam_or_wrong_number
- human_handoff

### Entity Extraction

Required entities:

- first_name
- last_name
- phone
- email
- company
- location
- service_interest
- product_interest
- budget
- timeline
- urgency
- preferred_callback_time
- existing_customer_identifier

### Sentiment Analysis

Sentiment labels:

- positive
- neutral
- confused
- frustrated
- angry

The sentiment should influence task priority and escalation.

### Dialogue Management

The agent should:

- greet naturally;
- identify caller goal;
- retrieve known CRM context when available;
- ask only missing qualification questions;
- avoid interrogating the caller;
- answer using approved knowledge base content;
- summarize the next step before ending the call;
- escalate to a human when confidence is low, caller is angry, or policy requires it.

## HubSpot CRM Update Strategy

HubSpot remains the source of truth.

For every valid call:

1. Find or create the contact.
2. Associate the call with contact, company, deal, or ticket where possible.
3. Create a structured note.
4. Create a task if follow-up is required.
5. Optionally update lead status, lifecycle stage, deal stage, or ticket status.

## HubSpot Note Template

```text
AI Voice Call Summary

Direction: {{direction}}
Caller Type: {{caller_type}}
Primary Intent: {{primary_intent}}
Sentiment: {{sentiment.label}} ({{sentiment.score}})
Lead Quality: {{lead_quality}}
Escalation Required: {{human_escalation_required}}

Summary:
{{summary}}

Captured Details:
Name: {{entities.name}}
Email: {{entities.email}}
Phone: {{entities.phone}}
Company: {{entities.company}}
Location: {{entities.location}}
Service Interest: {{entities.service_interest}}
Budget: {{entities.budget}}
Timeline: {{entities.timeline}}
Preferred Callback: {{entities.preferred_callback_time}}

Questions Asked:
{{questions_asked}}

Objections / Concerns:
{{objections}}

Recommended Next Step:
{{next_action}}

Recording:
{{recording_url}}

Transcript:
{{transcript_url}}
```

## Task Rules

Create a task when:

- lead quality is hot or warm;
- caller requests a callback;
- caller asks a question the AI cannot answer confidently;
- sentiment is frustrated or angry;
- an appointment, quote, or human action is needed.

Priority mapping:

- angry or complaint: urgent
- hot lead: high
- warm lead: medium
- cold lead: low

Default SLA:

- urgent: same business hour
- high: within 2 business hours
- medium: next business day
- low: within 3 business days

## Outbound Callback Strategy

Outbound callbacks should be triggered by:

- missed call;
- web enquiry;
- HubSpot lifecycle/status change;
- manual field such as `ai_callback_requested`;
- incomplete inbound conversation;
- task due soon.

Before calling, n8n must check:

- consent;
- do-not-call status;
- business hours;
- maximum attempt count;
- recent call attempts;
- customer timezone where known.

## Telegram Notifications

Send Telegram alerts for:

- hot lead captured;
- urgent sentiment or complaint;
- human handoff required;
- failed HubSpot update;
- failed outbound callback;
- workflow dead-letter event.

Example alert:

```text
New AI Voice Lead

Name: Sarah Lee
Phone: +61400000000
Intent: Pricing enquiry
Sentiment: Positive
Lead Quality: Hot
Next Step: Sales callback within 2 hours
HubSpot: {{contact_url}}
```

## Security And Compliance

- Store all secrets in n8n credentials.
- Do not hardcode API keys in workflow nodes.
- Use least-privilege HubSpot private app scopes.
- Verify webhook signatures where supported.
- Redact sensitive data before storing transcripts.
- Store raw transcripts outside HubSpot if large or sensitive.
- Log all automated CRM writes.
- Respect TCPA, DNC, ACMA, consent, quiet hours, and industry-specific compliance.
- Use separate credentials for dev, staging, and production.

## Production Readiness

Required before go-live:

- idempotency using `call_id`;
- retry policy with backoff;
- dead-letter workflow;
- duplicate task prevention;
- webhook payload logging;
- error notifications;
- business-hours guardrails;
- confidence thresholds;
- human fallback routing;
- monitoring dashboard.

