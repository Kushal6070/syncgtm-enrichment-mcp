---
name: sync-mcp
description: "Use when asked to enrich contacts (emails, phones, mobile), scrape LinkedIn posts/commenters/engagers, look up LinkedIn from email, enrich companies, check job changes, or use any SyncGTM MCP tool. Use when the user says 'enrich emails', 'find phone numbers', 'scrape LinkedIn posts', 'find work email', 'enrich company', 'get LinkedIn posts', 'check job change', 'find mobile', or 'use SyncGTM'. Handles credit checks, column guards, LinkedIn URL requirements, and structured CSV output."
metadata:
  version: 1.1.0
allowed-tools: mcp__syncgtm__check_credits, mcp__syncgtm__enrich_person, mcp__syncgtm__find_work_email, mcp__syncgtm__find_personal_email, mcp__syncgtm__find_work_phone, mcp__syncgtm__find_mobile_number, mcp__syncgtm__verify_email, mcp__syncgtm__validate_whatsapp, mcp__syncgtm__find_linkedin_from_work_email, mcp__syncgtm__find_linkedin_from_personal_email, mcp__syncgtm__linkedin_profile_enrich, mcp__syncgtm__linkedin_profile_posts, mcp__syncgtm__enrich_organization, mcp__syncgtm__find_company_techstack, mcp__syncgtm__find_company_website_traffic, mcp__syncgtm__find_people_within_company, mcp__syncgtm__company_job_listings, mcp__syncgtm__search_job_listings, mcp__syncgtm__linkedin_page_jobs, mcp__syncgtm__linkedin_employee_finder, mcp__syncgtm__search_company_by_techstack, mcp__syncgtm__enrich_linkedin_page, mcp__syncgtm__premium_linkedin_page_insights, mcp__syncgtm__linkedin_page_posts, mcp__syncgtm__job_openings_growth_rate, mcp__syncgtm__head_count_growth_rate, mcp__syncgtm__check_job_change, mcp__syncgtm__check_promotions, Read, Edit, Write, Bash(python:*), Bash(python3:*)
---

# sync-mcp — SyncGTM MCP Enrichment Skill

Use SyncGTM MCP tools to enrich contacts and companies from a CSV, a single row, or a list. All tools use waterfall enrichment across Apollo, RocketReach, LeadMagic, Datagma, and others automatically.

---

## CSV First — Always Work From a File

**I work better with a CSV.** Before starting any enrichment or list-building task, always ask the user to provide or create a CSV:

> "I work better with a CSV — can you create one (or share the file you have) so I can track results, skip already-enriched rows, and write back in batches? Even a single-column CSV with company names/domains/LinkedIn URLs is enough to get started."

Only skip this prompt if the user explicitly says they want a one-off single-entity lookup and don't need results saved.

---

**Server prefix:** `mcp__syncgtm__`  
**Credit reference:** `syncgtm-mcp-tools.txt` at repo root.

---

## Pre-flight — Run This Before Every Enrichment

### 1. Check Credits

Always call `mcp__syncgtm__check_credits` before any paid operation. If balance is below the estimated cost, report the deficit and ask the user before continuing.

**Cost estimate formula:** `N rows × cost per tool`. If estimated cost > 50 credits, show the breakdown and ask for confirmation.

### 2. Column Existence Guard

Before enriching a column, read the CSV headers and scan the data:

- If the **target column already exists** → skip rows where that cell is **non-empty**. Never overwrite existing data.
- If the target column does not exist → append it.
- Report: "X rows already have [column] — skipping those. Enriching Y rows."

### 3. Identifier Check

Different tools require different identifiers. Check what's available before calling:

| Needed for | Minimum required | Fallback |
|---|---|---|
| `find_work_email`, `find_personal_email`, `find_work_phone`, `find_mobile_number`, `linkedin_profile_enrich`, `linkedin_profile_posts` | `linkedin_url` | None — must have LinkedIn URL |
| `enrich_person` | Any one of: `first_name`, `last_name`, `email`, `linkedin_url`, `organization_name` | — |
| `verify_email` | `email` | — |
| `validate_whatsapp` | `mobile` | — |
| `find_linkedin_from_work_email` | `email` (work) | — |
| `find_linkedin_from_personal_email` | `email` (personal) | — |
| `enrich_organization`, `find_company_techstack`, `find_company_website_traffic` | `domain` | — |
| `search_company_by_techstack` | technology name(s) (no domain needed) | — |
| `find_people_within_company`, `company_job_listings` | `domain` or company name | — |
| `search_job_listings` | job title / keywords (no domain needed) | — |
| `linkedin_page_jobs`, `linkedin_employee_finder` | LinkedIn company URL or name | — |
| `enrich_linkedin_page`, `linkedin_page_posts`, `premium_linkedin_page_insights`, `job_openings_growth_rate`, `head_count_growth_rate` | LinkedIn company URL or name | — |

