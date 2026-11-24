# Docker Postgres Central Setup

One shared PostgreSQL 16 container for all your local projects. Run it once, then give every project its own database and user, with no new YAML per project.

## How it works

- A single container, `shared-postgres`, runs Postgres and keeps its data in a named volume.
- Each project gets its own **database** and **user** inside it, so projects cannot see each other's data.
- The container joins a Docker network called `shared-db`. Other Docker projects join the same network and connect by the name `shared-postgres`.
- The port is published only on `127.0.0.1:5433`, so nothing outside your machine can reach it.

## Quick start

```bash
git clone <your-repo-url> docker-postgres-central-setup
cd docker-postgres-central-setup
cp .env.example .env        # then edit the password
docker compose up -d
docker compose ps           # wait for "healthy"
```

Create a database for your first project:

```bash
docker exec -it shared-postgres psql -U admin -d postgres
```

```sql
CREATE USER project1 WITH PASSWORD 'strong-password';
CREATE DATABASE project1 OWNER project1;
\q
```

Connect with:

```
postgresql://project1:strong-password@localhost:5433/project1
```

## Documentation

| Guide | What it covers |
|-------|----------------|
| [Installation](docs/installation.md) | Requirements, first start, verifying, updating |
| [Database management](docs/database-management.md) | Creating, listing, changing and dropping databases and users |
| [Project connection](docs/project-connection.md) | Connecting from your host and from other Docker projects |
| [Backup and restore](docs/backup-restore.md) | Dumps, restores, and a daily backup job |
| [Troubleshooting](docs/troubleshooting.md) | Common errors and fixes |

## Repository layout

```
docker-postgres-central-setup/
├── README.md
├── docker-compose.yml
├── .env.example
├── .gitignore
└── docs/
```

## Security notes

- Keep `.env` out of Git. `.gitignore` already excludes it, along with SQL dumps.
- Use a different user and password for each project, and never use the `admin` account in an app.
- The password in `.env` is only read the first time the volume is created. Changing it later does not change the existing password (see [Troubleshooting](docs/troubleshooting.md)).
- Do not remove `127.0.0.1` from the `ports` line unless you understand the risk of exposing the database to your network.
