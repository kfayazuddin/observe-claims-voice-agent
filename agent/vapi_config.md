# Vapi configuration

These settings are stored in the Vapi dashboard and are recorded here so the build is reproducible.

## Assistant: "Ava - Observe Claims"

| Setting | Value |
|---|---|
| Model | OpenAI gpt-4o-mini (Vapi cluster), low temperature (about 0.3) |
| First message | "Thank you for calling Observe Insurance, this is Ava. How can I help you today?" |
| First message mode | Assistant speaks first |
| System prompt | `agent/system_prompt.md` |
| Voice | A calm, natural-sounding voice from Vapi's voice list |
| Transcriber | Deepgram, newest model, English |
| Endpointing | Extra wait after numbers (about 1.5–2 s) so callers can pause between digit groups |
| Server URL | `https://worksp.app.n8n.cloud/webhook/end-of-call` |
| Server messages | `end-of-call-report` (other event types are ignored by the workflow) |

## Tools

| Tool | Type | Target |
|---|---|---|
| `lookup_customer` | Function | POST `https://worksp.app.n8n.cloud/webhook/lookup-customer` |
| `verify_and_get_claim` | Function | POST `https://worksp.app.n8n.cloud/webhook/verify-claim` |
| `faq_lookup` | Query (knowledge base) | Vapi file upload of `data/faq.md` |
| `end_call` | End call | Built-in |
| `transfer_to_representative` | Transfer call / Function | Live transfer needs a phone-call leg; see README "Known limitations" and `n8n/workflow_escalate.json` for the mock |

### lookup_customer

Description: Looks up a customer account by the phone number they give. Call this after the caller has confirmed their phone number. Returns whether an account was found and the customer's first name. Does not return claim details.

```json
{
  "type": "object",
  "properties": {
    "phone": { "type": "string", "description": "The caller's phone number, digits only, including area code" }
  },
  "required": ["phone"]
}
```

### verify_and_get_claim

Description: Verifies the caller's identity using the ZIP code on their policy, and returns their claim status only if the ZIP matches. Call this after lookup_customer returns found true and the caller has given their ZIP code.

```json
{
  "type": "object",
  "properties": {
    "phone": { "type": "string", "description": "The caller's confirmed phone number" },
    "zip_code": { "type": "string", "description": "The 5-digit ZIP code on the caller's policy" }
  },
  "required": ["phone", "zip_code"]
}
```

### faq_lookup

Description: Searches the Observe Insurance FAQ. Use for office hours, mailing address, how to start a new claim, how to submit documents, and the general claims process.

## Post-call analysis (Structured Output `call_record`)

Attached to the assistant, extracted from the transcript after each call and delivered in the end-of-call report under `message.artifact.structuredOutputs`.

```json
{
  "type": "object",
  "properties": {
    "caller_name": { "type": "string", "description": "The caller's full name if the assistant stated it after verification, otherwise 'Unknown'" },
    "phone": { "type": "string", "description": "The phone number the caller gave, in +1XXXXXXXXXX format, or 'Unknown'" },
    "sentiment": { "type": "string", "enum": ["positive", "neutral", "negative"], "description": "The caller's overall sentiment" },
    "outcome": { "type": "string", "enum": ["resolved", "escalated", "auth_failed", "not_found", "emergency", "info_only"], "description": "How the call ended" },
    "summary": { "type": "string", "description": "A 1 to 2 sentence summary of the call: what the caller wanted and how it ended. Do not include the ZIP code." }
  }
}
```
