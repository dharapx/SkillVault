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
From the cached/resolved Markdown text, extract:
- **Job titles** held in the past
- **Technical skills** (languages, frameworks, tools, databases)
- **Years of experience** per skill domain
- **Education** (degrees, fields, certifications)
- **Industries** worked in

Store this as a structured dictionary — referenced for every job comparison.

---

## Workflow Overview

For each site in `config.jobSites`, run the searches defined in `config.searches[]`:

- **linkedin** → Step 3 (LinkedIn Playwright workflow)
- **naukri** → Step 4 (Naukri Playwright workflow)
- **glassdoor** → Step 5 (Glassdoor Playwright workflow)

All jobs from all sites flow into the same **Scoring** (Step 6), **Dedup** (Step 7), and **Presentation** (Step 8) steps.

---

## Step 3: LinkedIn Scraping (via Playwright)

### 3a. Launch Playwright and Log In

1. Open LinkedIn Jobs: `https://www.linkedin.com/jobs/`
2. Ask the user to log in manually, wait for confirmation

### 3b. For Each Search Profile, Search LinkedIn

Navigate to: `https://www.linkedin.com/jobs/search/?keywords={URL_ENCODED_KEYWORDS}&location={URL_ENCODED_LOCATION}`

Use the first location from the profile's `locations` array.

### 3c. Apply Filters

For each filter in the profile's `filters`:
- **Remote**: Click "Remote" filter toggle if `remote: true`
- **Date Posted**: Select matching option (e.g. "Past week")
- **Experience Level**: Check matching levels

### 3d. Scrape LinkedIn Job Cards

Extract for every job: **title**, **company**, **company page URL**, **location**, **job link** (the full `href` from the job title anchor, e.g. `https://www.linkedin.com/jobs/view/{JOB_ID}`), **posted date text**.

**MANDATORY**: Every job result MUST have a non-empty `url` field. Extract the `href` attribute from the job title `<a>` element — do not generate or construct URLs. If the URL cannot be extracted, log a warning and skip that job.

Limit: first **25 job listings** per search profile.

### 3e. Parse Posted Date & Filter by Age

Convert to days since posted — if > `config.jobPostMaxDays` (3), skip.

### 3f. Get Job Details

Click each remaining job card, expand description, extract full text.

### 3g. Get Company Details

Navigate to company page, extract company size and founded year.

---

## Step 4: Naukri.com Scraping (via Playwright)

Naukri.com does not require login for browsing jobs, but logged-in users get more results.

### 4a. Navigate to Naukri Search

For each search profile, navigate to:
`https://www.naukri.com/{URL_ENCODED_KEYWORDS}-jobs-in-{URL_ENCODED_LOCATION}`

If the URL pattern above doesn't work, use the search page:
`https://www.naukri.com/jobs?k={URL_ENCODED_KEYWORDS}&l={URL_ENCODED_LOCATION}`

### 4b. Apply Naukri Filters

Look for filter sections on the left sidebar:
- **Experience**: Select the matching experience range
- **Posted Date**: Look for "Posted by" filter — select "Last 3 days" or "Last week"
- **Remote / Work From Home**: Check the "Work from Home" or "Remote" checkbox
- **Salary**: Optional — can skip
- **Company Type**: Optional — can skip

### 4c. Scrape Naukri Job Cards

Use Playwright to extract job cards from the search results. Each card typically has:
- `class="jobTuple"` or `data-job-id` or similar container

For each job card extract:
- **Job title** — usually in an `<a>` tag with class `title` or inside `class="job-title"`
- **Company name** — usually in an `<a>` tag with class `subTitle` or `comp-name`
- **Location** — text near a location icon or class `loc`
- **Job link** — the **full absolute `href`** on the title anchor (usually `https://www.naukri.com/job/...`)
- **Posted date** — text like "Posted X days ago", "Just posted", "Posted today"
- **Salary** — if visible (optional)
- **Experience required** — if visible (optional)
- **Job description snippet** — short preview text

