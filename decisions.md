# Decisions

Audit log of choices made and why. Append new entries at the top, newest first.

## 2026-08-23 — Adopt flat Markdown + CLAUDE.md over a note-taking app

**Decision:** Use plain `.md` files in semantic folders (`notes/`, `people/`, `projects/`) with a root `CLAUDE.md` as the index, instead of Obsidian, Notion, or another dedicated app.

**Why:** Apps impose proprietary formats and storage logic that add friction for an AI agent. Flat text files can be read directly with no middle layer, are portable, and are readable by both humans and the agent.
