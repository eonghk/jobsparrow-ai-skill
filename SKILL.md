---
name: jobsparrow
description: >
  JobSparrow AI career assistant — help the user apply for jobs with tailored resumes and cover letters.
  Use when the user wants to: apply for a job, tailor a resume to a job description, write a cover letter,
  get gap analysis on a job application, download a resume/cover letter PDF, or track application status.
  Requires a JobSparrow API key from a paid account (Essential, Care, or Premium plan).
---

# JobSparrow — AI Career Assistant

You are helping the user apply for jobs using JobSparrow's AI-powered career platform.
JobSparrow analyses job descriptions, identifies skill gaps, tailors resumes and cover letters,
and produces styled PDFs — all through a single API.

## API Key

Before doing anything, check if you have the user's JobSparrow API key.

- If you don't have it, ask the user for it.
- Tell them: "You can create an API key at https://app.jobsparrow.ai/ai-agent — it requires an Essential, Care, or Premium plan."
- The key looks like `jsp_live_...`
- Once provided, use it for all subsequent API calls in this session.
- Do NOT store or log the key beyond this session.

## API Base URL

```
https://jobsparrow.piimobile.com
```

All endpoints are under `/api/v1/public/`.

## Authentication

Every request (except `/help`) needs:

```
Authorization: Bearer <user's API key>
```

## What You Can Do

| Action | How |
|--------|-----|
| Check user's subscription | `GET /profile` |
| Get user's base CV | `GET /cv` |
| Create a job application (from URL or JD text) | `POST /applications` |
| List applications (paginated) | `GET /applications?limit=20&lastKey=...` → `{ items, lastKey }` |
| Get application details (full JD, match analysis, company info, gaps) | `GET /applications/:id` |
| Answer gaps before generating (repeatable) | `POST /applications/:id/enhance` |
| Finish gap-filling, unlock generation | `POST /applications/:id/finish-enhancing` |
| Generate tailored resume | `POST /applications/:id/resume` |
| Generate cover letter | `POST /applications/:id/cover-letter` |
| Answer the gaps still open after generating | `POST /applications/:id/resume/gap-answers` |
| Iterate on resume/cover letter with feedback (resume: also notes on single lines) | `POST /applications/:id/resume/feedback` or `/cover-letter/feedback` |
| Lock final versions | `POST /applications/:id/finalize` |
| Download resume PDF | `GET /applications/:id/resume.pdf` |
| Download cover letter PDF | `GET /applications/:id/cover-letter.pdf` |
| Update application status | `PATCH /applications/:id` |
| List resume PDF templates | `GET /resume-templates` |
| API docs (no auth) | `GET /help`, `GET /openapi.json`, `GET /docs` |

## Reading Application Detail

`GET /applications/:id` returns far more than the gaps. Use these fields to ground
your tailoring, talking points, and cover letter. They are populated by the
analysis step, so they are empty while `status` is `analyzing` and filled once
`status` reaches `enhancing` (read them then, or after the SSE `update`).

- `job_description` — the full job description text (scraped from the URL, or as
  pasted). The source of truth for what the role wants; quote/reference it.
- `company_info` — researched at application creation:
  `{ description, industry, products_services[], culture_values[], website, size, headquarters }`.
  Use for cover letter tone and culture-fit framing.
