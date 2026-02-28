# 🏥 Claude UHC Plugin

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that automates filling out UHC (United Healthcare) Direct Medical Reimbursement forms. Hand it a superbill PDF and watch it go! 🚀

## ✨ What It Does

1. 📄 **Reads your superbill** — extracts provider info, CPT codes, ICD-10 diagnoses, dates, and charges from a PDF or image
2. 🧠 **Maps submission type** — automatically determines the correct UHC category (Mental Health, Vision, Acupuncture, etc.)
3. 🌐 **Opens the form** — launches the UHC reimbursement form in your browser
4. ✍️ **Fills everything out** — subscriber info, patient info, payer details, provider lookup, service lines, and attachments
5. 👀 **Hands control back to you** — stops at the review page so you can verify and submit

> ⚠️ **Safety first**: The plugin never clicks Submit. You always have final review and control.

## 📋 Prerequisites

### 1. Chrome DevTools MCP Server

This plugin uses browser automation via the Chrome DevTools MCP server. Add it to your Claude Code config:

```bash
claude mcp add chrome-devtools --scope user -- npx chrome-devtools-mcp@latest
```

### 2. Chrome Browser

Have Google Chrome installed and running. The MCP server will connect to it for automation.

## 🛠️ Installation

```bash
claude plugin add https://github.com/jlintz/claude_uhc_plugins.git
```

## ⚙️ Configuration

Copy the example config and fill in your details:

```bash
cp plugins/claude-uhc-plugin/config/member.example.json plugins/claude-uhc-plugin/config/member.json
```

Edit `member.json` with your subscriber information:

```json
{
  "subscriber": {
    "memberId": "YOUR_MEMBER_ID",
    "groupNumber": "YOUR_GROUP_NUMBER",
    "firstName": "YOUR_FIRST_NAME",
    "lastName": "YOUR_LAST_NAME",
    "dateOfBirth": "MM/DD/YYYY"
  },
  "defaults": {
    "patientRelationship": "Subscriber",
    "otherInsurance": false,
    "foreignServices": false
  }
}
```

> 🔒 `member.json` is gitignored — your personal info stays local.

## 🚀 Usage

In Claude Code, trigger the skill and provide your superbill:

```
> /submit-claim ~/Downloads/superbill-feb-2026.pdf
```

Or use any of these natural language triggers:

- `submit a claim`
- `fill out medical claim form`
- `submit insurance claim`
- `process a superbill`

The plugin will extract the data, confirm it with you, then fill the entire form — stopping at the review page for you to verify and submit.

## 🩺 Supported Submission Types

| CPT Code Range | Submission Type            |
|----------------|----------------------------|
| 90791–90899    | 🧠 Mental Health           |
| 97810–97814    | 📍 Acupuncture             |
| 92002–92499    | 👁️ Vision                  |
| 69700–69799    | 👂 Hearing Aid             |
| 97140, 97010–97039 | 💆 Massage Therapy    |
| E0100–E9999    | 🦽 Durable Medical Equipment |

If the CPT code doesn't match a known range, the plugin will ask you to select the correct type.

## 📄 License

MIT — see [LICENSE](LICENSE) for details.
