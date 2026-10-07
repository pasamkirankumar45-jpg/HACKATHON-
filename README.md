# CodeSync — Real-Time Collaborative Python IDE

CodeSync is a production-quality, distributed-systems-inspired real-time collaborative Python IDE featuring Conflict-Free Replicated Data Type (CRDT) consistency, transparent offline editing with local operation queuing, client-side End-to-End Encryption (AES-GCM-256), isolated code execution sandbox, and an integrated AI coding assistant.

---

## 1. Project Overview

CodeSync allows software engineering teams to collaboratively write, debug, and run Python code simultaneously. Unlike naive operational transformation (OT) systems that rely on a single centralized lock or overwrite models, CodeSync employs a Replicated Growable Array (RGA) CRDT with logical vector clocks to mathematically guarantee **Strong Eventual Consistency (SEC)**.

When a network split or temporary disconnection occurs, users can continue editing offline. Pending edits are buffered in an IndexedDB/localStorage queue and seamlessly merged upon reconnection without losing concurrent edits from other peers.

---

## 2. High-Level Architecture

```
                    ┌─────────────────────────┐
                    │      Web Browser        │
                    │                         │
                    │ React 19 + TypeScript   │
                    │ Monaco Editor           │
                    │ RGA CRDT Engine         │
                    │ Web Crypto API (E2EE)   │
                    └───────────┬─────────────┘
                                │
                        REST + WebSockets
                                │
                                ▼
                    ┌─────────────────────────┐
                    │       API Gateway       │
                    │                         │
                    │ Django REST Framework   │
                    │ Daphne ASGI Server      │
                    │ JWT Auth & RBAC Guard   │
                    └───────────┬─────────────┘
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
             ▼                  ▼                  ▼
      ┌────────────┐     ┌────────────┐     ┌────────────┐
      │ PostgreSQL │     │   Redis    │     │  Celery    │
      │            │     │            │     │            │
      │ Users      │     │ Pub/Sub    │     │ Background │
      │ Workspaces │     │ Channels   │     │ Sandboxed  │
      │ Files      │     │ Presence   │     │ Execution  │
      │ Snapshots  │     └────────────┘     └────────────┘
      └────────────┘            │
                                ▼
                    ┌─────────────────────────┐
                    │  WebSocket Sync Room    │
                    │                         │
                    │ AsyncJsonWebsocket      │
                    │ Causal Vector Clocks    │
                    │ Deduplication Filter    │
                    └───────────┬─────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │  Isolated Python Runner │
                    │                         │
                    │ AST Syscall Inspection  │
                    │ RLIMIT_AS / RLIMIT_CPU  │
                    │ Subprocess Sandbox      │
                    └─────────────────────────┘
```

---

## 3. Core Features

1. **Authentication & Authorization:** JWT access and refresh tokens, Argon2 password hashing, RBAC (OWNER, ADMIN, EDITOR, VIEWER).
2. **Workspace & File System:** Nested folders, files, tabs, rename, delete, and instant preview.
3. **CRDT Collaboration:** Real-time multi-user concurrent editing powered by character-level RGA CRDT with logical vector clocks.
4. **Presence & Cursors:** Remote caret decorations, line/column tracking, and collaborator activity indicators.
5. **Offline Support:** Automatic network disconnection detection, local persistent queue, and replay synchronization on reconnect.
6. **End-to-End Encryption (E2EE):** Authenticated client-side AES-GCM (256-bit) encryption using the Web Crypto API.
7. **Version History:** Snapshot timeline with diff preview and rollback capabilities.
8. **Isolated Code Execution:** Sandboxed Python execution with timeout (5s), memory limits, and static AST security checks.
9. **IDE Bottom Panel:** Integrated Terminal, Output (stdout, stderr, exit code, execution time), Problems diagnostics, and Test Suite.
10. **AI Coding Assistant:** Gemini 3.8 Flash server-side integration for code explanation, bug detection, pytest generation, and refactoring with user-confirmed diff application.
11. **Security & Threat Model Inspector:** Built-in auditor displaying security boundaries, replay defense metrics, and audit logs.
12. **Collaborator Simulator:** In-app multi-user simulator to test concurrent race conditions (e.g. Rahul and Sai typing simultaneously).