**MANDATORY**: Every job result MUST have a non-empty `url` field with the full absolute URL to the job detail page. If the URL cannot be extracted, skip that job.

### 4d. Parse Posted Date (Naukri)

Naukri displays dates as:
- "Just posted" or "Posted today" → 0 days
- "Posted 1 day ago" → 1 day
- "Posted X days ago" → X days
- "Posted X weeks ago" → X × 7 days
- "Posted on DD MMM YYYY" → compute from date

If `days > config.jobPostMaxDays (3)`, skip immediately.

### 4e. Get Naukri Job Details

Click each remaining job card or navigate to its detail URL:

1. Navigate to the job detail page URL
2. Wait 3 seconds for the page to load
3. Extract the **full job description** — usually in a `div` with class `job-detail` or `job-description`
4. Extract **company name** and **company info** from the detail page
5. Look for **company size/revenue** info — often in the "About Company" section
6. Look for **company founded year** if available

For company eligibility on Naukri:
- **Company age**: Search for "Founded", "Since", "Incorporated" text. If not found, try searching the company name on Google or skip age check with note.
- **Employee count**: Search for "employees", "headcount", "team size". If not found, estimate from company description or skip with note.

### 4f. Navigate Back

Call `playwright_navigate_back()` to return to search results for the next job.

---

## Step 5: Glassdoor Scraping (via Playwright)

Glassdoor requires login to view full job details. Open in headed mode for manual login.

### 5a. Open Glassdoor Jobs

Navigate to: `https://www.glassdoor.co.in/Job/index.htm`

Ask the user: *"A browser window has opened. Please log into Glassdoor in that window. Once logged in, type 'done' to continue."*

Wait for the user to confirm they are logged in.

### 5b. For Each Search Profile, Search Glassdoor

Navigate to:
`https://www.glassdoor.co.in/Job/{URL_ENCODED_LOCATION}/{URL_ENCODED_KEYWORDS}-jobs-SRCH_IL.0,{LOCATION_LEN}_KO{KEYWORDS_START},{KEYWORDS_END}.htm`

If the above URL is complex, use the simpler search:
`https://www.glassdoor.co.in/Job/jobs.htm?sc.keyword={URL_ENCODED_KEYWORDS}&locT=C&locId={LOCATION_ID}`

Use the first location from the profile's `locations` array.

### 5c. Apply Glassdoor Filters

Look for filter sections:
- **Date Posted**: Look for "Posted" or "Date" filter — select "Last 3 days" or "Past week"
- **Remote**: Look for "Remote" or "Work From Home" filter toggle
- **Salary**: Optional — can skip
- **Company Rating**: Optional — can skip
- **Experience Level**: Look for "Experience" filter if available

### 5d. Scrape Glassdoor Job Cards

Extract job cards from search results. Glassdoor cards typically have:
- `class="jobListing"` or `data-id` or `job-container` or similar

For each job card extract:
- **Job title** — usually in a link with `class="jobLink"` or inside heading element
- **Company name** — usually near `class="jobEmpolyerName"` or similar
- **Location** — class `jobLocation` or similar
- **Job link** — the **full absolute `href`** on the job title link (full URL to job detail, e.g. `https://www.glassdoor.co.in/Job/...`)
- **Posted date** — text like "Posted X days ago", "Posted today", "24h ago", "30d+"
- **Salary estimate** — if visible (optional)
- **Company rating** — star rating if visible (optional)

**MANDATORY**: Every job result MUST have a non-empty `url` field with the full absolute URL. If the URL cannot be extracted, skip that job.

Limit: first **25 job listings** per search profile.

### 5e. Parse Posted Date (Glassdoor)

Glassdoor displays dates as:
- "Posted today" or "Just posted" → 0 days
- "24h ago" or "Posted 1 day ago" → 1 day
- "Posted X days ago" → X days
- "Posted X weeks ago" → X × 7 days
- "30d+" or "30+ days ago" → treat as 31
- "Posted on [Month] [DD], [YYYY]" → compute from date

