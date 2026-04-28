# Baxter Character Files

The system prompt is the soul of Baxter. This directory mirrors the character files from the private app repo so they survive any single point of failure, stay visible to future Claude sessions that don't have private repo access, and preserve the archeological record of how Baxter evolved over time.

---

## Canonical Location

The authoritative files live in the **private app repo** at:

```
~/Projects/home-control/WallPanel/WallPanel/Resources/
```

The files here are mirrors. Always edit the canonical files; run `tools/sync-character.sh` to update this directory.

---

## Versioning Discipline

- Files are named `baxter-system-prompt-vN.md`
- **Never replace** an existing version file — bump N instead
- The app (`ClaudeVoiceAssistant.swift`) loads only the latest version
- Old versions stay permanently as character archaeology

| Version | Notes |
|---------|-------|
| v1 | Original Woadhouse prompt — the founding character brief |
| v2 | Three calibrations: wit restraint, expanded length rule, Baxter name/backronym |

---

## Character Iteration Workflow

1. Edit `baxter-system-prompt-vN.md` **or** create `baxter-system-prompt-v(N+1).md` in the private app repo's `Resources/`
2. Update `ClaudeVoiceAssistant.loadSystemPrompt()` to reference the new version name
3. Build the app and test in lived use — evaluate whether the changes feel right in conversation
4. When satisfied, from the app repo root:
   ```bash
   bash tools/sync-character.sh
   ```
5. Both repos updated; character history preserved

---

## Sync Script

`tools/sync-character.sh` in the app repo:

- Copies all `baxter-system-prompt-v*.md` from `WallPanel/WallPanel/Resources/` to this directory
- `git add baxter-character/`
- If changes staged: commits "Mirror Baxter character files" and pushes
- If no changes: exits cleanly without a commit (idempotent)

Run it from the app repo root: `cd ~/Projects/home-control && bash tools/sync-character.sh`
