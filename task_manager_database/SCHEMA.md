# Task Manager Database Schema (PostgreSQL)

This container uses PostgreSQL. The schema is created by executing SQL via `psql` using the connection string in `db_connection.txt` (one statement at a time).

## Extensions
- `pgcrypto` (UUID generation via `gen_random_uuid()`)
- `citext` (case-insensitive email)
- `pg_trgm` (fast `ILIKE` search via trigram GIN indexes)

## Enums
- `task_status`: `todo`, `in_progress`, `done`, `archived`
- `task_priority`: `low`, `medium`, `high`, `urgent`

## Core tables

### `users`
Authentication + profile.
- `id uuid PK`
- `email citext UNIQUE NOT NULL`
- `password_hash text NOT NULL`
- `display_name text`
- `is_active boolean DEFAULT true`
- `created_at`, `updated_at` (`updated_at` maintained by trigger)

### `tasks`
User tasks with status/priority/due dates.
- `id uuid PK`
- `user_id uuid FK -> users(id) ON DELETE CASCADE`
- `project_id uuid FK -> projects(id) ON DELETE SET NULL` (optional)
- `title text NOT NULL`
- `description text`
- `status task_status DEFAULT 'todo'`
- `priority task_priority DEFAULT 'medium'`
- `due_at timestamptz NULL`
- `completed_at timestamptz NULL`
- `created_at`, `updated_at` (`updated_at` maintained by trigger)

Constraints:
- `due_at` must be `NULL` or `>= created_at`
- if `status='done'` then `completed_at IS NOT NULL`

Search:
- trigram GIN index over `title + description` for fast `ILIKE '%q%'`

### `tags`
User-defined tags.
- `id uuid PK`
- `user_id uuid FK -> users(id) ON DELETE CASCADE`
- `name text NOT NULL`
- `color text NULL`
- `created_at timestamptz`

Uniqueness:
- unique (expression) index on `(user_id, lower(name))` to prevent duplicates like `Work` vs `work`.

### `task_tags`
Join table between tasks and tags.
- `task_id uuid FK -> tasks(id) ON DELETE CASCADE`
- `tag_id uuid FK -> tags(id) ON DELETE CASCADE`
- composite PK `(task_id, tag_id)` prevents duplicates
- index on `tag_id` for reverse lookups

## Optional tables included

### `projects`
- `id uuid PK`
- `owner_id uuid FK -> users(id) ON DELETE CASCADE`
- `name text NOT NULL`
- unique `(owner_id, name)`
- `created_at`, `updated_at` (trigger)

### `comments`
- `id uuid PK`
- `task_id uuid FK -> tasks(id) ON DELETE CASCADE`
- `user_id uuid FK -> users(id) ON DELETE CASCADE`
- `body text NOT NULL`
- `created_at timestamptz`
- index `(task_id, created_at)` for chronological listing

### `activity`
- `id uuid PK`
- `user_id uuid FK -> users(id) ON DELETE CASCADE`
- `task_id uuid FK -> tasks(id) ON DELETE CASCADE` (nullable)
- `action text NOT NULL`
- `details jsonb DEFAULT '{}'`
- `created_at timestamptz`
- index `(user_id, created_at desc)` for recent activity feed

### `notifications`
- `id uuid PK`
- `user_id uuid FK -> users(id) ON DELETE CASCADE`
- `type text NOT NULL`
- `title text NOT NULL`
- `body text`
- `data jsonb DEFAULT '{}'`
- `is_read boolean DEFAULT false`
- `created_at`, `read_at`
- index `(user_id, is_read, created_at desc)` for unread + recent listing

## Convenience view

### `v_tasks_with_tags`
Aggregates tags into a JSONB array per task:
- Columns: all `tasks.*` plus `tags` (JSONB array of `{id,name,color}`)

Useful for API responses to avoid N+1 queries.

## Key indexes (performance)
- `tasks(user_id, status, due_at NULLS LAST)` — filter by status + sort by due date
- `tasks(user_id, priority)` — filter/sort by priority
- `tasks(user_id, created_at DESC)` — recent-first lists
- partial: `tasks(user_id, due_at) WHERE due_at IS NOT NULL` — due-date-only queries
- `tasks` trigram GIN search index — fast `ILIKE` search
- `tags` trigram GIN search index — fast tag search/autocomplete
- `task_tags(tag_id)` — tasks-by-tag queries
