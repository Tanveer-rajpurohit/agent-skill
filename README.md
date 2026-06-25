# Tanveer's Agent Skill Library

Universal AI coding instructions and skills for Claude Code, agy CLI, 
Cursor, Codex, Gemini CLI — and any other coding agent.

## Structure

```
.
├── AGENTS.md                        ← source of truth (all agents read this)
├── CLAUDE.md                        ← mirrors AGENTS.md (Claude Code specific)
│
├── instructions/                    ← always-on rules, loaded every session
│   ├── file-operations.md           ← built-in file tools only, never bash for files
│   └── clean-code.md                ← naming, functions, constants, comments
│
└── skills/                          ← load on demand, by topic
    ├── frontend/                    ← React, Next.js App Router, Tailwind, TypeScript
    ├── ui-design/                   ← Design systems, visual quality, spacing, color
    ├── gsap/                        ← GSAP + ScrollTrigger animations
    ├── go-backend/                  ← Go services, chi, sqlc, JWT, Redis, scalability
    ├── api-design/                  ← REST/GraphQL contracts, versioning, errors
    ├── code-review/                 ← Staff engineer level bugs + security + perf review
    ├── git-operations/              ← Commits, revert, merge conflicts, history, PR prep
    ├── testing/                     ← Go table tests, Vitest, RTL, mocking strategy
    ├── ai-rag-engineering/          ← RAG pipelines, pgvector, streaming, citations
    ├── devops-deploy/               ← Docker, GitHub Actions, AWS, zero-downtime
    ├── mediasoup-webrtc/            ← WebRTC SFU, MediaSoup, transports, producers
    ├── system-design/               ← Architecture, distributed systems, scale
    ├── improve-codebase-architecture/ ← Refactoring, deep modules, testability
    └── caveman/                     ← Token-efficient ultra-brief mode
```

## How to Use

### For Claude Code
Global setup (applies to all projects):
```powershell
Copy-Item "AGENTS.md" "$env:USERPROFILE\.claude\CLAUDE.md"
Copy-Item -Recurse "instructions" "$env:USERPROFILE\.claude\instructions"
Copy-Item -Recurse "skills" "$env:USERPROFILE\.claude\skills"
```

Project-specific (in any repo):
```bash
cp AGENTS.md ./AGENTS.md
cp CLAUDE.md ./CLAUDE.md
```

### For agy CLI (Antigravity)
```bash
# Global
cp AGENTS.md ~/.gemini/GEMINI.md
cp -r instructions ~/.gemini/instructions/
cp -r skills ~/.agents/skills/

# Project
cp AGENTS.md ./AGENTS.md
```

### For Cursor / Windsurf / Codex
```bash
cp AGENTS.md .cursorrules   # or .windsurfrules
cp -r skills .agents/skills/
```

## Triggering Skills

Skills auto-trigger when Claude/agy reads the AGENTS.md skill table and
matches your request to a skill description. You can also trigger manually:

```
/code-review     ← review current file
/git-operations  ← git help
/mediasoup-webrtc ← WebRTC help
/go-backend      ← Go patterns
```

## Sources

Skills built from:
- Tanveer's production experience (CoNestify WebRTC SFU, LawVriksh RAG, TaxCopilot)
- Tanveer's own MediaSoup beginner guide
- PatrickJS/awesome-cursorrules (clean-code, gitflow, docker, go, tailwind, react-nextjs)
- alirezarezvani/claude-skills (api-design-reviewer)
- garrytan skills (staff engineer code review patterns)
- VoltAgent/awesome-agent-skills (fullstack, Next.js patterns)
