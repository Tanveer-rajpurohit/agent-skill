# Portable AI Skills + Instructions

A drop-anywhere library of engineering **skills** and a universal **operating
instructions** file (`AGENTS.md` / `CLAUDE.md`) that makes any AI assistant write
code the way you want — without re-explaining yourself every chat.

Built for a Go / TypeScript / React / Next.js / AWS stack, but the skills are
general SDE craft.

## What's here

```
.
├── AGENTS.md      # Universal operating instructions (who you are, stack, standards)
├── CLAUDE.md      # Mirror of AGENTS.md for tools that look for CLAUDE.md
└── skills/
    ├── frontend/                      React / Next.js / TypeScript
    ├── ui-design/                     Visual quality, design systems, layout
    ├── gsap/                          GSAP + ScrollTrigger animation
    ├── go-backend/                    Idiomatic Go services & concurrency
    ├── api-design/                    REST/API contracts, errors, versioning
    ├── testing/                       Go table-driven, Vitest/RTL, integration
    ├── ai-rag-engineering/            RAG pipelines, streaming, citations
    ├── devops-deploy/                 Docker, CI/CD, AWS, rollback-safe deploys
    ├── system-design/                 Scaling & distributed systems
    ├── improve-codebase-architecture/ Refactoring toward deep modules
    └── caveman/                       Token-efficient responses
```

Each skill is a self-contained folder with a `SKILL.md` (and sometimes reference
files). A `SKILL.md` has YAML frontmatter (`name`, `description`) that tells the
AI **when** to load it, followed by the actual guidance and code patterns.

## How to use it in any project

**1. Drop the instructions at the repo root.** Copy `AGENTS.md` (and `CLAUDE.md`)
into the project you're working on. That's the "never re-explain" layer.

**2. Bring the skills you need.** Either:
- copy the whole `skills/` folder in, or
- copy only the relevant skill folders (e.g. just `skills/frontend` for a UI repo).

**3. Works across AI tools** because the format is plain Markdown:

| Tool | Looks for | What to do |
|---|---|---|
| Claude Code | `CLAUDE.md`, `.claude/skills/` | Copy `CLAUDE.md` to root; put skills in `.claude/skills/` |
| Cursor | `AGENTS.md`, `.cursor/rules/` | Copy `AGENTS.md` to root |
| Codex / Gemini CLI | `AGENTS.md` | Copy `AGENTS.md` to root |
| Windsurf / others | varies | Point the tool at `AGENTS.md`, or paste the relevant `SKILL.md` |

For tools without auto-loading, just paste the relevant `SKILL.md` into the chat
when you start a task — the content is the value, the filename is just discovery.

## Keeping a single source of truth

Keep the master copy of this library in one place (e.g. this repo, or
`~/.agent-skills`). When starting a new project, copy in what you need. Update the
master, re-copy. `AGENTS.md` is the source of truth; `CLAUDE.md` mirrors it.

## Authoring new skills

1. `skills/<name>/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: <kebab-case>
   description: >
     One paragraph that lists the exact triggers — topics, phrases, and tasks —
     that should make an AI load this skill.
   ---
   ```
2. Body: lead with the rules, then **real runnable code patterns**, idioms,
   anti-patterns, and trade-offs. Dense over chatty. Numbers over adjectives.
3. Split long material into reference files (`PATTERNS.md`, etc.) and route to
   them from `SKILL.md`, like `skills/system-design` does.