- `analysis` — the match report:
  - `match_score` (0-1) and `recommendation` (`strong_match` ≥0.8, `good_match`
    0.6-0.8, `weak_match` 0.4-0.6, `poor_match` <0.4).
  - `parsed_requirements` — `{ required_skills[], preferred_skills[],
    experience_years, education_level, certifications[], key_responsibilities[] }`.
  - `strengths[]` — CV signals that already align with the JD (positive talking points).
  - `gaps[]` — a unified list of "areas to address" (structural gaps + recruiter-eye
    findings): `{ id, category, severity, item, suggestion, source?, remedy?, evidence?,
    user_input?, user_status?, work_eligibility? }`.
    Only ask about items with `remedy: "needs_input"` (or no `remedy`, on older
    applications); items with `remedy: "auto_rewrite"` are fixed automatically during
    resume generation — do NOT ask the user about them. **Never ask about items with
    `work_eligibility: true`** (visa, citizenship, residency): the resume states work
    rights only when the user's own CV or notes do. If the user volunteers their work
    rights, send their words as the answer to that gap. `user_status: "dismissed"` means the
    user said they don't have it; don't ask again.
  - `recruiter_verdict` — one-line recruiter-eye overall take on the CV for this role.
- `resume` — once a resume exists: `current_version`, `content`, `tier` (writing tier of
  that version) and `gap_coverage` (see step 10):
  `{ version, checked, pending, counts, items: [{gap_id, item, severity, status, quote?, question?, answered_adjacent?}], next_question_ids }`.
  - `content` is the text of `current_version`, the JSON Resume document the web renders:
    `basics.summary`, `basics.summaryHighlights[]`, `work[].highlights[]`,
    `projects[].highlights[]`, plus `education`, `skills` and `certificates`. Read it to show
    the user their resume and to copy lines for `line_notes` (step 11); never scrape the PDF.
    It only appears on `GET /applications/:id` (not the list), and while a new version is
    generating it is still the previous one, until `current_version` moves.

If a field is null/missing, the analysis has not finished yet — re-read after
`status` is `enhancing`.

## Typical Workflow

When the user says "help me apply for [job]":

1. **Check subscription** — `GET /profile`. If 403, tell them they need a paid plan.
2. **Check CV exists** — `GET /cv`. If empty, tell them to upload a base CV at app.jobsparrow.ai first.
3. **Create application** — `POST /applications` with either:
   - `{"url": "https://..."}` — JobSparrow scrapes the job page
   - `{"company_name": "...", "job_title": "...", "job_description": "..."}` — user pastes the JD
   Response: `{application_id, status: "analyzing"}`
4. **Wait for analysis** — `GET /applications/:id` until status is `"enhancing"` (analysis takes 30-60s, poll every 5s).
5. **Read the analysis** — The response carries the full picture (see "Reading Application Detail"): `job_description`, `company_info`, `analysis.match_score` + `recommendation`, `analysis.parsed_requirements`, `analysis.strengths[]`, `analysis.recruiter_verdict`, and `analysis.gaps[]`. Use the match report + company info to ground your tailoring, not just the gaps.
6. **Ask the user** — Show only the areas with `remedy: "needs_input"` (especially `critical` and `important` ones) and ask them to fill in details. Skip `remedy: "auto_rewrite"` areas (the generator fixes those) and `work_eligibility: true` areas (never ask about work rights).
7. **Enhance** — `POST /applications/:id/enhance` with the user's answers. An answer to a specific gap goes in `gap_answers: [{"gap_id": "<gaps[].id>", "answer": "<their words>"}]`; gaps they don't have go in `dismissed_gap_ids: [...]` (final). Other material maps to fields:
   - `skill` → `additional_skills: string[]`
   - `experience` → `additional_experience: string`
   - `certification` → `certifications_in_progress: string[]`
   - project-shaped → `relevant_projects: string[]`
   - accomplishments → `additional_achievements: string[]`
   - anything about ONE specific job → `role_notes: [{role_index, role_company, role_position, text}]`
   - free-form that spans no single job → `custom_notes: string`

   **Use `role_notes`, not `custom_notes`, whenever the answer is about a particular job.**
   Unattributed text leaves the resume generator to guess which role a claim belongs to, and a
   wrong guess prints one job's outcomes under another — which reads to a recruiter as the
   candidate overstating themselves. A real customer cancelled over exactly this. Copy
   `role_company` and `role_position` verbatim from the CV's experience entry (the index alone
   is unsafe: re-parsing the CV shifts it). Use `role_index: -1` for a genuinely general note.

   `role_notes` **replaces** the stored set rather than appending, so send the complete list on
   each call — that is what lets you move a note you filed against the wrong role instead of
   ending up with the claim on both. Every other field still accumulates, and you can call
   `/enhance` multiple times to add more before moving on.
