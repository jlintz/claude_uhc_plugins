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

### Step 5: Email Verification (User Action)

Inform the user:

> "Please navigate to https://memberforms.uhc.com/DirectMedicalReimbursement.html in your Chrome browser, complete the email verification step, and let me know when you're past it. I'll then fill out the rest of the form."

Wait for the user to confirm they've completed email verification before proceeding.

### Step 6: Fill Subscriber Information

Use Chrome DevTools MCP tools:

1. `take_snapshot` to see current form state
2. Identify the subscriber information fields
3. Use `fill_form` to fill:
   - **Member ID**: from config `subscriber.memberId`
   - **Date of Birth**: from config `subscriber.dateOfBirth`
   - **Group Number**: from config `subscriber.groupNumber`
   - **First Name**: from config `subscriber.firstName`
   - **Last Name**: from config `subscriber.lastName`
4. `take_snapshot` to verify fields were filled correctly
5. Click the "Next" or "Continue" button to advance

If any field fails validation, `take_screenshot` and report to user.

### Step 7: Fill Patient Information

1. `take_snapshot` to see form state
2. Use `fill_form` to fill:
   - **Relationship to Subscriber**: from config `defaults.patientRelationship` (typically "Subscriber")
   - **Patient First Name**: from config `subscriber.firstName`
   - **Patient Date of Birth**: from config `subscriber.dateOfBirth`
3. If the superbill shows a different patient name than the subscriber, ask the user to clarify who the patient is
4. `take_snapshot` to verify
5. Click "Next" to advance

### Step 8: Fill Payer Information

1. `take_snapshot` to see form state
2. Use `fill` to select:
   - **Other medical insurance**: "No" (from config `defaults.otherInsurance`)
3. Skip EOB attachment upload
4. `take_snapshot` to verify
5. Click "Next" to advance

### Step 9: Select Submission Type

1. `take_snapshot` to see form state
2. Use `fill` to select:
   - **Foreign/cruise ship services**: "No" (from config `defaults.foreignServices`)
3. Use `fill` to select the **Submission Type** from the dropdown, using the type determined in Step 3
4. `take_snapshot` to verify the correct type is selected
5. Click "Next" to advance

### Step 10: Fill Service Line Details

This is the most complex step. The form allows multiple service lines.

**For the first service line:**

1. `take_snapshot` to see the service detail fields
2. Use `fill_form` to fill:
   - **ICD-10 Diagnosis Code(s)**: from extracted data. Use "Add Diagnosis Code" if multiple codes.
   - **CPT/HCPCS Procedure Code**: from extracted data
   - **Modifier Code(s)**: from extracted data (if any). Use "Add Modifier" if multiple.
   - **Units/Quantity**: from extracted data
   - **Service Description**: from extracted data
   - **Date of Service**: from extracted data (MM/DD/YYYY format)
   - **Charge Amount**: from extracted data (numeric, no $ sign)
3. `take_snapshot` to verify all fields filled correctly

**For additional service lines (if multiple lines on the superbill):**

4. Click "Add Service Item" button
5. Repeat step 2-3 for each additional service line
6. Continue until all service lines from the superbill are entered

**After all service lines are entered:**

7. `take_snapshot` for a final review of all service lines
8. Click "Next" to advance

Important: The form supports up to 18-19 service items. If the superbill has more, inform the user they'll need to submit multiple claims.

### Step 11: Attachments (User Action)

Inform the user:

> "I've filled out all the claim details. Please now:
> 1. Upload your superbill/receipt as a proof of payment attachment
> 2. Check any applicable boxes (corrected claim, provider report, etc.)
> 3. Let me know when the attachment is uploaded so I can continue."

Wait for user confirmation before proceeding.

### Step 12: Review and Submit (User Action)

After the user confirms the attachment is uploaded:

1. Click "Next" to advance to the review/summary page
2. `take_snapshot` to capture the review summary
3. Present the summary to the user

Inform the user:

> "The form is ready for your review. Please:
> 1. Review all the information on the summary page
> 2. If anything needs correction, use the edit links on the form
> 3. Sign electronically and accept the terms
> 4. Click Submit when ready
>
> I will NOT click Submit — that's your action to take."

## Error Handling

Throughout the form filling process:

- **Field fill failure**: If `fill` or `fill_form` doesn't work on a field, `take_screenshot` to see the field state, then try alternative approaches (clicking first, using `type_text`, etc.). If still failing, report the specific field to the user for manual entry.
- **Validation errors**: If the form shows validation errors after clicking "Next", `take_snapshot` to identify which fields have errors, attempt to fix them, and retry.
- **Navigation issues**: If the form doesn't advance after clicking "Next", `take_snapshot` to check for blocking validation errors or popups.
- **Unexpected form state**: If the form doesn't match expected structure (UHC may update their form), `take_snapshot` and describe what you see to the user. Attempt to adapt, or ask user for guidance.

## Notes

- The form URL is: `https://memberforms.uhc.com/DirectMedicalReimbursement.html`
- Processing takes 10-15 business days after submission
- This skill does not guarantee reimbursement — it only automates form filling
- Always verify extracted data with the user before filling the form
- Keep the browser window open throughout the process