---

## 4. Technology Stack

- **Backend:** Python 3.12, Django 5.0, Django REST Framework, Django Channels 4.1, Daphne, Celery, SimpleJWT.
- **Frontend:** React 19, Vite, TypeScript, Monaco Editor (`@monaco-editor/react`), Tailwind CSS, Lucide icons.
- **Database & Broker:** PostgreSQL 16, Redis 7.
- **Cryptography:** Web Crypto API (`AES-GCM`, `PBKDF2`, `SHA-256`), Python `cryptography`.
- **AI Engine:** Google Gemini API (`@google/genai` with `gemini-3.8-flash`).

---

## 5. Environment Setup

Copy `.env.example` to `.env`:

```bash
cp .env.example .env
```

Key environment variables:

| Variable | Description | Default |
|---|---|---|
| `PORT` | Node.js web server port | `3000` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:postgrespassword@localhost:5432/codesync` |
| `REDIS_URL` | Redis URL for Channels & Celery | `redis://localhost:6379/0` |
| `JWT_SECRET` | Secret key for signing JWT tokens | `dev-jwt-secret-key-32chars` |
| `GEMINI_API_KEY` | Google Gemini API key for AI Assistant | Injected automatically |

---

## 6. Running with Docker Compose (Recommended)

Start all services (PostgreSQL, Redis, Django Channels backend, Celery worker, React full-stack frontend) with one command:

```bash
docker compose up --build
```

Access the application in your browser at `http://localhost:3000`.

---

## 7. Running Locally for Development

### 7.1. Start Redis & PostgreSQL

```bash
# Using Docker for services
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=postgrespassword postgres:16-alpine
docker run -d -p 6379:6379 redis:7-alpine
```

### 7.2. Start Django Backend

```bash
cd codesync/backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
daphne -b 0.0.0.0 -p 8000 config.asgi:application
```

### 7.3. Start Celery Worker

```bash
cd codesync/backend
celery -A config worker --loglevel=info
```

### 7.4. Start React Frontend & IDE Server

```bash
npm install
npm run dev
```

Server will run on `http://localhost:3000`.

---

## 8. Running Automated Tests

### Backend Pytest Suite:
```bash
cd codesync/backend
pytest -v
```

### In-App Interactive Test Suite:
Navigate to the **TEST SUITE** tab in the bottom panel of CodeSync and click **Run All Tests** to execute:
1. 3-Way Concurrent CRDT Convergence
2. Offline Disconnection & Queue Synchronization
3. Replay Attack & Duplicate Operation Rejection
4. Vector Clock Causality & Monotonic Clocks
5. RBAC Access Control Enforcement

---

## 9. REST API Documentation

### Authentication
- `POST /api/auth/register/` — Register new user account.
- `POST /api/auth/login/` — Authenticate and receive JWT access and refresh tokens.
- `POST /api/auth/refresh/` — Obtain fresh access token.
- `GET /api/auth/me/` — Retrieve authenticated user profile.

### Workspaces / Projects
- `GET /api/projects/` — List user's workspaces.
- `POST /api/projects/` — Create new workspace with optional E2EE mode.
- `GET /api/projects/{id}/` — Get workspace details.
- `DELETE /api/projects/{id}/` — Delete workspace (Owner only).
- `GET /api/projects/{id}/members/` — List project members and roles.
- `POST /api/projects/{id}/members/` — Invite member and assign role.

### Files
- `GET /api/projects/{id}/files/` — List files and folders in workspace.
- `POST /api/projects/{id}/files/` — Create file or folder.
- `GET /api/files/{id}/` — Get file content and metadata.
- `PUT /api/files/{id}/` — Update file content and increment version.
- `DELETE /api/files/{id}/` — Delete file (Admin/Owner).
- `GET /api/files/{id}/history/` — List historical document snapshots.
- `POST /api/files/{id}/history/restore/` — Restore document to specific snapshot.

