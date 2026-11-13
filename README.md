# Triage Engine — AI screening for unstructured inbound

An n8n workflow that reads a messy, unstructured request (here: a job ad), checks hard blockers first, scores fit on five weighted criteria, and routes the result — saved to Notion with an alert, or rejected with a one-line reason.

I built it to screen my own job search. The same engine works for **candidate screening, inbound lead qualification, or partnership requests** — only the criteria change.

---

## Demo

▶ [Watch the 77-second demo](TriageEngine_final.mp4): a live yes, a live no with its reason, and the 30-case test run.

## Results

| | |
|---|---|
| Real jobs screened | **40+** (live since October 2026) |
| Labelled test set | **28 / 30** verdicts matched my manual judgement |
| Time per item | **~8 min → ~10 sec of my time** (estimate: reading + deciding by hand vs. paste and read the reply; the bot answers in about a minute) |
| Running cost | **€0** (free models via OpenRouter, self-hosted n8n) |

Full test data: [`test_set_30.csv`](test_set_30.csv) (inputs + expected outcome) and [`test_results.csv`](test_results.csv) (expected vs. actual).

---

## How it works

```
Telegram message  ─┐
                   ├─► 1. Gate check (AI)          ── blocked? ──► Telegram: "Not eligible" + reason
Test runner / API ─┘    German level, seniority
                              │ passes
                              ▼
                     2. Score (AI rates, code counts)
                        role fit 0–3 · stage 0–2 · founder proximity 0–2
                        skills 0–2 · location 0–1  →  score 1.0–5.0
                              │
                ┌─────────────┴─────────────┐
           score < 4                    score ≥ 4
     Telegram: low score + reason   3. Save to Notion (+ JD if ≤ 2,000 chars)
                                       Telegram: "Saved" + Notion link
```

**Two entry points:** a Telegram bot for daily use, and an "Execute Workflow" trigger so a separate Test Runner can push a whole labelled dataset through the same logic.

---

## Decisions I made

1. **Cheap check before expensive scoring.** Language and seniority are hard blockers. A short first AI call rejects them before the full CV-vs-JD comparison runs — fewer calls, faster answers.
2. **The AI rates, code does the maths.** The model returns five sub-scores; a Code node clamps each to its range and computes the final score. Models are unreliable at arithmetic, so the total is deterministic.
3. **Every rejection carries a reason.** I can audit the bot in seconds instead of trusting it blindly.
4. **Fail safe, not silent.** If the model returns an empty or broken reply, the job is flagged "AI scoring failed — re-run" instead of crashing the batch. AI nodes retry 3× with a 5-second wait.
5. **€0 by design.** Free models are good enough for a 1–5 triage; the trade-off is occasional empty replies (handled above) and a daily request cap.

---

## What testing found

I wrote 30 fictional job ads with a known correct answer — clear fits, clear misfits, German-language ads, senior roles hiding behind junior titles, a duplicate, a vague two-line ad — and ran them through the workflow.

| Finding | What happened | Status |
|---|---|---|
| Long-JD branch crashed | Ads over 2,000 characters were routed to a "save without JD" branch — which still sent the JD to Notion and failed. | **Fixed** and re-tested (T05, T09 pass) |
| Empty AI replies | A free model occasionally returned nothing; the scoring step crashed. | **Fixed** — retries + safe fallback |
| Vague ads pass | A two-line "startup seeks motivated person" ad scored 4.6 — the model invented a company and role. | Next: minimum-length / missing-info guard |
| Duplicates saved twice | The same job URL sent twice was scored and saved twice (also seen with a real job). | Next: URL check against the log before scoring |
| Score drift between runs | The same SDR ad scored 4.2 on one run and 3.0 on another. | Next: lower temperature; treat 3.6–4.2 as "review" |

Strong cases held up: all 5 language blockers and all 4 senior roles were rejected — including a "Chief of Staff" title that needed 5+ years — and a German-language ad that only required B1 was correctly **not** rejected.

---

## Reuse it for

| Use case | Gate check | Scoring criteria |
|---|---|---|
| Candidate screening | Work permit, location | Role experience, stage fit, skills, salary range |
| Inbound lead triage | Country, company size | ICP fit, budget signal, urgency, decision-maker |
| Partnership requests | Spam, conflict of interest | Audience overlap, reach, effort required |

---

## Setup

1. Import `workflow.json` into n8n (⋯ → Import from file).
2. Add credentials: **OpenRouter** (both model nodes), **Telegram** (trigger + 4 send nodes), **Notion**.
3. Replace the placeholders:
   - `YOUR_TELEGRAM_CHAT_ID` in the 4 Telegram send nodes
   - `YOUR_NOTION_DATABASE_ID` in both Notion nodes — database columns: Company (title), Role, MatchScore (number), Description, CreatedDate (date), JD
   - `YOUR_DATA_TABLE_ID` in the "Insert row" node (an n8n Data table with a `job_url` column)
4. Paste your own profile into the `Master_cv` field of the **InputCvJD** node.
5. Activate, then send a job ad to your bot.

**To run the test set:** import `test_set_30.csv` as an n8n Data table, create a workflow with *Manual Trigger → Data table (Get rows) → Loop Over Items → Execute Workflow (Triage Engine)*, and run it. Free OpenRouter models have a daily request cap, so run 15 at a time.

---

## Stack

n8n (self-hosted, Cloudflare tunnel) · OpenRouter (NVIDIA Nemotron, free tier) · Telegram Bot API · Notion API

*Built by Deep Methaniya — [deepmethaniya.eu](https://www.deepmethaniya.eu) · [LinkedIn](https://www.linkedin.com/in/deepmethaniya)*
