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

## Future Extensions (not in scope now)
- Keyword filtering in job titles
- Multiple locations
- Slack/Telegram notification option
- Job deduplication across runs
