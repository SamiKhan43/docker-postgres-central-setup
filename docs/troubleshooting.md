# Troubleshooting

Start with the logs and the status:

```bash
docker compose ps
docker compose logs --tail 50 postgres
```

## Port is already allocated

```
Bind for 127.0.0.1:5433 failed: port is already allocated
```

Something else uses the port. Find it:

```bash
sudo ss -tulpn | grep 5433
```

Stop that program, or change the left number in the `ports` line of `docker-compose.yml`.

## Variable is not set, or the container exits at once

Docker Compose prints "variable is not set" or Postgres says the superuser password is not specified. The `.env` file is missing or in the wrong folder.

- It must be named exactly `.env` and sit next to `docker-compose.yml`.
- Check with `ls -a`. Make sure it is not `.env.txt`.
- In Portainer, set the variables in the stack's **Environment variables** section instead.

## Password authentication failed

```
FATAL: password authentication failed for user "project1"
```

- Check the user, password and database in your connection string for typos.
- If the password has special characters, URL-encode them.
- If you changed `POSTGRES_PASSWORD` in `.env` after the first start, the old password is still in use. Postgres only reads that variable when the volume is first created. Change the password directly:

  ```bash
  docker exec -it shared-postgres psql -U admin -d postgres -c "ALTER USER admin WITH PASSWORD 'new-password';"
  ```

- Starting from scratch is also an option, but it deletes all data: `docker compose down -v`, then `docker compose up -d`.

## Connection refused

- Check the container is running and healthy: `docker compose ps`.
- On your machine, use `localhost` and port `5433`.
- From another container, use `shared-postgres` and port `5432`, and make sure the container is on the `shared-db` network.
- Check that `pg_isready` answers:

  ```bash
  docker exec shared-postgres pg_isready -U admin
  ```

## Network shared-db declared as external, but could not be found

Start this repo's compose file first. It creates the network. Or create it by hand:

```bash
docker network create shared-db
```

## Permission denied on /var/run/docker.sock

Add your user to the docker group, then log out and back in:

```bash
sudo usermod -aG docker $USER
```

## Database is being accessed by other users

You cannot drop a database with active connections. Stop the project's app, or disconnect everyone:

```sql
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'project1' AND pid <> pg_backend_pid();
```

## Container is unhealthy

Read the logs. Common causes are a full disk (`df -h`, `docker system df`), a corrupted volume, or a changed major version on an existing volume. A major version mismatch shows "database files are incompatible with server". Switch the image back, or dump and restore (see [Backup and restore](backup-restore.md)).

## Permission denied for schema public

This appears when a project user is not the owner of its database. Create databases with `OWNER project1`, or fix an existing one:

```sql
ALTER DATABASE project1 OWNER TO project1;
```

## Disk space is filling up

```bash
docker system df
docker image prune
```

Do not run `docker system prune --volumes` here. It can delete volumes that are not attached to a running container.

## Reset everything

This deletes all databases:

```bash
docker compose down -v
docker compose up -d
```

Take a backup first (see [Backup and restore](backup-restore.md)).