If `days > config.jobPostMaxDays (3)`, skip immediately.

### 5f. Get Glassdoor Job Details

For each remaining job listing:

1. Click the job card or navigate to its detail URL
2. Wait 3 seconds for the detail panel to load
3. Click "Show more" or expand the description if a "More" button is present
4. Extract the **full job description** — usually in a `div` with class `jobDescriptionContent` or `desc`
5. Extract **company info** from the detail page or company section:
   - **Company size** — look for "Size" or "Company Size" text (e.g., "1001 to 5000 employees")
   - **Founded year** — look for "Founded" text (e.g., "Founded 2010")
6. Also check the Glassdoor company overview section for these details

### 5g. Get Glassdoor Company Details

If company size/founded year not found in job detail:

1. Click the company name link to go to the company's Glassdoor page
2. Extract company size from the "Overview" section
3. Extract founded year from the "Overview" section
4. Call `playwright_navigate_back()` to return

### 5h. Company Eligibility (Glassdoor)

Apply the same eligibility filters:
- **Company age < `config.companyEligibility.minAgeYears`** → skip
- **Employee count lower bound < `config.companyEligibility.minEmployees`** → skip

If company details cannot be found on Glassdoor, try searching the company on LinkedIn or Google, or skip with a note.

---

## Step 6: Score Each Job Against the Resume

**IMPORTANT**: During scoring, every job result must retain its `url` (direct link to the job posting), `source` (linkedin/naukri/glassdoor), and `postedDate` fields. These are required for the output table and JSON file.

### 6a. Identify Core vs Nice-to-Have Requirements

Parse the job description:
- **Core**: Under "Requirements", "Required", "Qualifications", "Must have", "Minimum qualifications", "Responsibilities" (key skills)
- **Nice-to-Have**: Under "Nice to have", "Preferred", "Good to have", "Bonus points", "Plus", "Desirable"

### 6b. Score Core Requirements

For each core requirement, check Resume Profile for a match.
**Core Match %** = (core requirements met / total core requirements) × 100

Pass threshold: `>= config.scoring.coreMatchThreshold` (default 90%)

### 6c. Score Nice-to-Have Requirements

**Nice-to-Have Match %** = (nice-to-have met / total nice-to-have) × 100

Pass threshold: `>= config.scoring.niceToHaveThreshold` (default 70%)

### 6d. Overall Decision

Passes **both** filters → add to results.

---

## Step 7: Deduplicate Results

Merge jobs from LinkedIn, Naukri, and Glassdoor. Same company + same title = duplicate (keep the one with higher core match).

---

## Step 8: Present Results

### Terminal Output

Ranked table sorted by **Core Match %** descending, showing the source site:

```
┌────┬─────────────────────────────┬──────────────────┬──────────┬──────────┬──────────┬──────────┬────────────┬──────────┐
│ #  │ Job Title                   │ Company          │ Core %   │ Nice %   │ Age yrs  │ Size     │ Posted     │ Source   │
├────┼─────────────────────────────┼──────────────────┼──────────┼──────────┼──────────┼──────────┼────────────┼──────────┤
│  1 │ DevOps Lead                 │ Acme Corp        │    95%   │    80%   │      14  │  5000+   │ 2 days ago│ LinkedIn │
│  2 │ Sr Platform Engineer        │ Beta Inc         │    92%   │    75%   │      22  │  10000+  │ 1 day ago │ Naukri   │
│  3 │ Cloud Engineer              │ Gamma Ltd        │    90%   │    72%   │      12  │  5000+   │ 2 days ago│ Glassdoor│
└────┴─────────────────────────────┴──────────────────┴──────────┴──────────┴──────────┴──────────┴────────────┴──────────┘
```

**MANDATORY**: Always include a **Job Links** section after the table with the direct URL for every passing job:

```
Job Links:
  1. https://www.linkedin.com/jobs/view/4418297213  (LinkedIn)
  2. https://www.naukri.com/job/12345  (Naukri)
  3. https://www.glassdoor.co.in/Job/12345  (Glassdoor)
```

