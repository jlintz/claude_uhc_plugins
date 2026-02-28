---
name: submit-claim
description: Use this skill when the user asks to "submit a claim", "fill out medical claim form", "submit insurance claim", "process a superbill", or mentions submitting claims to UHC/United Healthcare. This automates filling out UHC's Direct Medical Reimbursement form.
version: 1.0.0
---

# UHC Direct Medical Reimbursement — Claim Submission

This skill automates filling out UHC's Direct Medical Reimbursement form at `https://memberforms.uhc.com/DirectMedicalReimbursement.html` using data extracted from a superbill or medical receipt.

## Prerequisites

- User must be logged in / have completed email verification on the form already
- Chrome browser open with Chrome DevTools MCP connected
- `config/member.json` populated with subscriber details

## When to Use

This skill activates when the user requests to:
- Submit a medical claim or insurance claim
- Fill out the UHC reimbursement form
- Process a superbill or medical receipt for insurance
- Submit an out-of-network claim

## Workflow

### Step 1: Get Superbill File

Use AskUserQuestion to ask the user for the superbill file path:

> "Please provide the path to your superbill or medical receipt (PDF or image)."

### Step 2: Read Config

Read the config file at `config/member.json` (relative to this plugin's root directory). This contains subscriber details needed for form filling.

### Step 3: Extract Data from Superbill

Read the provided file (PDF or image) and extract the following structured data:

**Provider Information:**
- Provider name (individual or practice)
- NPI number (10-digit National Provider Identifier)
- TIN/EIN (Tax Identification Number)

**Service Lines** (extract ALL lines — there may be multiple):
For each service line, extract:
- Date of service (MM/DD/YYYY)
- CPT/HCPCS procedure code (5 characters, e.g., "90837")
- ICD-10 diagnosis code(s) (e.g., "F41.1")
- Modifier codes if present (2-digit, e.g., "95" for telehealth)
- Units/quantity (default to 1 if not specified)
- Service description (e.g., "Psychotherapy, 60 min")
- Charge amount (dollar amount for this line)

**Determine Submission Type** by mapping the primary CPT code:

| CPT Code Range | UHC Submission Type |
|---|---|
| 90791-90899 | Mental Health |
| 97810-97814 | Acupuncture |
| 92002-92499 | Vision |
| 69700-69799 | Hearing Aid |
| 97140, 97010-97039 | Massage Therapy |
| E0100-E9999 | Durable Medical Equipment |

If the CPT code doesn't match any range above, ask the user to select the submission type from UHC's dropdown options.

### Step 4: Confirm Extracted Data

Present a summary table to the user and ask for confirmation before proceeding:

```
**Provider:** [name] (NPI: [npi])
**Submission Type:** [type]

| Date | CPT | ICD-10 | Description | Amount |
|------|-----|--------|-------------|--------|
| [date] | [cpt] | [icd10] | [desc] | $[amt] |

**Total:** $[total]
```

Use AskUserQuestion: "Does this look correct? Should I proceed to fill the form?"

Do NOT proceed to form filling until the user confirms.
