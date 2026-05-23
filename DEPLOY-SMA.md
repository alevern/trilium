# Trilium — Deploy / Seed / Verify / Mutate / Assert

Baseline pinned at `7d77d2400b007f8e8ed2a77a41e67e66dc4a76e3`.

## Build

```bash
# Multi-stage Docker build (pnpm install + server:build → dist/)
docker build -f Dockerfile.tester-env -t tester-env-trilium .
```

Build time: ~3-4 minutes (pnpm install + frontend + backend bundling).
Builder stage: `node:24.15.0-bullseye-slim`, runtime stage: `node:24.15.0-bullseye-slim`.

## Run

```bash
# No-auth mode (TRILIUM_GENERAL_NOAUTHENTICATION=true skips all password prompts)
docker run -d --name tester-env-trilium \
  -p 38080:8080 \
  -v trilium-data:/home/node/trilium-data \
  -e TRILIUM_GENERAL_NOAUTHENTICATION=true \
  tester-env-trilium
```

## Database Initialisation

```bash
# One-time init (after first run on empty volume)
curl -X POST http://localhost:38080/api/setup/new-document
```

## Seed

```bash
# ETAPI-based deterministic seed — creates ACME Corp knowledge base (33 notes)
./tester-env seed
```

## Verify

```bash
# 33 content/structure/attribute/search checks
./tester-env verify
```

## Reset

```bash
docker stop tester-env-trilium
docker rm tester-env-trilium
docker volume rm trilium-data
```

## Default Credentials

None. `TRILIUM_GENERAL_NOAUTHENTICATION=true` bypasses all login.

## URL

`http://localhost:38080` (host port mapping from container port 8080).

## Seeded Data Overview

- 33 total seed notes: 6 top-level book sections, 27 child notes (text + code)
- Structure: ACME Corp Knowledge Base with 6 sections:
  - Getting Started (3 notes)
  - Project Planning (5 notes + 1 sub-book)
  - Meeting Notes (3 notes)
  - Technical Documentation (8 notes + 1 sub-book)
  - Engineering Standards (3 notes)
  - HR & Admin (3 notes)
- Note types: 13 text, 2 book, 5 code (bash, json, sql, sh)
- 9 labels on 4 notes (priority, department, audience, status, type, quarter, year, environment, iconClass)
- Searchable unique phrase: `Q2 Retrospective` returns exactly 1 result (noteQ2Retro)
- Fixed timestamps: all notes created with `dateCreated=2024-06-15 11:00:00.000+0100`
- All noteIds are deterministic (acmeRoot, sectGettingStarted, noteWelcome, etc.)

## Deterministic Assertions

1. All 33 noteIds resolve to expected titles via ETAPI
2. Key content phrases present (e.g., "ACME Corporation", "harassment-free", "helpdesk@acmecorp.com")
3. acmeRoot has exactly 6 child section books
4. 6 attribute/label checks (priority, department, status, type, quarter, environment)
5. Search for "Q2 Retrospective" returns noteQ2Retro as unique result
6. 3 full reset+deploy+seed cycles confirmed: 33/33 verify checks pass each time

## Mutation Smoke (baseline reference)

Passed by mutation-smoke subagent:
- Source file changed: `apps/client/index.html` line 10
- Before: `<title>Trilium Notes</title>`
- After: `<title>MUTANT Trilium Notes</title>`
- Docker image rebuilt successfully
- Mutated title visible via curl
- Source restored to baseline after verification

## Browser Smoke (baseline reference)

Passed by browser-checker subagent:
- Page loaded without errors
- Demo notes visible: Trilium Demo tree with Journal, Inbox, Formatting examples
- Full CRUD cycle works: create, edit, verify, delete note via right-click menus
- Note persistence confirmed via navigation

## Architecture Notes

- Self-contained: SQLite (better-sqlite3), no external services
- Server: Node.js, serves SPA frontend on port 8080
- ETAPI: External Trilium API for deterministic seeding (`/etapi/create-note`, `/etapi/notes`, etc.)
- No auth mode: `etapi_utils.ts` line 17 bypasses token validation when `TRILIUM_GENERAL_NOAUTHENTICATION=true`

## CLI Reference

```
./tester-env deploy    # Build from source, start container, init DB
./tester-env seed      # Populate with deterministic ACME Corp knowledge base
./tester-env verify    # Check seed data integrity (33 checks)
./tester-env reset     # Stop container, remove container and data volume
./tester-env stop      # Stop container (preserves data)
./tester-env logs      # Tail container logs
./tester-env status    # Show container status and URL
./tester-env help      # Show help message

Options:
  --port <port>     Host port (default: 38080)
  --run-id <id>     Isolate container/image/volume names for parallel runs
```