**LinkedIn URL gate:** If the user asks for email/phone/profile enrichment and `linkedin_url` column is missing or empty for the target rows — **do not silently skip**. Ask:

> "LinkedIn URL is required for this tool but is missing for [N] rows. Want me to find LinkedIn profiles from their work email first? (`find_linkedin_from_work_email` — 2 credits/row)"

---

## Execution Model — Parallel Batches & Retries

Applies to **every CSV/table enrichment** — any mode that loops over rows. Single-row or one-off list requests run directly, no batching.

### Parallelism — 20 rows per batch

When a CSV or table is attached for enrichment, process rows in **parallel batches of 20**:

- Run pre-flight first (credits → column guard → identifier check). Build the to-do list = rows that pass the guard and have the required identifier.
- Fire **up to 20 MCP tool calls concurrently** — one per row, all in a single message (20 tool_use blocks).
- Wait for the whole batch to return, write the 20 results back to the CSV, then start the next batch of 20.
- Repeat until the to-do list is exhausted.

### Retry — up to 4 attempts, 10s apart

Most failures are provider **rate limits**. Any tool call that errors, times out, or returns a rate-limit signal is **retried up to 4 times with a 10-second gap between attempts** before the error is surfaced:

```
attempt 1 → fail → wait 10s → retry 1 → fail → wait 10s → retry 2
→ wait 10s → retry 3 → wait 10s → retry 4 → still failing → throw to user
```

- Retry only the **failed call(s)**, not the whole batch. Rows that already succeeded are written back regardless.
- A null / empty result is **not** an error — it means "not found." Leave that cell blank, mark `Enrichment Failed` in `Enrichment Status`, do **not** retry.
- Only after the 4th retry still fails do you stop and report the error to the user (with completed-row count). Auth / insufficient-credit errors are systemic — don't burn retries; report immediately.

---

## Mode Selection

| User says | Run section |
|---|---|
| "find work email", "enrich emails", "get emails" | Contact Enrichment → Work Email |
| "find personal email" | Contact Enrichment → Personal Email |
| "find phone", "work phone" | Contact Enrichment → Work Phone |
| "find mobile", "mobile number" | Contact Enrichment → Mobile |
| "verify email", "check email validity" | Contact Enrichment → Verify Email |
| "check WhatsApp", "validate WhatsApp" | Contact Enrichment → WhatsApp |
| "enrich person", "full contact enrichment" | Contact Enrichment → Enrich Person |
| "find LinkedIn from email", "LinkedIn lookup" | Reverse Lookup |
| "enrich LinkedIn profile" | Reverse Lookup → Profile Enrich |
| "scrape LinkedIn posts", "get LinkedIn posts", "person posts" | LinkedIn Profile Posts |
| "get post commenters", "who commented", "commenters" | LinkedIn Post Commenters/Engagers |
| "get post engagers", "who liked", "engagers" | LinkedIn Post Commenters/Engagers |
| "enrich company", "company data" | Company Enrichment |
| "find tech stack", "techstack", "what tech does X use" — **has domain** | Company Enrichment → `find_company_techstack` (STRICTLY) |
| "build company list by tech", "find companies using X tech", "outbound list by technology" | Company Enrichment → `search_company_by_techstack` (STRICTLY) |
| "website traffic", "company traffic" | Company Enrichment → Traffic |
| "find people at company", "employees", "people in company" | Company Enrichment → People |
| "find decision makers", "decision makers", "who to contact" — **from LinkedIn** | Find Decision Makers → `linkedin_employee_finder` |
| "find decision makers", "employees" — **company has domain** | Find Decision Makers → `find_people_within_company` |
| "job listings", "hiring signals", "open roles" — **company has domain** | `company_job_listings` (STRICTLY) |
| "build lead list from job listings", "find companies hiring for X" | `search_job_listings` (STRICTLY) |
| "job listings from company LinkedIn", "company LinkedIn jobs" | `linkedin_page_jobs` (STRICTLY) |
| "google maps", "maps listings", "local businesses" | Google Maps Scraper |
| "LinkedIn company page", "company page data" | LinkedIn Company Page |
| "company posts", "LinkedIn page posts" | LinkedIn Company Page → Posts |
| "headcount growth", "hiring growth" | LinkedIn Company Page → Growth Signals |
| "check job change", "detect job change" | Job / Signal Checks |
| "check promotions", "who got promoted" | Job / Signal Checks |
| "check credits", "how many credits" | Run `mcp__syncgtm__check_credits` |

