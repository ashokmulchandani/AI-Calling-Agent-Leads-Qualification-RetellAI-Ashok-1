# AI Calling Agent — Leads Qualification with Retell AI

An AI voice agent that handles inbound calls, qualifies leads, updates HubSpot CRM, creates follow-up tasks, and performs outbound callbacks — orchestrated by n8n Cloud.

## Confirmed Stack

| Layer | Tool |
|-------|------|
| Telephony | Aircall |
| Voice AI | Retell AI |
| Orchestration | n8n Cloud |
| CRM | HubSpot |
| Post-Call AI | Azure OpenAI (GPT-4o-mini) |
| Knowledge Base | Pinecone or Supabase pgvector |
| Notifications | Telegram |
| Storage | Supabase or AWS S3 |

## Architecture

```
Inbound Caller
  → Aircall (SIP trunk / forward)
  → Retell AI (real-time STT + dialogue + TTS)
      → call.started → n8n: Inbound Context → HubSpot lookup → dynamic variables back to agent
      → call.ended  → n8n: Post-Call Processing → Azure OpenAI → HubSpot note + task → Telegram
  
Outbound:
  HubSpot trigger / schedule
  → n8n: Outbound Callback (consent + DNC + hours check)
  → Retell AI POST /v2/create-phone-call
  → same post-call pipeline
```

## Workflows

| Workflow | File | Webhook Path |
|----------|------|-------------|
| Inbound Context | `n8n/workflows/inbound-call-context.workflow.json` | `POST /webhook/ai-voice/inbound-context` |
| Post-Call Processing | `n8n/workflows/post-call-processing.workflow.json` | `POST /webhook/ai-voice/post-call` |
| Outbound Callback | `n8n/workflows/outbound-callback.workflow.json` | Manual / HubSpot / Schedule |

## Files

- [docs/ai-voice-agent-design.md](docs/ai-voice-agent-design.md) — business + technical design
- [docs/architecture.md](docs/architecture.md) — system architecture and data flows
- [docs/implementation-plan.md](docs/implementation-plan.md) — phased rollout plan
- [config/environment.template.md](config/environment.template.md) — credentials checklist
- [n8n/prompts/voice-agent-system-prompt.md](n8n/prompts/voice-agent-system-prompt.md) — Retell AI agent prompt
- [n8n/prompts/post-call-analysis-prompt.md](n8n/prompts/post-call-analysis-prompt.md) — Azure OpenAI extraction prompt
- [n8n/schemas/call-analysis.schema.json](n8n/schemas/call-analysis.schema.json) — post-call JSON schema

## Implementation Order

1. ✅ Stack confirmed — Aircall + Retell AI + n8n + HubSpot + Azure OpenAI
2. Configure Retell AI inbound + outbound agents, set webhook URLs to n8n
3. Configure Aircall SIP trunk / forwarding to Retell AI number
4. Import 3 workflow JSONs into n8n, replace credential IDs
5. Test inbound call end-to-end — verify HubSpot note + task + Telegram
6. Test outbound callback — verify AI calls back eligible contacts
7. Add knowledge base (Pinecone / Supabase) for grounded answers
8. Add error handling, idempotency, DLQ, monitoring

## License

Private — internal use.
