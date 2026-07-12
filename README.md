# Note App - Docker Deployment

Docker Compose configuration for the Note App - a markdown note-taking application.

## Architecture

- **frontend** - React app served by Nginx with HTTP Basic Auth
- **backend** - Express 5 REST API (Node.js)
- **db** - PostgreSQL 18

## Prerequisites

- Docker and Docker Compose
- Git

## Repository layout

This repo (`notes-app-infra`) holds the Compose files and orchestration only. `notes-app-frontend/` and `notes-app-backend/` are separate git repositories checked out as subdirectories (they're listed in `.gitignore` here, so this repo doesn't track their contents — each has its own `.git`).

## Setup

### 1. Clone this repository and its dependencies

```bash
git clone git@github.com:JanC02/notes-app-infra.git
cd notes-app-infra
git clone git@github.com:JanC02/notes-app-frontend.git
git clone git@github.com:JanC02/notes-app-backend.git
```

### 2. Configure environment variables

There is no single root `.env` — each service reads its own env file:

| File | Used by | Template |
|---|---|---|
| `db.env` | `notes-app-db` | [`db.env.example`](db.env.example) |
| `notes-app-backend/.env` | `notes-app-backend` | [`notes-app-backend/.env.example`](notes-app-backend/.env.example) — see [backend README](notes-app-backend/README.md) |
| `notes-app-frontend/.env` | — | not needed — the frontend has no runtime env vars (Axios `baseURL` is relative `/api`, proxied by Vite/Nginx), see [frontend README](notes-app-frontend/README.md) |

Copy each template and fill in your values:
```bash
cp db.env.example db.env
cp notes-app-backend/.env.example notes-app-backend/.env
```

None of the real env files are committed (`.env` and `db.env` are gitignored). `DB_USER`/`DB_PASSWORD`/`DB_DATABASE` in `notes-app-backend/.env` must match `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` in `db.env`.

Generate strong JWT secrets instead of typing your own:
```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### 3. Generate HTTP Basic Auth credentials (production only)

Used by Nginx in `docker-compose.prod.yml` to protect the frontend. Not needed for local dev.

```bash
openssl passwd -apr1 "your_password"
```
Paste the result as `login:hash` into `notes-app-frontend/.htpasswd`. Or with `htpasswd` (bcrypt, stronger):
```bash
htpasswd -Bc notes-app-frontend/.htpasswd your_username
```
Requires `apache2-utils` (`sudo apt install apache2-utils`) or use Docker:
```bash
docker run --rm httpd:alpine htpasswd -nbB your_username your_password > notes-app-frontend/.htpasswd
```

### 4. Run

**Development** (hot reload, Vite dev server, bind-mounted source):
```bash
docker compose up --build
```
Frontend: `http://localhost:5173` · Backend: `http://localhost:3000` · DB exposed on `5432`.

**Production** (multi-stage builds, compiled backend, Nginx + Basic Auth serving the frontend):
```bash
docker compose -f docker-compose.prod.yml up -d --build
```
App available at `http://localhost:80`. Backend and DB are not exposed to the host — only reachable inside the `notes-app` Docker network.

## Useful commands

```bash
# Stop
docker compose down                                       # dev
docker compose -f docker-compose.prod.yml down             # prod

# Stop and remove database volume
docker compose down -v

# Rebuild after code changes
docker compose up --build                                  # dev
docker compose -f docker-compose.prod.yml up -d --build     # prod

# View logs
docker compose logs -f                                     # dev
docker compose -f docker-compose.prod.yml logs -f           # prod
```
