# Implementation Plan

## Confirmed Stack

- Telephony: Aircall
- Voice AI: Retell AI
- Orchestration: n8n Cloud
- CRM: HubSpot
- Post-Call AI: Azure OpenAI (GPT-4o-mini)
- Knowledge Base: Pinecone or Supabase pgvector
- Notifications: Telegram
- Storage: Supabase or AWS S3

## Workflow Status

| Workflow | File | Status |
|----------|------|--------|
| Inbound Context | `inbound-call-context.workflow.json` | ✅ Built — ready to import |
| Post-Call Processing | `post-call-processing.workflow.json` | ✅ Built — ready to import |
| Outbound Callback | `outbound-callback.workflow.json` | ✅ Built — ready to import |

## Phase 1: MVP

Goal: handle inbound calls and update HubSpot after the call.

Tasks:

- Create Retell/Vapi voice agent.
- Configure inbound phone number.
- Create n8n webhook for post-call events.
- Add Azure OpenAI structured analysis.
- Create HubSpot note.
- Create HubSpot task.
- Send Telegram alert.

Acceptance criteria:

- A test call produces a transcript.
- HubSpot contact is found or created.
- A segmented note is attached to the contact.
- A task is created when follow-up is needed.
- Telegram receives a summary.

## Phase 2: CRM-Aware Inbound Agent

Goal: make the agent aware of whether the caller is new or existing.

Tasks:

- Add inbound context webhook.
- Search HubSpot by phone.
- Return contact context to Retell/Vapi.
- Use different opening logic for leads and customers.
- Add human escalation flags.

Acceptance criteria:

- Existing customers are greeted with context.
- New leads are qualified.
- Unknown callers are handled gracefully.

## Phase 3: Outbound Callback

Goal: allow AI to call back leads/customers.

Tasks:

- Create HubSpot trigger or scheduled callback workflow.
- Check consent, DNC, timezone, and business hours.
- Call Retell/Vapi outbound API.
- Pass dynamic variables.
- Reuse post-call processing workflow.

Acceptance criteria:

- Eligible contacts receive callback attempts.
- Attempts are logged.
- Duplicate calls are prevented.

## Phase 4: Knowledge Base

Goal: ground answers in approved company information.

Tasks:

- Choose Pinecone, Supabase pgvector, Azure AI Search, or AWS Bedrock Knowledge Bases.
- Ingest FAQs, product docs, pricing rules, policies, and objection handling.
- Add metadata filters by product, region, and customer type.
- Add confidence threshold and fallback response.

Acceptance criteria:

- Agent answers common questions from approved knowledge.
- Low-confidence answers are escalated.

## Phase 5: Production Hardening

Goal: prepare for scale and reliability.

Tasks:

- Add idempotency.
- Add dead-letter workflow.
- Add replay process.
- Add monitoring dashboard.
- Add prompt versioning.
- Add compliance review.
- Add redaction.
- Add QA sampling.

Acceptance criteria:

- Failed executions are visible and recoverable.
- Duplicate notes/tasks are prevented.
- Sensitive data handling is documented.
- Operations team can monitor outcomes.

