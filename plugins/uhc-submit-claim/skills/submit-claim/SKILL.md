---
name: submit-claim
description: Use this skill when the user asks to "submit a claim", "fill out medical claim form", "submit insurance claim", "process a superbill", or mentions submitting claims to UHC/United Healthcare. This automates filling out UHC's Direct Medical Reimbursement form.
version: 1.7.0
---

# UHC Direct Medical Reimbursement — Claim Submission

This skill automates filling out UHC's Direct Medical Reimbursement form at `https://memberforms.uhc.com/DirectMedicalReimbursement.html` using data extracted from a superbill or medical receipt.

## Prerequisites

- Chrome browser open with Chrome DevTools MCP connected
- `config/member.json` populated with subscriber details including email address (will prompt to create if missing)

## When to Use

This skill activates when the user requests to:
- Submit a medical claim or insurance claim
- Fill out the UHC reimbursement form
- Process a superbill or medical receipt for insurance
- Submit an out-of-network claim

## Critical: The form lies about whether a field is filled

This is an AEM adaptive form. Every input has **two** states: the raw DOM `input.value`, and the framework's *bound* value that validation actually reads. `fill`/`fill_form` can set the first without committing the second. When that happens the field looks correct in a `take_snapshot` and on screen, but "Next" or "Search" silently does nothing — **no error text, no `aria-invalid` anywhere on the page**.

**The tell is the wrapper class.** `input.closest('.guideFieldNode').className` must contain `validation-success af-field-filled`. An uncommitted field stays `af-field-empty`, usually alongside `guideActiveField` (meaning it is still focused, so `blur` never fired).

**The most common cause: `fill_form` leaves its LAST field focused and uncommitted.** It blurs each field as it moves to the next, but nothing blurs the final one. In practice, expect to fix the last field of every batch — it is the default case, not the exception. Fields observed hitting this: Group number, provider search ZIP, provider manual-entry ZIP, Charge amount, Provider TIN, added-line Date of service.

**The fix** (works for every field — TIN, ZIP, dates, amounts) via `evaluate_script`:

```js
() => {
  const n = [...document.querySelectorAll('.guideFieldNode')].find(x => x.className.includes('FIELD_CLASS'));
  const el = n.querySelector('input');
  const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
  el.focus();
  setter.call(el, 'VALUE');
  ['input','change','blur'].forEach(t => el.dispatchEvent(new Event(t, { bubbles: true })));
  el.blur();
  return { val: el.value, cls: n.className };
}
```

**Sweep after every fill, before every "Next"** — filter to *visible* fields or you will drown in hidden nodes from form branches you never took (dependent info, other-insurance, other submission types, provider group):

```js
() => {
  const bad = [];
  document.querySelectorAll('.guideFieldNode').forEach(n => {
    const i = n.querySelector('input,select,textarea');
    if (!i || !i.offsetParent) return;            // visible only
    const c = n.className;
    if (/validation-failure/.test(c) || (/af-field-empty/.test(c) && !/validation-success/.test(c))) {
      bad.push({ label: (i.getAttribute('aria-label') || i.id).slice(0, 45), val: i.value, cls: c });
    }
  });
  return bad;
}
```

Optional fields left intentionally blank (Modifier, Address 2) show up as `af-field-empty` and are fine. Anything required and empty-classed must be fixed before advancing.

**The inverse trap — do not over-correct.** The DOM can also *under*-report a field that is genuinely fine. The Attachment tab's **Attached:** input reads `"0"` even after a successful upload, while the page visibly renders "Attached: 1". Rule of thumb: **wrapper class** is the truth for text/select bindings, **screenshot** is the truth for the upload widget, and raw `input.value` is authoritative for neither.

## Workflow

### Step 1: Get Superbill File

Use AskUserQuestion to ask the user for the superbill file path:

> "Please provide the path to your superbill or medical receipt (PDF or image)."

### Step 2: Read Config

Locate `config/member.json` within the plugin's **installed** directory. The installed plugin lives at `~/.claude/plugins/<marketplace>/<plugin>/` — for example `~/.claude/plugins/jlintz-uhc-marketplace/uhc-submit-claim/config/member.json`.

**IMPORTANT:** Do NOT read from the plugin cache directory (`~/.claude/plugins/cache/...`). The cache is read-only and may be stale. Always resolve paths under `~/.claude/plugins/<marketplace>/<plugin>/` (no `cache` segment).

**If the file is found:** read it and continue to Step 3.

**If the file is not found**, ask the user how they'd like to proceed with three options:

