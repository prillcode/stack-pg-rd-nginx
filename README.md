# Docker Stack - Postgres, Redis, Nginx

## Overview

This docker-compose file sets up a local development infrastructure stack with three services to support Angular and React application development.

## Services

### Always-Running Services

#### PostgreSQL (postgres:15-alpine)
- A lightweight PostgreSQL database server on port 5432
- Pre-configured with simple dev credentials (user: postgres, password: dev, database: devdb)
- Data persists in a named volume (`postgres_data`) so it survives container restarts
- Automatically restarts unless manually stopped

#### Redis (redis:7-alpine)
- An in-memory data store/cache on port 6379
- Useful for session management, API caching, rate limiting, or real-time features
- Connect from your backend (Node.js/Express/NestJS), not directly from frontend
- Data persists in `redis_data` volume
- Also set to auto-restart

### Opt-in Service

#### Nginx (nginx:alpine)
- Web server for testing production builds locally on port 8080
- Uses a Docker profile (`production-test`), so it won't start with `docker compose up`
- Serves static files from `./dist` directory (your built Angular/React apps)
- Two ways to use: run from project dir with `-f` flag, or set `PROJECT_DIST` env variable
- Read-only mount prevents accidental modifications

## Usage

Start the always-running services:
```bash
docker compose up -d
```

Test a production build with nginx (Option 1 - from project directory):
```bash
cd /path/to/your/project
docker compose -f /home/prill/code/stack-pg-rd-nginx/docker-compose.yml --profile production-test up nginx
```

Test a production build with nginx (Option 2 - using environment variable):
```bash
PROJECT_DIST=/path/to/your/project/dist docker compose --profile production-test up nginx
```

Stop all services:
```bash
docker compose down
```

For detailed examples, see [USAGE-EXAMPLES.md](USAGE-EXAMPLES.md).
