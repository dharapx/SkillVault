# 🎯 Find Your Job — AI-Powered Job Matcher

> **Scrape LinkedIn, Naukri & Glassdoor → Match against your resume → Get scored results in seconds.**

[![OpenCode Skill](https://img.shields.io/badge/OpenCode-Skill-6C47FF)](https://opencode.ai)
[![GitHub](https://img.shields.io/badge/GitHub-Repo-181717?logo=github)](https://github.com)

---

## ✨ What It Does

**Find Your Job** is an [OpenCode](https://opencode.ai) skill that automates the full job-hunting workflow:

1. **🔍 Scrapes** live job listings from **LinkedIn**, **Naukri.com**, and **Glassdoor** (3 search profiles × 3 sites)
2. **📄 Reads** your resume PDF and builds a comprehensive **Resume Profile** (skills, experience, education, domains)
3. **🎯 Scores** every job using **semantic matching** — core requirements (≥90%) and nice-to-haves (≥70%)
4. **🧹 Deduplicates** identical listings across sites (keeps the best match)
5. **📊 Presents** a ranked table with job titles, companies, match scores, posting dates, and **direct apply links**
6. **💾 Saves** full results to `linkedin-jobs-matched.json` for future reference

---

## 🚀 How to Use

### Prerequisites
- [OpenCode](https://opencode.ai) CLI installed
- Resume in PDF or Markdown format
- LinkedIn account (for LinkedIn scraping)
- Glassdoor account (optional, for full Glassdoor results)

### Quick Start

```bash
# Load the skill in OpenCode
# The skill auto-triggers on keywords like:
#   "find jobs", "job matcher", "match resume", "job search"

# Or just tell your agent:
"Find jobs matching my resume on LinkedIn, Naukri, and Glassdoor"
```

### Configuration

Edit `job-matcher-config.json` to customize:

```json
{
  "resume": "~/path/to/your-resume.pdf",
  "jobSites": ["linkedin", "naukri", "glassdoor"],
  "searches": [
    {
      "profile": "Your Job Title",
      "linkedin": {
        "keywords": "Keyword1 Keyword2",
        "locations": ["City1", "City2", "Remote"],
        "filters": { "remote": true, "datePosted": "past_week" }
      },
      "naukri": { ... },
      "glassdoor": { ... }
    }
  ],
  "scoring": {
    "coreMatchThreshold": 90,
    "niceToHaveThreshold": 70
  },
  "companyEligibility": {
    "minAgeYears": 10,
    "minEmployees": 1000
  },
  "jobPostMaxDays": 3
}
```

---

## ⚙️ How It Works

```
              ┌──────────────────────┐
              │   Do you have a      │
              │  Markdown resume?    │◄──── Ask user first
              └──────┬───────┬───────┘
                   Yes│       │No
              ┌───────┘       └──────────┐
              ▼                           ▼
     ┌──────────────────┐      ┌──────────────────┐
     │ Load markdown    │      │  Read PDF with   │
     │ from user/cache  │      │   Read tool      │
     └────────┬─────────┘      └────────┬─────────┘
              │                    Fail │ OK
              │              ┌──────────┘  │
              │              ▼             ▼
              │     ┌──────────────┐ ┌────────────┐
              │     │  MarkItDown  │ │  Save to   │
              │     │   fallback   │ │  cache     │
              │     └──────┬───────┘ └──────┬─────┘
              │            ▼                │
              │     ┌──────────────┐        │
              │     │  Save to     │────────┘
              │     │  cache       │
              │     └──────┬───────┘
              │            ▼
              └─────► ┌──────────────┐
                      │    Resume    │
                      │   Profile    │
                      └──────┬───────┘
                             ▼
    ┌─────────────────────────────────────┐
    │           Job Scraping              │
    ├──────────┬──────────┬───────────────┤
    │ LinkedIn │  Naukri  │  Glassdoor    │
    │ (25jobs) │ (25jobs) │  (25jobs)     │
    └────┬─────┴────┬─────┴──────┬────────┘
         ▼          ▼            ▼
    ┌─────────────────────────────────────┐
    │     Filter by Post Age (≤3d)        │
    │     Filter by Company Elig.         │
    └────────────────┬────────────────────┘
                     ▼
    ┌─────────────────────────────────────┐
    │          Semantic Scoring           │
    │  ┌──────────┐  ┌─────────────────┐  │
    │  │ Core ≥90%│  │ Nice-to-have≥70%│  │
    │  └──────────┘  └─────────────────┘  │
    └────────────────┬────────────────────┘
                     ▼
    ┌─────────────────────────────────────┐
    │        Dedup & Rank Results         │
    ├─────────────────────────────────────┤
    │  #  Title         Core  Source      │
    │  1  DevOps Lead    95%  LinkedIn    │
    │  2  Platform Eng   92%  Naukri      │
    │  3  Cloud Arch     90%  Glassdoor   │
    └─────────────────────────────────────┘
```

---

## 🧠 Smart Matching Features

| Feature | Description |
|---------|-------------|
| **Semantic equivalence** | "5+ years" ≈ "6 years", "React.js" ≈ "React" |
| **Partial skill credit** | Related/broader skills get 70%+ match weight |
| **Experience tolerance** | Within 1 year counts as full match |
| **Generous parsing** | Generic fluff ("team player") ignored |
| **Company eligibility** | Filters by min age (10 yrs) and size (1000 emp) |
| **Cross-site dedup** | Same job on LinkedIn + Naukri = kept once (best score) |

---

## 📁 File Structure

```
~/.config/opencode/skills/find-your-job/
├── SKILL.md                    # Skill instructions (OpenCode reads this)
├── job-matcher-config.json     # Your search profiles & thresholds
├── linkedin-jobs-matched.json  # Output — results from last run
└── README.md                   # This file
```

---

## 🌐 Sites Supported

| Site | Login Required | Max Jobs/Profile | Anti-Scrape Notes |
|------|:---:|:---:|:---|
| **LinkedIn** | ✅ Yes | 25 | 2-4s delays, human-like scrolling |
| **Naukri.com** | ❌ No (better with) | 25 | 3-5s delays, CAPTCHA possible |
| **Glassdoor** | ✅ Recommended | 25 | 3-5s delays, aggressive bot detection |

---

## 📊 Sample Output

```
┌─────┬──────────────────────────────┬──────────────────┬───────┬───────┬──────────┬──────────┐
│  #  │ Job Title                    │ Company          │ Core% │ Nice% │ Posted   │ Source   │
├─────┼──────────────────────────────┼──────────────────┼───────┼───────┼──────────┼──────────┤
│  1  │ Senior DevOps Lead           │ Acme Corp        │  95%  │  80%  │ 2d ago   │ linkedin │
│  2  │ Platform Engineer            │ Beta Inc         │  92%  │  75%  │ 1d ago   │ naukri   │
│  3  │ Cloud Architect              │ Gamma Ltd        │  90%  │  72%  │ 2d ago   │ glassdoor│
└─────┴──────────────────────────────┴──────────────────┴───────┴───────┴──────────┴──────────┘

Job Links:
  1. https://www.linkedin.com/jobs/view/4418297213  (LinkedIn)
  2. https://www.naukri.com/job/12345  (Naukri)
  3. https://www.glassdoor.co.in/Job/12345  (Glassdoor)
```

---

## 🤝 Contributing

Found a bug? Want to add a job site? Open an issue or PR on the repo.

---

## 📝 License

MIT — free to use, modify, and share.
