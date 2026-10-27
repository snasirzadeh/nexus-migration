# Nexus Repository H2-to-PostgreSQL Migration Guide

This guide migrates a Docker-based Nexus Repository Community Edition instance from its embedded H2 database to PostgreSQL using Sonatype's Database Migrator.

> Important: the migration is offline. Nexus must remain stopped from the point noted below until migration finishes.

## 1. Prerequisites

- Nexus Repository Community Edition **3.77.2 or later**. PostgreSQL support for Community Edition starts at 3.77.2.
- Docker and Docker Compose.
- A PostgreSQL server supported by Nexus Repository.
- At least 10 GB of free disk space and enough memory for the migration. Sonatype recommends 16 GB RAM; for very small databases, lower Java memory values may work, but test first.
- A current backup of the Nexus data volume and blob storage.

## 2. Define the database password

Create or update `.env` in the same directory as `compose.yml`:

```dotenv
DB_PASSWORD=use-a-long-unique-password
```

Keep this file private. Do not commit it to source control.

## 3. Configure Docker Compose

Use Nexus 3.77.2 or newer and configure the Nexus service with PostgreSQL connection values:

```yaml
services:
  nexus:
    image: sonatype/nexus3:3.77.2
    environment:
      NEXUS_DATASTORE_NEXUS_JDBCURL: jdbc:postgresql://db:5432/nexus?currentSchema=nexus
      NEXUS_DATASTORE_NEXUS_USERNAME: nexus
      NEXUS_DATASTORE_NEXUS_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env}
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:18.6
    hostname: db
    environment:
      POSTGRES_DB: nexus
      POSTGRES_USER: nexus
      POSTGRES_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env}
    command: ["postgres", "-c", "max_connections=400"]
```

Do not use `PGOPTIONS=-c max_connections=400`. `PGOPTIONS` is applied to client sessions, and PostgreSQL rejects attempts to change `max_connections` that way.

Validate the Compose file:

```bash
docker compose config -q
```

## 4. Create and prepare PostgreSQL

If this is a new PostgreSQL volume, `POSTGRES_DB=nexus` creates the database automatically. If the database is missing in an existing volume, create it manually.

Run these commands while PostgreSQL is running:

```bash
docker exec nexus-db env -u PGOPTIONS psql -U nexus -d postgres -c \
  "CREATE DATABASE nexus OWNER nexus;"

docker exec nexus-db env -u PGOPTIONS psql -v ON_ERROR_STOP=1 -U nexus -d nexus -c \
  "CREATE SCHEMA nexus AUTHORIZATION nexus;
   CREATE EXTENSION pg_trgm SCHEMA nexus;
   GRANT ALL PRIVILEGES ON DATABASE nexus TO nexus;
   GRANT ALL PRIVILEGES ON SCHEMA nexus TO nexus;
   ALTER ROLE nexus IN DATABASE nexus SET search_path = nexus, public;"
```

Verify:

```bash
docker exec nexus-db env -u PGOPTIONS psql -U nexus -d nexus -Atc \
  "SELECT current_database() || '|' || current_user || '|' || current_schema();"
```

Expected output:

```text
nexus|nexus|nexus
```

If an existing PostgreSQL volume was initialized with an older password, synchronize the role password with the current Compose secret before migration:

```bash
docker compose exec -T postgres sh -c \
  'printf "%s\n%s\n" "$POSTGRES_PASSWORD" "$POSTGRES_PASSWORD" | \
   env -u PGOPTIONS psql -U nexus -d postgres -c "\\password nexus"'
```

## 5. Download the matching migrator

Download the migrator matching the Nexus version into a temporary location:

```bash
curl --fail --location --output /tmp/nexus-db-migrator-3.77.2-02.jar \
  https://download.sonatype.com/nexus/nxrm3-migrator/nexus-db-migrator-3.77.2-02.jar

sha256sum /tmp/nexus-db-migrator-3.77.2-02.jar
```