8. **Finish enhancing** — `POST /applications/:id/finish-enhancing` (no body) once the user has nothing more to add. This advances status `enhancing` → `prepare_documents` and **unlocks generation**. It is required: `/resume` and `/cover-letter` are rejected while the application is still `enhancing`. Idempotent — safe to call again.
9. **Generate documents** — In parallel:
   - `POST /applications/:id/resume` with `{"tier": "standard"}` — `tier` is optional (see "Writing tiers"); default `standard`. `style` is optional too (default `professional`): it only sets the default PDF template and does not change the wording.
   - `POST /applications/:id/cover-letter` (no body; a cover letter takes no style or tone options)
   Both are async (~30-90s). Poll `GET /applications/:id` until `resume_generation_status` and `cover_letter_generation_status` are both `"success"`.
10. **Second round: what's still open** — About 20s after the resume succeeds (up to ~150s), `resume.gap_coverage` appears on `GET /applications/:id`. While `gap_coverage.pending` is `true`, poll again every 5-10s.
    - Each item is `covered` (with the resume line as `quote`), `auto` (handled by the writer), or `open` (with a `question`).
    - Ask the user **only** about `next_question_ids` — at most 3 at a time, the same 3 the web app shows. Use each item's `question`. Partly met items (below) come last.
    - Send their answers: `POST /applications/:id/resume/gap-answers` with `{"version": <the version you read>, "answers": [{"gap_id": "...", "answer": "..."}], "dismissed_gap_ids": [...]}` (`version` is optional; if sent, a newer version gets a 400 so you re-read the questions). Answers start ONE new version at the same tier (watch `resume_generation_status`); dismiss-only saves without regenerating.
    - When the new version succeeds, read `gap_coverage` again and repeat while the user wants to. Stop when nothing is open or they're done.
    - **"Added, but only partly met"**: an `open` item with `answered_adjacent: true` means their answer is on the resume (`quote`) but only sits next to the requirement (e.g. POS and Xero for "core banking system"). Tell them it was added and is still only partly met; never call it met. Ask again only if they have more.
11. **Optional: iterate** — `/resume/feedback` or `/cover-letter/feedback` with `{"feedback": "...", "version": <current>}`. For the resume, if the user says a specific line is wrong, send `line_notes` instead of (or with) `feedback`:
    ```json
    {"version": 2, "line_notes": [
      {"line": "Led social strategy across 3 venues", "kind": "not_true", "note": "I posted on their accounts"},
      {"line": "Built the monthly stock sheet", "kind": "wrong_role", "role": "Brew & Co"}
    ]}
    ```
    Copy `line` exactly from `resume.content` on `GET /applications/:id` (re-read it after every new version): one sentence of `basics.summary` (not the whole paragraph), one `basics.summaryHighlights` entry, or one `work[].highlights` / `projects[].highlights` bullet. Send `resume.current_version` as `version`. `kind`: `not_true`, `overstated`, `wrong_role` (the writer may move it to the right role) or `reword`. `note` is the user's words; `role` is only needed when the same words appear on two lines. The server writes the instruction, so with notes alone only those lines change. All notes go out as one new version. Max 10 versions per document (gap-answers with answers count too; dismiss-only doesn't).
12. **Finalize** — `POST /applications/:id/finalize` to lock versions.
13. **Download PDFs** — `GET /applications/:id/resume.pdf` (supports `?template=professional|two-column|elegant|colorful|contemporary`) and `GET /applications/:id/cover-letter.pdf`.
14. **Track status** — `PATCH /applications/:id` with `{"status": "applied"}` when the user actually applies.

