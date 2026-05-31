---
name: find-your-job
description: >-
  Scrape job postings from LinkedIn, Naukri.com, and Glassdoor, match them against the user's resume.
  Use when the user asks to find jobs matching their profile, run a job search, or match their resume against job listings.
  Trigger keywords: "job matcher", "find jobs", "match resume", "job search", "naukri", "glassdoor", "linkedin scrape", "career", "find your job".
---

# Job Matcher (LinkedIn + Naukri + Glassdoor)

Match job postings from LinkedIn, Naukri.com, and Glassdoor against the user's resume with semantic scoring.
Core (required) skills must match >= 90%; nice-to-have must match >= 70%.

---

## Before You Begin

All configuration and data files live in this skill's directory:
`~/.config/opencode/skills/find-your-job/`

| File | Purpose |
|---|---|
| `job-matcher-config.json` | Search profiles, locations, filters, thresholds, eligibility rules |
| `linkedin-jobs-matched.json` | Output — written by the agent after each run |
| `resume-cached.md` | Cached resume in Markdown |

---

## CRITICAL: Execution Model (MUST Follow Exactly)

This workflow uses a **strict sequential multi-agent model** for reliability:

```
Main Agent orchestrates:
  Step 1 → [Dedicated Agent: LinkedIn Scraper] → validates output
  Step 2 → [Dedicated Agent: Naukri Scraper]   → validates output
  Step 3 → [Dedicated Agent: Glassdoor Scraper] → validates output
  Step 4 → [Dedicated Agent: Scorer & Reporter] → final output
```

**RULES:**
1. Process ONE site at a time. Never launch two site scrapers in parallel.
2. Each site gets its own dedicated `general` Task agent with a clear, focused prompt.
3. After each site agent completes, the main agent MUST validate the output JSON file before proceeding to the next site.
4. Only after ALL 3 sites are scraped does the main agent launch the scoring agent.
5. Every intermediate file uses the SAME schema so agents can be swapped/re-run independently.

---

## Resume Profile

### Cached Resume
The skill saves the extracted resume as Markdown to:
```
~/.config/opencode/skills/find-your-job/resume-cached.md
```
On each run, check if this file exists. If it does, load it directly as the **Resume Profile** — skip all PDF extraction steps. The cache is invalidated only when the user provides a new/different resume path.

### Step 1 — Ask the user
Before any file extraction, ask:
> "Do you already have a Markdown version of your resume? If yes, paste the content or provide the path."

If they provide it, save it to `resume-cached.md` and build the profile from it.

