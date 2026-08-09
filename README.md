# Notes App — Infrastructure

Docker Compose configuration for the Notes App, a Markdown note-taking application with server-side session authentication.

## Architecture

The application consists of three services connected through the private `notes-app` Docker network:

- **Frontend:** React application built with Vite; served by the Vite development server in development and Nginx in production
- **Backend:** Express 5 REST API responsible for users, sessions, and notes
- **Database:** PostgreSQL 18 storing application data and server-side sessions

The browser communicates with the application through the frontend service. Requests to `/api/*` are forwarded to the backend, while the backend and database communicate only inside the Docker network in production.

```text
Browser
   |
   | application pages and /api requests
   v
Frontend (Vite or Nginx)
   |
   | /api proxy
   v
Backend (Express)
   |
   v
PostgreSQL
```

### Authentication layers

The production deployment has two independent authentication layers:

1. **Nginx HTTP Basic Auth** protects access to the frontend itself.
2. **Application authentication** uses an opaque session identifier in an HTTP-only cookie. The backend stores only its SHA-256 hash in PostgreSQL.

The application previously had a JWT-based implementation, which remains available on the `feature/jwt-auth` branches of the frontend and backend repositories. Server-side sessions are used by the current version because refresh-token rotation and client-side token coordination added unnecessary complexity for an application of this scale.

## Repository layout

This repository contains only Docker Compose configuration and orchestration. The frontend and backend are separate Git repositories checked out as ignored subdirectories:

```text
notes-app-infra/
  docker-compose.yml              # Development stack
  docker-compose.prod.yml         # Production stack
  db.env.example                  # PostgreSQL environment template
  notes-app-frontend/             # Separate frontend repository
  notes-app-backend/              # Separate backend repository
```

Each of the three repositories has its own `.git` directory, branches, and history. Changes inside the component repositories do not appear in the infrastructure repository's Git status.

## Requirements

- Docker Engine
- Docker Compose v2
- Git

## Setup

### 1. Clone the repositories

```bash
git clone git@github.com:JanC02/notes-app-infra.git
cd notes-app-infra
git clone git@github.com:JanC02/notes-app-frontend.git
git clone git@github.com:JanC02/notes-app-backend.git
```

The checked-out frontend and backend branches must use the same authentication model and API contract.

### 2. Configure the environment

There is no shared root `.env` file. PostgreSQL and the backend read separate files, while the frontend requires no environment variables:

| File | Service | Template |
|---|---|---|
| `db.env` | `notes-app-db` | [`db.env.example`](db.env.example) |
| `notes-app-backend/.env` | `notes-app-backend` | [`notes-app-backend/.env.example`](notes-app-backend/.env.example) |
| None | `notes-app-frontend` | The API uses the relative `/api` path |

Create the local configuration files:

```bash
cp db.env.example db.env
cp notes-app-backend/.env.example notes-app-backend/.env
```

PostgreSQL configuration in `db.env`:

```dotenv
POSTGRES_USER=your_db_user
POSTGRES_PASSWORD=your_db_password
POSTGRES_DB=notes_app
```

Matching backend configuration in `notes-app-backend/.env`:

```dotenv
PORT=3000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_HOST=notes-app-db
DB_PORT=5432
DB_DATABASE=notes_app
```

`DB_USER`, `DB_PASSWORD`, and `DB_DATABASE` must match their PostgreSQL equivalents. The real environment files are ignored by Git. The session-based implementation does not require JWT secrets or a CORS origin.

### 3. Create production Basic Auth credentials

The production frontend image expects `notes-app-frontend/.htpasswd`. This file is not needed by the development target.

Create it with `htpasswd`:

```bash
htpasswd -Bc notes-app-frontend/.htpasswd your_username
```

This command is provided by `apache2-utils` on Debian and Ubuntu. Alternatively, generate the entry with Docker:

```bash
docker run --rm httpd:alpine htpasswd -nbB your_username your_password > notes-app-frontend/.htpasswd
```

## Development

Start the development stack:

```bash
docker compose up --build
```

The development configuration uses bind mounts and anonymous `node_modules` volumes so Vite and `tsx watch` can reload source changes without rebuilding the images.

| Service | Host address |
|---|---|
| Frontend | `http://localhost:5173` |
| Backend | `http://localhost:3000` |
| PostgreSQL | `localhost:5432` |

Although the backend is exposed for development and debugging, normal browser requests still use the frontend's `/api` proxy.

## Production

Build and start the production stack:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Only the Nginx frontend is published on host port `80`. The backend and PostgreSQL have no host port mappings and are reachable only through the internal Docker network.

In the public deployment, an external reverse proxy provides the domain and terminates HTTPS before forwarding traffic to the Nginx container. Internal proxying between Nginx and Express remains HTTP. This is expected: the browser-facing connection is HTTPS, so the backend's production `Secure` session cookie can be used safely.

When starting the production Compose file locally, the Nginx entry point is `http://localhost:80`. The intended public production setup is the HTTPS address supplied by the hosting platform.

## Database initialization

The `pgdata` named volume persists PostgreSQL data between container restarts and rebuilds. The backend's `init.sql` file is mounted at `/docker-entrypoint-initdb.d/init.sql` and creates the initial `users`, `notes`, and `sessions` schema.

PostgreSQL executes initialization scripts only when creating a new data directory. This project does not currently use a migration system, so editing `init.sql` does not update an existing `pgdata` volume. During development, the database can be recreated with:

```bash
docker compose down -v
docker compose up --build
```

This deletes all data stored in the development database volume.

## Service details

### Development Compose

- Builds the `dev` stage of both application Dockerfiles
- Bind-mounts frontend and backend source code
- Exposes all three services to the host
- Sets backend `NODE_ENV=development`

### Production Compose

- Builds the compiled `prod` stages
- Serves static frontend assets through Nginx
- Protects the frontend with HTTP Basic Auth
- Exposes only Nginx on host port `80`
- Sets backend `NODE_ENV=production`, enabling secure session cookies

## Useful commands

```bash
# Start development in the foreground
docker compose up --build

# Start production in the background
docker compose -f docker-compose.prod.yml up -d --build

# Show service status
docker compose ps
docker compose -f docker-compose.prod.yml ps

# Follow logs
docker compose logs -f
docker compose -f docker-compose.prod.yml logs -f

# Stop containers while preserving database data
docker compose down
docker compose -f docker-compose.prod.yml down

# Rebuild a single development service
docker compose up --build notes-app-backend
docker compose up --build notes-app-frontend

# Delete the development database volume and all stored data
docker compose down -v
```

## Component documentation

- [Frontend README](notes-app-frontend/README.md)
- [Backend README](notes-app-backend/README.md)
