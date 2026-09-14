# Customizable Internship Hunter in n8n

An automated job-matching pipeline built on n8n. It ingests remote job postings from three public APIs (Remotive, Remote OK, Jobicy), normalizes them into a single schema, and applies a keyword pre-filter to reduce volume before invoking an LLM. A single batched call to Google Gemini scores every candidate job 0-10 against a configurable applicant profile, flags region-eligibility (screening out postings not open to applicants outside specific countries), and returns structured JSON via a schema-enforced output parser. Matches are persisted to a Google Sheet for tracking, high-scoring roles trigger real-time Telegram alerts, and a daily HTML digest is sent via Gmail. The workflow runs on a fixed schedule with no manual intervention and no paid infrastructure.

Built during a workshop by Khwarzime Lebanon.

<img width="2670" height="840" alt="n" src="https://github.com/user-attachments/assets/a376132a-4aad-40d3-8fe1-51c23b963b1e" />


## Requirements

- n8n account: sign up for the free trial at n8n.io, or download it locally: follow documentation at https://github.com/n8n-io/n8n
- Telegram bot: message @BotFather on Telegram, send `/newbot`, follow the prompts, and it'll give you a token. Telegram bots can't message you first. Open a chat with your new bot and send it literally anything


## 🕵️ STAGE 1 — Fetch & Clean Jobs

### 1. Run Manually (`manualTrigger`)
Starting point for manual execution.

### 2. Every Morning 8:00 (`scheduleTrigger`)
Runs daily at 08:00 (Asia/Beirut).

### 3. My Profile (`set`)
Central config. Fields:

| Field | Value |
|---|---|
| `name` | Your Name |
| `email` | you@example.com |
| `telegram_chat_id` | PASTE_YOUR_CHAT_ID |
| `level` | 3rd-year Computer Science student in Lebanon, looking for an internship or junior role |
| `skills` | Java, Python, OOP, Data Structures, SQL, Git, HTML, CSS, JavaScript, basic React, Linux |
| `interests` | backend, web development, data, AI and automation |
| `keywords` | intern, internship, junior, entry level, graduate, trainee, java, python, javascript, typescript, react, node, backend, frontend, full stack, sql, data, ai, machine learning, automation, qa |
| `max_jobs_for_ai` | 20 |
| `min_score` | 6 |

### 4–6. Remotive Pipeline
- **Remotive API** — `GET https://remotive.com/api/remote-jobs?category=software-dev&limit=60`
- **Split Remotive Jobs** — splits `jobs` array
- **Clean Remotive** — normalizes fields:

| Field | Expression |
|---|---|
| source | `Remotive` |
| title | `$json.title` |
| company | `$json.company_name` |
| location | `$json.candidate_required_location` or `Not specified` |
| tags | `$json.tags.join(', ')` |
| posted | first 10 chars of `publication_date` |
| url | `$json.url` |
| description | HTML-stripped, entity-decoded, whitespace-collapsed, sliced to 600 chars |

### 7–8. Remote OK Pipeline
- **Remote OK API** — `GET https://remoteok.com/api` with `User-Agent: Mozilla/5.0 (n8n student workshop)`
- **Clean Remote OK** — normalizes:

| Field | Expression |
|---|---|
| source | `Remote OK` |
| title | `$json.position` |
| company | `$json.company` |
| location | `$json.location` or `Worldwide` |
| tags | `$json.tags.join(', ')` |
| posted | first 10 chars of `$json.date` |
| url | `$json.url` or `$json.apply_url` |
| description | HTML-stripped, sliced to 600 chars |

### 9–11. Jobicy Pipeline
- **Jobicy API (EMEA)** — `GET https://jobicy.com/api/v2/remote-jobs?count=50&geo=emea`
- **Split Jobicy Jobs** — splits `jobs` array
- **Clean Jobicy** — normalizes:

| Field | Expression |
|---|---|
| source | `Jobicy` |
| title | `jobTitle` with `&#8211;`→`-`, `&amp;`→`&` |
| company | `companyName` |
| location | `jobGeo` or `EMEA` |
| tags | `jobIndustry` joined |
| posted | first 10 chars of `pubDate` |
| url | `$json.url` |
| description | HTML-stripped from `jobExcerpt`/`jobDescription`, sliced to 600 chars |

### 12. Combine Sources (`merge`, append, 3 inputs)
Merges the three cleaned streams.

### 13. Smart Filter (`code`)
Runs **before AI** to save tokens/rate limits:
1. Drops broken items (missing url/title)
2. Removes duplicates (same URL)
3. Keeps jobs matching keywords (word-boundary regex)
4. Ranks by number of keyword hits
5. Returns only top `max_jobs_for_ai` (20)

Adds fields: `keyword_hits`, `keywords_found`.

---

## 🧠 STAGE 2 — AI Brain

### 14. Pack Jobs for AI (`code`)
Collapses N job items into **ONE item** so the AI scores all in a single request (avoids Gemini free-tier rate limits). Builds `jobs_text` with format:

```
[id] Title @ Company
Location: X | Source: Y | Tags: Z
description
---
```