### Step 2 — Read PDF via Read tool
If the user has no Markdown version, resolve the PDF path from `config.resume` (always resolve `~` to the user's home directory). Use the **Read tool** first — it can natively extract text from PDFs.

If the Read tool returns text content successfully, save it to `resume-cached.md`.

### Step 3 — Fallback to MarkItDown
If the Read tool cannot parse the PDF (returns an error or empty output), use Microsoft's `markitdown`:

**CLI** (preferred):
```
pip install 'markitdown[pdf]'
markitdown "<path-to-pdf>" > resume-output.md
```

**Python API**:
```python
from markitdown import MarkItDown
md = MarkItDown()
result = md.convert("<path-to-pdf>")
print(result.text_content)
```

Save the output to `resume-cached.md`.

### Step 4 — If all fails
If neither the Read tool, MarkItDown CLI, nor MarkItDown Python works, ask the user:
- "Please provide a plain-text or Markdown version of your resume."
- If the user provides text inline or a path, save it to `resume-cached.md`.

### Parse the Resume Profile
From the cached/resolved Markdown text, extract these fields into a structured dictionary:

```json
{
  "name": "Pulakesh Dhara",
  "currentTitle": "DevOps Lead / Platform Engineer",
  "totalYearsExperience": 9,
  "skills": {
    "cloud": ["AWS", "Azure", ...],
    "iaC": ["Terraform", "CloudFormation"],
    "containers": ["Docker", "Kubernetes", "EKS", "ECS"],
    "observability": ["Grafana", "Prometheus", ...],
    "cicd": ["Azure DevOps", "TeamCity"],
    "programming": ["Python", "Bash", "PowerShell", ...],
    "ai": ["AI Agents", "Prompt Engineering"]
  },
  "education": [...],
  "locations": ["Pune"]
}
```

THIS structured dictionary is passed to every scraping agent and the scoring agent.

---

## Intermediate Data Format

Every scraper agent writes its output to a shared intermediate file at:
`~/.config/opencode/skills/find-your-job/raw-jobs.json`

Schema:
```json
{
  "source": "linkedin" | "naukri" | "glassdoor",
  "searchProfile": "DevOps Lead / Platform Engineer",
  "jobs": [
    {
      "title": "string (required)",
      "company": "string (required)",
      "url": "string (required) — full absolute URL to job posting",
      "companyPageUrl": "string or null",
      "location": "string (required)",
      "postedDate": "string (required) — e.g. '2 days ago'",
      "postedDateDays": "number (required) — days since posted",
      "description": "string (required) — full job description text",
      "companyAge": "number or null — years since founded",
      "companySize": "string or null — e.g. '5,001-10,000 employees'",
      "companyEligible": "bool (required) — based on config thresholds"
    }
  ]
}
```

**VALIDATION RULE:** Before returning, the scraper agent MUST verify every job has a non-empty `url`, `title`, `company`, `location`, `postedDate`, `postedDateDays`, and `description`. If any fail, skip those jobs.

---

## Site-by-Site Workflow (Main Agent Orchestration)

For EACH site in `config.jobSites` (linkedin → naukri → glassdoor), in this exact order:

### 1. Launch a dedicated Task agent for that site

The main agent creates a `general` Task agent with this prompt pattern. The prompt must include:
- The raw-jobs.json file path and schema
- The config file path and contents
- The resume profile (structured skills dictionary)
- The site-specific scraping instructions from below
- The exact search URLs to use (the main agent computes the URLs from config)
- A clear instruction to write results to raw-jobs.json **appending** to any existing data

### 2. Wait for agent to complete

### 3. Validate output

After the agent returns, the main agent MUST:
a. Read raw-jobs.json
b. Count jobs from that site
c. Verify ALL required fields are non-empty for every job from that site
d. If `< 3` jobs scraped, re-run with a warning to the scraper

### 4. Proceed to next site

---

## Step 1: LinkedIn Scraping (via Dedicated Agent)

### 1a. Prepare URLs

For each search profile in config.searches[], compute:
```
https://www.linkedin.com/jobs/search/?keywords={URL_ENCODED_KEYWORDS}&location={URL_ENCODED_FIRST_LOCATION}
```

### 1b. Agent Instructions

The LinkedIn scraper agent receives:
- A LinkedIn browser session that is already logged in (main agent handles login)
- List of search URLs to process
- Resume profile for context
- raw-jobs.json path

It MUST:
1. For each search URL:
   - Navigate to the URL
   - Wait 3 seconds
   - Apply filters (date posted, remote, experience level)
   - Wait for results to load
   - Extract ALL visible job cards (title, company, location, url, posted date)
   - For each card where days <= 3:
     - Click the job card
     - Wait 2 seconds
     - Extract full description text (click "show more" if needed)
     - Get company details from the right panel
   - Limit: 25 jobs per search profile
2. Write ALL collected jobs to raw-jobs.json (appending mode)
3. Return stats: how many scraped, how many filtered, how many passed

### 1c. LinkedIn Filters

- **Date Posted**: Click filter button → select "Past week" → click "Show results"
- **Remote**: Click filter button → check "Remote" and "Hybrid" → click "Show results"
- **Experience Level**: Click filter button → check "Mid-Senior" and "Director" → click "Show results"

### 1d. Date Parsing

LinkedIn shows dates like:
- "7 hours ago", "15 hours ago" → 0 days
- "1 day ago", "2 days ago", "3 days ago" → 1, 2, 3 days
- "1 week ago", "2 weeks ago", "1 month ago" → 7, 14, 30 days

---

## Step 2: Naukri.com Scraping (via Dedicated Agent)

### 2a. Prepare URLs

For each search profile, compute:
```
https://www.naukri.com/jobs?k={URL_ENCODED_KEYWORDS}&l={URL_ENCODED_LOCATION}
```

### 2b. Agent Instructions

1. Navigate to Naukri search page
2. Apply filters:
   - **Posted Date**: Look for "Posted by" or "Date" filter — select "Last 3 days"
   - **Remote/WFH**: Check "Work from Home" or "Remote" checkbox
3. Wait 3 seconds for results
4. Extract job cards (class `jobTuple` or `data-job-id` or similar):
   - Title, Company, Location, URL, Posted date, Salary, Experience
5. For each job with days <= 3:
   - Navigate to job detail URL
   - Wait 3 seconds
   - Extract full description
   - Extract company info (size, founded year)
6. Write to raw-jobs.json (append mode)
7. Return stats

### 2c. Date Parsing

Naukri shows:
- "Just posted", "Posted today" → 0 days
- "Posted 1 day ago" → 1 day
- "Posted X days ago" → X days
- "Posted X weeks ago" → X * 7 days

### 2d. Company Lookup

If company size/founded year not found on Naukri:
- Search company name on Google with "founded" and "employees"
- Use `webfetch` or `websearch` tool
- If still not found, set `companyEligible` to `true` (pass) with a note

---

## Step 3: Glassdoor Scraping (via Dedicated Agent)

### 3a. Login

Glassdoor requires login. The main agent:
1. Opens a new browser tab to `https://www.glassdoor.co.in/Job/index.htm`
2. Asks the user to log in manually
3. Confirms login before launching the Glassdoor agent

### 3b. Prepare URLs

Use Glassdoor search:
```
https://www.glassdoor.co.in/Job/jobs.htm?sc.keyword={URL_ENCODED_KEYWORDS}
```

### 3c. Agent Instructions

1. Navigate to Glassdoor search URL
2. Apply filters:
   - **Date Posted**: Select "Last 3 days" or "Past week"
   - **Remote**: Toggle "Remote" filter if available
3. Wait 3 seconds
4. Extract job cards:
   - Title, Company, Location, URL, Posted date, Salary
5. For each job with days <= 3:
   - Click the job card
   - Wait 3 seconds
   - Click "Show more" to expand description
   - Extract full description
   - Extract company info (size, founded) from the detail panel
6. Write to raw-jobs.json (append mode)
7. Return stats

---

## Step 4: Scoring & Reporting (via Dedicated Agent)

### 4a. Agent Instructions

The scoring agent receives:
- raw-jobs.json with all collected jobs from all 3 sites
- Resume profile (structured skills dictionary)
- Config file thresholds

It MUST:

1. **Parse each job description** into core requirements and nice-to-have requirements:
   - **Core**: Under sections titled "Requirements", "Required", "Qualifications", "Must have", "Minimum qualifications", "Key Responsibilities" (specific technical skills only)
   - **Nice-to-Have**: Under sections titled "Nice to have", "Preferred", "Good to have", "Bonus points", "Plus", "Desirable"
   - **Skip generic phrases**: "team player", "communication skills", "attention to detail", "problem-solving", "work in a team"

2. **Score core requirements**:
   - For each core requirement, check if it matches any skill in the resume
   - Be generous with semantic matching (e.g., "AWS" matches "AWS", "Kubernetes" matches "K8s", "Python" matches "python")
   - Core Match % = (matched core / total core) × 100

3. **Score nice-to-have requirements**:
   - Same matching logic
   - Nice-to-Have Match % = (matched nice-to-have / total nice-to-have) × 100

4. **Apply thresholds**:
   - Core >= 90% AND Nice-to-Have >= 70% → PASS
   - Otherwise → skip

5. **Deduplicate**: Same company + same title = keep higher core match

6. **Generate output**:

### 4b. Terminal Output

Ranked table sorted by Core Match % descending:

```
┌────┬─────────────────────────────┬──────────────────┬──────────┬──────────┬──────────┬──────────┬────────────┬──────────┐
│ #  │ Job Title                   │ Company          │ Core %   │ Nice %   │ Age yrs  │ Size     │ Posted     │ Source   │
├────┼─────────────────────────────┼──────────────────┼──────────┼──────────┼──────────┼──────────┼────────────┼──────────┤
│  1 │ DevOps Lead                 │ Acme Corp        │    95%   │    80%   │      14  │  5000+   │ 2 days ago │ LinkedIn │
└────┴─────────────────────────────┴──────────────────┴──────────┴──────────┴──────────┴──────────┴────────────┴──────────┘
```

Then **Job Links** section with direct URLs:
```
Job Links:
  1. https://www.linkedin.com/jobs/view/4418297213  (LinkedIn)
```

Then **Summary** section with full stats:
```
Summary: Found N matching jobs across 3 search profiles and 3 job sites.
  - LinkedIn scraped: X, filtered: Y, matched: Z
  - Naukri scraped: X, filtered: Y, matched: Z
  - Glassdoor scraped: X, filtered: Y, matched: Z
  - Final matches: N
```

### 4c. Save Full Results

Write to `linkedin-jobs-matched.json` with EXACT schema:

```json
{
  "generatedAt": "ISO timestamp",
  "resume": "Pulakesh Dhara",
  "config": {
    "coreThreshold": 90,
    "niceToHaveThreshold": 70,
    "minCompanyAge": 10,
    "minEmployees": 1000,
    "maxPostDays": 3
  },
  "stats": {
    "totalScraped": 0,
    "linkedinScraped": 0,
    "linkedinFiltered": 0,
    "naukriScraped": 0,
    "naukriFiltered": 0,
    "glassdoorScraped": 0,
    "glassdoorFiltered": 0,
    "finalMatches": 0
  },
  "results": [
    {
      "title": "string (required)",
      "company": "string (required)",
      "url": "string (required) — full absolute URL, NEVER empty",
      "companyPageUrl": "string or null",
      "location": "string (required)",
      "postedDate": "string (required)",
      "companyAge": "number or null",
      "companySize": "string or null",
      "companyEligible": "bool (required)",
      "matchScores": { "core": 90, "niceToHave": 80, "overall": 85 },
      "source": "linkedin" | "naukri" | "glassdoor",
      "searchProfile": "string (required)"
    }
  ]
}
```

**VALIDATION RULE:** Before writing, verify EVERY result has a non-empty `url`. If any are missing, skip those entries.

---

## Important Guidelines (All Sites)

### URL Consistency (CRITICAL - Do Not Skip)
- Every job result across ALL stages MUST carry a non-empty `url` field.
- The `url` is the full absolute URL to the job posting page on the source site.
- Never construct or guess a URL — always extract from the page's HTML `href` attribute.
- If URL extraction fails, skip that job with a warning.
- Verify `raw-jobs.json` has non-empty URLs before passing to scoring.

### Error Handling
- If Playwright fails to find an element, try 2 alternative selectors
- If a job detail page fails to load, skip and continue
- If CAPTCHA appears, stop and ask the user to solve it
- If a site blocks scraping, note it in stats and continue to next site
- If `< 3` jobs come back from a search, the scraper agent should re-scroll and try again

### Anti-Scraping Delays (MANDATORY)
- LinkedIn: 2-4 seconds between actions
- Naukri: 3-5 seconds between actions
- Glassdoor: 3-5 seconds between actions

### Matching Judgment
- Be generous: "5+ years" matches "6 years", "React.js" matches "React"
- Within 1 year of experience = full match
- Related/broader skill = 70% match
- Do NOT count generic soft skills as requirements
