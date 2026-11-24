# Backup and Restore

Docker volumes are not backups. If the volume is deleted or damaged, the data is gone. Take regular dumps and store copies outside this machine.

## Back up everything

Saves all databases, users and passwords:

```bash
docker exec -t shared-postgres pg_dumpall -U admin > all-databases.sql
ls -lh all-databases.sql
```

## Back up one project

```bash
docker exec -t shared-postgres pg_dump -U admin project1 > project1.sql
```

Compressed custom format (smaller, and allows selective restore):

```bash
docker exec -t shared-postgres pg_dump -U admin -Fc project1 > project1.dump
```

## Restore

**A full dump** (`pg_dumpall`) into a fresh container:

```bash
cat all-databases.sql | docker exec -i shared-postgres psql -U admin -d postgres
```

Messages such as "role already exists" are normal when restoring into a server that has the admin user.

**One project** from a plain SQL dump. Create the empty database first:

```bash
docker exec -it shared-postgres psql -U admin -d postgres \
  -c "CREATE USER project1 WITH PASSWORD 'strong-password';" \
  -c "CREATE DATABASE project1 OWNER project1;"

cat project1.sql | docker exec -i shared-postgres psql -U admin -d project1
```

**One project** from a `.dump` file:

```bash
cat project1.dump | docker exec -i shared-postgres pg_restore -U admin -d project1 --no-owner --role=project1
```

## Daily automatic backup

Create `backup.sh` outside the repo, or keep it in a git-ignored `backups/` folder:

```bash
#!/bin/bash
set -e
DIR="$HOME/pg-backups"
mkdir -p "$DIR"
docker exec -t shared-postgres pg_dumpall -U admin | gzip > "$DIR/all-$(date +%F).sql.gz"
# Keep the last 14 days
find "$DIR" -name 'all-*.sql.gz' -mtime +14 -delete
```

Make it executable and test it:

```bash
chmod +x backup.sh
./backup.sh
```

Schedule it with `crontab -e` (every day at 02:00):

```
0 2 * * * /home/youruser/backup.sh
```

## Test your backups

A backup you have never restored is only a guess. Once in a while, restore into a temporary container and check the data:

```bash
docker run -d --name pg-restore-test -e POSTGRES_PASSWORD=test postgres:16
sleep 5
gunzip -c all-2025-01-01.sql.gz | docker exec -i pg-restore-test psql -U postgres
docker rm -f pg-restore-test
```

Use the real file name of one of your backups.

## Moving to a new major version

1. Take a full dump with `pg_dumpall`.
2. Run `docker compose down -v` (this deletes the old data, so be sure the dump is good).
3. Change the image in `docker-compose.yml`, for example to `postgres:17`.
4. Run `docker compose up -d` and wait until it is healthy.
5. Restore the dump.

## Never commit dumps

`.gitignore` excludes `*.sql`, `*.dump`, `*.sql.gz` and `backups/`. Dumps contain your data and password hashes.
