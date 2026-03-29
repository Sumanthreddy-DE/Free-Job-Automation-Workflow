# Bosch Job Alert Automation — Design Spec

## Problem
Manually checking the Bosch careers portal for new student/graduate jobs in Reutlingen is tedious. Automate it with a daily email digest.

## Solution
A Python script that runs daily via GitHub Actions, fetches new job postings from the SmartRecruiters public API, filters them, and sends an email with clickable links.

## API
- **Endpoint:** `GET https://api.smartrecruiters.com/v1/companies/BoschGroup/postings`
- **Auth:** None required (public API)
- **Rate limit:** 10 req/sec (we make 1-2 requests total)
- **Query params:** `country=de`, `city=Reutlingen`, `limit=100`
- **Response fields used:** `name`, `releasedDate`, `experienceLevel`, `id`, `ref`, `location`, `department`

## Filters
- **Location:** Reutlingen, Germany
- **Time window:** Jobs posted in the last 72 hours
- **Experience levels:** `internship`, `entry_level`, `associate`, `not_applicable` (covers Praktikum, Werkstudent, Thesis, PreMaster)
- **Keywords:** None for now. Config placeholder for future use — will match against job title (case-insensitive, partial match)

## Email
- **Transport:** Gmail SMTP (`smtp.gmail.com:587`, STARTTLS)
- **Auth:** Gmail App Password (not account password)
- **Format:** HTML email with a table of jobs — title (as hyperlink), location, date posted, experience level
- **Subject:** `Bosch Reutlingen — X new jobs (Mar 29)`
- **No jobs case:** Send a short "No new jobs in the last 72 hours" email so the user knows the script ran
- **Recipient:** Same as sender (self-notification)

## Job Link Format
`https://jobs.smartrecruiters.com/BoschGroup/{posting_id}-{slugified-name}`

The API response includes a `ref` field and `id`. We construct the link from the API detail URL pattern.

## Config (top of main.py)
```python
CONFIG = {
    "company": "BoschGroup",
    "country": "de",
    "city": "Reutlingen",
    "hours_window": 72,
    "experience_levels": ["internship", "entry_level", "associate", "not_applicable"],
    "keywords": [],  # empty = all jobs, future: ["software", "data", "embedded"]
}
```

## Scheduler
- **Platform:** GitHub Actions
- **Cron:** `0 5 * * *` (5:00 UTC = 7:00 AM CEST)
- **Manual trigger:** `workflow_dispatch` enabled for testing
- **Secrets:** `GMAIL_ADDRESS`, `GMAIL_APP_PASSWORD`

## Error Handling
- API request failure → log error, exit with non-zero code (GitHub Actions marks run as failed)
- Zero jobs after filtering → send "no new jobs" email
- Email send failure → log error, exit with non-zero code
- No retry logic needed (runs daily, transient failures resolve next run)

## Project Structure
```
Automation-test/
├── main.py              # ~80-100 lines: fetch, filter, format, email
├── requirements.txt     # requests
├── .github/workflows/
│   └── notify.yml       # Daily cron + manual trigger
├── .gitignore
└── README.md
```

## Implementation Status (Mar 29, 2026)

### Completed
- [x] Project scaffolded at `C:\Users\suman\Desktop\Docs\Job\Projects\Automation-test`
- [x] `main.py` — fetch, filter, HTML email, Gmail SMTP send (~100 lines)
- [x] `requirements.txt` — `requests>=2.31.0`
- [x] `.github/workflows/notify.yml` — daily cron (5 AM UTC) + manual trigger
- [x] GitHub repo created: `Sumanthreddy-DE/Free-Job-Automation-Workflow`
- [x] Code pushed to `main` branch
- [x] GitHub secrets configured: `GMAIL_ADDRESS`, `GMAIL_APP_PASSWORD`
- [x] First manual workflow run — successful, email received
- [x] Job links verified working (SmartRecruiters URLs resolve correctly)

### Key Decisions Made
- **No scraping needed** — Bosch uses SmartRecruiters which has a free public API
- **Job link format:** `https://jobs.smartrecruiters.com/BoschGroup/{posting_id}` (just the ID works, no slug needed)
- **Experience level field:** `experienceLevel.label` (not `.name`) for display
- **Department column removed** from email — API list endpoint returns empty department, not worth an extra API call per job
- **Email columns:** Job Title (clickable link), Level, Posted date

### API Notes
- SmartRecruiters public API: `https://api.smartrecruiters.com/v1/companies/BoschGroup/postings`
- No auth, no API key needed
- Rate limit: 10 req/sec, 8 concurrent (we use 1 request)
- Single job detail: `https://api.smartrecruiters.com/v1/companies/BoschGroup/postings/{id}`
- Reutlingen currently has ~57 total jobs, mostly internships
- Experience level values: `internship` (Praktikum), `entry_level` (PreMaster/Graduate), `associate` (PreMaster), `not_applicable` (Werkstudent/Thesis)

## Future Extensions (not yet implemented)
- **Keyword filtering** — CONFIG already has `keywords: []` placeholder. Add keywords to match in job titles (case-insensitive, partial). Code already handles this — just populate the list.
- **Multiple locations** — extend `city` param or make multiple API calls
- **Multiple companies** — SmartRecruiters hosts many companies, same API pattern works
- **Slack/Telegram notifications** — add as alternative to email
- **Job deduplication** — track seen job IDs across runs to avoid repeat notifications
- **Configurable experience levels** — allow filtering by specific levels
- **Weekly digest mode** — option for weekly summary instead of daily
