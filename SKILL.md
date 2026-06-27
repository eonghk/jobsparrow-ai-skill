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
| List all applications | `GET /applications` |
| Get application details (full JD, match analysis, company info, gaps) | `GET /applications/:id` |
| Fill in skill/experience gaps | `POST /applications/:id/enhance` |
| Generate tailored resume | `POST /applications/:id/resume` |
| Generate cover letter | `POST /applications/:id/cover-letter` |
| Iterate on resume/cover letter with feedback | `POST /applications/:id/resume/feedback` or `/cover-letter/feedback` |
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
  - `gaps[]` — `{ category, severity, item, suggestion }` (drive the `/enhance` flow).

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
5. **Read the analysis** — The response carries the full picture (see "Reading Application Detail"): `job_description`, `company_info`, `analysis.match_score` + `recommendation`, `analysis.parsed_requirements`, `analysis.strengths[]`, and `analysis.gaps[]` (each gap has `category`, `severity`, `item`, `suggestion`). Use the match report + company info to ground your tailoring, not just the gaps.
6. **Ask the user** — Show the gaps (especially `critical` and `important` ones) and ask them to fill in details.
7. **Enhance** — `POST /applications/:id/enhance` with the user's answers. Map gap categories to fields:
   - `skill` → `additional_skills: string[]`
   - `experience` → `additional_experience: string`
   - `certification` → `certifications_in_progress: string[]`
   - project-shaped → `relevant_projects: string[]`
   - accomplishments → `additional_achievements: string[]`
   - free-form → `custom_notes: string`
8. **Generate documents** — In parallel:
   - `POST /applications/:id/resume` with `{"style": "professional" | "creative" | "minimal"}`
   - `POST /applications/:id/cover-letter` with `{"tone": "professional" | "enthusiastic" | "formal"}`
   Both are async (~30s). Poll `GET /applications/:id` until `resume_generation_status` and `cover_letter_generation_status` are both `"success"`.
9. **Optional: iterate** — If the user wants changes, use `/resume/feedback` or `/cover-letter/feedback` with `{"feedback": "...", "version": <current>}`. Max 10 versions per document.
10. **Finalize** — `POST /applications/:id/finalize` to lock versions.
11. **Download PDFs** — `GET /applications/:id/resume.pdf` (supports `?template=professional|two-column|elegant|colorful|contemporary`) and `GET /applications/:id/cover-letter.pdf`.
12. **Track status** — `PATCH /applications/:id` with `{"status": "applied"}` when the user actually applies.

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
| 400 — "… already exists … Use the feedback endpoint" | Called `/resume` again after success — switch to `/resume/feedback` |
| 400 — "Invalid version number" | `/feedback` version mismatch — re-read `GET /applications/:id` |
| 400 — "feedback cap reached (max 10)" | Hit iteration limit |
| 401 — "Invalid API key" | Wrong or revoked token |
| 403 — "subscription_required" | User needs a paid plan |
| 404 | Wrong application_id or belongs to another user |

## Important Notes

- **Resume `style` ≠ PDF `template`**: `style` controls the LLM writing tone (professional/creative/minimal), `template` controls the visual layout of the PDF (5 options).
- **Don't poll `/applications/:id` in a tight loop** — generation takes 30s, poll every 5-10s.
- **`/resume` and `/cover-letter` can only be called once** per application. After that, use the `/feedback` endpoints to iterate.
- **`/enhance` requires at least one non-empty field** — empty body returns 400.
- **`/finalize` is idempotent** — safe to retry.
- **PDF download is a binary stream** — save the response body directly to a file.
