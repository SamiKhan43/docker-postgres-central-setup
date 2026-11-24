# Docker PostgreSQL Central Setup

A reusable centralized PostgreSQL setup using Docker Compose.

The goal of this project is simple: run one PostgreSQL server on your machine and use it across multiple projects instead of creating a separate PostgreSQL container for every project.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Docker Configuration](#docker-configuration)
- [Starting PostgreSQL](#starting-postgresql)
- [PostgreSQL Management](#postgresql-management)
- [Database Management](#database-management)
- [User and Permission Management](#user-and-permission-management)
- [Connecting Projects](#connecting-projects)
- [Connecting Dockerized Projects](#connecting-dockerized-projects)
- [Backup and Restore](#backup-and-restore)
- [Database Inspection](#database-inspection)
- [Docker Volume Management](#docker-volume-management)
- [Updating PostgreSQL](#updating-postgresql)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)
- [Useful Commands](#useful-commands)
- [Recommended Workflow](#recommended-workflow)
- [License](#license)

## Overview

Normally, when starting a new backend project, you might create a separate PostgreSQL container for it:

```text
Project A
└── PostgreSQL Container A

Project B
└── PostgreSQL Container B

Project C
└── PostgreSQL Container C
```

This means running and maintaining multiple PostgreSQL containers.

Instead, this setup uses one centralized PostgreSQL server that manages multiple independent databases. Each project gets its own database, username, password, and permissions, while all projects share the same PostgreSQL server.

## Architecture

```text
                 Local Machine
                      |
                 Docker Engine
                      |
               Shared PostgreSQL
                  PostgreSQL 16
                      |
          +-----------+-----------+
          |           |           |
      Database A  Database B  Database C
          |           |           |
       Project A   Project B   Project C
```

The container joins a Docker network called `shared-db`. Projects running in Docker join the same network and reach PostgreSQL by the name `shared-postgres`. Projects running directly on your machine connect through `localhost`.

## Features

- PostgreSQL 16 using Docker.
- One centralized PostgreSQL container.
- Multiple independent databases.
- Separate database users and permissions.
- Persistent database storage.
- Docker health checks.
- Shared Docker network.
- Localhost-only database access.
- Backup and restore support.
- Compatible with Node.js, Python, Django, FastAPI, Express, NestJS, Prisma, and other PostgreSQL clients.

## Requirements

| Tool | Purpose |
|---|---|
| Docker | Container runtime |
| Docker Compose V2 | Container orchestration |
| Git | Version control |
| PostgreSQL Client | Optional database management |

Check your installations:

```bash
docker --version
docker compose version
git --version
```

## Project Structure

```text
docker-postgres-central-setup/
│
├── README.md
├── docker-compose.yml
├── .env.example
├── .gitignore
└── LICENSE
```

Create a `.gitignore` so passwords and backups are never committed:

```text
.env
*.sql
*.dump
*.sql.gz
backups/
```

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/docker-postgres-central-setup.git
```

Replace `YOUR_USERNAME` with your GitHub username.

### 2. Navigate into the project

```bash
cd docker-postgres-central-setup
```

### 3. Create your environment file

```bash
cp .env.example .env
```

### 4. Configure environment variables

```bash
nano .env
```

Example:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_strong_password
POSTGRES_DB=postgres
```

Replace the password with a strong, unique password. If you change `POSTGRES_USER`, replace `-U postgres` in every command in this README with your user.

## Docker Configuration

The `docker-compose.yml` file:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: shared-postgres
    restart: unless-stopped

    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

    ports:
      - "127.0.0.1:5433:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

    networks:
      - shared-db

volumes:
  postgres_data:

networks:
  shared-db:
    name: shared-db
    driver: bridge
```

### Configuration explanation

| Configuration | Description |
|---|---|
| `postgres:16` | PostgreSQL 16 Docker image |
| `shared-postgres` | Container name |
| `restart: unless-stopped` | Automatically restarts the container |
| `POSTGRES_USER` | Initial PostgreSQL administrator |
| `POSTGRES_PASSWORD` | Initial PostgreSQL password |
| `POSTGRES_DB` | Initial database |
| `127.0.0.1:5433:5432` | Host port 5433 (localhost only) maps to PostgreSQL's port 5432 inside the container |
| `postgres_data` | Persistent database volume |
| `shared-db` | Shared Docker network |
| `healthcheck` | Checks PostgreSQL readiness |

The port is bound to `127.0.0.1`, so PostgreSQL is accessible from your local machine without being exposed to other network interfaces.

Port `5433` is used on the host so it does not clash with another PostgreSQL on `5432`. If `5432` is free on your machine, you can use `"127.0.0.1:5432:5432"` instead and change `5433` to `5432` in the local connection strings below.

## Starting PostgreSQL

```bash
docker compose up -d
```

Check running containers (wait for `healthy`):

```bash
docker ps
```

Check container logs:

```bash
docker logs shared-postgres
```

Check PostgreSQL readiness:

```bash
docker exec shared-postgres pg_isready -U postgres
```

Expected output:

```text
/var/run/postgresql:5432 - accepting connections
```

## PostgreSQL Management

```bash
docker start shared-postgres      # start
docker stop shared-postgres       # stop
docker restart shared-postgres    # restart
docker logs -f shared-postgres    # follow logs
docker ps -a                      # container status
```

Connect to PostgreSQL:

```bash
docker exec -it shared-postgres psql -U postgres
```

Exit PostgreSQL:

```sql
\q
```

## Database Management

Connect first:

```bash
docker exec -it shared-postgres psql -U postgres
```

### Create a database

```sql
CREATE DATABASE my_project;
```

To create a database together with its own user, see [User and Permission Management](#user-and-permission-management).

### List all databases

```sql
\l
```

### Connect to a database

```sql
\c my_project
```

### Rename a database

```sql
ALTER DATABASE old_name RENAME TO new_name;
```

### Delete a database

```sql
DROP DATABASE my_project;
```

Dropping a database permanently deletes its contents. Back it up first.

## User and Permission Management

Create a separate PostgreSQL user for every project instead of using the PostgreSQL administrator account in application code.

### Recommended: create a user that owns its database

```sql
CREATE USER my_project_user WITH PASSWORD 'strong_project_password';
CREATE DATABASE my_project OWNER my_project_user;
```

The owner can create tables in its database without any extra grants.

### Isolate the database

By default any user may try to connect to any database. Restrict access so each project can only use its own:

```sql
REVOKE CONNECT ON DATABASE my_project FROM PUBLIC;
GRANT CONNECT ON DATABASE my_project TO my_project_user;
```

### If the database already exists

Grant access to a user on an existing database:

```sql
GRANT ALL PRIVILEGES ON DATABASE my_project TO my_project_user;
```

Then connect to the project database and grant schema permissions:

```sql
\c my_project
GRANT USAGE, CREATE ON SCHEMA public TO my_project_user;
```

### Grant access to existing tables and sequences

If the database already contains tables created by another user:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA public
TO my_project_user;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA public
TO my_project_user;
```

### Set default privileges for future tables

Run this as the user that will create future tables:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES
TO my_project_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT USAGE, SELECT, UPDATE ON SEQUENCES
TO my_project_user;
```

### List PostgreSQL users

```sql
\du
```

### Change a user's password

```sql
ALTER USER my_project_user WITH PASSWORD 'new_password';
```

### Delete a user

```sql
DROP USER my_project_user;
```

A user that still owns a database cannot be dropped. Drop or reassign the database first.

## Connecting Projects

All your backend projects can connect to the same PostgreSQL server. The database name, username, and password are different for each project.

Example for one project:

```sql
CREATE USER project_a_user WITH PASSWORD 'project_a_password';
CREATE DATABASE project_a OWNER project_a_user;
REVOKE CONNECT ON DATABASE project_a FROM PUBLIC;
GRANT CONNECT ON DATABASE project_a TO project_a_user;
```

Connection string for a project running on your machine:

```env
DATABASE_URL=postgresql://project_a_user:project_a_password@localhost:5433/project_a
```

Repeat the same steps for every other project, changing the names and passwords:

| Project | Database | User | Connection string |
|---|---|---|---|
| A | `project_a` | `project_a_user` | `postgresql://project_a_user:project_a_password@localhost:5433/project_a` |
| B | `project_b` | `project_b_user` | `postgresql://project_b_user:project_b_password@localhost:5433/project_b` |
| C | `project_c` | `project_c_user` | `postgresql://project_c_user:project_c_password@localhost:5433/project_c` |

If a password contains special characters such as `@`, `/`, `:` or `#`, URL-encode them (`@` becomes `%40`), or the connection string will break. Passwords made of letters, numbers, and dashes avoid the problem.

## Connecting Dockerized Projects

If your backend also runs inside Docker, there is one important difference.

Inside a Docker container, `localhost` refers to that container, not your PostgreSQL container. Use the container name instead:

```text
shared-postgres
```

### Connect your backend to the shared network

Example backend `docker-compose.yml`:

```yaml
services:
  backend:
    build: .
    container_name: my-backend

    environment:
      DATABASE_URL: postgresql://my_project_user:password@shared-postgres:5432/my_project

    networks:
      - shared-db

networks:
  shared-db:
    external: true
```

`external: true` means the network already exists. It is created when you start this repository's Compose file, so start `shared-postgres` first. If needed, create it by hand with `docker network create shared-db`.

Start your backend:

```bash
docker compose up -d
```

For container-to-container connections, always use:

```text
shared-postgres:5432
```

The host port mapping (`5433`) is not used for communication between containers on this network.

## Backup and Restore

Regular backups are important because your databases contain project data. A Docker volume is not a backup.

### Backup a single database

```bash
docker exec shared-postgres pg_dump -U postgres my_project > my_project_backup.sql
```

### Backup using custom format

```bash
docker exec shared-postgres pg_dump -U postgres -Fc my_project > my_project_backup.dump
```

### Backup all databases

```bash
docker exec shared-postgres pg_dumpall -U postgres > full_backup.sql
```

### Restore a SQL backup

Create the database:

```bash
docker exec shared-postgres createdb -U postgres my_project
```

Restore:

```bash
docker exec -i shared-postgres psql -U postgres my_project < my_project_backup.sql
```

### Restore a custom-format backup

Create the target database first, then:

```bash
docker exec -i shared-postgres pg_restore -U postgres -d my_project < my_project_backup.dump
```

Use `-i` only, not `-it`, whenever you redirect a file into `docker exec`.

### Restore a full backup

```bash
docker exec -i shared-postgres psql -U postgres < full_backup.sql
```

Messages such as "role already exists" are normal.

Keep backups somewhere safe outside the Docker volume, and never commit them to Git. The `.gitignore` above excludes common backup files.

## Database Inspection

```sql
\l                              -- list all databases
\du                             -- list all users
\dt                             -- show tables
\d table_name                   -- show table structure
SELECT current_database();      -- current database
SELECT current_user;            -- current user
SELECT * FROM pg_stat_activity; -- active connections
```

Show database sizes:

```sql
SELECT
    datname AS database_name,
    pg_size_pretty(pg_database_size(datname)) AS database_size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

## Docker Volume Management

PostgreSQL stores its data in a persistent Docker named volume. Its name starts with your Compose project name, which is the folder name by default, so for this repository it is usually:

```text
docker-postgres-central-setup_postgres_data
```

Find the real name on your machine:

```bash
docker volume ls | grep postgres_data
```

Inspect it:

```bash
docker volume inspect docker-postgres-central-setup_postgres_data
```

Stop PostgreSQL without deleting data:

```bash
docker compose down
```

Start it again:

```bash
docker compose up -d
```

Your data remains available because the named volume is preserved.

Never run this unless you intentionally want to delete all database storage:

```bash
docker compose down -v
```

The `-v` flag removes the Compose-managed named volume and all your data.

## Updating PostgreSQL

Pull the latest image for your configured major version:

```bash
docker compose pull
docker compose up -d
docker ps
```

Changing from PostgreSQL 16 to another major version requires a database upgrade procedure. Simply changing `postgres:16` to `postgres:17` does not migrate your existing database files, and the container will refuse to start on the old data.

To change major versions: take a full backup with `pg_dumpall`, verify it, run `docker compose down -v`, change the image, start the new container, and restore the backup.

## Troubleshooting

### Port already in use

```bash
sudo ss -ltnp | grep 5433
```

Stop the conflicting service or change the host port mapping, for example:

```yaml
ports:
  - "127.0.0.1:5434:5432"
```

Your local connection string would then use port `5434`. Dockerized projects on the shared network still use port `5432`.

### PostgreSQL container is not starting

```bash
docker logs shared-postgres
docker ps -a
```

If the logs say a variable is not set, your `.env` file is missing or misnamed. It must be called exactly `.env` and sit next to `docker-compose.yml`.

### Connection refused

```bash
docker exec shared-postgres pg_isready -U postgres
```

Make sure the container is running and healthy.

### Password authentication failed

Check the credentials in your connection string and `.env` file.

PostgreSQL initialization variables only apply to a new, empty data directory. Changing `POSTGRES_PASSWORD` in `.env` does not change an existing user's password. Change it inside PostgreSQL:

```sql
ALTER USER postgres WITH PASSWORD 'new_password';
```

### Docker network not found

```bash
docker network ls
docker network create shared-db
```

### Backend cannot connect to PostgreSQL

If your backend runs on your machine, use:

```text
localhost:5433
```

If your backend runs inside Docker, use:

```text
shared-postgres:5432
```

Also verify that both containers are on the `shared-db` network:

```bash
docker network inspect shared-db
```

### Permission denied

Connect as the PostgreSQL administrator and inspect:

```sql
\du
\l
```

If a project user cannot create tables, make it the owner of its database:

```sql
ALTER DATABASE my_project OWNER TO my_project_user;
```

### Permission denied on the Docker socket

```bash
sudo usermod -aG docker $USER
```

Then log out and back in.

## Security Best Practices

- Never commit `.env` files to GitHub.
- Use strong and unique database passwords.
- Create separate users for separate projects.
- Avoid using the PostgreSQL superuser in application code.
- Revoke `PUBLIC` connect access so projects cannot reach each other's databases.
- Keep PostgreSQL bound to localhost for local development.
- Don't expose the PostgreSQL port publicly.
- Back up important databases regularly and test restoring them.
- Keep Docker and PostgreSQL updated.
- Don't delete persistent volumes without checking their contents.
- Use dedicated database instances for projects requiring stronger isolation.

## Useful Commands

| Action | Command |
|---|---|
| Start PostgreSQL | `docker start shared-postgres` |
| Stop PostgreSQL | `docker stop shared-postgres` |
| Restart PostgreSQL | `docker restart shared-postgres` |
| View logs | `docker logs -f shared-postgres` |
| Check status | `docker ps` |
| Connect to PostgreSQL | `docker exec -it shared-postgres psql -U postgres` |
| List databases | `docker exec -it shared-postgres psql -U postgres -c "\l"` |
| List users | `docker exec -it shared-postgres psql -U postgres -c "\du"` |
| Check readiness | `docker exec shared-postgres pg_isready -U postgres` |
| List volumes | `docker volume ls` |
| Inspect network | `docker network inspect shared-db` |
| Stop Compose stack | `docker compose down` |
| Start Compose stack | `docker compose up -d` |
| View Compose logs | `docker compose logs -f` |

## Recommended Workflow

Whenever you start a new backend project:

**1. Create a user and a database it owns**

```sql
CREATE USER new_project_user WITH PASSWORD 'strong_password';
CREATE DATABASE new_project OWNER new_project_user;
```

**2. Isolate the database**

```sql
REVOKE CONNECT ON DATABASE new_project FROM PUBLIC;
GRANT CONNECT ON DATABASE new_project TO new_project_user;
```

**3. Add the connection string to your project's `.env`**

For a backend running on your machine:

```env
DATABASE_URL=postgresql://new_project_user:strong_password@localhost:5433/new_project
```

For a Dockerized backend (also add the `shared-db` network to its Compose file):

```env
DATABASE_URL=postgresql://new_project_user:strong_password@shared-postgres:5432/new_project
```

**4. Start developing.**

You don't need another PostgreSQL container for your new project.

---

## License

This project is licensed under the MIT License. See the `LICENSE` file.

---

A simple and reusable PostgreSQL development setup for managing multiple projects with one centralized database server.
