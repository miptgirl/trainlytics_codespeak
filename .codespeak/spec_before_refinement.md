# Database Models & Schema

## Overview

PostgreSQL database accessed via SQLAlchemy (async) with Alembic for migrations. All data is scoped per user via a `user_id` string on every top-level entity. The ORM uses a shared `Base` class; an async session factory is provided as a FastAPI dependency (`get_db`).

Migrations are managed by Alembic with both offline and online (async) modes supported. The `DATABASE_URL` environment variable overrides the URL in `alembic.ini`, enabling Docker/CI usage without file edits.

---

## Domain Entities

### Exercise
- User-owned exercise definition with a name and optional notes.
- Can have zero or more **exercise types** (tags) associated via a many-to-many join table (`exercise_exercise_types`). Deleting an exercise or a type removes the association; deleting an exercise that is referenced by a session or template is restricted.

### Exercise Type
- A user-defined label/tag (e.g. "push", "legs") that can be attached to exercises.
- Per-user, name only.

### Cardio Activity Type
- A user-defined label for a cardio activity (e.g. "running", "cycling").
- Per-user, name only.

---

## Workout Sessions

Sessions use a **single-table inheritance-like** pattern: every session is a row in `workout_sessions` (with `type` = `"cardio"` or `"strength"`), plus exactly one child row in either `cardio_sessions` or `strength_sessions`. Deleting the parent cascades to the child.

### WorkoutSession (base)
- `user_id`, `type`, `date` (timestamp with timezone), optional `title`, `notes`, `calories`.

### Cardio Session
- Links to an optional `CardioActivityType` (set to NULL if the type is deleted).
- Optional `total_duration_seconds`.
- Has an ordered list of **CardioSegments**.

### Cardio Segment
- Represents an interval within a cardio session (e.g. a lap or a pace zone).
- Fields: `order`, `duration_seconds` (required), optional `distance_meters`, `pace_seconds_per_km`, `heart_rate_avg`, `title`.

### Strength Session
- Optional `duration_seconds`.
- Has an ordered list of **StrengthExerciseEntries**.

### Strength Exercise Entry
- References an `Exercise` (delete restricted — the exercise cannot be removed while it is used in a session).
- Has an `order` and an ordered list of **StrengthSets**.

### Strength Set
- `set_number`, optional `reps`, `weight` (float), optional `notes`.

---

## Strength Templates

Reusable workout blueprints, parallel in structure to strength sessions.

### StrengthTemplate
- Per-user, has `name`, optional `notes`, `created_at`, `updated_at`.
- Contains an ordered list of **StrengthTemplateExercises**.

### StrengthTemplateExercise
- References an `Exercise` (delete restricted).
- Contains an ordered list of **StrengthTemplateSets**.

### StrengthTemplateSet
- `set_number`, optional `reps`, `weight_kg` (float), optional `notes`.
- Note: template sets store weight as `weight_kg` while session sets store it as `weight` (unit-agnostic).

---

## API Schemas (Pydantic)

Each entity exposes three schema variants:
- **Create / Input** — fields required for creation.
- **Patch / Update** — all fields optional, for partial updates.
- **Out / Read** — full representation returned to the client.

### Session List & Analytics
- **SessionSummaryOut** — lightweight row for history lists; includes computed summary fields (total duration, distance, set count, exercise count, total volume) so the list view doesn't need to load full session detail.
- **SessionListOut** — paginated wrapper (`items`, `total`, `page`, `page_size`).
- **WeeklySummaryOut** — aggregated minutes and calories split by cardio and strength for the current week.
- **TrainingTrendPoint** — per-week data point with cardio/strength minutes and calories, for trend charts.

### StrengthExerciseEntryOut
- Includes `exercise_name` (denormalized) alongside `exercise_id` to avoid a separate lookup in the UI.

### StrengthTemplateSummary
- Lightweight template list row including `exercise_count` (computed).

---

## Migration History

| Revision | Description |
|----------|-------------|
| 0001 | `exercises` table |
| 0002 | `cardio_activity_types` table |
| 0003 | `workout_sessions`, `cardio_sessions`, `cardio_segments` tables |
| 0004 | `strength_sessions`, `strength_exercise_entries`, `strength_sets` tables |
| 0005 | `strength_templates`, `strength_template_exercises`, `strength_template_sets` tables |
| 0006 | Added `title` and `calories` to `workout_sessions`; `title` to `cardio_segments`; `duration_seconds` to `strength_sessions`; changed `workout_sessions.date` from `Date` to `DateTime(timezone=True)` |
| 0007 | `exercise_types` table and `exercise_exercise_types` join table |
