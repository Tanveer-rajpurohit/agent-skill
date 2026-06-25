# AGENTS.md — Universal Operating Instructions

> Portable instruction file for **any** AI coding assistant (Claude Code, Cursor,
> Codex, Gemini CLI, agy, Windsurf, Copilot Workspace). Drop this at a repo root,
> or keep the master copy in your skills library and copy it in. `CLAUDE.md` is a
> mirror of this file. **Read this once at the start of every session; do not ask
> the user to re-explain anything written here.**

---

## 1. Who you are working with

**Tanveer Singh** — Software Engineer. Builds real-time systems, AI-powered
apps, and cloud-native infra. Ships production (500+ concurrent users, AI
workflows for 1,000+ users). Built a custom WebRTC SFU and an interpreted
language in Go from scratch.

**Treat him as a strong mid/senior engineer, not a beginner.** He knows the
fundamentals. Skip 101 explanations unless he asks. When he asks "why," he wants
the *engineering trade-off*, not a definition.

**Default stack (assume these unless the repo says otherwise):**

| Layer | Default |
|---|---|
| Language (backend) | **Go** (idiomatic, stdlib-first) or **Node.js/TypeScript** |
| Language (frontend) | **TypeScript** (never plain JS for new code) |
| Frontend framework | **Next.js (App Router)** + React, **Tailwind CSS** |
| State | Server components first; **Zustand**/Context for client state; Redux only if repo already uses it |
| Animation | **GSAP** (+ ScrollTrigger); Framer Motion for simple React-local transitions |
| Backend (Node) | Node.js, **Prisma** or raw SQL, JWT auth |
| Datastores | **PostgreSQL** primary, **Redis** cache/pubsub, pgvector for embeddings |
| Streaming/queues | **Kafka** for event streaming, Redis Pub/Sub for fanout |
| Realtime | **WebSockets**, **WebRTC (MediaSoup)** for media |
| AI/LLM | RAG pipelines, streaming responses, **citation validation**, Gemini / AWS Bedrock |
| Cloud/infra | **AWS** (EC2, SES, Textract, Bedrock), **Docker** + Compose, Kubernetes, Nginx |
| CI/CD | **GitHub Actions**, Jenkins; rollback-safe deploys |
| Observability | Prometheus + Grafana, structured logs (slog in Go, pino in Node) |

---

## 2. How to write code (non-negotiable defaults)

These are the standards. Apply them **without being asked**.

1. **Write real, runnable, idiomatic code — not pseudocode or sketches.** If you
   show a function, it should compile/run. Imports included. No `// ... rest of
   logic here`. If something is genuinely out of scope, say so in one line, don't
   fake it.

2. **Match the surrounding codebase.** Read neighboring files first. Mirror their
   naming, error handling, import style, and structure. Consistency beats your
   personal preference every time.

3. **Strong typing always.** TypeScript: no `any` (use `unknown` + narrowing).
   Go: explicit error returns, no naked `interface{}` unless justified. Model the
   domain in types so illegal states are unrepresentable.

4. **Errors are values, handle them.** No swallowed errors. Go: `if err != nil`
   with wrapped context (`fmt.Errorf("doing X: %w", err)`). TS: typed errors or
   discriminated `Result` unions at boundaries; never `catch {}` silently.

5. **Production-minded by default.** Think about: the failure path, concurrency,
   the N+1 query, the unbounded loop, the missing index, the retry/idempotency,
   input validation at the boundary. Mention the trade-off you chose and why.

6. **Security is not optional.** Parameterized queries (never string-concat SQL),
   validate/escape all external input, no secrets in code, auth checks at the
   boundary, least privilege. Flag injection/XSS/SSRF risks when you see them.

7. **Small surface, deep modules.** A small interface hiding real behavior beats
   a thin pass-through. Don't add abstraction until something actually varies.

8. **Explain the decision, not the syntax.** A short "why this approach over the
   alternative" beats a paragraph narrating what the code obviously does.

9. **Numbers over adjectives.** "Redis ~100K ops/sec vs Postgres ~5K writes/sec,"
   not "Redis is fast." Back claims with real figures.

10. **Teach while you build.** When you implement something non-trivial, leave a
    1–3 line note on *why* it's done this way and what the alternative trade-off
    was — but never at the cost of cluttering the code.

---

## 3. File Operations — Read the instruction first

**Read `instructions/file-operations.md` before touching any file.**

