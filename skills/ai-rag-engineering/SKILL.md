---
name: ai-rag-engineering
description: >
  Build production-grade AI/LLM-powered features: RAG pipelines, streaming
  responses, agentic workflows, citation validation, OCR ingestion, embedding +
  vector search (pgvector/Pinecone), prompt engineering, Gemini/OpenAI/AWS Bedrock
  APIs, structured output / tool use, retry on hallucination, and real-time AI
  response UX. Trigger for: "build a RAG pipeline", "stream LLM response",
  "citation validation", "AI document editor", "vector search", "embed documents",
  "prompt template", "structured output", "agentic", "tool use/function calling",
  "reduce AI latency", "OCR + AI", GenAI architecture questions, and anything that
  involves plugging an LLM into a real product feature.
---

# AI/RAG Engineering Skill — Production LLM Features

You build LLM features that work in production, not just demos. That means:
streaming UX, latency budgets, deterministic citation validation, retry logic,
cost awareness, and fallback chains. No "just call the API and return the response."

## The RAG pipeline (full architecture)

```
Document Ingestion (offline / on upload):
  Raw file (PDF/docx/scan)
    → OCR if scanned (AWS Textract / pytesseract)
    → Clean + chunk (fixed-size with overlap, or semantic/markdown-aware)
    → Embed chunks (text-embedding-3-small / Gemini embedding-004)
    → Store in vector DB (pgvector / Pinecone) with metadata (source, page, chunk_id)

Query (online, per user request):
  User query
    → Embed query (same model as ingestion — must match)
    → Vector search → top-K chunks (k=5–10, rerank if needed)
    → Build context window: system prompt + retrieved chunks + conversation history
    → LLM call (stream response)
    → Citation validation (verify every reference)
    → Stream to client
```

## Chunking — the part that kills most RAG implementations

```python
# Semantic/markdown-aware chunking (better than fixed-size for structured docs)
from langchain.text_splitter import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

headers = [("#", "h1"), ("##", "h2"), ("###", "h3")]
md_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers)
chunks = md_splitter.split_text(markdown_doc)

# Then size-limit each chunk, with overlap to preserve context across boundaries
char_splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=150)
final_chunks = char_splitter.split_documents(chunks)

# Rules:
# - chunk_size: ~800 chars (~200 tokens) for detail-heavy docs; ~1500 for narrative
# - chunk_overlap: 15–20% of chunk_size (preserves context at boundaries)
# - Always store metadata: source_file, page_number, chunk_index, section_header
# - Embed BOTH the chunk AND a generated question it answers (HyDE) for better retrieval
```

## Embeddings + pgvector (production setup)

```sql
-- Enable extension and create table
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE document_chunks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id   UUID NOT NULL REFERENCES documents(id),
    chunk_text  TEXT NOT NULL,
    metadata    JSONB,
    embedding   vector(1536),    -- dimension matches your embedding model
    created_at  TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);          -- lists ≈ sqrt(row_count); rebuild after bulk inserts
```

```typescript
// Similarity search (cosine — best for normalized embeddings)
async function searchChunks(queryEmbedding: number[], k = 5, sourceId?: string) {
  const vec = `[${queryEmbedding.join(",")}]`
  const filter = sourceId ? `AND source_id = $3` : ""
  const params = sourceId ? [vec, k, sourceId] : [vec, k]
  return db.query(
    `SELECT id, chunk_text, metadata, 1 - (embedding <=> $1::vector) AS score
     FROM document_chunks
     WHERE 1 - (embedding <=> $1::vector) > 0.75   ${filter}
     ORDER BY embedding <=> $1::vector
     LIMIT $2`,
    params
  )
}
// Score > 0.75 threshold filters noise. Tune per domain.
// Hybrid search: combine cosine score with BM25 (text search) for best recall.
```

## Streaming responses (the UX that matters)

**Backend (Node.js/Fastify):**
```typescript
import Anthropic from "@anthropic-ai/sdk"

async function streamCompletion(req: FastifyRequest, reply: FastifyReply) {
  const { prompt, context } = req.body as Body
  reply.raw.setHeader("Content-Type", "text/event-stream")
  reply.raw.setHeader("Cache-Control", "no-cache")
  reply.raw.setHeader("Connection", "keep-alive")

  const stream = await anthropic.messages.stream({
    model: "claude-sonnet-4-6",
    max_tokens: 2048,
    messages: [{ role: "user", content: buildPrompt(prompt, context) }],
  })

  for await (const chunk of stream) {
    if (chunk.type === "content_block_delta" && chunk.delta.type === "text_delta") {
      reply.raw.write(`data: ${JSON.stringify({ text: chunk.delta.text })}\n\n`)
    }
  }
  reply.raw.write("data: [DONE]\n\n")
  reply.raw.end()
}
```

**Frontend (React):**
```tsx
"use client"
import { useState } from "react"

export function AIResponseStream() {
  const [text, setText] = useState("")
  const [loading, setLoading] = useState(false)

  async function ask(prompt: string) {
    setLoading(true)
    setText("")
    const res = await fetch("/api/stream", { method: "POST", body: JSON.stringify({ prompt }) })
    const reader = res.body!.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      const lines = decoder.decode(value).split("\n").filter(l => l.startsWith("data:"))
      for (const line of lines) {
        const data = line.slice(5).trim()
        if (data === "[DONE]") { setLoading(false); return }
        try {
          const { text: delta } = JSON.parse(data)
          setText(prev => prev + delta)
        } catch {}
      }
    }
    setLoading(false)
  }

  return (
    <div>
      <button onClick={() => ask("Summarise the doc")} disabled={loading}>Ask</button>
      <div className="whitespace-pre-wrap font-mono text-sm">{text}</div>
    </div>
  )
}
```
Latency optimization: stream starts in < 500ms, **first token in < 2s**. Use
`max_tokens` budgets. Consider response caching for identical prompts (Redis,
hash the normalized prompt).

