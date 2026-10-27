# Changes

- Updated Nexus Repository from 3.77.1 to 3.77.2, the first Community Edition release that supports PostgreSQL.
- Configured Nexus to connect to the `nexus` PostgreSQL database and `nexus` schema using credentials from `DB_PASSWORD` in `.env`.
- Replaced the invalid client-side `PGOPTIONS=-c max_connections=400` setting with a PostgreSQL server command-line setting.

## Migration record

- Created the missing PostgreSQL `nexus` database and `nexus` schema, installed `pg_trgm`, and set the `nexus` role's database search path.
- Synchronized the existing PostgreSQL role password with the current `DB_PASSWORD` value in `.env`. The persistent database volume was initialized with an earlier password, which caused the initial migrator connection attempt to fail before it wrote data.
- Preserved the original H2 files at `/tmp/nexus-pre-postgres-nexus.mv.db` and `/tmp/nexus-pre-postgres-nexus.trace.db` before the migration.
- Ran the official `nexus-db-migrator-3.77.2-02.jar` offline with `h2_to_postgres`. It migrated 63 records with 0 skips. Four scheduled-task records were intentionally filtered by the migrator.
- Started Nexus successfully on PostgreSQL. The target has 188 tables in the `nexus` schema; Nexus completed its one-time browse and search rebuild tasks.
