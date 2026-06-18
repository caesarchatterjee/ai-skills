# Claims Processing Agent

## When to use
Use this skill when building an agent that automates insurance claims intake, from initial FNOL (First Notice of Loss) through triage and routing.

## Workflow

1. **Extract claim data** from the submission (email, form, call transcript)
2. **Validate coverage** by looking up the policy number against the policy database
3. **Assess complexity** based on claim amount, type, and fraud indicators
4. **Route** to the appropriate handler

## Implementation

```python
from pydantic import BaseModel

class ClaimData(BaseModel):
    policy_number: str
    claimant_name: str
    loss_date: str
    loss_type: str
    description: str
    estimated_amount: float

class TriageResult(BaseModel):
    priority: str  # "auto", "standard", "complex", "fraud_review"
    handler_queue: str
    reason: str
```

### Step 1: Parse the claim

Use structured output to extract claim fields from unstructured input. The LLM should identify the policy number, loss details, and estimated amount.

### Step 2: Validate coverage

```python
# Query the policy API
response = policy_api.get(f"/policies/{claim.policy_number}")
if response.status_code != 200:
    return TriageResult(priority="complex", handler_queue="manual_review",
                        reason="Policy not found")

policy = response.json()
if claim.loss_type not in policy["covered_perils"]:
    return TriageResult(priority="standard", handler_queue="coverage_review",
                        reason=f"Loss type '{claim.loss_type}' not in covered perils")
```

### Step 3: Triage rules

| Condition | Priority | Queue |
|-----------|----------|-------|
| Amount < CHF 5,000 | auto | auto_process |
| Amount < CHF 50,000 | standard | claims_adjuster |
| Amount >= CHF 50,000 | complex | senior_adjuster |
| Fraud score > 0.7 | complex | fraud_investigation |
| Multiple claims in 90 days | complex | fraud_investigation |

## Gotchas

- Policy numbers have format `ZUR-{country}-{8 digits}`. Validate the format before API lookup.
- The `loss_date` must be within the policy's active period. Backdated claims get routed to fraud review.
- Always log the full claim payload before routing — needed for audit trail.
- Motor claims require a separate vehicle identification step (VIN lookup).