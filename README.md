# ⚙️ HireIQ Automation — n8n Workflow Engine

![n8n](https://img.shields.io/badge/Built%20With-n8n-ef4136?style=for-the-badge&logo=n8n&logoColor=white)
![OpenAI](https://img.shields.io/badge/AI-OpenAI%20GPT--4o-412991?style=for-the-badge&logo=openai&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Database-Google%20Sheets-34a853?style=for-the-badge&logo=googlesheets&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-22c55e?style=for-the-badge)

> The backend automation engine powering HireIQ — an AI-powered resume screening tool. This repo contains the complete n8n workflow, setup documentation, prompt engineering templates, and API reference.

🌐 **Frontend Repo:** [hireiq-frontend](https://github.com/yourusername/hireiq-smart-candidate-screening)
🔗 **Live App:** [hireiq-smart-screen.lovable.app](https://hireiq-smart-screen.lovable.app)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Architecture](#workflow-architecture)
- [Tech Stack](#tech-stack)
- [Quick Start](#quick-start)
- [Workflow Nodes Explained](#workflow-nodes-explained)
- [AI Prompt Engineering](#ai-prompt-engineering)
- [API Reference](#api-reference)
- [Google Sheets Schema](#google-sheets-schema)
- [Security](#security)
- [Environment Variables](#environment-variables)
- [Troubleshooting](#troubleshooting)

---

## 📌 Overview

This repository contains the n8n automation workflow that powers the HireIQ resume screening engine. When a recruiter uploads resumes and a job description through the frontend, this workflow:

1. Receives the files via webhook
2. Enforces rate limiting per IP
3. Extracts text from each PDF
4. Sends each resume to OpenAI for AI scoring
5. Ranks and sorts all candidates
6. Logs results to Google Sheets
7. Returns structured JSON to the frontend

---

## 🏗️ Workflow Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    HireIQ Frontend                           │
│         POST /webhook/screen-resumes                        │
│         multipart/form-data                                 │
│         { job_description, resumes0, resumes1... }          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  1. WEBHOOK NODE                                            │
│     Receives POST request with binary PDF files             │
│     Field Name for Binary Data: data                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  2. CODE NODE — Rate Limiter                                │
│     Tracks requests per IP using workflow static data       │
│     Max: 10 requests per 60 minutes per IP                  │
│     Preserves binary data for downstream nodes              │
└──────────────┬───────────────────────────┬──────────────────┘
               │ Under limit               │ Over limit
               ▼                           ▼
┌──────────────────────┐      ┌────────────────────────────┐
│  3. IF NODE          │      │  Respond to Webhook        │
│     Routes request   │      │  429 Too Many Requests     │
│     based on count   │      └────────────────────────────┘
└──────────┬───────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│  4. CODE NODE — PDF Splitter                                │
│     Splits resumes0, resumes1... into individual items      │
│     Renames each binary field to 'data'                     │
│     Preserves job_description in each item                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  5. EXTRACT FROM FILE NODE                                  │
│     Operation: Extract From PDF                             │
│     Input Binary Field: data                                │
│     Outputs: text (raw resume content)                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  6. EDIT FIELDS NODE                                        │
│     Maps: resume_text → $json.text                         │
│     Maps: job_description → Webhook body                   │
│     Maps: file_name → $json.original_filename              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  7. LOOP OVER ITEMS                                         │
│     Batch Size: 1                                           │
│     Processes one resume at a time through AI               │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  8. MESSAGE A MODEL (OpenAI)                                │
│     Model: GPT-4o                                           │
│     Structured JSON prompt                                  │
│     Returns scored candidate object                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  9. CODE NODE — Parser + Ranker                             │
│     Parses OpenAI JSON response                             │
│     Sorts candidates by score descending                    │
│     Assigns rank and tier to each candidate                 │
│     Flattens arrays to comma-separated strings              │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  10. GOOGLE SHEETS           │
│      Append Row              │
│      Sheet: Rankings         │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  11. CODE NODE               │
│      Batch Summary Builder   │
│      Calculates session stats│
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  12. GOOGLE SHEETS           │
│      Append Row              │
│      Sheet: Batch Summary    │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  13. CODE NODE               │
│      Final Response Builder  │
│      Combines candidates +   │
│      summary into one object │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  14. RESPOND TO WEBHOOK      │
│      Returns JSON to         │
│      HireIQ frontend         │
└──────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation engine |
| **OpenAI GPT-4o** | Resume scoring and analysis |
| **Google Sheets** | Results storage and logging |
| **Webhook Node** | REST API endpoint |
| **PDF Extraction** | Parse resume text from PDFs |

---

## 🚀 Quick Start

### Prerequisites

- n8n cloud account or self-hosted instance (v2.8+)
- OpenAI API key with GPT-4o access
- Google account with Sheets API enabled
- HireIQ frontend deployed

### Step 1 — Import the Workflow

1. Go to your n8n instance
2. Click **Workflows** → **Import from file**
3. Upload `workflows/hireiq-resume-screener.json`
4. Click **Import**

> ⚠️ After importing the workflow, replace all
> `YOUR_GOOGLE_SHEET_ID_HERE` values with your
> actual Google Sheet ID, and reconnect your
> OpenAI and Google Sheets credentials in n8n.

### Step 2 — Configure Credentials

#### OpenAI

1. In n8n go to **Credentials** → **Add**
2. Select **OpenAI**
3. Paste your API key
4. Save as `OpenAI HireIQ`

#### Google Sheets

1. In n8n go to **Credentials** → **Add**
2. Select **Google Sheets OAuth2**
3. Follow the OAuth flow
4. Grant access to your Google Sheets
5. Save as `Google Sheets HireIQ`

### Step 3 — Create Google Sheet

Create a new Google Sheet with two tabs:

**Tab 1: Rankings**

```
Rank | Candidate Name | Score | Tier | Recommendation | 
Summary | Strengths | Weaknesses | Skills Found | 
Skills Missing | Experience Match | Red Flags | Processed At
```

**Tab 2: Batch Summary**

```
Job Title | Total Candidates | Fast Track | Review | 
Rejected | Average Score | Top Candidate | Run Date
```

### Step 4 — Update Node Settings

- In the **Google Sheets** nodes, select your Sheet ID
- In the **Webhook** node, copy the **Production URL**

### Step 5 — Activate the Workflow

Toggle the workflow to **Active** in the top right corner

### Step 6 — Connect to Frontend

Add the webhook URL to your frontend environment:

```
VITE_N8N_WEBHOOK_URL=https://your-instance.app.n8n.cloud/webhook/screen-resumes
```

---

## 🔍 Workflow Nodes Explained

### Rate Limiter (Code Node)

Tracks requests per IP address using n8n's workflow static data. Prevents abuse by limiting each IP to 10 requests per 60-minute window.

```javascript
const staticData = $getWorkflowStaticData('global');
const ip = $json.headers['x-forwarded-for'] || 'unknown';
// tracks timestamps per IP and filters within time window
```

### PDF Splitter (Code Node)

When multiple resumes are uploaded, they arrive as `resumes0`, `resumes1`, `resumes2` etc. This node splits them into individual items each with a standardised `data` binary field.

```javascript
const binaryKeys = Object.keys(firstItem.binary || {})
  .filter(key => key.startsWith('resumes'));
return binaryKeys.map(key => ({
  json: { job_description, original_filename },
  binary: { data: firstItem.binary[key] }
}));
```

### Parser + Ranker (Code Node)

Parses the raw OpenAI text response into structured JSON, sorts by score, assigns ranks and tiers.

```javascript
// Tier assignment
score >= 75 ? '🟢 Fast Track' 
: score >= 50 ? '🟡 Review' 
: '🔴 Reject'
```

---

## 🧠 AI Prompt Engineering

The core prompt sent to GPT-4o for each resume:

```
You are a senior technical recruiter with 10+ years of experience 
evaluating candidates across tech, data, and operations roles.

You will be given a job description and a candidate's resume. 
Your task is to objectively score and evaluate the candidate's fit.

---
JOB DESCRIPTION:
{{ $json.job_description }}

---
CANDIDATE RESUME:
{{ $json.resume_text }}

---
Respond ONLY with a valid JSON object in this exact format:

{
  "candidate_name": "extracted from resume or 'Unknown'",
  "score": <integer 0-100>,
  "recommendation": "<Strong Hire | Lean Hire | Maybe | Lean Reject | Reject>",
  "summary": "<2-3 sentence overall assessment>",
  "strengths": ["<strength 1>", "<strength 2>", "<strength 3>"],
  "weaknesses": ["<weakness 1>", "<weakness 2>", "<weakness 3>"],
  "skill_match": {
    "required_skills_found": ["<skill1>", "<skill2>"],
    "required_skills_missing": ["<skill1>", "<skill2>"],
    "bonus_skills_found": ["<skill1>"]
  },
  "experience_match": "<Exceeds | Meets | Partially Meets | Does Not Meet>",
  "culture_fit_notes": "<brief note on soft skills and trajectory>",
  "red_flags": ["<concern>"] or []
}
```

### Prompt Design Decisions

- **Role priming** — "senior technical recruiter" improves scoring accuracy
- **Structured JSON output** — eliminates parsing errors
- **5-tier recommendation** — more nuanced than pass/fail
- **Skill match breakdown** — separates found vs missing skills
- **Red flags array** — empty array if none, avoids null issues

---

## 📡 API Reference

### POST /webhook/screen-resumes

Accepts multipart/form-data with PDF resumes and job description.

**Request**

```
Content-Type: multipart/form-data

Fields:
  job_description  (string)   — Full job description text
  resumes0         (file)     — First PDF resume
  resumes1         (file)     — Second PDF resume (optional)
  resumes2         (file)     — Third PDF resume (optional)
  ...
```

**Response 200**

```json
{
  "success": true,
  "total": 2,
  "fast_track": 1,
  "review": 0,
  "rejected": 1,
  "average_score": 53,
  "top_candidate": "Sarah Chen",
  "run_date": "2026-03-05",
  "candidates": [
    {
      "rank": 1,
      "candidate_name": "Sarah Chen",
      "score": 91,
      "tier": "🟢 Fast Track",
      "recommendation": "Strong Hire",
      "summary": "Sarah demonstrates a solid background...",
      "strengths": "3 years SQL experience, Tableau certified",
      "weaknesses": "Limited AWS exposure",
      "skills_found": "SQL, Python, Tableau",
      "skills_missing": "AWS",
      "experience_match": "Exceeds Requirements",
      "red_flags": "",
      "processed_at": "2026-03-05T09:22:34.242Z"
    },
    {
      "rank": 2,
      "candidate_name": "Tom Bradley",
      "score": 10,
      "tier": "🔴 Reject",
      "recommendation": "Reject",
      "summary": "Tom lacks the required technical skills...",
      "strengths": "Good communication skills",
      "weaknesses": "No SQL, Python or data experience",
      "skills_found": "",
      "skills_missing": "SQL, Python, Tableau, AWS",
      "experience_match": "Does Not Meet",
      "red_flags": "No relevant data experience",
      "processed_at": "2026-03-05T09:22:34.242Z"
    }
  ]
}
```

**Response 429 — Rate Limited**

```json
{
  "error": "Too many requests",
  "message": "Max 10 requests per hour per IP"
}
```

---

## 📊 Google Sheets Schema

### Rankings Sheet

| Column | Field | Type | Example |
|---|---|---|---|
| A | Rank | Number | 1 |
| B | Candidate Name | Text | Sarah Chen |
| C | Score | Number | 91 |
| D | Tier | Text | 🟢 Fast Track |
| E | Recommendation | Text | Strong Hire |
| F | Summary | Text | Sarah demonstrates... |
| G | Strengths | Text | SQL, Python, Tableau |
| H | Weaknesses | Text | Limited AWS... |
| I | Skills Found | Text | SQL, Python |
| J | Skills Missing | Text | AWS |
| K | Experience Match | Text | Exceeds Requirements |
| L | Red Flags | Text | — |
| M | Processed At | DateTime | 2026-03-05T09:22:34Z |

### Batch Summary Sheet

| Column | Field | Type | Example |
|---|---|---|---|
| A | Job Title | Text | Data Analyst |
| B | Total Candidates | Number | 2 |
| C | Fast Track | Number | 1 |
| D | Review | Number | 0 |
| E | Rejected | Number | 1 |
| F | Average Score | Number | 53 |
| G | Top Candidate | Text | Sarah Chen |
| H | Run Date | Date | 2026-03-05 |

---

## 🔐 Security

### IP Rate Limiting

Uses n8n workflow static data to track request timestamps per IP. Requests exceeding 10 per hour receive a 429 response before hitting the AI.

### Webhook Authentication

Add a secret header to your webhook node:

```
Header Name:  x-api-key
Header Value: your_secret_key_here
```

Then send from frontend:

```javascript
headers: { 'x-api-key': import.meta.env.VITE_N8N_API_KEY }
```

### CORS Configuration

In the Webhook node Settings, set Allowed Origins:

```
https://hireiq-smart-screen.lovable.app
```

---

## 🔧 Environment Variables

Create a `.env` file (never commit this):

```env
# n8n instance
N8N_WEBHOOK_PATH=screen-resumes
N8N_API_KEY=your_secret_webhook_key

# OpenAI
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx

# Google Sheets
GOOGLE_SHEET_Link=https://docs.google.com/spreadsheets/d/1ib2P-STwBtLCAVJ6dTAcEBF6tuUCuoFKVLmoJYV2t-Q/edit?usp=sharing
```

See `.env.example` for the full template with placeholder values.

---

## 🐛 Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `binary file 'data' not found` | PDF splitter not passing binary | Add `binary: item.binary` to Code node return |
| `No Respond to Webhook node` | Missing terminal node | Add Respond to Webhook at end of flow |
| `Column to Match On required` | Wrong Sheets operation | Change to "Append Row" not "Append or Update" |
| Loop never reaches Done branch | OpenAI node failing | Check API key and model name |
| Blank screen on frontend | Wrong response shape | Ensure candidates array exists in response |
| CORS error from frontend | Origin not whitelisted | Add Lovable URL to Webhook Allowed Origins |

---

## 🔮 Planned Improvements

- [ ] Add retry logic for OpenAI failures
- [ ] Support DOCX resume format
- [ ] Slack notification for Fast Track candidates
- [ ] Webhook signature verification (HMAC)
- [ ] Batch processing queue for 50+ resumes
- [ ] Custom scoring weights per job type

---

## 📁 Repository Structure

```
hireiq-automation/
├── README.md                       ← You are here
├── .env.example                    ← Environment variable template
├── workflows/
│   └── hireiq-resume-screener.json  ← export from n8n
└── docs/
    ├── api-reference.md             ← Full API docs
    ├── setup-guide.md               ← write this next
    └── assets/
        ├── upload-screen.png        ← screenshot
        ├── results-dashboard.png    ← screenshot
        ├── n8n-workflow.png         ← screenshot
        └── google-sheets.png        ← screenshot
```

---

## 👤 Author

**Adeniran Precious Adebayo**

- GitHub: [@adeniranprecious002-ux](https://github.com//adeniranprecious002-ux)
- LinkedIn: [Adeniran Precious Adebayo](https://www.linkedin.com/in/precious-adeniran-842b58294)
- Frontend Repo: [hireiq-frontend](https://github.com/adeniranprecious002-ux/hireiq-smart-candidate-screening)

---

## 📄 License

MIT License — feel free to fork and adapt for your own projects.

---

⭐ **Star this repo if it helped you understand n8n automation!**
