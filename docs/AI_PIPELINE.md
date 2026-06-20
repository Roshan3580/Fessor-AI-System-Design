# AI Pipeline

Fessor routes work across multiple AI providers depending on the task. The live product orchestrates OpenAI (GPT-4), Claude, Gemini, and Grok (xAI) so that each stage of book creation can use the model best suited to that work.

The pipeline favors modularity and asynchronous execution. Long-running generation steps run in Celery workers; interactive assistant flows use realtime streaming. See [System Design](SYSTEM_DESIGN.md) for request paths and [Async Workflows](ASYNC_WORKFLOWS.md) for worker behavior.

---

## Pipeline Overview

```text
User prompt (+ optional attachments)
        ↓
   Credit check
        ↓
   Planning
        ↓
   Chapter / section generation
        ↓
   Image generation
        ↓
   Quiz / flashcard / exam creation
        ↓
   Validation + persistence
        ↓
   Reader experience
```

Each stage produces structured output that is stored incrementally so the reader can surface progress before the full book is complete.

---

## Pipeline Stages

### 1. Intake

**Input**

- Topic prompt
- Optional generation settings (difficulty, included materials, model preferences)
- Optional file attachments (PDF, image)

**Output**

- Validated generation request
- Reserved or checked credit balance
- Job record enqueued to Celery

**Notes**

The API accepts the request synchronously and returns quickly. Actual model calls happen in workers so HTTP timeouts do not bound generation time.

---

### 2. Planning

**Input**

- User prompt
- Attachment summaries (if present)
- Book-level configuration

**Output**

- Book outline: chapters, sections, learning objectives
- Generation plan used to drive downstream tasks

**Notes**

Planning establishes structure before content writing begins. This reduces incoherent long-form output and gives the frontend a skeleton to display while sections are still generating.

---

### 3. Chapter and Section Generation

**Input**

- Approved outline
- Section-level context (prior sections, book metadata)
- Attachment-derived context where relevant

**Output**

- Structured section content (headings, explanations, examples, learning objectives)
- Incremental database writes per section

**Notes**

Sections are generated as discrete units so partial failure does not require restarting the entire book. Completed sections remain available in the reader while remaining work continues.

---

### 4. Image Generation

**Input**

- Section content and topic context
- Image placement metadata from the generation plan

**Output**

- Generated images stored in Cloud Storage
- References attached to the relevant section or book record

**Notes**

Image generation is treated as a separate async concern so text generation is not blocked by slower image provider latency.

---

### 5. Quiz, Flashcard, and Exam Creation

**Input**

- Final or near-final section content
- Section and chapter boundaries

**Output**

- Multiple-choice and open-ended quiz items
- Flashcard decks tied to sections
- Midterm and final exam question sets

**Notes**

Study artifacts are derived from generated content so assessments stay aligned with the material the student actually reads.

---

### 6. Validation and Persistence

**Input**

- Model-generated text, images, and assessment items
- Expected schema for each content type

**Output**

- Normalized records in PostgreSQL
- Media references in Cloud Storage
- Book readiness state updated for the frontend

**Notes**

Validation covers structural correctness, sanitization, and basic content integrity before content is shown to users.

---

### 7. Reader Handoff

**Input**

- Persisted book, section, and study artifacts
- User session and ownership context

**Output**

- Reader UI with table of contents, section content, assistant context, and study modes

**Notes**

The reader consumes the same structured records produced by the pipeline. The page assistant is scoped to the active section rather than re-querying raw generation output.

---

## Model Routing

Fessor does not rely on a single model for all tasks. Routing is task-based: the orchestration layer selects a provider based on the work being performed and the configured generation mode.

