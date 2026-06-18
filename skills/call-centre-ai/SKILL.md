# Call Centre AI

## When to use
Use this skill when building AI capabilities for Zurich's call centres — automated call handling, real-time agent assist, post-call summarization, or intent detection.

## Architecture Overview

```
Caller → Telephony (Genesys/Avaya) → Google CCAI → Agent Assist / Virtual Agent
                                         ↓
                                    Speech-to-Text
                                         ↓
                                   Intent Detection
                                         ↓
                              ┌──────────┴──────────┐
                              │                     │
                         Virtual Agent          Agent Assist
                         (self-service)     (overlay for human agent)
```

## Intent Detection

Define intents for common call types:

| Intent | Example Utterances | Action |
|--------|-------------------|--------|
| `claim.status` | "Where is my claim?", "Claim update" | Look up claim, return status |
| `claim.new` | "I need to file a claim", "I had an accident" | Start FNOL flow |
| `policy.question` | "What does my policy cover?" | RAG over policy document |
| `billing.payment` | "I want to pay my bill" | Transfer to payment IVR |
| `escalation.agent` | "I want to talk to a person" | Immediate transfer |

## Real-Time Agent Assist

Provide agents with:

1. **Live transcript** — STT output displayed in real-time
2. **Suggested responses** — LLM generates response suggestions based on conversation context
3. **Knowledge cards** — Relevant policy/procedure documents surfaced automatically
4. **Compliance alerts** — Flag when agent or customer mentions regulated topics

```python
async def generate_suggestion(transcript: list[dict], customer_context: dict) -> str:
    messages = [
        {"role": "system", "content": AGENT_ASSIST_PROMPT},
        {"role": "user", "content": format_transcript(transcript)},
    ]
    response = await client.chat.completions.create(
        model="eu.anthropic.claude-sonnet-4-6",
        messages=messages,
        max_tokens=300,
    )
    return response.choices[0].message.content
```

## Post-Call Summarization

Generate structured call summaries:

```python
class CallSummary(BaseModel):
    caller_intent: str
    resolution: str
    follow_up_required: bool
    follow_up_action: str | None = None
    sentiment: str  # positive, neutral, negative
    compliance_flags: list[str] = []
    duration_seconds: int
```

## Gotchas

- Google CCAI supports 30+ languages but Zurich primarily needs EN, DE, FR, IT, ES, PT. Configure language detection for the region.
- Call recordings must be stored according to local regulations (some jurisdictions require explicit consent).
- Agent assist suggestions must never include coverage confirmations or claim approvals — those require human judgment.
- Silence detection: a pause > 3 seconds may indicate the caller is waiting. Don't interpret it as end-of-utterance.
- Transfer warm vs. cold: always attempt warm transfer with context summary. Cold transfers lose conversation context.