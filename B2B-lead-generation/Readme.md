# B2B AI Lead Discovery (n8n Workflow)

An n8n workflow that finds possible B2B leads from public GitHub organizations, scores them, writes a personal first outreach email for each one with Google Gemini, and saves everything to Google Sheets for review.

> **Human in the loop:** This workflow never sends emails. Every lead is saved with `approval_status = "Needs review"` and `outreach_status = "Not sent"`.

---

## Features

- **Lead discovery**: searches the GitHub REST API for organizations matching a keyword query (default: `type:org AI automation`).
- **Profile enrichment**: fetches each organization's full public profile (name, bio, website, location, public repos).
- **Lead scoring**: a simple, readable score from 0 to 100 with a status label.
- **AI email drafts**: Google Gemini (`gemini-2.5-flash`) writes a short email (under 120 words) using only facts from the profile. The prompt tells it not to invent anything.
- **Safe parsing**: if the AI's JSON can't be read, the lead is still saved with a fallback draft and `Low` confidence.
- **No duplicates**: leads are added or updated in Google Sheets using `github_url` as the unique key.

---

## Workflow Architecture

```
Manual Trigger
   │
   ▼
Search GitHub Organizations   (GET /search/users?q=type:org AI automation)
   │
   ▼
Split Out                     (one item per organization)
   │
   ▼
Get Organization Profile      (GET /users/{login})
   │
   ▼
Normalize & Score             (clean up fields + score the lead)
   │
   ▼
Prepare AI Request            (build the prompt)
   │
   ▼
Gemini AI                     (gemini-2.5-flash, temperature 0.3)
   │
   ▼
Combine AI Draft              (read the JSON, merge it with the lead data)
   │
   ▼
Save Leads                    (Google Sheets: add or update by github_url)
```

---

## Lead Scoring

| Criteria                                                              | Points |
|-----------------------------------------------------------------------|--------|
| Name, bio or company matches AI / automation / software / SaaS / ML / tech keywords | +40 |
| Has a website                                                         | +20    |
| Bio is longer than 40 characters                                      | +20    |
| Has a location                                                        | +10    |
| Has at least 1 public repository                                      | +10    |

| Score   | Status                 |
|---------|------------------------|
| ≥ 70    | `Qualified for review` |
| 40 – 69 | `Review`               |
| < 40    | `Low priority`         |

---

## Output Columns (Google Sheets)

| Column            | Description                                       |
|-------------------|---------------------------------------------------|
| `github_url`      | Organization's GitHub URL (unique key)            |
| `company_name`    | Display name, or the login if there's no name     |
| `github_username` | GitHub login                                      |
| `website`         | Website from the profile                          |
| `description`     | Bio from the profile                              |
| `location`        | Location from the profile                         |
| `public_repos`    | Number of public repositories                     |
| `lead_score`      | Score from 0 to 100                               |
| `lead_status`     | Qualified for review / Review / Low priority      |
| `source`          | `GitHub REST API`                                 |
| `source_url`      | Link to the source profile                        |
| `ai_subject`      | Email subject written by the AI                   |
| `ai_email_draft`  | Email body written by the AI                      |
| `ai_reason`       | The AI's explanation of why the lead is relevant  |
| `ai_confidence`   | Low / Medium / High                               |
| `approval_status` | Starts as `Needs review`                          |
| `outreach_status` | Starts as `Not sent`                              |

---

## Prerequisites

- An [n8n](https://n8n.io) instance (Cloud or self-hosted)
- A **GitHub** personal access token (for the GitHub API credential)
- A **Google Gemini** API key ([Google AI Studio](https://aistudio.google.com/))
- A **Google Sheets** account (OAuth2 credential in n8n)

---

## Setup

1. **Import the workflow**
   In n8n, go to **Workflows → Import from File** and choose `workflow.json` from this repo.

2. **Create credentials in n8n**
   - `GitHub API`: used by *Search GitHub Organizations* and *Get Organization Profile*
   - `Google Gemini (PaLM) API`: used by *Gemini AI*
   - `Google Sheets OAuth2 API`: used by *Save Leads*

3. **Prepare the Google Sheet**
   Create a spreadsheet with a sheet called `Sheet1` whose header row matches the column names in [Output Columns](#output-columns-google-sheets). Then choose that spreadsheet in the **Save Leads** node.

4. **(Optional) Customize the search**
   In **Search GitHub Organizations**, change:
   - `q`: for example `type:org fintech`, `type:org SaaS location:India`
   - `per_page`: how many organizations to get per run (default `10`, max `100`)

5. **Run it**
   Click **Execute workflow**, then review the leads in your Google Sheet.

---

## Customization Ideas

- Replace the **Manual Trigger** with a **Schedule Trigger** to find leads automatically.
- Change the keyword regex and point values in **Normalize & Score** to fit your ideal customer.
- Change the prompt in **Prepare AI Request** to match your service and tone.
- Add an approval step (Slack, email, or a sheet filter) before sending emails with Gmail or Outlook.

---

## Responsible Use

- Only public GitHub profile data is used.
- The AI is told not to invent facts, recipients, or email addresses.
- Review every draft before you contact anyone, and follow anti-spam and data-protection laws (CAN-SPAM, GDPR, etc.).
- Respect [GitHub's API rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) and Terms of Service.

---

## Tech Stack

- [n8n](https://n8n.io): workflow automation
- [GitHub REST API](https://docs.github.com/en/rest): lead source
- [Google Gemini](https://ai.google.dev/): AI email drafts
- [Google Sheets](https://www.google.com/sheets/about/): lead storage

---