For a different Nexus version, use the corresponding official Database Migrator release.

## 6. Preserve the H2 recovery files

Before stopping Nexus, copy the active H2 files somewhere outside the Nexus volume:

```bash
docker cp nexus-repository:/nexus-data/db/nexus.mv.db \
  /tmp/nexus-pre-postgres-nexus.mv.db
docker cp nexus-repository:/nexus-data/db/nexus.trace.db \
  /tmp/nexus-pre-postgres-nexus.trace.db
```

Also retain a full backup of the Nexus data volume and blob storage. Do not delete the old `db` directory until the migrated service has been fully verified.

## 7. Stop Nexus

```bash
docker stop nexus-repository
```

Confirm it is stopped before continuing:

```bash
docker ps --filter name=nexus-repository
```

## 8. Run the offline H2-to-PostgreSQL migration

Run the migrator from the Nexus `db` directory. The following command uses the Docker network named `nexus_default`; adjust it if your Compose project uses a different network.

```bash
docker run --rm \
  --network nexus_default \
  --env-file .env \
  -v nexus-data:/nexus-data \
  -v /tmp/nexus-db-migrator-3.77.2-02.jar:/tmp/nexus-db-migrator.jar:ro \
  --entrypoint sh sonatype/nexus3:3.77.2 \
  -c 'cd /nexus-data/db && java -Xms1G -Xmx3G -XX:+UseG1GC -XX:MaxDirectMemorySize=1G \
    -jar /tmp/nexus-db-migrator.jar \
    --migration_type=h2_to_postgres \
    --db_url="jdbc:postgresql://nexus-db:5432/nexus?user=nexus&password=${DB_PASSWORD}&currentSchema=nexus" \
    --yes'
```

Successful output includes a summary like:

```text
Migration job finished
... records were migrated
Created 'Rebuild repository browse' and 'Rebuild repository search' tasks
```

Do not run Nexus if the migration command fails. Fix the reported issue, clean or recreate the target PostgreSQL database/schema if necessary, then retry.

## 9. Start Nexus on PostgreSQL

```bash
docker compose up -d postgres nexus
```

Watch the startup log:

```bash
docker logs -f nexus-repository
```

The migrator creates one-time browse and search rebuild tasks. Let them finish; do not restart Nexus during these tasks.

## 10. Verify the migration

Check the containers:

```bash
docker compose ps
```

Check the Nexus HTTP endpoint:

```bash
curl --silent --show-error --fail http://127.0.0.1:8081/ >/dev/null && echo "Nexus is reachable"
```

Check PostgreSQL:

```bash
docker compose exec -T postgres sh -c \
  'env -u PGOPTIONS psql -U nexus -d nexus -Atc \
  "SELECT current_database() || '\''|'\'' || current_user || '\''|'\'' || current_schema();
    SELECT count(*) FROM information_schema.tables WHERE table_schema = '\''nexus'\'';
    SHOW max_connections;"'
```

Finally, sign in to Nexus and confirm repositories, users, blob stores, and hosted artifacts are present. Review the task list until the `Rebuild repository browse` and `Rebuild repository search` tasks are successful.

## 11. After verification

Keep the H2 recovery copy and a backup of the old data for an appropriate retention period. Once the PostgreSQL-backed instance is validated, archive the legacy `/nexus-data/db` directory rather than deleting it immediately.

## Troubleshooting

### `database "nexus" does not exist`

The PostgreSQL volume already existed, so the Docker image initialization script did not rerun. Create the database manually as described in step 4.

### `password authentication failed for user "nexus"`

The password stored in an existing PostgreSQL role differs from the current `.env` value. Reset the role password using the command in step 4, then rerun the migrator.

### `parameter "max_connections" cannot be changed without restarting the server`

Remove `PGOPTIONS=-c max_connections=...`. Set it as the PostgreSQL server command instead:

```yaml
command: ["postgres", "-c", "max_connections=400"]
```