If the request is ambiguous, ask which mode before proceeding.

---

## Contact Enrichment

### Work Email — 1 credit/row

**Tool:** `mcp__syncgtm__find_work_email`  
**Requires:** `linkedin_url`  
**Output column:** `Work Email`

```
Pre-flight → check_credits → column guard (skip if Work Email already filled) → LinkedIn gate
Process target rows in parallel batches of 20 (see Execution Model):
  For each batch of ≤20 rows with linkedin_url and empty Work Email:
    fire 20 find_work_email(linkedin_url=..., first_name=..., last_name=..., organization_name=...) calls concurrently (one message, 20 tool calls)
    apply the retry policy to any call that errors / rate-limits (up to 4 retries, 10s gaps)
    write all returned results to the Work Email column
    write the batch back to CSV before starting the next batch
```

After finding emails, offer to verify them: "Want me to verify deliverability? (0.3 credits/email)"

---

### Personal Email — 3 credits/row

**Tool:** `mcp__syncgtm__find_personal_email`  
**Requires:** `linkedin_url`  
**Output column:** `Personal Email`

Same flow as work email. Higher cost — confirm if > 10 rows: "This will cost ~[N×3] credits. Continue?"

---

### Work Phone — 15 credits/row

**Tool:** `mcp__syncgtm__find_work_phone`  
**Requires:** `linkedin_url`  
**Output column:** `Work Phone`

High cost. Always confirm: "Finding work phones costs 15 credits/row. [N] rows = [N×15] credits. Continue?"

---

### Mobile Number — 15 credits/row

**Tool:** `mcp__syncgtm__find_mobile_number`  
**Requires:** `linkedin_url`  
**Output column:** `Mobile`

High cost. Always confirm before running. After finding mobiles, offer WhatsApp validation: "Want me to check which numbers are on WhatsApp? (0.5 credits/number)"

---

### Verify Email — 0.3 credits/row

**Tool:** `mcp__syncgtm__verify_email`  
**Requires:** `email`  
**Output columns:** `Email Verified` (true/false), `Email Status` (valid/invalid/risky/unknown)

Run after any email enrichment, or standalone when user has emails already. Skip rows where `Email Verified` is already filled.

---

### Validate WhatsApp — 0.5 credits/row

**Tool:** `mcp__syncgtm__validate_whatsapp`  
**Requires:** `mobile`  
**Output column:** `WhatsApp Valid` (true/false)

Check `Mobile` column exists and is non-empty before running. Skip rows where `WhatsApp Valid` is already filled.

---

### Enrich Person (Full Profile) — 2 credits/row

**Tool:** `mcp__syncgtm__enrich_person`  
**Requires:** at least one of `first_name`, `last_name`, `email`, `linkedin_url`, `organization_name`  
**Output columns:** write whatever fields are returned that don't already have values — Title, Company, LinkedIn URL, Work Email, Industry, City, Country, etc.

Use this when the user needs broad contact data, not just a single field. It returns a richer payload than targeted tools.

Column mapping from response:
| Response field | CSV column |
|---|---|
| title / job_title | Title |
| organization_name | Company Name |
| linkedin_url | Person LinkedIn Url |
| email | Work Email |
| phone | Work Phone |
| city | City |
| country | Country |
| industry | Industry |

Never overwrite cells that already have data.

---

## Reverse Lookup — Email → LinkedIn

### From Work Email — 2 credits/row

**Tool:** `mcp__syncgtm__find_linkedin_from_work_email`  
**Requires:** `email`  
**Output column:** `Person LinkedIn Url`

Use when the CSV has emails but is missing LinkedIn URLs. Enables all other linkedin_url-gated tools.

### From Personal Email — 2 credits/row

**Tool:** `mcp__syncgtm__find_linkedin_from_personal_email`  
**Requires:** `email` (personal), optional `work_email`  
**Output column:** `Person LinkedIn Url`

### LinkedIn Profile Enrich — 1 credit/row

