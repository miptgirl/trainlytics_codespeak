# Backend API Test Suite

Async integration tests covering the full backend HTTP API. Tests run against a real FastAPI application using an in-memory SQLite database, exercising actual HTTP request/response cycles.

## Test Infrastructure

### Database

Each test that touches the DB uses the `db_session` fixture, which spins up a fresh in-memory SQLite database and wires it into the app's dependency injection. The DB is torn down after every test, ensuring full isolation between tests.

### Users

Authentication is driven by a patched `settings.users` value — no real user store is needed. Two test users are available (`testuser` / `otheruser`) with bcrypt hashes generated at low cost (`rounds=4`) to keep tests fast.

Three client fixtures are provided:

- `client` — unauthenticated
- `auth_client` — authenticated as `testuser`
- `auth_client_2` — authenticated as `otheruser`

## Auth (`/api/auth`)

- `POST /api/auth/login` with valid credentials returns `200`, a Bearer `access_token` in the body, and a `refresh_token` HTTP-only cookie.
- Wrong password or unknown user returns `401`.
- `POST /api/auth/refresh` with a valid cookie returns a new `access_token`; without a cookie returns `401`.
- `POST /api/auth/logout` returns `200` and clears the `refresh_token` cookie.
- JWT `sub` claim encodes the username — tokens are user-scoped.

## Exercise Types (`/api/exercise-types`)

A named category that can be attached to exercises (e.g. "Push", "Compound").

- CRUD: create (201), list (200), rename via PATCH (200), delete (204).
- Created resource includes `id`, `name`, `user_id`, `created_at`.
- Non-existent ID returns `404`.
- All endpoints require authentication; unauthenticated requests return `401`.
- **User isolation**: each user only sees and can modify their own types. Attempting to edit another user's type returns `404`.
- **Cascade on delete**: deleting an exercise type removes it from any exercises that referenced it (join table rows removed); the exercises themselves remain.
- **Reverse cascade**: deleting an exercise removes its join table entries without affecting the type.

## Cardio Types (`/api/cardio-types`)

Same shape and rules as exercise types, without the cascade behaviour.

- CRUD: create (201), list (200), rename via PATCH (200), delete (204).
- Created resource includes `id`, `name`, `user_id`, `created_at`.
- Non-existent ID returns `404`.
- All endpoints require authentication; unauthenticated requests return `401`.
- **User isolation**: each user only sees and can modify their own types. Attempting to edit another user's type returns `404`.

## Exercises (`/api/exercises`)

A named movement in the user's exercise library, optionally annotated with notes and tagged with exercise types.

- CRUD: create (201), list (200), update via PATCH (200), delete (204).
- Created resource includes `id`, `name`, `notes` (nullable), `user_id`, `created_at`, and a `types` list.
- `type_ids` on create/patch sets the exercise's type associations; passing `[]` clears them.
- Referencing a non-existent or another user's `type_id` returns `400`.
- Non-existent exercise ID returns `404`.
- Partial PATCH: only supplied fields are updated; unmentioned fields stay unchanged.
- All endpoints require authentication; unauthenticated requests return `401`.
- **User isolation**: exercises are not visible to other users. Attempting to edit another user's exercise returns `404`.

## Sessions (`/api/sessions`)

Two session types: **cardio** and **strength**. Both share the same detail/delete endpoints.

### Cardio Sessions (`POST /api/sessions/cardio`)

- Fields: `date`, `title` (nullable), `notes` (nullable), `calories` (nullable), `total_duration_seconds` (nullable), `activity_type_id` (nullable), `segments`.
- Each segment has: `order`, `duration_seconds`, `distance_meters` (nullable), `pace_seconds_per_km` (nullable), `heart_rate_avg` (nullable), `title` (nullable).
- Response includes all fields plus `id`; `null` optionals are preserved as `null`.

### Strength Sessions (`POST /api/sessions/strength`)

- Fields: `date`, `title` (nullable), `notes` (nullable), `calories` (nullable), `duration_seconds` (nullable), `exercises`.
- Each session exercise has: `exercise_id`, `order`, `sets`.
- Each set has: `set_number`, `reps`, `weight`.
- Response `type` field is `"strength"`, and each exercise entry includes `exercise_name`.

### Shared Behaviour

- `GET /api/sessions/{id}` — returns full detail; `404` if not found or belongs to another user.
- `PATCH /api/sessions/{id}` — replaces the session's mutable fields; for strength sessions, the exercises/sets list is fully replaced.
- `DELETE /api/sessions/{id}` — returns `204`; subsequent GET returns `404`.
- All endpoints require authentication; unauthenticated requests return `401`.
- **User isolation**: sessions are not visible to other users; get/delete of another user's session returns `404`.

### Session History (`GET /api/sessions`)

- Returns `{ items, total }` with summary entries for both session types.
- Summary for cardio includes `total_duration_seconds`, `title`, `calories`.
- Summary for strength includes `total_sets`, `title`, `calories`, `duration_seconds`.
- Filters: `type` (cardio/strength), `date_from`, `date_to`.
- Pagination: `page` and `page_size` query params; `total` always reflects the unfiltered count.
- Results are ordered reverse-chronologically (newest first).
- User-isolated: users only see their own sessions.

### Weekly Summary (`GET /api/sessions/weekly-summary?week_start=YYYY-MM-DD`)

- Returns `{ cardio: { minutes, calories }, strength: { minutes, calories } }` for the 7-day window starting at `week_start`.
- Sessions outside that window are excluded.
- `null` calories are treated as 0 in the aggregate.
- User-isolated.

### Training Trends (`GET /api/sessions/training-trends?weeks=N`)

- Returns a list of `N+1` data points (the last N complete weeks plus the current in-progress week).
- Each point has `week_start`, `cardio_minutes`, `strength_minutes`, `cardio_calories`, `strength_calories`.
- Points are ordered chronologically (oldest first).
- Weeks with no sessions have all-zero values.
- User-isolated.

## Strength Templates (`/api/templates/strength`)

Saved workout blueprints consisting of ordered exercises with prescribed sets.

- CRUD: create (201), list (200), get detail (200), update via PATCH (200), delete (204).
- Template fields: `name`, `notes` (nullable), `exercises`, `id`, `created_at`, `updated_at`.
- Each template exercise has `exercise_id`, `order`, `sets`; each set has `set_number`, `reps`, `weight_kg`.
- Response includes `exercise_name` resolved from the library.
- List response includes `exercise_count` (number of distinct exercises) per template.
- Referencing an unknown `exercise_id` returns `400`.
- `updated_at` is updated on PATCH.
- Partial PATCH: supplying only `name` leaves exercises unchanged.
- On PATCH with exercises, the exercises/sets list is fully replaced.
- Non-existent template ID returns `404`.
- All endpoints require authentication.
- **User isolation**: templates are not visible to other users; get/patch/delete of another user's template returns `404`; list returns empty for other user.
- **Cascade on delete**: deleting a template removes its exercise and set rows; the underlying exercises in the library are unaffected.
- Deleting an exercise from the library does not cascade-delete any templates (exercises remain referenced by name).