**MANDATORY**: Always include a **Summary** section:

```
Summary: Found 8 matching jobs across 3 search profiles and 3 job sites.
  - LinkedIn scraped: 45, filtered: 30, matched: 5
  - Naukri scraped: 30, filtered: 22, matched: 2
  - Glassdoor scraped: 20, filtered: 15, matched: 1
  - Final matches: 8
```

### Save Full Results

Write to `linkedin-jobs-matched.json`. Every result object MUST include these fields:

| Field | Required | Description |
|---|---|---|
| `title` | yes | Job title |
| `company` | yes | Company name |
| `url` | **yes** | **Full absolute URL to the job posting** — extracted during scrape, never generated/guessed |
| `companyPageUrl` | if available | URL to company page |
| `location` | yes | Job location |
| `postedDate` | yes | Relative posted date (e.g. "2 days ago") |
| `companyAge` | if available | Company age in years |
| `companySize` | if available | Employee count range |
| `companyEligible` | yes | `true`/`false` based on config thresholds |
| `matchScores` | yes | Object with `core`, `niceToHave`, `overall` |
| `source` | yes | `"linkedin"`, `"naukri"`, or `"glassdoor"` |
| `searchProfile` | yes | Which search profile matched this job |

**MANDATORY RULE**: Every result in `results[]` MUST have a non-empty `url` field pointing to the actual job posting page. Skip any job where the URL cannot be extracted. Do not construct or guess URLs.

---

## Step 9: Site-Specific Anti-Scraping Guidelines

### LinkedIn
- Add **2-4 second delays** between every Playwright action
- Do NOT scroll too fast — mimic human reading speed
- Do not scrape more than **25 jobs per search profile** per run
- If LinkedIn shows "Sign in to see more results", ask the user to sign in again

### Naukri
- Add **3-5 second delays** between page navigations
- Naukri may show a CAPTCHA after rapid requests — if hit, stop and ask the user to solve it
- Naukri may require login to view full job descriptions — if a login wall appears, ask the user to log in
- Some Naukri job cards have multiple `data-job-id` attributes — prefer exact selectors
- Naukri pagination uses "Next" buttons at the bottom — limit to page 1 (first 20-25 jobs)

### Glassdoor
- Add **3-5 second delays** between page navigations
- Glassdoor is aggressive with anti-bot measures — if blocked, stop and inform the user
- Some Glassdoor pages require JavaScript rendering — Playwright handles this natively
- Glassdoor may show a sign-in wall for full job descriptions — ask the user to log in if needed
- Company details (size, founded) are often on the company overview page, not the job listing
- Glassdoor URLs use location IDs — prefer navigating via search page rather than constructing URLs manually
- If CAPTCHA appears, ask the user to solve it

---

## Important Guidelines (All Sites)

### Matching Judgment
- Be generous with semantic equivalence: "5+ years" matches "6 years", "React.js" matches "React"
- For years of experience, a match within 1 year counts as a full match
- For skills, consider a related or broader skill as a partial match (70%+)
- Do not count generic requirements like "team player" against the user

### URL Consistency (CRITICAL)
- Every job result across all stages (scraping, scoring, output JSON, terminal display) MUST carry its `url` field.
- The `url` field is the full absolute URL to the job posting page on the source site.
- Never construct or guess a URL — always extract it from the page's HTML `href` attribute.
- Before writing results to `linkedin-jobs-matched.json`, verify that every entry has a non-empty `url`. If any are missing, re-scrape or skip those entries.
- In the terminal output table, the "Job Links" numbered section is **mandatory** — every passing job must have its URL listed.

### Error Handling
- If Playwright fails to find an element, try an alternative selector
- If a job detail page fails to load, skip and continue
- If the resume PDF cannot be read, ask the user for the correct path

### Config File Location
`~/.config/opencode/skills/find-your-job/job-matcher-config.json`

Always resolve the `resume` path in the config relative to the user's home directory (`~`).