**Tool:** `mcp__syncgtm__linkedin_profile_enrich`  
**Requires:** `profile_url` (the LinkedIn URL)  
**Output columns:** Title, Company, Work Experience (most recent), Education, Skills

Use when the user wants structured profile data beyond what `enrich_person` returns (full work history, education, skills).

---

## LinkedIn Profile Posts

**Tool:** `mcp__syncgtm__linkedin_profile_posts`  
**Cost:** 1 credit/row  
**Requires:** `linkedin_url` (person's LinkedIn profile URL)  
**Returns:** up to 10 most recent posts with content, engagement, timestamps

### Output Column Naming

Create one column per post:

| Column | Content |
|---|---|
| `Post 1` | Full text of post 1 (truncate at 500 chars, append `[truncated]` if cut) |
| `Post 2` | Full text of post 2 |
| … | … |
| `Post 10` | Full text of post 10 |

If fewer than 10 posts exist, leave remaining Post N columns empty (or omit if user hasn't asked for a fixed width).

**Engagement columns** — only add these if the user explicitly asks for engagement data:
- `Post 1 Likes`, `Post 1 Comments`, `Post 1 Shares` (repeat per post index)

**Pre-flight guard:** Check if `Post 1` column already exists and is filled → skip those rows.

**LinkedIn URL gate:** If `linkedin_url` missing → ask if user wants reverse lookup first.

---

## LinkedIn Post Commenters / Engagers

> **Status:** These tools are marked "coming soon" in syncgtm-mcp-tools.txt. If they are not yet available, report: "LinkedIn Post Commenter/Engager tools are not yet live — check syncgtm-mcp-tools.txt for availability." Suggest the user check back or use the posts data from LinkedIn Profile Posts instead.

**Planned tools:**
- `linkedin_post_commenters` — 0.3 credits/commenter — requires `post_url`
- `linkedin_post_engagers` — 0.3 credits/engager — requires `post_url`

### Pre-flight Gate — Check for Posts First

Before scraping commenters or engagers:

1. Check if `Post 1` through `Post N` columns exist **and contain post URLs or content**.
2. If posts columns are missing or empty → ask:
   > "No LinkedIn posts found in the CSV. Want me to scrape LinkedIn posts first? (`linkedin_profile_posts` — 1 credit/row)"
3. If the user provides a specific post URL directly, use that instead.

### Output Schema

Results are **one row per commenter/engager**, linked to the source post. Append these columns:

| Column | Content |
|---|---|
| `Post URL` | The LinkedIn post URL that was scraped |
| `Commenter Name` | Full name of the person who commented |
| `Commenter LinkedIn` | LinkedIn profile URL of the commenter (if returned) |
| `Comment` | Text of the comment |
| `Comment Date` | Date the comment was posted |
| `Reaction Type` | (Engagers only) Like / Celebrate / Support / Love / Insightful / Curious |

Each source contact row may expand to multiple commenter rows. Clarify with the user how they want the output structured (separate sheet/file vs. expanding the original CSV).

---

## Company Enrichment

### Enrich Organization — 2 credits/row

**Tool:** `mcp__syncgtm__enrich_organization`  
**Requires:** `domain`  
**Output columns:** Industry, Company Size, Annual Revenue, Total Funding, Latest Funding, Technologies, HQ City, HQ Country

Column guard: skip rows where these columns are already filled.

### Find Company Techstack — 1 credit/row

> **STRICT RULE:** If the user has existing domains and wants to check what tech those companies use, use THIS tool ONLY — never `search_company_by_techstack`.

**Tool:** `mcp__syncgtm__find_company_techstack`  
**Requires:** `domain`  
**Output column:** `Tech Stack` (comma-separated list of technologies)

---

### Build Outbound List by Technology — 2 credits/result

> **STRICT RULE:** If the user wants to find companies that use a specific technology (to build a prospecting list), use THIS tool ONLY — never `find_company_techstack`.

**Tool:** `mcp__syncgtm__search_company_by_techstack`  
**Use case:** Technographic prospecting — find companies running a specific stack (e.g. "find companies using Salesforce", "find companies on HubSpot").  
**Requires:** technology name(s)  
**Optional:** `industry`, `company_size`, `location`, `max_results`  
**Cost:** 2 credits/result — always show estimate and confirm if > 25 results.

**Output columns** (one row per company):

| Column | Content |
|---|---|
| `Company Name` | Company name |
| `Domain` | Company website |
| `Industry` | Company industry |
| `Company Size` | Employee range |
| `Technologies` | Full tech stack returned |
| `HQ Country` | Headquarters country |

After running, offer: "Want me to find decision makers at these companies? I can filter by your ICP title."

---

### Company Website Traffic — 1 credit/row

**Tool:** `mcp__syncgtm__find_company_website_traffic`  
**Requires:** `domain`  
**Output columns:** `Monthly Visits`, `Traffic Growth`

### Find People Within Company — 0.5 credits/result

**Tool:** `mcp__syncgtm__find_people_within_company`  
**Requires:** `domain`  
**Optional:** `title`, `company_name`, `first_name`, `last_name`, `country`, `max_results` (default 5)

Returns a list of people. Each result becomes its own row. Output as a new CSV or append rows to the existing file — ask the user which they prefer.

**Output columns** (one row per person returned):

| Column | Content |
|---|---|
| `First Name` | Person's first name |
| `Last Name` | Person's last name |
| `Title` | Current job title |
| `Person LinkedIn Url` | LinkedIn profile URL |
| `Company Name` | The company that was searched |
| `Website` | Company domain used for the lookup |

After outputting results, ask: "Want me to find decision makers at these companies? I can filter by ICP title/role."

---

### Find Decision Makers

Triggered when user says "find decision makers", "who should I contact", "find the right person", "find employees", or after job listings / people search surfaces target companies.

**Tool selection is STRICT — do not mix these up:**

| Signal | Tool |
|---|---|
| User has a **domain / website** for the company | **STRICTLY** `mcp__syncgtm__find_people_within_company` |
| User has a **LinkedIn company URL or name** | **STRICTLY** `mcp__syncgtm__linkedin_employee_finder` |

**Workflow:**

1. **Determine identifier type** — does the user/CSV have a domain or a LinkedIn company URL?

2. **Ask who their ICP is:**
   > "Who is your ideal contact at these companies? (e.g. VP of Sales, Head of RevOps, CTO, Founder) — I'll filter results by title."

3a. **If domain available → `mcp__syncgtm__find_people_within_company`**  
   **Requires:** `domain`  
   **Pass:** `title=[ICP title]`, `max_results=5` (or ask user how many per company)

3b. **If LinkedIn company URL/name available → `mcp__syncgtm__linkedin_employee_finder`**  
   **Requires:** LinkedIn company URL or name  
   **Pass:** `title=[ICP title]`, `max_results=5`

4. **Output columns** (one row per result):

| Column | Content |
|---|---|
| `First Name` | Decision maker's first name |
| `Last Name` | Decision maker's last name |
| `Title` | Their current title |
| `Person LinkedIn Url` | LinkedIn profile URL |
| `Company Name` | Company |
| `Website` | Domain used |

5. After outputting, offer next steps:
   > "Decision makers found. Want me to: (a) find their work emails, (b) enrich their LinkedIn profiles, or (c) build personalized outreach with sync-gtme?"

---

### Company Job Listings — 0.5 credits/result

> **STRICT RULE:** If the user asks for job listings of a company **that has a domain**, use THIS tool ONLY — never `linkedin_page_jobs` or `search_job_listings`.

**Tool:** `mcp__syncgtm__company_job_listings`  
**Requires:** company domain or name  
**Optional:** `company_linkedin_url`, `workplace_type`, `max_results`, `job_titles_include`, `job_titles_exclude`, `posted_after`, `posted_before`, `keywords`, `remote`

**Output columns** (one row per job listing):

| Column | Content |
|---|---|
| `Job Title` | Title of the open role |
| `Job URL` | Direct URL to the job posting |
| `Job Description` | Full or truncated job description (500 chars, `[truncated]`) |
| `Hiring Signal` | yes (has active listings) / no |
| `Job Dept` | Department / function (Engineering, Sales, etc.) |
| `Posted Date` | When the job was posted |

After running, ask: "Want me to find decision makers at these companies? I can look for the hiring manager or the buyer persona you're targeting."

---

### Build Lead List from Job Listings — Search Across Companies

> **STRICT RULE:** If the user wants to **build a lead list by searching job listings** (e.g. "find companies hiring SDRs", "find companies that have open roles in RevOps"), use THIS tool ONLY — never `company_job_listings`.

**Tool:** `mcp__syncgtm__search_job_listings`  
**Use case:** Prospect discovery — find companies actively hiring for roles that signal a buying need (e.g. "Hiring a VP of Sales" → likely needs sales tools).  
**Optional filters:** `job_title`, `keywords`, `company_size`, `industry`, `location`, `posted_after`, `max_results`

**Output columns** (one row per matching job listing):

| Column | Content |
|---|---|
| `Company Name` | Company with the open role |
| `Domain` | Company website |
| `Job Title` | Title of the open role |
| `Job URL` | Direct link to the posting |
| `Job Description` | Truncated description (500 chars, `[truncated]`) |
| `Posted Date` | When the role was posted |
| `Location` | Job location |

After running, offer: "Want me to find decision makers at these companies? I can filter by your ICP title."

---

### LinkedIn Company Page Job Listings

> **STRICT RULE:** If the user wants to find job listings via a **company's LinkedIn page** (has a LinkedIn company URL or name, not a domain), use THIS tool ONLY — never `company_job_listings`.

**Tool:** `mcp__syncgtm__linkedin_page_jobs`  
**Requires:** LinkedIn company URL or company name  
**Optional:** `max_results`, `job_titles_include`, `posted_after`, `keywords`, `remote`

**Output columns** (one row per job listing):

| Column | Content |
|---|---|
| `Job Title` | Title of the open role |
| `Job URL` | Direct URL to the LinkedIn job posting |
| `Job Description` | Truncated description (500 chars, `[truncated]`) |
| `Posted Date` | When the job was posted |
| `Location` | Job location |
| `Workplace Type` | Remote / Hybrid / On-site |

After running, ask: "Want me to find decision makers at this company via their LinkedIn page?"

---

## LinkedIn Company Page

### Enrich LinkedIn Page — 0.5 credits

**Tool:** `mcp__syncgtm__enrich_linkedin_page`  
**Requires:** LinkedIn company ID, URL, or name  
**Output columns:** `Company Description`, `Specialties`, `Employee Count (LinkedIn)`, `Founded Year`

### Premium LinkedIn Page Insights — 2 credits

**Tool:** `mcp__syncgtm__premium_linkedin_page_insights`  
**Requires:** LinkedIn company ID, URL, or name  
**Output columns:** `Headcount Growth %`, `Hiring Trend`, `Employee Distribution`

### LinkedIn Page Posts — 1 credit

**Tool:** `mcp__syncgtm__linkedin_page_posts`  
**Requires:** `company_url` (LinkedIn URL or company name)  
**Optional:** `max_posts` (default 10, max 100)  
**Output columns:** `Company Post 1`, `Company Post 2`, … `Company Post N`

Same truncation rule as profile posts (500 chars, `[truncated]`).

### Job Openings Growth Rate — 2 credits

**Tool:** `mcp__syncgtm__job_openings_growth_rate`  
**Requires:** LinkedIn page URL (**required** — must be the full LinkedIn company page URL, e.g. `https://www.linkedin.com/company/stripe`; domain alone is not sufficient)

**Output columns** — create all of these:

| Column | Content |
|---|---|
| `Job Openings Growth 1y` | % change in open roles over the past 12 months |
| `Job Openings Growth 6m` | % change over the past 6 months |
| `Job Openings Growth 2y` | % change over the past 24 months |
| `Job Openings Growth 1y - Engineering` | 1-year growth for Engineering dept |
| `Job Openings Growth 1y - Sales` | 1-year growth for Sales dept |
| `Job Openings Growth 1y - Marketing` | 1-year growth for Marketing dept |
| `Job Openings Growth 1y - Product` | 1-year growth for Product dept |
| `Job Openings Growth 1y - Operations` | 1-year growth for Operations dept |

Only create department columns for departments returned in the response — leave others blank. If the response doesn't break down by department, note "No department breakdown returned" and leave department columns empty.

**LinkedIn page URL gate:** If the CSV column holds a domain (e.g. `stripe.com`) instead of a LinkedIn URL, ask: "This tool requires a LinkedIn company page URL (e.g. linkedin.com/company/stripe). Should I try to enrich the LinkedIn page URL first using `enrich_linkedin_page`?"

### Head Count Growth Rate — 2 credits

**Tool:** `mcp__syncgtm__head_count_growth_rate`  
**Requires:** LinkedIn page URL (**required** — full LinkedIn company page URL; domain alone is not sufficient)

**Output columns** — create all of these:

| Column | Content |
|---|---|
| `Headcount Growth 1y` | % change in total headcount over the past 12 months |
| `Headcount Growth 6m` | % change over the past 6 months |
| `Headcount Growth 2y` | % change over the past 24 months |
| `Headcount Growth 1y - Engineering` | 1-year headcount growth for Engineering dept |
| `Headcount Growth 1y - Sales` | 1-year headcount growth for Sales dept |
| `Headcount Growth 1y - Marketing` | 1-year headcount growth for Marketing dept |
| `Headcount Growth 1y - Product` | 1-year headcount growth for Product dept |
| `Headcount Growth 1y - Operations` | 1-year headcount growth for Operations dept |

Only create department columns for departments returned in the response. Same LinkedIn URL gate as job openings growth — ask to enrich the LinkedIn URL first if only a domain is available.

---

## Google Maps Scraper

> **Status:** Marked "coming soon" in syncgtm-mcp-tools.txt. If the tool is not yet available, report: "Google Maps scraper is not yet live — check syncgtm-mcp-tools.txt for availability." and stop.

**Planned tool:** `google_maps_listings`  
**Cost:** 0.3 credits/business  
**Requires:** `query` or `place_id`  
**Optional:** `location`, `max_results`, `include_reviews` (bool)

Use for local business prospecting, finding businesses by category/location, or building lead lists from Google Maps search results.

### Output Columns

Create all of the following columns (one row per business result):

| Column | Content |
|---|---|
| `Name` | Business name |
| `Reviews` | Most recent or top review text (first review, truncated at 300 chars) |
| `Stars` | Average star rating (e.g. `4.5`) |
| `Description` | Business description or category from Maps |
| `Address` | Full street address |
| `Phone` | Phone number |
| `Website` | Business website URL |
| `Opening Hours` | Hours of operation (all days, semicolon-separated, e.g. `Mon: 9am–5pm; Tue: 9am–5pm`) |
| `Reviews Count` | Total number of reviews |
| `Images Count` | Number of photos on the listing |
| `Search Page URL` | The Google Maps search URL that returned this result |

**Pre-flight:** Ask the user for the search query and location if not provided (e.g. "plumbers in Austin, TX"). Ask for `max_results` if scraping at scale — estimate cost before running.

---

## Job / Signal Checks

### Check Job Change

**Tool:** `mcp__syncgtm__check_job_change`  
**Output column:** `Job Changed` (yes/no), `New Company`, `New Title`, `Job Change Date`

### Check Promotions

**Tool:** `mcp__syncgtm__check_promotions`  
**Output column:** `Promoted` (yes/no), `New Title`, `Promotion Date`

Use these to identify warm outbound signals before launching a campaign. Integrate with sync-gtme — rows where `Job Changed = yes` map to the "Job changed" outbound signal.

---

## CSV Write-back Pattern

Run **20 rows per batch in parallel**, then write that batch back before starting the next — never hold all enrichment until the end. This way a partial run (error, credit exhaustion, user interrupt) keeps every completed batch. Each call carries its own retry policy (4 retries, 10s gaps).

```python
import csv, time
from concurrent.futures import ThreadPoolExecutor

BATCH_SIZE = 20      # rows enriched concurrently
MAX_RETRIES = 4      # retries after the first attempt
RETRY_GAP = 10       # seconds between attempts

def with_retry(fn, *args):
    """Call fn; on any error retry up to MAX_RETRIES times, 10s apart, then raise."""
    last_err = None
    for attempt in range(MAX_RETRIES + 1):
        try:
            return fn(*args)
        except Exception as e:            # rate limit, timeout, transient provider error
            last_err = e
            if attempt < MAX_RETRIES:
                time.sleep(RETRY_GAP)
    raise last_err                        # all 4 retries exhausted — surface to user

def enrich_csv(path, linkedin_col, target_col, enrich_fn):
    with open(path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        rows = list(reader)
        fieldnames = reader.fieldnames[:]

    if target_col not in fieldnames:
        fieldnames.append(target_col)

    # column guard + identifier check → rows still needing enrichment
    todo = [r for r in rows
            if not r.get(target_col) and r.get(linkedin_col, "").strip()]

    for start in range(0, len(todo), BATCH_SIZE):
        batch = todo[start:start + BATCH_SIZE]

        # up to 20 rows concurrently, each wrapped in the retry policy
        with ThreadPoolExecutor(max_workers=BATCH_SIZE) as pool:
            results = list(pool.map(
                lambda r: with_retry(enrich_fn, r[linkedin_col].strip()), batch))
        for r, result in zip(batch, results):
            r[target_col] = result or ""

        # write back after each batch so partial runs keep completed rows
        with open(path, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
            writer.writeheader()
            writer.writerows(rows)

        print(f"Batch {start // BATCH_SIZE + 1}: {len(batch)} rows written")
```

Save this pattern to `temp/` if the task requires a standalone script. Adapt `enrich_fn` to whichever MCP tool is being called. When enriching by issuing MCP tool calls directly (no script), apply the same shape: 20 concurrent tool calls per message, retry each failure 4× with 10s gaps, write back per batch.

---

## Cost Estimator

Before any bulk run (N > 1 row), compute and display:

```
--- cost estimate ---
tool          : find_work_email
cost per row  : 1 credit
rows to enrich: 47
total cost    : 47 credits
current balance: [from check_credits]
```

If `total cost > 50 credits`, ask for explicit confirmation before proceeding.

---

## Edge Cases

| Situation | Action |
|---|---|
| `linkedin_url` missing for email/phone tools | Ask if user wants reverse lookup (email→LinkedIn) first. Never skip silently. |
| Tool returns null / empty | Leave cell blank, append `Enrichment Failed` to a `Enrichment Status` column |
| Row already has value in target column | Skip that row — never overwrite |
| Credit balance too low | Stop, report how many rows completed, how many remain |
| "Coming soon" tool called | Report unavailability, suggest closest available alternative |
| No `domain` column for company tools | Ask user which column holds the company domain |
| Rate limit / timeout / transient error from provider | Retry that call up to 4 times with 10s gaps (see Execution Model). If still failing after the 4th retry, stop and report completed count + the error to the user. |
| Mixed identifiers (some rows have LinkedIn, some have email only) | Segment rows — run linkedin_url-based tools on rows that have it, run reverse lookup on email-only rows, then re-run |

---

## Quick Reference — All Tools + Costs

| Tool (mcp__syncgtm__) | Cost | Requires | Use for |
|---|---|---|---|
| `check_credits` | free | — | Always run first |
| `enrich_person` | 2cr | any identifier | Full contact data |
| `find_work_email` | 1cr | linkedin_url | Work email |
| `find_personal_email` | 3cr | linkedin_url | Personal email |
| `find_work_phone` | 15cr | linkedin_url | Work phone |
| `find_mobile_number` | 15cr | linkedin_url | Mobile number |
| `verify_email` | 0.3cr | email | Email deliverability |
| `validate_whatsapp` | 0.5cr | mobile | WhatsApp check |
| `find_linkedin_from_work_email` | 2cr | email | LinkedIn from work email |
| `find_linkedin_from_personal_email` | 2cr | email | LinkedIn from personal email |
| `linkedin_profile_enrich` | 1cr | profile_url | Full LinkedIn profile |
| `linkedin_profile_posts` | 1cr | profile_url | Person's recent posts |
| `enrich_organization` | 2cr | domain | Company profile |
| `find_company_techstack` | 1cr | domain | Check tech stack of existing domains (STRICT) |
| `search_company_by_techstack` | 2cr/result | technology name | Build outbound list by technology (STRICT) |
| `find_company_website_traffic` | 1cr | domain | Traffic data |
| `find_people_within_company` | 0.5cr/result | domain | Decision makers/employees when company has domain |
| `linkedin_employee_finder` | 0.5cr/result | LinkedIn company URL or name | Decision makers/employees from LinkedIn |
| `company_job_listings` | 0.5cr/result | domain or name | Job listings when company has domain (STRICT) |
| `search_job_listings` | 0.5cr/result | job title/keywords | Build lead list from job listings (STRICT) |
| `linkedin_page_jobs` | 0.5cr/result | LinkedIn company URL or name | Job listings from company's LinkedIn (STRICT) |
| `enrich_linkedin_page` | 0.5cr | LinkedIn id/URL/name | Company page data |
| `premium_linkedin_page_insights` | 2cr | LinkedIn id/URL/name | Growth trends |
| `linkedin_page_posts` | 1cr | company_url | Company page posts |
| `job_openings_growth_rate` | 2cr | LinkedIn page URL (required) | Job openings 1y/6m/2y + by dept |
| `head_count_growth_rate` | 2cr | LinkedIn page URL (required) | Headcount 1y/6m/2y + by dept |
| `check_job_change` | — | — | Job change signal |
| `check_promotions` | — | — | Promotion signal |
| `google_maps_listings` *(coming soon)* | 0.3cr/business | query or place_id | Local business leads from Maps |
