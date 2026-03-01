# Note App - Docker Deployment

Docker Compose configuration for the Note App - a markdown note-taking application.

## Architecture

- **frontend** - React app served by Nginx with HTTP Basic Auth
- **backend** - Express 5 REST API (Node.js)
- **db** - PostgreSQL 18.1

## Prerequisites

- Docker and Docker Compose
- Git

## Setup

### 1. Clone this repository and its dependencies

```bash
git clone git@github.com:JanC02/notes-app.git
cd notes-app
git clone git@github.com:JanC02/notes-app-frontend.git
git clone git@github.com:JanC02/notes-app-backend.git
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

| Variable | Description |
|---|---|
| `CORS_ORIGIN` | Frontend URL (e.g. `http://localhost:30155`) |
| `DB_USER` | PostgreSQL username |
| `DB_PASSWORD` | PostgreSQL password |
| `DB_DATABASE` | Database name |
| `ACCESS_TOKEN_SECRET` | Secret for JWT access tokens |
| `REFRESH_TOKEN_SECRET` | Secret for JWT refresh tokens |

### 3. Generate HTTP Basic Auth credentials

```bash
htpasswd -cB .htpasswd your_username
```

Requires `apache2-utils` (`sudo apt install apache2-utils`) or use Docker:

```bash
docker run --rm httpd:alpine htpasswd -nbB your_username your_password > .htpasswd
```

### 4. Run

```bash
docker compose up --build
```

The app will be available at `http://localhost:30155`.

## Useful commands

```bash
# Stop
docker compose down

# Stop and remove database volume
docker compose down -v

# Rebuild after code changes
docker compose up --build

# View logs
docker compose logs -f
```
