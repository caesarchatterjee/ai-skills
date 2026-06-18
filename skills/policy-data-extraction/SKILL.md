# Policy Data Extraction

## When to use
Use this skill to extract structured data from insurance policy PDFs — declarations pages, schedules, endorsements, and coverage tables.

## Extraction Workflow

1. Read the PDF and identify document sections
2. Extract key-value pairs from the declarations page
3. Parse coverage tables into structured format
4. Validate extracted data against expected schema

## Step 1: Read and Section the PDF

```python
import pdfplumber

def extract_sections(pdf_path: str) -> dict[str, str]:
    sections = {}
    current_section = "header"
    current_text = []

    with pdfplumber.open(pdf_path) as pdf:
        for page in pdf.pages:
            text = page.extract_text() or ""
            for line in text.split("\n"):
                upper = line.strip().upper()
                if upper in KNOWN_SECTIONS:
                    if current_text:
                        sections[current_section] = "\n".join(current_text)
                    current_section = upper.lower()
                    current_text = []
                else:
                    current_text.append(line)

    if current_text:
        sections[current_section] = "\n".join(current_text)
    return sections

KNOWN_SECTIONS = [
    "DECLARATIONS", "SCHEDULE", "INSURING AGREEMENT",
    "EXCLUSIONS", "CONDITIONS", "ENDORSEMENTS", "DEFINITIONS",
]
```

## Step 2: Parse Declarations

Use an LLM with structured output to extract key fields:

```python
from pydantic import BaseModel
from typing import Optional

class PolicyDeclarations(BaseModel):
    policy_number: str
    insured_name: str
    effective_date: str
    expiration_date: str
    premium: Optional[float] = None
    deductible: Optional[float] = None
    coverage_limit: Optional[float] = None
    property_address: Optional[str] = None
    policy_type: str  # "homeowner", "auto", "commercial", "life"
```

## Step 3: Parse Coverage Tables

```python
def extract_tables(pdf_path: str) -> list[list[list[str]]]:
    all_tables = []
    with pdfplumber.open(pdf_path) as pdf:
        for page in pdf.pages:
            tables = page.extract_tables()
            all_tables.extend(tables)
    return all_tables
```

## Gotchas

- Zurich policy PDFs use varying layouts by country and product line. Do not hardcode column positions.
- Some PDFs are scanned images — check if `extract_text()` returns empty. Fall back to OCR (pytesseract + pdf2image).
- Currency values may be in CHF, EUR, USD, GBP, or JPY depending on the entity. Always extract the currency symbol.
- Endorsement amendments override base policy terms. Process endorsements AFTER the base policy and apply as patches.
- Policy numbers follow country-specific formats. Do not validate format globally.