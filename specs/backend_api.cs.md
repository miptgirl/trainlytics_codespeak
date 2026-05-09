---
import specs/data_layer.cs.md
---
# Trainlytics Backend API

FastAPI application serving the Trainlytics workout tracking backend. All routes are prefixed with `/api`.

## Configuration & Setup

- CORS is enabled for `http://localhost:5173`
- Exposes a `GET /api/health` endpoint returning `{"status": "ok"}`
- Settings are loaded from environment variables / `.env` file:
  - `DATABASE_URL` — PostgreSQL async connection string
  - `SECRET_KEY` — JWT signing secret (must be changed in production)
  - `ALGORITHM` — JWT algorithm (default: HS256)
  - `ACCESS_TOKEN_EXPIRE_MINUTES` — default 30
  - `REFRESH_TOKEN_EXPIRE_DAYS` — default 30
  - `USERS` — comma-separated `username:bcrypt_hash` pairs defining valid users (e.g. `alice:$2b$...,bob:$2b$...`)

## Authentication

Users are not stored in the database — they are defined entirely via the `USERS` config value. Passwords are stored as bcrypt hashes.

### Tokens

- **Access token**: short-lived JWT, returned in the response body as `{ access_token, token_type: "bearer" }`
- **Refresh token**: long-lived JWT, set as an `httponly`, `SameSite=lax` cookie named `refresh_token`

### Endpoints (`/api/auth`)

- `POST /login` — accepts `{ username, password }`, returns access token and sets refresh cookie
- `POST /refresh` — reads refresh cookie, issues a new access token
- `POST /logout` — clears the refresh cookie

### Route Protection

All non-auth endpoints require a valid Bearer access token. The authenticated username is derived from the token's `sub` claim and used to scope all data queries — users can only access their own data.

## Domain Resources

All resources are user-scoped: every read, write, and delete operation is restricted to the currently authenticated user.

### Exercise Types (`/api/exercise-types`)

User-defined categories/labels for strength exercises (e.g. "Push", "Legs").

- `GET /` — list all, ordered by creation time
- `POST /` — create; body: `{ name }`
- `PATCH /{type_id}` — partial update
- `DELETE /{type_id}` — delete

### Cardio Activity Types (`/api/cardio-types`)

User-defined categories for cardio activities (e.g. "Running", "Cycling").

- `GET /` — list all, ordered by creation time
- `POST /` — create; body: `{ name }`
- `PATCH /{type_id}` — partial update
- `DELETE /{type_id}` — delete

### Exercises (`/api/exercises`)

Named movements that can be logged in strength sessions. Each exercise can have zero or more exercise types (tags) associated with it.

- `GET /` — list all, ordered by creation time
- `POST /` — create; body: `{ name, notes?, type_ids[] }` — `type_ids` must reference the user's own exercise types
- `PATCH /{exercise_id}` — partial update; supplying `type_ids` replaces the full set of associated types
- `DELETE /{exercise_id}` — delete

### Workout Sessions (`/api/sessions`)

Recorded workouts. Two session types are supported: **cardio** and **strength**. All sessions share common fields: `date`, `title`, `notes`, `calories`.

#### Cardio Sessions

A cardio session records an activity type, total duration, and an ordered list of **segments**. Each segment can carry pace, distance, heart rate, and other metrics.

- `POST /cardio` — create with segments
- `PATCH /{session_id}` — update; supplying `segments` replaces all existing segments entirely

#### Strength Sessions

A strength session records a duration and an ordered list of **exercise entries**. Each entry references an exercise and contains an ordered list of **sets** (reps, weight, notes).

- `POST /strength` — create with exercise entries and sets
- `PATCH /{session_id}` — update; supplying `exercises` replaces all exercise entries and sets entirely

#### Shared Session Endpoints

- `GET /` — paginated session list with optional filters: `type`, `date_from`, `date_to`, `page`, `page_size` (max 100). Returns summaries:
  - Cardio summaries include: total duration, total distance (summed across segments)
  - Strength summaries include: exercise count, total sets, total volume (weight × reps), duration
- `GET /{session_id}` — full session detail (cardio or strength, determined at runtime)
- `DELETE /{session_id}` — delete

#### Analytics

- `GET /weekly-summary?week_start=<ISO date>` — aggregated cardio and strength totals (minutes, calories) for the Monday–Sunday week containing the given date
- `GET /training-trends?weeks=<N>` — per-week breakdown of cardio and strength minutes and calories for the last N weeks (1–52) plus the current in-progress week

### Strength Templates (`/api/templates/strength`)

Reusable workout plans. A template has a name, optional notes, and an ordered list of exercise entries with predefined sets (set number, reps, weight in kg, notes).

- `GET /` — list summaries (includes exercise count, timestamps)
- `GET /{template_id}` — full template with exercises and sets
- `POST /` — create
- `PATCH /{template_id}` — partial update; supplying `exercises` replaces all exercise entries and sets entirely; `updated_at` is refreshed on every update
- `DELETE /{template_id}` — delete
