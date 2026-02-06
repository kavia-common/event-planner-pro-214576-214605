# Event Planner Pro — PostgreSQL Schema + Seed (Applied)

This document records the **PostgreSQL schema and seed data** that were applied to the `event_database` container.

Per container rules, all DDL/DML was executed **one statement at a time** via `psql -c "..."` using the connection string in `db_connection.txt`:

`psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Tables

### `users`
- `id` UUID PK (default `gen_random_uuid()`)
- `email` unique
- `password_hash` (demo data uses `"demo"`)
- `full_name`
- `created_at`, `updated_at`

### `events`
- `id` UUID PK (default `gen_random_uuid()`)
- `creator_user_id` FK → `users(id)` ON DELETE CASCADE
- `title`, `description`, `location`
- `starts_at`, `ends_at`
- `capacity` (nullable, non-negative check)
- checks:
  - `capacity IS NULL OR capacity >= 0`
  - `ends_at IS NULL OR ends_at >= starts_at`

### `rsvps`
- `id` UUID PK (default `gen_random_uuid()`)
- `event_id` FK → `events(id)` ON DELETE CASCADE
- `user_id` FK → `users(id)` ON DELETE CASCADE
- `status` constrained to: `going | interested | declined`
- unique: `(event_id, user_id)` (one RSVP per user per event)

## Indexes
- `idx_events_creator_user_id` on `events(creator_user_id)`
- `idx_events_starts_at` on `events(starts_at)`
- `idx_rsvps_event_id` on `rsvps(event_id)`
- `idx_rsvps_user_id` on `rsvps(user_id)`

## How to re-run (manual)

Use the connection command from `db_connection.txt` and run each statement individually:

### Drop (clean slate)
1. `DROP TABLE IF EXISTS rsvps`
2. `DROP TABLE IF EXISTS events`
3. `DROP TABLE IF EXISTS users`

### Extension
4. `CREATE EXTENSION IF NOT EXISTS pgcrypto`

### Create tables
5. `CREATE TABLE users (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), email TEXT NOT NULL UNIQUE, password_hash TEXT NOT NULL, full_name TEXT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now())`

6. `CREATE TABLE events (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), creator_user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE, title TEXT NOT NULL, description TEXT, location TEXT, starts_at TIMESTAMPTZ NOT NULL, ends_at TIMESTAMPTZ, capacity INTEGER, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now(), CONSTRAINT events_capacity_nonnegative CHECK (capacity IS NULL OR capacity >= 0), CONSTRAINT events_ends_after_starts CHECK (ends_at IS NULL OR ends_at >= starts_at))`

7. `CREATE TABLE rsvps (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE, user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE, status TEXT NOT NULL DEFAULT 'going', created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now(), CONSTRAINT rsvps_status_valid CHECK (status IN ('going','interested','declined')), CONSTRAINT rsvps_event_user_unique UNIQUE (event_id, user_id))`

### Indexes
8. `CREATE INDEX idx_events_creator_user_id ON events(creator_user_id)`
9. `CREATE INDEX idx_events_starts_at ON events(starts_at)`
10. `CREATE INDEX idx_rsvps_event_id ON rsvps(event_id)`
11. `CREATE INDEX idx_rsvps_user_id ON rsvps(user_id)`

### Seed users (deterministic UUIDs)
12. `INSERT INTO users (id, email, password_hash, full_name) VALUES ('00000000-0000-0000-0000-000000000001','alice@example.com','demo','Alice Example')`
13. `INSERT INTO users (id, email, password_hash, full_name) VALUES ('00000000-0000-0000-0000-000000000002','bob@example.com','demo','Bob Example')`
14. `INSERT INTO users (id, email, password_hash, full_name) VALUES ('00000000-0000-0000-0000-000000000003','carol@example.com','demo','Carol Example')`

### Seed events (deterministic UUIDs)
15. `INSERT INTO events (id, creator_user_id, title, description, location, starts_at, ends_at, capacity) VALUES ('10000000-0000-0000-0000-000000000001','00000000-0000-0000-0000-000000000001','Retro Game Night','Bring your favorite classic game or console.','Arcade Bar Downtown', now() + interval '7 days', now() + interval '7 days' + interval '3 hours', 20)`

16. `INSERT INTO events (id, creator_user_id, title, description, location, starts_at, ends_at, capacity) VALUES ('10000000-0000-0000-0000-000000000002','00000000-0000-0000-0000-000000000002','Synthwave Coding Meetup','Short lightning talks, then pair-programming.','Community Lab', now() + interval '14 days', now() + interval '14 days' + interval '2 hours', 50)`

17. `INSERT INTO events (id, creator_user_id, title, description, location, starts_at, ends_at, capacity) VALUES ('10000000-0000-0000-0000-000000000003','00000000-0000-0000-0000-000000000003','80s Movie Marathon','We vote on the lineup at the start. Snacks provided.','Carol''s Place', now() + interval '3 days', now() + interval '3 days' + interval '5 hours', 12)`

### Seed RSVPs
18. `INSERT INTO rsvps (event_id, user_id, status) VALUES ('10000000-0000-0000-0000-000000000001','00000000-0000-0000-0000-000000000002','going')`
19. `INSERT INTO rsvps (event_id, user_id, status) VALUES ('10000000-0000-0000-0000-000000000001','00000000-0000-0000-0000-000000000003','interested')`
20. `INSERT INTO rsvps (event_id, user_id, status) VALUES ('10000000-0000-0000-0000-000000000002','00000000-0000-0000-0000-000000000001','going')`
21. `INSERT INTO rsvps (event_id, user_id, status) VALUES ('10000000-0000-0000-0000-000000000002','00000000-0000-0000-0000-000000000003','declined')`

### Verification
- `SELECT (SELECT count(*) FROM users) AS users_count, (SELECT count(*) FROM events) AS events_count, (SELECT count(*) FROM rsvps) AS rsvps_count`

Expected counts after seeding: `users=3`, `events=3`, `rsvps=4`.
