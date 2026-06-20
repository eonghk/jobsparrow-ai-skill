# JobSparrow AI Skill

An AI agent skill that lets your AI assistant help you apply for jobs — with tailored resumes, cover letters, and gap analysis powered by [JobSparrow](https://jobsparrow.ai).

## What It Does

- Analyse job descriptions and identify skill gaps
- Tailor your resume to specific job postings
- Generate cover letters matched to the role
- Iterate on documents with AI feedback
- Download styled PDFs (5 resume templates)
- Track application status

## Requirements

- A **JobSparrow account** on Essential, Care, or Premium plan
- A **JobSparrow API key** — create one at [app.jobsparrow.ai/ai-agent](https://app.jobsparrow.ai/ai-agent)

## Installation

### Claude Desktop

1. Download the `SKILL.md` file from this repo
2. Open Claude Desktop → Settings → Customize → Skills
3. Upload the `SKILL.md` file
4. Start a conversation and ask for help applying to a job — Claude will ask for your API key

### OpenClaw

The skill is available as `jobsparrow-api`. If not already active, add it to your skills config.

## How It Works

Once installed, just tell your AI assistant things like:

- "Help me apply for this job: [paste URL or job description]"
- "Tailor my resume for this position"
- "Write a cover letter for this role"
- "What gaps do I have for this job?"

The assistant will guide you through the process step by step.

## Compatibility

Works with any AI agent that supports the SKILL.md format:
- Claude Desktop
- OpenClaw
- Claude Code
- Codex CLI
- Cursor
- And other SKILL.md-compatible agents

## License

MIT