| Task | Typical routing | Rationale |
|------|-----------------|-----------|
| Book planning and outline | Strong reasoning model (OpenAI GPT-4, Claude, or Gemini) | Requires coherent long-horizon structure |
| Section writing | Task-selected writing model across OpenAI, Claude, Gemini, or Grok | Different models suit different content styles and topics |
| Attachment summarization | Model with strong document understanding | Grounds generation in uploaded source material |
| OCR / image text extraction | Vision-capable provider | Converts uploaded images and scanned content into usable text |
| Image generation | Dedicated image provider path | Separated from text LLM workflow |
| Page assistant Q&A | Streaming-capable assistant model | Low-latency interactive responses with section context |
| Assistant edit actions | Tool-aware model with edit workflow support | Rewrites or transforms active section content |
| Quiz / flashcard / exam generation | Structured-output-friendly model | Produces assessable items with predictable formatting |

The live product exposes model choice during customization. The orchestration layer maps that selection and task type to the appropriate provider rather than hard-coding a single model per feature.

---

## Attachment Handling

Fessor supports PDF and image attachments as generation inputs.

**Upload path**

1. User attaches a file to a generation or assistant request.
2. API stores the file in Cloud Storage and records metadata in PostgreSQL.
3. Worker tasks extract usable text or visual information from the attachment.

**Processing**

- **PDFs** — Text is extracted and chunked for inclusion in planning and section prompts.
- **Images** — OCR or vision-based extraction captures text and visual context where applicable.

**Downstream use**

- Attachment context is injected into planning and section-generation prompts.
- The page assistant can reference uploaded files during in-section conversations.
- Sanitization and ownership checks apply before attachment content influences generation.

Attachment-aware generation increases grounding but also raises token usage and credit consumption because more context is sent to providers.

---

## Validation and Fallbacks

**Validation**

- Enforce expected JSON or block structure for generated sections and assessments
- Sanitize HTML and rendered content before persistence
- Reject or flag malformed assistant output before it reaches the editor
- Verify attachment ownership and book ownership on read/write paths

**Fallbacks**

- If a section generation attempt fails validation, the worker can retry with the same or an alternate model depending on job configuration
- If image generation fails, the book can still complete with text content; image slots may remain empty or be retried separately
- If a provider is temporarily unavailable, the job can be retried rather than failing the entire book synchronously
- Partial book state is preserved so retries target failed units instead of restarting from scratch

The system is designed to degrade gracefully: a failed quiz batch should not invalidate an otherwise complete chapter.

---

## Cost and Credit Implications

Fessor uses a credit-based usage model. Credits abstract provider token and generation costs for the user.

**What consumes credits**

- Book planning and section generation (scales with book size and prompt context)
- Image generation (typically charged per image or per image batch)
- Page assistant interactions (scales with conversation length and attached context)
- Attachment processing (OCR and summarization add context length)
- Regeneration and assistant-driven edits (re-run all or part of the pipeline for a section)

**Design implications**

- Longer books and larger attachments increase provider cost; credits gate usage accordingly
- Async execution allows the API to enforce balance checks before work starts
- Separating image generation from text generation makes cost attribution clearer
- Streaming assistant responses still accumulate usage over the full conversation

Credit checks occur before expensive work is enqueued so users are not left with partially generated books they cannot afford to finish.

---

## Failure Handling

| Failure type | Behavior |
|--------------|----------|
| Provider timeout | Worker retry with backoff; job remains resumable |
| Invalid structured output | Validation rejects output; stage retries or marks section failed |
| Insufficient credits | Request rejected before enqueue; no partial charge for unstarted work |
| Attachment extraction failure | Generation may continue without attachment context; user-visible error where appropriate |
| Image generation failure | Text content still persists; image task may retry independently |
| Worker crash | Celery redelivery via Redis broker; idempotent writes prevent duplicate sections where possible |

Failed stages surface status to the frontend through realtime progress channels where applicable. See [Realtime Architecture](REALTIME_ARCHITECTURE.md) for how progress and errors reach the client.

---

## Related Documents

- [System Design](SYSTEM_DESIGN.md) — Sync vs async paths, ownership, and credit metering
- [Architecture](ARCHITECTURE.md) — Component-level view of the platform
- [Async Workflows](ASYNC_WORKFLOWS.md) — Celery tasks, queues, retries, and scaling
- [Realtime Architecture](REALTIME_ARCHITECTURE.md) — Streaming assistant and progress delivery
