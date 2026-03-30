# HireIQ API Reference

This document describes the webhook API exposed by the HireIQ n8n automation workflow.

---

## Base URL

```
https://your-instance.app.n8n.cloud/webhook
```

---

## Authentication

All requests must include a secret key in the request header:

```
x-api-key: your_secret_key_here
```

Requests without this header will be rejected with a `401 Unauthorized` response.

---

## Endpoints

### POST /screen-resumes

Accepts a job description and one or more resume PDF files. Returns AI-scored and ranked candidates.

---

#### Request

**Headers**

```
Content-Type: multipart/form-data
x-api-key: your_secret_key_here
```

**Body (multipart/form-data)**

| Field | Type | Required | Description |
|---|---|---|---|
| `job_description` | string | ✅ Yes | Full job description text including requirements and responsibilities |
| `resumes0` | file (PDF) | ✅ Yes | First resume PDF file |
| `resumes1` | file (PDF) | ❌ Optional | Second resume PDF file |
| `resumes2` | file (PDF) | ❌ Optional | Third resume PDF file |
| `resumesN` | file (PDF) | ❌ Optional | Additional resumes (increment N) |

**Constraints**

- Maximum file size per resume: **5MB**
- Accepted file types: **PDF only**
- Maximum resumes per request: **20**
- Rate limit: **10 requests per hour per IP**

---

#### Example Request (JavaScript)

```javascript
const formData = new FormData();
formData.append('job_description', jobDescriptionText);

resumeFiles.forEach((file, index) => {
  formData.append(`resumes${index}`, file, file.name);
});

const response = await fetch('https://your-instance.app.n8n.cloud/webhook/screen-resumes', {
  method: 'POST',
  headers: {
    'x-api-key': 'your_secret_key_here'
  },
  body: formData
});

const data = await response.json();
```

#### Example Request (Python)

```python
import requests

url = "https://your-instance.app.n8n.cloud/webhook/screen-resumes"

headers = {
    "x-api-key": "your_secret_key_here"
}

files = [
    ("resumes0", ("sarah_chen.pdf", open("sarah_chen.pdf", "rb"), "application/pdf")),
    ("resumes1", ("tom_bradley.pdf", open("tom_bradley.pdf", "rb"), "application/pdf")),
]

data = {
    "job_description": "Data Analyst role requiring SQL, Python, Tableau..."
}

response = requests.post(url, headers=headers, files=files, data=data)
print(response.json())
```

#### Example Request (cURL)

```bash
curl -X POST \
  https://your-instance.app.n8n.cloud/webhook/screen-resumes \
  -H "x-api-key: your_secret_key_here" \
  -F "job_description=Data Analyst role requiring SQL, Python, Tableau..." \
  -F "resumes0=@/path/to/sarah_chen.pdf" \
  -F "resumes1=@/path/to/tom_bradley.pdf"
```

---

#### Response

