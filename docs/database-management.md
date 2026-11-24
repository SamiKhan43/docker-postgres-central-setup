# Database Management

Each project gets its own database and its own user. All commands assume the container is named `shared-postgres` and the admin user is `admin`.

## Open a SQL prompt

```bash
docker exec -it shared-postgres psql -U admin -d postgres
```

Useful commands inside `psql`:

| Command | Meaning |
|---------|---------|
| `\l` | List databases |
| `\du` | List users (roles) |
| `\c dbname` | Switch to a database |
| `\dt` | List tables in the current database |
| `\q` | Quit |

## Create a database for a new project

```sql
CREATE USER project1 WITH PASSWORD 'strong-password';
CREATE DATABASE project1 OWNER project1;
```

Or in one line from the shell:

```bash
docker exec -it shared-postgres psql -U admin -d postgres \
  -c "CREATE USER project2 WITH PASSWORD 'strong-password';" \
  -c "CREATE DATABASE project2 OWNER project2;"
```

## Keep projects isolated

By default any user may try to connect to any database. Block that so each project can only use its own:

```sql
REVOKE CONNECT ON DATABASE project1 FROM PUBLIC;
GRANT CONNECT ON DATABASE project1 TO project1;
```

Run the same pair for every database you create.

## Check that it works

```bash
docker exec -it shared-postgres psql -U project1 -d project1 -c "SELECT current_user, current_database();"
```

## Change a user's password

```sql
ALTER USER project1 WITH PASSWORD 'new-strong-password';
```

Update the password in the project's own configuration too.

## Create a read-only user

Useful for reports or dashboards:

```sql
CREATE USER project1_readonly WITH PASSWORD 'another-password';
GRANT CONNECT ON DATABASE project1 TO project1_readonly;
\c project1
GRANT USAGE ON SCHEMA public TO project1_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO project1_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE project1 IN SCHEMA public GRANT SELECT ON TABLES TO project1_readonly;
```

## Delete a project

Back it up first (see [Backup and restore](backup-restore.md)). This cannot be undone.

```sql
DROP DATABASE project1;
DROP USER project1;
```

If it says the database is being accessed by other users, stop the project's app first.

## See how much space each database uses

```sql
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY pg_database_size(datname) DESC;
```

## Using a GUI

Tools such as pgAdmin or DBeaver can connect from your machine with host `localhost`, port `5433`, and the user and password of a project (or the admin account for management tasks).
