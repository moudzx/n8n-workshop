# Customizable Internship Hunter in n8n

An automated job-matching pipeline built on n8n. It ingests remote job postings from three public APIs (Remotive, Remote OK, Jobicy), normalizes them into a single schema, and applies a keyword pre-filter to reduce volume before invoking an LLM. A single batched call to Google Gemini scores every candidate job 0-10 against a configurable applicant profile, flags region-eligibility (screening out postings not open to applicants outside specific countries), and returns structured JSON via a schema-enforced output parser. Matches are persisted to a Google Sheet for tracking, high-scoring roles trigger real-time Telegram alerts, and a daily HTML digest is sent via Gmail. The workflow runs on a fixed schedule with no manual intervention and no paid infrastructure.

Built during a workshop by Khwarzime Lebanon.

<img width="2670" height="840" alt="n" src="https://github.com/user-attachments/assets/a376132a-4aad-40d3-8fe1-51c23b963b1e" />


## Requirements

- n8n account: sign up for the free trial at n8n.io, or download it locally: follow documentation at https://github.com/n8n-io/n8n
- Telegram bot: message @BotFather on Telegram, send `/newbot`, follow the prompts, and it'll give you a token. Telegram bots can't message you first. Open a chat with your new bot and send it literally anything

## Setting it up

1. Import `workflow.json` into n8n (Workflows, Add workflow, Import from File)
2. Open the **My Profile1** node near the start and fill in your real info: your skills, what kind of jobs you want, your Telegram chat ID, your email
3. Add your Gemini API key as a credential and attach it to the **Gemini Chat Model** node
4. Connect your own Google account to the **Job Tracker** (Sheets) and **Gmail Digest** nodes, and point the sheet at a real spreadsheet you've made
5. Connect your Telegram bot token to the **Telegram Alert** node
6. Hit the manual test button and see if a message shows up in your Telegram chat
7. Once it works, switch the workflow to Active so it runs on its own every morning



