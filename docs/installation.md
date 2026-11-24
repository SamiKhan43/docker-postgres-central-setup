# Installation

How to start the shared Postgres container for the first time.

## Requirements

- Linux, macOS, or Windows with Docker Engine and the Docker Compose plugin
- Port `5433` free on your machine

Check that Docker works:

```bash
docker --version
docker compose version
```

If a command needs `sudo`, add yourself to the docker group once, then log out and back in:

```bash
sudo usermod -aG docker $USER
```

## Steps

1. Get the files:

   ```bash
   git clone <your-repo-url> docker-postgres-central-setup
   cd docker-postgres-central-setup
   ```

2. Create your `.env` file:

   ```bash
   cp .env.example .env
   nano .env
   ```

   Set a strong `POSTGRES_PASSWORD`. The defaults `admin` and `postgres` for the user and database are fine.

3. Start the container:

   ```bash
   docker compose up -d
   ```

4. Check the status. Wait until it shows `healthy`:

   ```bash
   docker compose ps
   ```

## Verify it works

```bash
docker exec -it shared-postgres psql -U admin -d postgres -c "SELECT version();"
```

You should see a PostgreSQL 16 version string.

## Using Portainer

Go to **Stacks, Add stack**, paste the contents of `docker-compose.yml`, and add the three variables from `.env.example` under **Environment variables**. Portainer does not read a `.env` file from disk. Then click **Deploy the stack**.

## Daily use

```bash
docker compose stop        # stop, data kept
docker compose start       # start again
docker compose logs -f     # follow logs
docker compose down        # remove container, data kept
docker compose down -v     # remove container AND DELETE ALL DATA
```

## Updating

Minor updates (for example 16.4 to 16.6) are safe. Back up first, then:

```bash
docker compose pull
docker compose up -d
```

A **major version** change (for example `postgres:16` to `postgres:17`) cannot reuse the old data folder. Take a full dump, start a new volume on the new version, and restore. See [Backup and restore](backup-restore.md).

## Changing the port

Edit the left number in the `ports` line of `docker-compose.yml`, for example `127.0.0.1:5432:5432`, then run `docker compose up -d`. Other Docker containers are not affected, because they use port `5432` on the `shared-db` network.