### 15. AI Recruiter (`chainLlm`)
- **System message:** "You are a brutally honest but kind tech recruiter. You help university Computer Science and Engineering students in Lebanon find internships and junior jobs they can REALLY get. You always answer in valid JSON."
- **Task:** Evaluate every job and return:
  - `id` — bracket number
  - `score` — 0–10 fit (senior/lead/4+ yrs must score ≤3)
  - `lebanon_ok` — true if Worldwide/Anywhere/EMEA/MENA/Middle East/Lebanon/no restriction
  - `reason` — one short punchy sentence
  - `matched_skills` — max 4
  - `missing_skills` — 1–3

### 16. Gemini (`lmChatGoogleGemini`)
- Model: `models/gemini-2.5-flash`
- Temperature: `0.2`

### 17. Scores Format (JSON) (`outputParserStructured`)
Example schema:
```json
{
  "matches": [
    {
      "id": 0,
      "score": 8,
      "lebanon_ok": true,
      "reason": "Junior React role open worldwide - strong match for your JS skills",
      "matched_skills": ["JavaScript", "React"],
      "missing_skills": ["TypeScript", "Testing"]
    }
  ]
}
```

### 18. Join Scores + Jobs (`code`)
Attaches AI score to each original job (matched by `id`). Output fields:
`date_found`, `score`, `title`, `company`, `location`, `lebanon_ok` (YES/NO), `reason`, `matched_skills`, `missing_skills`, `source`, `url`. Sorted by score descending.

### 19. Good Matches Only (`filter`)
Keeps jobs where:
- `score >= min_score` (6)
- `lebanon_ok == "YES"`

---

## 🚀 STAGE 3 — Deliver

### 20. Save to Job Tracker (`googleSheets`)
- Operation: `appendOrUpdate`
- Matching column: `url` (prevents duplicates)
- Document/Sheet: **empty — must be configured**

### 21. Hot Jobs (8+) (`filter`)
Keeps only `score >= 8`.

### 22. Top 3 Only (`limit`)
Max 3 items.

### 23. Telegram Alert (`telegram`)
Message format:
```
🔥 {score}/10 — {title}
🏢 {company}
📍 {location}

💡 {reason}
📚 Learn next: {missing_skills}

👉 {url}
(via {source})
```
Chat ID from `My Profile.telegram_chat_id`. Attribution disabled.

### 24. Build Email Digest (`code`)
Builds an HTML email with:
- Dark header (`#1f1f2e`) with title + count + date
- One card per job with colored score badge (green ≥8, yellow ≥6, gray <6)
- Title, company · location, reason, matched skills, missing skills, apply link
- Footer crediting n8n + Gemini + sources

Returns `{ to, subject, html }`.

### 25. Send Gmail Digest (`gmail`)
- To: `{{ $json.to }}`
- Subject: `{{ $json.subject }}`
- Type: HTML
- Message: `{{ $json.html }}`

---

## 🔗 Connection Flow

```
Run Manually ──┐
               ├──► My Profile ──┬──► Remotive API ──► Split Remotive Jobs ──► Clean Remotive ──┐
Every Morning ─┘                 ├──► Remote OK API ──────────────────────► Clean Remote OK ──┼──► Combine Sources ──► Smart Filter ──► Pack Jobs for AI ──► AI Recruiter ──► Join Scores + Jobs ──► Good Matches Only ──┬──► Save to Job Tracker
                                 └──► Jobicy API (EMEA) ──► Split Jobicy Jobs ──► Clean Jobicy ──┘                                                                                                                    ├──► Hot Jobs (8+) ──► Top 3 Only ──► Telegram Alert
                                                                                                                                                                                                                            └──► Build Email Digest ──► Send Gmail Digest

Gemini ──► (ai_languageModel) ──► AI Recruiter
Scores Format (JSON) ──► (ai_outputParser) ──► AI Recruiter
```

---

## ⚙️ Settings & Metadata

- `executionOrder`: `v1`
- `timezone`: `Asia/Beirut`
- `active`: `false`
- `templateCredsSetupCompleted`: `false`
- No tags

---

## ✅ Setup Checklist

1. **My Profile:** fill in `name`, `email`, `telegram_chat_id`
2. **Google Sheets node:** select Document ID + Sheet name
3. **Credentials:** Google Gemini API key, Telegram bot token, Gmail OAuth, Google Sheets OAuth
4. **Publish** the workflow to activate the 8:00 AM schedule

---

## 📝 Sticky Notes (in-canvas docs)

1. **Note 1 (Stage 1):** "🕵️ STAGE 1 — Fetch & clean jobs / Remotive + Remote OK + Jobicy (EMEA) → same fields → Smart Filter"
2. **Note 2 (Stage 2):** "🧠 STAGE 2 — AI brain / Pack → Gemini scores every job vs YOUR profile → Join → Good matches (Lebanon-friendly)"
3. **Note 3 (Stage 3):** "🚀 STAGE 3 — Deliver / 📊 Sheet = tracker (no duplicates: matched on url) / 📱 Telegram = top 3 hot jobs (8+) / 📧 Gmail = daily digest / ▶️ Then click **Publish** → runs every morning, forever."

---

## 📄 License

Free to use and adapt for personal or educational purposes.

---

## 🙌 Credits

Built with [n8n](https://n8n.io) + Google Gemini. Job data from [Remotive](https://remotive.com), [Remote OK](https://remoteok.com), and [Jobicy](https://jobicy.com) — always apply via the original link.