### Code Execution
- `POST /api/execution/run/` — Execute code in sandboxed subprocess.
  - Request: `{"code": "print('hello')", "timeoutMs": 5000}`
  - Response: `{"stdout": "hello\n", "stderr": "", "exitCode": 0, "executionTimeMs": 28}`

### AI Assistant
- `POST /api/gemini/assistant` — Gemini 3.8 Flash coding assistance.
  - Actions: `explain`, `find_bugs`, `improve`, `generate_tests`, `document`, `custom`

---

## 10. WebSocket Collaboration Protocol

Connect to:
`ws://localhost:3000/ws/projects/{project_id}/files/{file_id}/`

### Client → Server Events:
- `join_room`: Registers client, announces presence, triggers `sync_response`.
- `document_update`: Sends atomic CRDTOperation:
  ```json
  {
    "type": "document_update",
    "operation": {
      "operation_id": "op_17123984_8x29a",
      "user_id": "usr_manikanta_101",
      "document_id": "f_main",
      "timestamp": 1712398400000,
      "logical_clock": 15,
      "operation_type": "insert",
      "position": 25,
      "content": "def calculate():\n",
      "parent_version": 14
    }
  }
  ```
- `cursor_update`: Broadcasts caret coordinates `{"line": 12, "column": 8}`.
- `sync_request`: Requests latest snapshot and missing operation delta.
- `ping`: Keepalive heartbeat.

### Server → Client Events:
- `sync_response`: Full snapshot, active users, vector clock.
- `user_joined` / `user_left`: Presence changes.
- `document_update`: Broadcast of remote edit.
- `cursor_update`: Remote peer caret movement.

---

## 11. CRDT Consistency Model (RGA)

CodeSync uses a **Replicated Growable Array (RGA)** character-level CRDT:

1. **Atoms with Immutable IDs:** Each character is modeled as an atom:
   $$\text{atom} = \langle \text{id: clock@client}, \text{origin\_id}, \text{value}, \text{deleted} \rangle$$
2. **Tombstone Deletions:** Deleting a character marks it as a tombstone (`deleted = True`) without removing it from the atom chain. This ensures concurrent inserts referencing deleted characters can still locate their exact predecessor.
3. **Deterministic Sibling Resolution:** When two clients insert concurrently after the same predecessor atom, the conflict is resolved deterministically by comparing their logical clocks. If clocks match, the client ID string breaks the tie:
   $$(\text{clock}_A, \text{client}_A) > (\text{clock}_B, \text{client}_B)$$
4. **Strong Eventual Consistency:** Irrespective of packet order, all replicas converge to the identical sequence of visible characters.

---

## 12. Security Model & Byzantine Threat Analysis

| Threat Vector | Practical Defense Implemented | Theoretical Limit |
|---|---|---|
| **Replay Attacks** | Deduplication cache of `operation_id` sets; duplicate operations are rejected with HTTP 409 / WS error. | Relies on monotonic UUIDs and clock bounds. |
| **Out-of-Order Packets** | Vector clocks track causal readiness; operations arriving out of order are causally sequenced. | Network partitions require queue flush. |
| **Eavesdropping Relay** | Client-side AES-GCM 256 authenticated encryption via Web Crypto API. | Plaintext visible if room passphrase is compromised. |
| **Malicious Code Execution** | Subprocess resource limits (`RLIMIT_AS`, `RLIMIT_CPU`), timeout kill, AST blocking of `os.system` / `subprocess`. | Multi-tenant hostile untrusted code requires Firecracker microVMs in production. |
| **Unauthorized Escalation** | Server-side RBAC validation on all REST endpoints and WebSocket room joins. | Token theft mitigated by short-lived JWTs (60m). |

> **Byzantine Fault Tolerance Notice:** CodeSync provides practical CRDT convergence and cryptographic security. It does not claim formal $n \ge 3f + 1$ PBFT consensus for arbitrary hostile behavior; an authorized editor inserting syntactically invalid Python code will have that edit consistently replicated across all peers.
