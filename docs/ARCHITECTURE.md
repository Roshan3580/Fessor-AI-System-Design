# Fessor Architecture

## Overview

Fessor is an AI-powered educational platform that generates textbooks and interactive learning experiences.

The architecture separates:

- Frontend
- Backend APIs
- Async workers
- Databases
- AI providers
- Storage systems

---

# Frontend

Built with:

- Next.js 15
- React 19
- TypeScript

Responsibilities:

- Reader UI
- Editor
- Dashboard
- Library
- Quizzes
- Flashcards
- Exams
- Page Assistant

---

# Backend

Built with:

- Django 4.2
- Django Channels
- Daphne

Responsibilities:

- Authentication
- Credits
- Generation APIs
- Content management
- WebSockets
- User accounts

---

# Async Processing

Celery workers execute long-running jobs:

- Book generation
- Image generation
- Analytics
- Background processing

Celery Beat handles scheduled jobs.

---

# Databases

### PostgreSQL

Stores:

- Users
- Books
- Sections
- Chapters
- Exams
- Quiz attempts
- Credits

### Redis

Used for:

- Cache
- Celery broker
- Channel layer
- Realtime messaging

---

# AI Layer

Multiple providers are orchestrated depending on the task.

Examples:

- OpenAI
- Claude
- Gemini

Different models are used for:

- Planning
- Writing
- Attachments
- OCR
- Assistant interactions

---

# File Attachments

Supports:

- PDFs
- Images

Attachments are processed and incorporated into generation workflows.

---

# Deployment

Development:

Docker Compose

Production:

Google Cloud Platform

Components:

- Compute Engine
- Cloud SQL
- Memorystore
- Cloud Storage

---

# Realtime Features

Django Channels powers:

- Streaming responses
- Assistant interactions
- Progress updates

---

# Scaling Considerations

Horizontal worker scaling

Redis-backed messaging

External AI providers

Async task queues

Managed databases