# SyncGTM MCP

Reference guide for the [SyncGTM MCP server](https://docs.syncgtm.com/mcp_server/tools) — covers all available tools, credit costs, and real outbound use cases.

---

> **Tip:** Install the [sync-mcp agent](./agents/sync.md) for better formatting and experience.

## Prerequisites

- **SyncGTM account** — logged in at [syncgtm.com](https://syncgtm.com)

---

## Getting Started

For installation guides across different AI clients (Claude Code, Claude Desktop, Cursor, Windsurf, and more), see the [SyncGTM MCP documentation](https://docs.syncgtm.com/mcp_server).

---

## Use Case Examples

**1. Enrich a prospect from LinkedIn URL**
Find work email, job title, company info, and recent posts for a prospect before outreach.
→ `find_work_email` + `linkedin_profile_enrich` + `linkedin_profile_posts`

**2. Build a hiring signal list**
Identify companies actively hiring for RevOps or Sales roles — personalize outreach around growth.
→ `company_job_listings` + `enrich_organization`

**3. Score inbound leads by company size**
Check headcount and headcount growth rate for inbound signups to prioritize enterprise accounts.
→ `head_count_growth_rate` + `enrich_organization`

**4. Verify an email list before sending a cold campaign**
Bulk-validate emails to protect sender reputation before launching a sequence.
→ `verify_email`

**5. Find decision-makers at a target account**
Search for VPs and Directors at a company, then enrich with work emails.
→ `find_people_within_company` + `find_work_email`

**6. Research a company's tech stack before outreach**
Know what tools a prospect uses to tailor your pitch (e.g., "we integrate with HubSpot").
→ `find_company_techstack` + `enrich_organization`

**7. Reverse-lookup LinkedIn from a personal email**
Convert a personal email (from a form submission) into a LinkedIn profile for deeper enrichment.
→ `find_linkedin_from_personal_email` + `linkedin_profile_enrich`

---

## Available Tools

Server prefix: `mcp__syncgtm__`  
Most tools use **waterfall enrichment** — multiple providers tried in sequence until a result is found.

### Account

| Tool | Cost | Description |
|------|------|-------------|
| `check_credits` | Free | Check your remaining credit balance. Call before expensive tools. |

---

### Person Enrichment

| Tool | Cost | Description | Required | Optional |
|------|------|-------------|----------|----------|
| `enrich_person` | 2 credits | Enrich by name, email, LinkedIn URL, or org. Returns contact details, title, company info. | One of: `first_name`, `last_name`, `email`, `linkedin_url`, `organization_name` | Remaining identifiers |
| `find_work_email` | 1 credit | Find work email from LinkedIn URL. | `linkedin_url` | `first_name`, `last_name`, `organization_name` |
| `find_personal_email` | 3 credits | Find personal email from LinkedIn URL. | `linkedin_url` | `first_name`, `last_name`, `organization_name` |
| `find_work_phone` | 15 credits | Find direct work phone number. | `linkedin_url` | `first_name`, `last_name`, `organization_name` |
| `find_mobile_number` | 15 credits | Find mobile phone number. | `linkedin_url` | `first_name`, `last_name`, `organization_name` |
| `verify_email` | 0.3 credits | Verify email is valid and deliverable. | `email` | — |
| `validate_whatsapp` | 0.5 credits | Check if a phone number is registered on WhatsApp. | `mobile` | — |

---

### Reverse Lookup (Email → LinkedIn)

| Tool | Cost | Description | Required | Optional |
|------|------|-------------|----------|----------|
| `find_linkedin_from_work_email` | 2 credits | Find LinkedIn profile from work email. | `email` | — |
| `find_linkedin_from_personal_email` | 2 credits | Find LinkedIn profile from personal email. | `email` | `work_email` |
| `linkedin_profile_enrich` | 1 credit | Enrich a LinkedIn profile — work experience, education, skills. | `profile_url` | — |
| `linkedin_profile_posts` | 1 credit | Get 10 most recent posts and engagement from a LinkedIn profile. | `profile_url` | — |

---

### Company Enrichment

| Tool | Cost | Description | Required | Optional |
|------|------|-------------|----------|----------|
| `enrich_organization` | 2 credits | Enrich company by domain — industry, size, funding, technologies. | `domain` | — |
| `find_company_techstack` | 1 credit | Find the technology stack used by a company. | `domain` | — |
| `find_company_website_traffic` | 1 credit | Find website traffic data for a company. | `domain` | — |
| `find_people_within_company` | 0.5 credits/result | Search for people at a company across multiple providers (merged + deduped). | `domain` | `title`, `company_name`, `first_name`, `last_name`, `country`, `max_results` (default 5) |
| `company_job_listings` | 0.5 credits/result | Find open roles at a company — filter by title, location, department. | `domain` or company name | `company_linkedin_url`, `workplace_type`, `job_titles_include`, `job_titles_exclude`, `posted_after`, `posted_before`, `keywords`, `remote`, `max_results` (default 5) |

---

### LinkedIn Company / Page

| Tool | Cost | Description | Required | Optional |
|------|------|-------------|----------|----------|
| `enrich_linkedin_page` | 0.5 credits | Enrich a LinkedIn company page — description, specialties, employee count. | `identifier` (LinkedIn id, URL, or name) | — |
| `premium_linkedin_page_insights` | 2 credits | Deep company insights — growth trends, hiring patterns, employee distribution. | `identifier` (LinkedIn id, URL, or name) | — |
| `linkedin_page_posts` | 1 credit | Get recent posts from a LinkedIn company page. | `company_url` | `max_posts` (default 10, max 100) |
| `job_openings_growth_rate` | 2 credits | Track growth rate of a company's open job postings over time. | LinkedIn page URL | — |
| `head_count_growth_rate` | 2 credits | Track growth rate of a company's employee headcount over time. | LinkedIn page URL | — |
| `check_job_change` | — | Detect if a person has recently changed jobs. | `linkedin_url` | — |
| `check_promotions` | — | Detect if a person has been recently promoted. | `linkedin_url` | — |

---

### Coming Soon

| Tool | Est. Cost | Description |
|------|-----------|-------------|
| LinkedIn Job Listings | 0.3 credits/result | Search LinkedIn jobs by keyword, location, and filters |
| LinkedIn Post Commenters | 0.3 credits/commenter | People who commented on a LinkedIn post |
| LinkedIn Post Engagers | 0.3 credits/engager | People who reacted to a LinkedIn post |
| Find Companies by Techstack | 1 credit/company | Find companies using a specific technology (via BuiltWith) |
| Google Maps Listings | 0.3 credits/business | Business listings and reviews from Google Maps |
| Headcount by Department | 2 credits | Company headcount filtered by department |
| SemRush Insights | 1 credit | Traffic, top keywords, backlink data for a domain |
| SimilarWeb Insights | 3 credits | Detailed traffic stats and audience insights |
| Meta Ads from Page ID | 1 credit | Active ads running from a Facebook/Meta page |
| Instagram Posts | 0.3 credits/post | Recent posts from an Instagram profile |
| TikTok Posts | 0.3 credits/post | Recent posts from a TikTok profile |
| Revenue Estimate | 1 credit | AI-estimated annual revenue for a company |
| Funding Data | 1 credit | AI-estimated or scraped funding and last round data |
| Education | 1 credit | Person's education history from LinkedIn |
| Previous Job Titles | 1 credit | Person's previous titles and companies from LinkedIn |

---

## Credit Tips

- Always call `check_credits` before running a batch enrichment.
- Phone lookups (`find_work_phone`, `find_mobile_number`) cost 15 credits — use only when needed.
- `find_people_within_company` and `company_job_listings` are billed per result — set `max_results` to control spend.
- Verify emails first (`verify_email` at 0.3 credits) before paying for phone or personal email lookups.