## Writing tiers

`tier` on `POST /resume` sets how far the writer may move from the user's own wording. The user picks; default to `standard` unless they ask otherwise. Every later version keeps the tier (to change it, start a new application). `resume.tier` shows the current one.

- **precise** — keeps their sentences; only selects, orders and fixes. For senior or regulated roles and strong resumes. Gaps are left for the user to answer.
- **standard** (default) — tightens their material and aims it at the role. Rewritten in October 2026 ("Standard v2"): closer to the user's own words than before, so expect fewer job-ad keywords than older versions.
- **bold** — puts transferable experience forward harder, for thin experience.

Every tier keeps the same facts: no invented employers, titles, dates, degrees, numbers or skills.

## Resume Templates

5 visual templates available for resume PDF download:
- **professional** — Clean, traditional single-column with teal accent (Corporate, Finance, Legal)
- **two-column** — Modern two-column layout (Tech, Engineering, Design)
- **elegant** — Sophisticated serif layout (Executive, Academia, Law)
- **colorful** — Bold color accents (Creative, Marketing, Startups)
- **contemporary** — Minimal with geometric accents (Tech, Modern Industries)

Default is `professional` if not specified.

## Application Status Transitions

The server enforces allowed transitions. Don't guess — if PATCH returns 400, read the error for allowed next states.

```
analyzing → enhancing → prepare_documents → ready_to_apply → applied → interview → offer
                                                                  ↓         ↓         ↓
                                                               withdrawn  failed   withdrawn
```

- `analysis_failed` — terminal, create a new application
- `failed` — terminal
- `withdrawn` — terminal

## Common Errors

| Code | Meaning |
|------|---------|
| 400 — "Provide either url … OR …" | Mixed URL with JD fields, or partial JD |
| 400 — "Application is still being analyzed" | Called `/resume` too early, wait for status `enhancing` |
| 400 — "still in the `enhancing` phase … finish-enhancing" | Called `/resume` or `/cover-letter` before finishing gap-fill — call `POST /finish-enhancing` first |
| 400 — "… already exists … Use the feedback endpoint" | Called `/resume` again after success — switch to `/resume/feedback` |
| 400 — "Invalid version number" | `/feedback` version mismatch — re-read `GET /applications/:id` |
| 400 — "feedback cap reached (max 10)" | Hit iteration limit (dismiss-only `/resume/gap-answers` still works) |
| 400 — "Unknown gap id(s)" | Use `analysis.gaps[].id`; nothing was saved |
| 400 — "line_notes[i]: not a line of resume version N" | Re-read `resume.content` and copy one sentence or bullet exactly from it; add `role` if the words are on several lines |
| 400 — "style must be one of the following values: ..." | Use a listed style, or leave `style` out (defaults to `professional`) |
| 401 — "Invalid API key" | Wrong or revoked token |
| 403 — "subscription_required" | User needs a paid plan |
| 404 | Wrong application_id or belongs to another user |

## Important Notes

- **`tier` ≠ `style` ≠ PDF `template`**: `tier` sets the wording (see "Writing tiers"). `style` (optional, default `professional`) no longer changes the wording; it is stored as the default PDF template. `template` picks the PDF layout (5 options) and overrides it.
- **Don't poll `/applications/:id` in a tight loop** — generation takes 30s, poll every 5-10s.
- **`/resume` and `/cover-letter` can only be called once** per application. After that, use the `/feedback` endpoints to iterate.
- **`/enhance` requires at least one non-empty field** — empty body returns 400. It is repeatable while `enhancing`.
- **`/finish-enhancing` is required before generating** — the application stays in `enhancing` until you call it (it does not auto-advance, because `/enhance` is repeatable). `/resume` and `/cover-letter` are rejected until then. Idempotent.
- **`/finalize` is idempotent** — safe to retry.
- **PDF download is a binary stream** — save the response body directly to a file.