1. **Provide a path** — ask the user for the absolute path to their existing `member.json` file, then read it from that location.
2. **Create one now** — prompt the user for each required field:
   - Member ID
   - Group Number
   - First Name
   - Last Name
   - Date of Birth (MM/DD/YYYY)
   - Email address (used for form email verification)
   - Patient relationship to subscriber (default: Subscriber)
   - Other insurance? (default: no)
   - Foreign/cruise ship services? (default: no)

   Write the collected values to `~/.claude/plugins/<marketplace>/uhc-submit-claim/config/member.json` (the installed directory, NOT the cache), using the same format as `config/member.example.json`, then continue.
3. **Exit** — stop the skill and tell the user to populate `config/member.json` using `config/member.example.json` as a template before retrying.

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

### Step 5: Open Form and Email Verification

1. Use Chrome DevTools MCP `new_page` to open `https://memberforms.uhc.com/DirectMedicalReimbursement.html`
2. `take_snapshot`. The form opens on an **intro/landing screen**, not the email step. Click the **"Start new claim form"** button, then `take_snapshot` again to reveal the email verification form.
3. Use `fill` to enter the subscriber's email address from config `subscriber.email` into the email input field
4. `take_snapshot` to verify the email was entered correctly
5. Click the "Send Code" button to trigger the verification code email
6. Inform the user:

> "I've entered your email address and clicked Send Code. Please check your inbox for the verification code, enter it in the browser, and let me know when you're past the verification step."

7. Wait for the user to confirm they've completed email verification before proceeding.

### Step 6: Fill Subscriber Information

This tab has exactly **three** fields — there are no first/last name inputs here (the name is only used later, for the signature in Step 13).

1. `take_snapshot` to see current form state
2. Use `fill_form` to fill:
   - **Member ID**: from config `subscriber.memberId`
   - **Subscriber's date of birth**: from config `subscriber.dateOfBirth` (mm/dd/yyyy)
   - **Group number**: from config `subscriber.groupNumber`
3. Run the visible-field sweep (see **Critical: The form lies about whether a field is filled**). **Group number is the last field in this batch and routinely fails to commit** — expect to apply the native-setter fix to it.
4. Click "Next" to advance. This tab runs an async eligibility lookup, so it may stay on the Subscriber tab for a moment — `take_snapshot` again before assuming it failed.

If any field fails validation, `take_screenshot` and report to user.

### Step 7: Fill Patient Information

This step is a **single radio group** — "Please select who this submission is for" — with options **Subscriber**, **Spouse**, **Dependent**. There are no name/DOB fields when the patient is the subscriber.

1. `take_snapshot` to see form state
2. Click the radio matching `defaults.patientRelationship` (typically **Subscriber**). If the superbill shows a patient who is not the subscriber, ask the user whether it's the Spouse or a Dependent; selecting **Spouse**/**Dependent** reveals additional name/DOB fields to fill from the superbill.
3. `take_snapshot` to verify the selection
4. Click "Next" to advance

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
4. Selecting a submission type reveals a second dropdown, **"Where were the services rendered?"**. As of 2026-09-01 telehealth is **split into two options** (this dropdown previously offered a single "Telehealth"):

   | Option | Superbill POS |
   |---|---|
   | Telehealth – in the home | POS **10** |
   | Telehealth – not in the home | POS **02** |
   | Office | POS **11** |
   | Home | POS **12** |
   | Inpatient Hospital Doctor Bill | — |
   | Outpatient Hospital Doctor Bill | — |

   `take_snapshot`, then select the value implied by the superbill's POS code.

   > **Note:** the dash in the telehealth options is an **en dash (–), not a hyphen** — match the option value exactly or `fill` will not find it.

   POS 02 vs 10 is a real distinction (whether the patient was physically at home), and a mismatch against the attached superbill is something UHC can flag. The superbill's coded POS is authoritative, but it is worth one `AskUserQuestion` to confirm rather than assuming. If the superbill only says "telehealth" with no POS code, ask.
5. `take_snapshot` to verify both dropdowns are set
6. Click "Next" to advance

### Step 10: Fill Provider Information

Selecting a submission type adds a **Provider information** tab (this is where the NPI/TIN extracted in Step 3 are used).

1. `take_snapshot` to see the provider search fields.
2. Fill the search fields:
   - **Provider Tax Identification Number (TIN)** — see the TIN note below.
   - **Individual provider vs. Provider group** — select **Individual Provider** if the superbill names a person (e.g., "Jane Smith, PhD"); select **Provider Group** if it only lists an organization (e.g., "LabCorp").
   - **Last name / First name** (individual) or organization name (group)
   - **Zip code** — provider's zip from the superbill
3. Click **Search**.
   - **If a matching provider is returned**, select it and continue.
   - **If not found**, the form shows "please re-enter…". Click **Search** a second time with the same values. After the second failed search, a **manual-entry form** appears.
4. In the manual-entry form, `fill_form` the full provider details from the superbill: **Last name, First name, NPI (10 digits), Address 1, City, State (dropdown), ZIP code**. Leave Address 2 blank unless present.
5. Sweep for uncommitted fields, then click **Next**.

