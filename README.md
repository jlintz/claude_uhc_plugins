# 🏥 UHC Plugins Marketplace

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin marketplace for automating United Healthcare tasks.

## 🔌 Available Plugins

| Plugin | Description |
|--------|-------------|
| **uhc-submit-claim** | Automates filling out UHC Direct Medical Reimbursement forms using superbill data extraction and Chrome DevTools browser automation |

## 📋 Prerequisites

### 1. Chrome DevTools MCP Server

The plugins use browser automation via the Chrome DevTools MCP server. Add it to your Claude Code config:

```bash
claude mcp add chrome-devtools --scope user -- npx chrome-devtools-mcp@latest
```

### 2. Chrome Browser

Have Google Chrome installed and running. The MCP server will connect to it for automation.

## 🛠️ Installation

Add the marketplace and install a plugin:

```bash
claude plugin marketplace add https://github.com/jlintz/claude_uhc_plugins.git
claude plugin install uhc-submit-claim
```

## ⚙️ Configuration

Copy the example config and fill in your subscriber details:

```bash
cp plugins/uhc-submit-claim/config/member.example.json plugins/uhc-submit-claim/config/member.json
```

Edit `member.json` with your information:

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

---

## 🏥 uhc-submit-claim

Reads a superbill PDF/image, extracts provider info, CPT codes, ICD-10 diagnoses, dates, and charges, then fills out UHC's Direct Medical Reimbursement form via browser automation. Stops at the review page so you always have final control.

> ⚠️ **Safety first**: The plugin never clicks Submit. You always review and submit yourself.

### 🚀 Usage

In Claude Code, trigger the skill and provide your superbill:

```
> /submit-claim ~/Downloads/superbill-feb-2026.pdf
```

Or use any of these natural language triggers:

- `submit a claim`
- `fill out medical claim form`
- `submit insurance claim`
- `process a superbill`

### 🩺 Supported Submission Types

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
