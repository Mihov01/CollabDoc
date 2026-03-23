# CollabDoc

A production-grade real-time collaborative document editor. Multiple users can create, edit, and share documents simultaneously with live cursor presence, conflict-free concurrent edits, and full change history.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Nginx (reverse proxy)                │
└──────────────┬──────────────────────────┬───────────────────┘
               │ HTTP/WS                  │ Static assets
    ┌──────────▼──────────┐    ┌──────────▼──────────┐
    │   FastAPI Backend   │    │   React Frontend    │
    │   (stateless)       │    │   Vite + TypeScript │
    └──────┬──────┬───────┘    └─────────────────────┘
           │      │ WebSocket
    ┌──────▼──┐  ┌▼────────────────────┐
    │Postgres │  │  Redis Pub/Sub       │
    │         │  │  (WS broadcast layer)│
    └─────────┘  └─────────────────────┘
           │
    ┌──────▼──────┐
    │   Celery    │
    │ (snapshots, │
    │  indexing)  │
    └─────────────┘
```

The core challenge is keeping document state consistent across concurrent editors. This is solved using **Yjs** (a CRDT implementation) synced over WebSockets, with **Redis Pub/Sub** enabling the real-time broadcast layer to scale horizontally across multiple server instances.

## Key Engineering Decisions

**CRDT-based conflict resolution** — Yjs documents are conflict-free replicated data types. Concurrent edits from different users always converge to the same state without any server-side conflict arbitration.

**Relay-mode WebSocket server** — The backend operates as a relay: it receives Yjs binary updates from one client and broadcasts them to all others on the same document channel. This avoids the complexity of server-side Yjs state while remaining correct, since CRDT convergence happens on clients.

**Redis Pub/Sub for horizontal scaling** — Each server instance subscribes to a per-document Redis channel. Any instance can receive an update and fan it out globally, making the WS layer stateless and horizontally scalable.

**Event sourcing** — Every Yjs update is appended to `document_operations` as an immutable log. Periodic Celery tasks compact the log into `document_snapshots`. Point-in-time restoration replays ops from the nearest snapshot forward.

**Full-text search** — PostgreSQL `tsvector` column updated via trigger, queried with `to_tsquery`, indexed with GIN.

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite |
| Collaborative editing | Yjs, Tiptap, y-websocket |
| Backend | FastAPI, Python 3.12 |
| Database | PostgreSQL 16 |
| Cache / Pub-Sub | Redis 7 |
| Task queue | Celery |
| Migrations | Alembic |
| Deployment | Docker, Docker Compose, Nginx |

## Project Structure

```
CollabDoc/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   └── routes/         # Auth, documents, users, share links
│   │   ├── core/               # Config, security, JWT
│   │   ├── db/                 # Session factory, base model
│   │   ├── models/             # SQLAlchemy ORM models
│   │   ├── schemas/            # Pydantic request/response schemas
│   │   ├── services/           # Business logic (document, auth, search)
│   │   ├── tasks/              # Celery tasks (snapshots, indexing)
│   │   └── websocket/          # WS connection manager + Redis relay
│   ├── migrations/             # Alembic migration scripts
│   └── tests/
├── frontend/
│   └── src/
│       ├── api/                # Typed API client (fetch wrappers)
│       ├── components/
│       │   ├── editor/         # Tiptap editor, cursor overlays, toolbar
│       │   ├── layout/         # Shell, sidebar, nav
│       │   └── ui/             # Reusable primitives (button, modal, etc.)
│       ├── hooks/              # useDocument, usePresence, useAuth, etc.
│       ├── pages/              # Login, Register, Dashboard, Editor
│       ├── store/              # Auth state (Zustand)
│       └── types/              # Shared TypeScript types
├── nginx/                      # nginx.conf
├── scripts/                    # Dev helpers (seed, reset-db, etc.)
├── docker-compose.yml
└── .env.example
```

## Database Schema

```
users                    documents               document_access
─────────────────        ────────────────────    ───────────────────────
id (uuid, PK)            id (uuid, PK)           document_id (uuid, FK)
email (unique)           title (text)            user_id (uuid, FK)
hashed_password          owner_id (FK → users)   role (owner|editor|viewer)
created_at               content_snapshot (bytea) ──────────────────────
                         search_vector (tsvector)
                         created_at              share_links
                         updated_at              ──────────────────────
                                                 id (uuid, PK)
document_operations      document_snapshots      document_id (uuid, FK)
───────────────────      ──────────────────      token (text, unique)
id (uuid, PK)            id (uuid, PK)           role (editor|viewer)
document_id (uuid, FK)   document_id (uuid, FK)  expires_at
user_id (uuid, FK)       snapshot (bytea)        created_by (FK → users)
op_data (bytea)          version (int)
applied_at               created_at
```

## Getting Started

```bash
cp .env.example .env
docker compose up --build
```

API: http://localhost:8000
Frontend: http://localhost:5173
API docs: http://localhost:8000/docs

## Features

- Real-time collaborative editing with live cursors and user presence
- CRDT-based conflict resolution — concurrent edits never conflict
- Role-based access control: owner / editor / viewer
- Time-limited signed share links
- Full document version history with point-in-time restoration
- Full-text search across all accessible documents
- Horizontally scalable WebSocket layer via Redis Pub/Sub