> **TIN field truncation bug (IMPORTANT):** The Provider TIN input silently truncates/mangles dashed or formatted input — `84-4240960` becomes `84-424096`. Do **not** type the dashed form and do **not** use `fill`. Set the raw **9 digits** (`844240960`) with the native-setter snippet from **Critical: The form lies about whether a field is filled** (field class `Provider_Tax_identifier`).

> **Both ZIP fields need the same fix.** The provider *search* ZIP (`providerLookupZipcode`) and the manual-entry ZIP (`Provider_Zipcode`) are each the last field in their `fill_form` batch, so both routinely land `af-field-empty` and block **Search**/**Next** with no error. Fix them before clicking.

### Step 11: Fill Service Line Details

This is the most complex step (the **Submission details** tab). The form allows multiple service lines.

**For the first service line:**

1. `take_snapshot` to see the service detail fields
2. Use `fill_form` to fill:
   - **ICD-10 Diagnosis Code(s)**: from extracted data. Use "+ Add a Diagnosis Code" if multiple codes.
   - **CPT/HCPCS Procedure Code**: from extracted data
   - **Modifier Code(s)**: from extracted data (if any). Use "Add modifier" if multiple.
   - **Units/Quantity**: from extracted data
   - **Date of Service**: from extracted data (MM/DD/YYYY format)
   - **Charge Amount**: from extracted data (numeric, e.g. `250.00`; the form reformats it to `$250.00`)
   - Note: the first line has no separate "Service Description" field.
3. Run the visible-field sweep. **Charge amount (`phyTotalServiceAmtBh`) is the last field in this batch and routinely fails to commit** — when it does, the read-only **Total charge amount** also stays blank, which is the quickest signal. Apply the native-setter fix; a correct commit shows `$250.00` in *both* the charge field and the Total.

**For additional service lines (if multiple lines on the superbill):**

4. Click **"+ Add a Service Item"**. The new line first asks: **"Is the new service item for the same Procedure code, Charge amount and Units?"**
   - If the new line shares CPT + charge + units with the first (common for recurring therapy — only the date differs), select **Yes**. This pre-fills CPT/units/charge; you then only need to enter the **date of service** and re-enter the **Modifier** (the modifier does **not** carry over).
   - Otherwise select **No** and fill all fields as in step 2.
5. Repeat for each additional service line.

**Date-of-service picker bug (IMPORTANT) for added service lines:**

Each added line renders **three** date inputs, and only one of them gates validation. Setting the wrong one makes the visible date picker *display* the date correctly while the form still treats the field as empty — "Next" then silently fails with no error message and no `aria-invalid` anywhere on the page.

For an added line with generated id prefix `GUID<n>__`:

| Input | id suffix | Type | Role |
|---|---|---|---|
| decorative picker | `...guidedatepicker_copy___widget` | `date` | drives the visible spinbuttons — **not** what validates |
| **bound field** | `...guidedatepicker___widget` | `text` | **this is what validation reads** |

(The first service line's own `...guidedatepicker___widget` has no `GUID` prefix and accepts a normal `fill` with mm/dd/yyyy — only added lines need the workaround.)

Set the GUID-prefixed **text** input via `evaluate_script` in **mm/dd/yyyy** format:
```js
() => {
  const el = document.getElementById('GUID..._..._guidedatepicker___widget'); // type=text, NOT _copy_
  const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
  el.focus();
  setter.call(el, '07/29/2026'); // mm/dd/yyyy
  ['input','change','blur'].forEach(t => el.dispatchEvent(new Event(t, { bubbles: true })));
  return { value: el.value, wrapCls: el.closest('.guideFieldNode').className };
}
```

Locate the right input by listing visible text inputs whose `aria-label` matches `/date of service/i` and picking the empty one:
```js
() => {
  const out = [];
  document.querySelectorAll('input').forEach(i => {
    const al = i.getAttribute('aria-label') || '';
    if (/date of service/i.test(al) && i.offsetParent && i.type === 'text' && !i.value) {
      out.push({ id: i.id, wrapCls: i.closest('.guideFieldNode').className });
    }
  });
  return out;
}
```

**Verify by wrapper class, not by the spinbuttons.** `el.closest('.guideFieldNode').className` must flip to `validation-success af-field-filled`; an unset bound field stays `af-field-empty` (and shows `validation-failure` once touched). The spinbuttons read the decorative picker and will show a date even when the bound field is empty — do not trust them or a `take_snapshot` of them as confirmation.

**After all service lines are entered:**

6. `take_snapshot` and confirm the read-only **Total charge amount** matches the superbill total.
7. Click "Next" to advance. If it does not advance, re-check the added-line date pickers per the bug note above.

Important: The form supports up to 18-19 service items. If the superbill has more, inform the user they'll need to submit multiple claims.

### Step 12: Attachments (Automated Upload)

The superbill provided in Step 1 is uploaded automatically as the proof-of-payment attachment.

1. `take_snapshot` to locate the **"Upload attachment"** button.
2. Use the Chrome DevTools MCP `upload_file` tool with:
   - `uid`: the "Upload attachment" button (it opens the file chooser; `upload_file` handles the chooser without a blocking OS dialog)
   - `filePath`: the superbill path from Step 1
3. **`take_screenshot`** (not `take_snapshot`) and confirm the page shows a green check, the file name from Step 1, **"Upload complete."**, and **Attached: 1**.

   > **Do not verify this one from the DOM.** The **Attached:** input's `value` stays `"0"` after a successful upload while the page renders "Attached: 1". Reading `input.value` here will make a working upload look failed and send you into a pointless retry loop. Trust the screenshot.
4. Leave the "check all that apply" boxes (Proof of timely filing, Corrected Claims, Provider reports or records) **unchecked** by default — a standard first submission needs none. Only check one if the user explicitly asked for it.
5. Click "Next" to advance.

If `upload_file` fails (e.g. tool error or the count stays 0), fall back to asking the user to upload the file manually and confirm when done, then continue.

### Step 13: Review, Sign, and Hand Off (Do NOT Submit)

1. Click "Next" to advance to the review/summary page.
2. `take_snapshot` to capture the review summary and present it to the user.
3. **Fill the signature** with the subscriber's full name (`subscriber.firstName` + " " + `subscriber.lastName`). The signature field is guarded, so this requires enabling it first:
   - The **agreement checkbox** ("I agree to use electronic records and signatures") is `disabled` until the Terms & Conditions disclosure has been scrolled through. Scroll the disclosure container to the bottom (e.g. `evaluate_script` setting the scrollable terms element's `scrollTop = scrollHeight`, or scrolling the page to the checkbox), then `take_snapshot` to confirm the checkbox is now enabled.
   - Click the agreement checkbox to check it.
   - Fill the **"Submitter signature"** field (`Signature`) with the full name. Observed 2026-09-01: this input is **not** `readonly` once the checkbox is checked, but `fill` still may not commit the bound value — the native-setter snippet is the reliable path. The **Submission date** is prefilled by the form (`readonly`) — leave it.
   - `take_snapshot` to confirm the signature shows the full name and the checkbox is checked.
   - If the field still cannot be populated after these attempts, stop and ask the user to type their signature manually (do not keep retrying).
4. **Do NOT click Submit.** Hand off to the user:

> "The form is filled, signed with your name, the superbill is attached, and the terms are accepted. Please review everything on the summary page (use the Edit links for any changes) and click **Submit** when you're ready.
>
> I will NOT click Submit — that final step is yours."

## Error Handling

Throughout the form filling process:

- **Field fill failure**: If `fill` or `fill_form` doesn't work on a field, `take_screenshot` to see the field state, then try alternative approaches (clicking first, using `type_text`, etc.). If still failing, report the specific field to the user for manual entry.
- **Fields whose bound value doesn't commit**: This is the single most common failure on this form and it is **not limited to specific fields** — see **Critical: The form lies about whether a field is filled** for the mechanism, the sweep, and the fix. First suspects, in order: the **last field of whatever `fill_form` batch you just ran**, then the Provider TIN, then added-line dates of service.
- **"Next" doesn't advance with no visible error**: Almost always the above. Note that `aria-invalid` and on-page error text are usually **absent** in this state, so never conclude the form is valid from their absence. Run the visible-field sweep, fix what it returns, confirm each flips to `validation-success af-field-filled`, then retry.
- **A form control has more options than this skill documents**: UHC revises these dropdowns (the telehealth split in Step 9 landed this way). Read the actual options out of the snapshot rather than assuming the documented list, select by exact option value including punctuation, and tell the user the skill is out of date so it can be updated.
- **Validation errors**: If the form shows validation errors after clicking "Next", `take_snapshot` to identify which fields have errors, attempt to fix them, and retry.
- **Navigation issues**: If the form doesn't advance after clicking "Next", `take_snapshot` to check for blocking validation errors or popups.
- **Unexpected form state**: If the form doesn't match expected structure (UHC may update their form), `take_snapshot` and describe what you see to the user. Attempt to adapt, or ask user for guidance.

## Notes

- The form URL is: `https://memberforms.uhc.com/DirectMedicalReimbursement.html`
- Processing takes 10-15 business days after submission
- This skill does not guarantee reimbursement — it only automates form filling
- Always verify extracted data with the user before filling the form
- Keep the browser window open throughout the process
- Never click **Submit** — filling, attaching, and signing are automated; the final submission is always the user's action