**200 OK — Success**

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
      "summary": "Sarah demonstrates a solid background in data analysis with relevant experience and skills that align closely with the job requirements.",
      "strengths": "3 years SQL experience, Tableau certified, GCP experience",
      "weaknesses": "Limited AWS exposure, No machine learning experience",
      "skills_found": "SQL, Python, Tableau, GCP",
      "skills_missing": "AWS, Machine Learning",
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
      "summary": "Tom lacks the required technical skills and experience for the Data Analyst position. His background in sales does not align with the role requirements.",
      "strengths": "Good communication skills, Punctual and hardworking",
      "weaknesses": "No SQL experience, No Python or data tools, No relevant education",
      "skills_found": "",
      "skills_missing": "SQL, Python, Tableau, AWS, Statistics",
      "experience_match": "Does Not Meet",
      "red_flags": "No relevant data experience, Unrelated work background",
      "processed_at": "2026-03-05T09:22:34.242Z"
    }
  ]
}
```

---

#### Response Fields

**Top-level fields**

| Field | Type | Description |
|---|---|---|
| `success` | boolean | Whether the request was processed successfully |
| `total` | number | Total number of candidates screened |
| `fast_track` | number | Candidates scoring 75 or above |
| `review` | number | Candidates scoring between 50 and 74 |
| `rejected` | number | Candidates scoring below 50 |
| `average_score` | number | Mean score across all candidates |
| `top_candidate` | string | Name of the highest scoring candidate |
| `run_date` | string | Date the screening was run (YYYY-MM-DD) |
| `candidates` | array | Array of scored candidate objects (see below) |

**Candidate object fields**

| Field | Type | Description |
|---|---|---|
| `rank` | number | Position in ranked list (1 = best) |
| `candidate_name` | string | Extracted from resume by AI |
| `score` | number | AI score from 0 to 100 |
| `tier` | string | 🟢 Fast Track / 🟡 Review / 🔴 Reject |
| `recommendation` | string | Strong Hire / Lean Hire / Maybe / Lean Reject / Reject |
| `summary` | string | 2-3 sentence AI assessment |
| `strengths` | string | Comma-separated list of strengths |
| `weaknesses` | string | Comma-separated list of weaknesses |
| `skills_found` | string | Required skills present in resume |
| `skills_missing` | string | Required skills absent from resume |
| `experience_match` | string | Exceeds / Meets / Partially Meets / Does Not Meet |
| `red_flags` | string | Any concerns identified, empty string if none |
| `processed_at` | string | ISO 8601 timestamp of when this resume was processed |

---

#### Tier Classification

| Tier | Score Range | Meaning |
|---|---|---|
| 🟢 Fast Track | 75 – 100 | Strong candidate, prioritise for interview |
| 🟡 Review | 50 – 74 | Potential candidate, review manually |
| 🔴 Reject | 0 – 49 | Does not meet requirements |

---

#### Recommendation Scale

| Recommendation | Meaning |
|---|---|
| Strong Hire | Excellent fit, move forward immediately |
| Lean Hire | Good fit with minor gaps |
| Maybe | Borderline, needs further review |
| Lean Reject | Significant gaps, unlikely to succeed |
| Reject | Does not meet minimum requirements |

---

### Error Responses

**429 Too Many Requests — Rate Limited**

```json
{
  "success": false,
  "error": "Too many requests",
  "message": "Maximum 10 requests per hour per IP address",
  "retry_after": "60 minutes"
}
```

**401 Unauthorized — Missing or Invalid API Key**

```json
{
  "success": false,
  "error": "Unauthorized",
  "message": "Missing or invalid x-api-key header"
}
```

**400 Bad Request — Missing Required Fields**

```json
{
  "success": false,
  "error": "Bad Request",
  "message": "job_description and at least one resume file are required"
}
```

**500 Internal Server Error**

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An unexpected error occurred while processing your request"
}
```

---

## Scoring Methodology

Each resume is evaluated by GPT-4o against the provided job description across these dimensions:

| Dimension | Weight | Description |
|---|---|---|
| Skills Match | High | Required skills present vs missing |
| Experience Level | High | Years and relevance of work history |
| Education | Medium | Degree relevance and level |
| Career Trajectory | Medium | Growth and progression pattern |
| Soft Skills | Low | Communication, leadership indicators |
| Red Flags | Negative | Gaps, irrelevant history, concerns |

The final score (0-100) is a holistic AI judgment combining all dimensions.

---

## Rate Limits

| Limit | Value |
|---|---|
| Requests per IP per hour | 10 |
| Max file size per resume | 5MB |
| Max resumes per request | 20 |
| Max job description length | 5,000 characters |

---

## Testing the API

Use [Hoppscotch](https://hoppscotch.io) (free, browser-based) to test:

1. Go to **hoppscotch.io**
2. Set method to **POST**
3. Set URL to your webhook URL
4. Go to **Headers** → add `x-api-key: your_key`
5. Go to **Body** → select **Multipart Form**
6. Add field `job_description` with your job description text
7. Add field `resumes0` → select a PDF file
8. Click **Send**

---

## Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-03-05 | Initial release with single and multi-resume support |
