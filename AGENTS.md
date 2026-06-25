# AGENTS.md — Universal Operating Instructions

> Portable instruction file for **any** AI coding assistant (Claude Code, Cursor,
> Codex, Gemini, Windsurf, Copilot Workspace). Drop this at a repo root, or keep
> the master copy in your skills library and copy it in. `CLAUDE.md` is a mirror
> of this file. **Read this once at the start of every session; do not ask the
> user to re-explain anything written here.**

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
| Observability | Prometheus + Grafana, structured logs |

---

## 2. How to write code (non-negotiable defaults)

These are the standards. Apply them **without being asked**. They are the reason
this file exists.

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
   (See the `improve-codebase-architecture` skill for the full vocabulary.)

8. **Explain the decision, not the syntax.** A short "why this approach over the
   alternative" beats a paragraph narrating what the code obviously does.

9. **Numbers over adjectives.** "Redis ~100K ops/sec vs Postgres ~5K writes/sec,"
   not "Redis is fast." Back claims with real figures.

10. **Teach while you build.** Tanveer uses these skills to *learn SDE-level
    craft*. When you implement something non-trivial, leave a 1–3 line note on
    *why* it's done this way and what the alternative trade-off was — but never at
    the cost of cluttering the code. Insight in prose, clean code in the file.

---

## 3. How to respond

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

---

## 4. Definition of done

Before you call work complete:

- [ ] Code compiles / type-checks (no `any`, no unused, no dead imports).
- [ ] Error paths handled, inputs validated at the boundary.
- [ ] No secrets, no SQL string-concat, no obvious injection/XSS.
- [ ] Matches existing code style and structure.
- [ ] Tests added/updated for new logic where a test harness exists.
- [ ] You stated what you changed, what you verified, and any trade-off taken.
- [ ] If you skipped or stubbed something, you said so explicitly.

---

## 5. The skills library

Specialized skills live in `skills/`. Each is a portable `SKILL.md` (plus
reference files) that any AI can read. Load the relevant one by topic:

| Topic | Skill |
|---|---|
| React / Next.js / TS frontend | `skills/frontend` |
| Visual quality, design systems, layout | `skills/ui-design` |
| GSAP / ScrollTrigger animation | `skills/gsap` |
| Go services, concurrency, idioms | `skills/go-backend` |
| REST/API contracts, versioning, errors | `skills/api-design` |
| Testing (Go table-driven, Vitest/RTL) | `skills/testing` |
| RAG / LLM pipelines, streaming, citations | `skills/ai-rag-engineering` |
| Docker / CI-CD / AWS deploys | `skills/devops-deploy` |
| Scaling, distributed systems, system design | `skills/system-design` |
| Refactoring toward deep modules | `skills/improve-codebase-architecture` |
| Token-efficient replies | `skills/caveman` |

**When a task matches a skill, read that skill's `SKILL.md` first, then apply
it.** Skills encode the "how Tanveer wants it done" so this file stays short.
