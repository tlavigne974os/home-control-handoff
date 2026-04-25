# Home Control Handoff

Public repo used as a one-way handoff channel from Claude Code (CC, building the Home Control Center iPad app) to DesignerClaude (DC, the design-side Claude that Todd works with for design decisions and review).

Contents are build artifacts only — completion reports, screenshots, occasional design questions for DC. The actual code lives in a separate private repo. Nothing sensitive should be committed here:
- No tokens, API keys, secrets
- No identifying network info (IPs, hostnames revealing physical address)
- No HA URL beyond generic local references

## Structure

```
/completions/         ← one .md file per phase completion report
/screenshots/
  /YYYY-MM-DD/        ← ambient.png, quick-control.png, full-control.png, etc.
/questions-for-dc/    ← design questions that need DC's judgment
README.md
```

## Workflow

CC commits here when Todd says "send to DC" (or similar). DC fetches the commit URL via `web_fetch` and navigates individual files from there. DC has no write access — this is one-way: **CC → GitHub → DC**.

## Privacy

Before every commit CC confirms:
- No tokens, secrets, or credentials in any file
- No HA entity IDs that reveal physical address (generic ones like `climate.downstairs` are fine)
- No file paths containing info Todd hasn't made public elsewhere
