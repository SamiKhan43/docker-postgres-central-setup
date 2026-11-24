# Project Connection

How a project connects to the shared database. First create a database and user for it (see [Database management](database-management.md)).

## Which address to use

| Where your app runs | Host | Port |
|---------------------|------|------|
| On your machine (not in Docker) | `localhost` | `5433` |
| In a Docker container on the `shared-db` network | `shared-postgres` | `5432` |

Inside Docker, always use the container name and the internal port `5432`. Using `localhost` inside a container points to the container itself, not to Postgres.

## Connection string format

```
postgresql://USER:PASSWORD@HOST:PORT/DATABASE
```

Examples:

```
postgresql://project1:strong-password@localhost:5433/project1
postgresql://project1:strong-password@shared-postgres:5432/project1
```

If the password contains special characters such as `@`, `/`, `:` or `#`, URL-encode them (`@` becomes `%40`). Letters, numbers and dashes avoid the problem.

## From an app on your machine

Put the string in the project's own `.env` file:

```
DATABASE_URL=postgresql://project1:strong-password@localhost:5433/project1
```

Keep that `.env` out of Git.

## From another Docker Compose project

Join the existing network in that project's `docker-compose.yml`:

```yaml
services:
  app:
    image: your-app-image
    environment:
      DATABASE_URL: postgresql://project1:strong-password@shared-postgres:5432/project1
    networks:
      - shared-db

networks:
  shared-db:
    external: true
```

`external: true` tells Compose the network already exists (created by this repo's compose file), so start `shared-postgres` first.

## From a plain `docker run`

```bash
docker run --rm -it --network shared-db postgres:16 \
  psql postgresql://project1:strong-password@shared-postgres:5432/project1
```

## Wait for the database on startup

If your app starts at the same time as Postgres, it may fail on its first connection. Either let the app retry, or keep `shared-postgres` always running (it has `restart: unless-stopped`, so it comes back after a reboot).

## Migrations

Run your migration tool (Alembic, Prisma, Django, Flyway, and so on) with the project's own `DATABASE_URL`. The project user owns its database, so it can create tables.

## Quick checks

```bash
# Is the container on the network?
docker network inspect shared-db

# Can a container reach it?
docker run --rm --network shared-db postgres:16 pg_isready -h shared-postgres -p 5432
```
