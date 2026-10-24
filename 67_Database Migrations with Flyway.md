# Database Migrations with Flyway

> **Part of:** Spring gap topics (63–71) · 🟦 NestJS: TypeORM/Prisma migrations ≈ Flyway/Liquibase
> **Replaces:** `spring.jpa.hibernate.ddl-auto=update` ([19](19_Spring%20Data%20JPA%20-%20Entity%2C%20Repository%20and%20Database%20Config.md))

## Why this topic matters
`ddl-auto=update` is fine for demos but unacceptable in production. Interviewers ask how you manage schema changes, how migrations run on deploy, and how to do **zero-downtime** changes.

---

## 1. Why not `ddl-auto=update`?
- It never drops or renames safely and can't migrate data.
- Changes aren't reviewed, versioned, or reproducible across environments.
- Behaviour depends on entity code at startup (dangerous with multiple instances).
- There's no rollback story.
**Migrations** = versioned SQL scripts in Git, applied in order, tracked in a DB table. The same history applies to dev, CI, staging, and prod.

## 2. Setup (Spring Boot auto-runs Flyway on startup)
```xml
<dependency><groupId>org.flywaydb</groupId><artifactId>flyway-core</artifactId></dependency>
<dependency><groupId>org.flywaydb</groupId><artifactId>flyway-mysql</artifactId></dependency>   <!-- DB-specific module (Flyway 9+) -->
```
```properties
spring.jpa.hibernate.ddl-auto=validate          # Hibernate only CHECKS the schema matches entities
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true          # for an existing DB without Flyway history
```
Boot runs Flyway **before** the JPA EntityManagerFactory starts, so the schema is ready when Hibernate validates.

## 3. Naming and types
```
src/main/resources/db/migration/
├── V1__create_students_table.sql
├── V2__add_email_unique_index.sql
├── V3__create_departments_and_fk.sql
├── V4__backfill_student_status.sql
└── R__refresh_reporting_views.sql
```
| Prefix | Meaning |
|---|---|
| `V<version>__<desc>.sql` | **Versioned**: runs once, in order (two underscores!) |
| `R__<desc>.sql` | **Repeatable**: re-runs whenever its checksum changes (views, functions) |
| `U<version>__…` | Undo (paid editions only) |
Java-based migrations: a class extending `BaseJavaMigration` for complex data transforms.

```sql
-- V1__create_students_table.sql
CREATE TABLE students (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(150) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uk_students_email UNIQUE (email)
);
```

## 4. How Flyway tracks state
Table **`flyway_schema_history`**: version, description, script, **checksum**, installed_on, execution time, success.
On startup: scan scripts → compare with history → **validate** checksums → apply pending versions in order.
- **Never edit an applied migration.** The checksum mismatch fails startup (`Validate failed: Migration checksum mismatch`). Add a new version instead.
- Out-of-order versions (from parallel branches) are rejected unless `outOfOrder=true`.
- A failed migration leaves a failed row: fix it, then `flyway repair` (MySQL DDL isn't transactional, so partial changes may need manual cleanup).
- Concurrent startup of several pods: Flyway takes a **DB lock**, so only one applies the migrations. Many teams instead run migrations as a separate deploy step (init container / CI job).

## 5. Zero-downtime changes: expand → migrate → contract
Old and new app versions run **at the same time** during a rolling deploy, so every schema change must work with **both**.
**Renaming `name` → `full_name`:**
1. **Expand** (release N): `V5__add_full_name.sql` adds a nullable `full_name`; the app **writes both** columns, reads `name`.
2. **Migrate**: backfill `UPDATE students SET full_name = name WHERE full_name IS NULL` (in batches for big tables).
3. Release N+1: the app reads and writes `full_name` only.
4. **Contract** (release N+2): `V7__drop_name.sql` drops the old column.
Rules of thumb:
- Add columns as **nullable** or with defaults; add NOT NULL later.
- Never rename or drop in the same release that stops using a column.
- Big tables: create indexes online (`ALGORITHM=INPLACE, LOCK=NONE` in MySQL / `CREATE INDEX CONCURRENTLY` in PostgreSQL), and batch backfills.

## 6. Flyway vs Liquibase
| Flyway | Liquibase |
|---|---|
| Plain **SQL** files (Java optional) | Changelogs in XML/YAML/JSON/SQL |
| Simple, convention-based | DB-agnostic abstractions, preconditions, rollback blocks |
| Very common with Spring Boot | Common in enterprises needing multi-DB support |

## 7. Testing
Run migrations in integration tests against a **real DB via Testcontainers** (H2 may accept or reject SQL that differs from MySQL). Use `@DataJpaTest` + `@AutoConfigureTestDatabase(replace = NONE)` + Testcontainers, so Flyway + `ddl-auto=validate` catch drift in CI.

---

## Interview Q&A
**Q1. Why use Flyway/Liquibase instead of `ddl-auto=update`?**
Versioned, reviewed, repeatable, data-capable migrations with a history table. `update` is unsafe and untracked.

**Q2. How does Flyway know which migrations to run?**
It compares the scripts against `flyway_schema_history` (version + checksum) and applies the pending ones in order.

**Q3. What happens if you edit an already-applied migration?**
Checksum validation fails on startup. Create a new migration instead (or `repair` only to fix history metadata).

**Q4. Versioned vs repeatable migrations?**
Run once in version order vs re-run when the content changes (views, procedures).

**Q5. How do you rename a column with zero downtime?**
Expand/contract: add the new column, dual-write, backfill, switch reads, then drop the old column in a later release.

**Q6. What should `ddl-auto` be in production?**
`validate` (or `none`), with the schema owned by migrations.

**Q7. Multiple instances start at once: do migrations run multiple times?**
No. Flyway uses a lock and the history table. Many teams still run migrations as a separate pipeline step.

## Quick revision
- Migrations = versioned SQL in Git + `flyway_schema_history`; prod uses `ddl-auto=validate`.
- `V1__desc.sql` (once, ordered), `R__desc.sql` (on change); never edit applied ones.
- Zero downtime = expand → migrate (backfill) → contract over several releases.
- Test migrations with Testcontainers on the real DB engine.
