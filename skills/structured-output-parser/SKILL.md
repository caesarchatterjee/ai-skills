# Structured Output Parser

## When to use
Use this skill when you need an LLM to return structured, validated data instead of free-form text. The parser sends the schema to the LLM, validates the response, and retries on failure.

## Implementation

```python
import json
from typing import TypeVar
from pydantic import BaseModel, ValidationError
from openai import OpenAI

T = TypeVar("T", bound=BaseModel)

def parse_structured(
    prompt: str,
    schema: type[T],
    model: str = "eu.anthropic.claude-sonnet-4-6",
    max_retries: int = 2,
) -> T:
    client = OpenAI()
    schema_json = json.dumps(schema.model_json_schema(), indent=2)

    messages = [
        {"role": "system", "content": f"Respond with valid JSON matching this schema:\n{schema_json}"},
        {"role": "user", "content": prompt},
    ]

    for attempt in range(max_retries + 1):
        response = client.chat.completions.create(
            model=model, messages=messages, max_tokens=2000,
        )
        raw = response.choices[0].message.content.strip()

        # Strip markdown code fences if present
        if raw.startswith("```"):
            raw = raw.split("\n", 1)[1].rsplit("```", 1)[0].strip()

        try:
            return schema.model_validate_json(raw)
        except (ValidationError, json.JSONDecodeError) as e:
            if attempt == max_retries:
                raise
            messages.append({"role": "assistant", "content": raw})
            messages.append({"role": "user", "content": f"Invalid: {e}. Return valid JSON only."})
```

## Usage Example

```python
class SentimentResult(BaseModel):
    sentiment: str  # "positive", "negative", "neutral"
    confidence: float
    key_phrases: list[str]

result = parse_structured(
    prompt="Analyze sentiment: 'The claim was resolved quickly, very happy with the service'",
    schema=SentimentResult,
)
print(result.sentiment)  # "positive"
print(result.confidence)  # 0.95
```

## When to Use This vs. Native Structured Output

- **Use this skill** when working through a proxy (LiteLLM) or when you need custom retry logic
- **Use native structured output** (Anthropic's tool_use or OpenAI's response_format) when available and the schema is simple

## Gotchas

- Some models wrap JSON in markdown code fences (```` ```json ... ``` ````) — the parser handles this.
- Nested Pydantic models work but increase the chance of validation failure. Keep schemas flat when possible.
- The retry loop appends the failed attempt to the conversation, giving the model context to self-correct.
- Set `max_retries=0` for latency-sensitive paths and handle the error upstream.