## Citation validation — the production requirement

Every legal/financial/research RAG MUST verify references. Never trust the LLM
to cite accurately.

```typescript
interface Citation { section: string; text: string }

async function validateCitations(
  aiResponse: string,
  sourceChunks: { id: string; text: string; metadata: { section: string } }[],
): Promise<{ valid: boolean; invalidCitations: string[] }> {
  // Extract citations the model claimed (e.g. [Section 12(1)(a)] patterns)
  const claimed = [...aiResponse.matchAll(/\[([^\]]+)\]/g)].map(m => m[1])

  const invalid: string[] = []
  for (const ref of claimed) {
    // Check if ANY source chunk actually contains this reference
    const found = sourceChunks.some(
      c => c.metadata.section === ref || c.text.includes(ref)
    )
    if (!found) invalid.push(ref)
  }

  return { valid: invalid.length === 0, invalidCitations: invalid }
}

// In pipeline: if invalid.length > 0, retry with a stricter prompt or
// return partial response + flag to user. Never silently serve hallucinated citations.
async function generateWithValidation(prompt: string, chunks: Chunk[], retries = 2) {
  for (let i = 0; i <= retries; i++) {
    const response = await callLLM(prompt + (i > 0 ? "\nOnly cite sections present in context." : ""), chunks)
    const { valid } = await validateCitations(response, chunks)
    if (valid) return response
  }
  throw new Error("citation validation failed after retries")
}
```

## Structured output / tool use (for agentic flows)

```typescript
// Force the model to return valid JSON matching your schema
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  tools: [{
    name: "extract_entities",
    description: "Extract named entities from the text",
    input_schema: {
      type: "object",
      properties: {
        parties: { type: "array", items: { type: "string" } },
        dates: { type: "array", items: { type: "string" } },
        amounts: { type: "array", items: { type: "number" } },
      },
      required: ["parties", "dates", "amounts"],
    },
  }],
  tool_choice: { type: "tool", name: "extract_entities" },  // force tool use
  messages: [{ role: "user", content: text }],
})

const toolUse = response.content.find(b => b.type === "tool_use")
const entities = toolUse?.input  // guaranteed to match the schema or SDK throws
```
Tool use / function calling = structured output with a type contract. Use it
whenever you need the LLM's output to be machine-readable. Never parse free-text
JSON from the model — it will hallucinate format.

## Prompt engineering rules

```
1. SYSTEM prompt = persona + constraints + output format. Be specific.
2. Inject retrieved context between "---CONTEXT---" delimiters so the model
   knows exactly where retrieved text ends and user input begins.
3. Instruct: "Only use information from the CONTEXT above. If not present, say so."
4. For citations: "Every factual claim must cite [Section X] from the context."
5. Shorter, more specific prompts > long, vague ones. Vague → garbage.
6. Few-shot examples beat instructions for format compliance.
7. Temperature 0.1–0.3 for factual/legal tasks (low hallucination); 0.7–0.9
   for creative. Default 0 for extraction.

Context window management:
  - Prioritize: system + retrieved context + user message. History is last.
  - If history > 6 turns, summarize old turns into one "prior context" block.
  - Track token count; truncate history before context if budget is tight.
```

## Latency budget (what to optimize)

```
Target for an interactive AI feature: < 3s to first meaningful token on screen.

  OCR (if scanned)    → parallelise across pages; cache results
  Embedding query     → < 100ms (fast model, local or cached)
  Vector search       → < 50ms (pgvector with index)
  LLM call            → stream; first token < 1.5s with Sonnet/Gemini Flash
  Citation check      → < 100ms (string matching, not LLM)

Caching layers:
  Embed cache: hash(text) → vector (skip re-embedding same chunk)
  Response cache: hash(prompt + context_ids) → cached response (for repeated queries)
  Both in Redis with TTL matching your freshness requirement.
```

## Fallback chain (Gemini → Bedrock → error)

```typescript
async function callWithFallback(prompt: string, context: string): Promise<string> {
  const models = [
    () => callGemini("gemini-2.0-flash", prompt, context),
    () => callBedrock("anthropic.claude-sonnet-4-6-v1", prompt, context),
  ]
  for (const attempt of models) {
    try { return await withTimeout(attempt(), 30_000) }
    catch (err) { console.error("model attempt failed:", err) }
  }
  throw new Error("all AI providers failed")
}

function withTimeout<T>(p: Promise<T>, ms: number): Promise<T> {
  return Promise.race([p, new Promise<never>((_, rej) => setTimeout(() => rej(new Error("timeout")), ms))])
}
```

## Anti-patterns

```
❌ No streaming → users stare at spinner for 10s for long answers
❌ Fixed chunk size with no overlap → cuts concepts in half at boundaries
❌ Different embedding model for ingestion vs query → garbage retrieval
❌ Trust citations without validation → legal/financial hallucination disaster
❌ Putting all history in context → costs tokens, degrades relevance
❌ Temperature 0.8 for factual extraction → high hallucination rate
❌ No retry / fallback for model errors → single point of failure
❌ Free-text JSON parsing from LLM → use tool use / structured output
❌ Logging PII/document content → GDPR/compliance failure
```

## Response format

1. State the pipeline stage you're addressing (ingestion / retrieval / generation / validation).
2. Give runnable code for that stage with real API calls (not pseudocode).
3. Name the latency + cost trade-off of the approach chosen.
4. Call out any hallucination/citation risk and how it's mitigated.
