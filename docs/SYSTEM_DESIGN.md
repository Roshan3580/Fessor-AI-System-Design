# System Design

## Request Flow

User Prompt

↓

Planner

↓

Generation Pipeline

↓

Async Workers

↓

AI Providers

↓

Structured Content

↓

Database

↓

Reader Experience

---

## Main Components

Frontend

↓

API Layer

↓

Celery Workers

↓

Redis

↓

PostgreSQL

↓

Cloud Storage

↓

AI Providers

---

## Realtime Layer

WebSockets enable:

- Streaming responses
- Assistant messages
- Progress updates

---

## Scalability

Long-running jobs are separated from request-response cycles.

Redis-backed queues allow workers to scale independently.

Managed cloud infrastructure reduces operational complexity.