Short version: **never use bash to read or edit files.** Use built-in file tools:
`read_file`, `edit_file`, `write_file`, `create_file`. Bash is OK for running
lint/build/tests and git operations only.

---

## 4. How to respond

- **Lead with the answer / the code.** No "Great question!", no restating the
  prompt, no filler preamble.
- **Be concise but complete.** Dense, skimmable. Use tables, ASCII diagrams, and
  code over walls of prose.
- **Always surface trade-offs** for any design choice (pros AND cons).
- When the request is ambiguous in a way that changes the output, ask **one**
  sharp question. Otherwise pick the sensible default, state it in one line, and
  proceed.
- Don't hedge on things you can verify — check the code instead of guessing.
- If something is a bad idea, say so directly and propose the better path.
- **Never invent changes** beyond what was asked.
- **Never remove existing code** unless specifically asked.
- Make **one file change at a time** — show it, then continue.

---

## 5. Definition of done

Before you call work complete:

- [ ] Code compiles / type-checks (no `any`, no unused, no dead imports).
- [ ] Error paths handled, inputs validated at the boundary.
- [ ] No secrets, no SQL string-concat, no obvious injection/XSS.
- [ ] Matches existing code style and structure.
- [ ] Tests added/updated for new logic where a test harness exists.
- [ ] You stated what you changed, what you verified, and any trade-off taken.
- [ ] If you skipped or stubbed something, you said so explicitly.

---

## 6. Always-on instruction files

These files are loaded every session. Read them before starting any task:

| File | What it covers |
|---|---|
| `instructions/file-operations.md` | Built-in tools only — never bash for file edits |
| `instructions/clean-code.md` | Names, functions, constants, comments baseline |

---

## 7. The skills library

Specialized skills live in `skills/`. Each is a portable `SKILL.md` that loads
only when relevant — saves context tokens. Load by topic:

| Topic | Skill folder | Triggers |
|---|---|---|
| React / Next.js / Tailwind frontend | `skills/frontend` | component, Next.js page, App Router, re-render, form |
| Visual quality, design system, layout | `skills/ui-design` | "make it look good", design system, spacing, polish |
| GSAP / ScrollTrigger animation | `skills/gsap` | animate, scroll animation, parallax, timeline, stagger |
| Go services, concurrency, scalability | `skills/go-backend` | Go handler, chi route, sqlc, JWT, rate limit, goroutine |
| REST/GraphQL API contracts | `skills/api-design` | design endpoint, status code, versioning, pagination, error format |
| Code review (staff engineer level) | `skills/code-review` | review my code, check this, any issues, audit this |
| Git operations & workflow | `skills/git-operations` | commit, revert, undo, merge conflict, git history, branch, PR |
| Testing (Go + Vitest/RTL) | `skills/testing` | write tests, table-driven, mock, test coverage, brittle test |
| RAG / LLM pipelines / streaming | `skills/ai-rag-engineering` | RAG pipeline, vector search, streaming AI, citation, pgvector, LLM |
| Docker / CI-CD / AWS deploys | `skills/devops-deploy` | Dockerfile, GitHub Actions, deploy to AWS, zero-downtime, Nginx |
| WebRTC / MediaSoup SFU | `skills/mediasoup-webrtc` | WebRTC, SFU, mediasoup, video call, transport, producer, consumer |
| System design & architecture | `skills/system-design` | design a system, scale, distributed, CAP, microservices vs monolith |
| Codebase refactoring | `skills/improve-codebase-architecture` | refactor, architecture review, deep modules, decouple |
| Token-efficient replies | `skills/caveman` | "caveman mode", "less tokens", "be brief" |

**When a task matches a skill, read that skill's `SKILL.md` first, then apply
it.** Skills encode the "how Tanveer wants it done" for that domain.

---

## 8. Permissions — what you can do without asking

**OK without asking:**
- Read any file in the current project
- Edit files using built-in file tools
- Run: `npm run lint`, `npm run build`, `npm run test`
- Run: `go build ./...`, `go test ./...`, `go vet ./...`
- Run: `git status`, `git diff`, `git log`
- Search for files: `find . -name "*.go"`

**Ask before doing:**
- Installing any npm/go package
- Deleting files
- Creating new directories or changing project structure
- Running any network requests from the terminal
- Committing or pushing to git
- Any command that modifies system config

**Never:**
- Access files outside the current project directory
- Use bash to read or write file content (use file tools)
- Run arbitrary scripts you haven't shown to Tanveer
- Access root-level filesystem paths
