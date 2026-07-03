# Project 5: Mixtape Bug Hunt — Submission

## Milestone 1: Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for a social music app. There is no
frontend — every endpoint returns JSON. The architecture is a clean three-layer
stack, and the same shape repeats everywhere:

```
HTTP request → routes/ (parse input, format response) → services/ (business logic) → models.py (data)
```

### The main files and what each one does

**`app.py`** — The application factory (`create_app`). It configures SQLAlchemy
(SQLite at `mixtape.db`), initializes the shared `db` object, registers the four
blueprints under URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and
calls `db.create_all()`. Note: it must be launched with
`FLASK_APP=app:create_app flask run`, **not** `python app.py`, to avoid a
double-import of the module (which would register models twice).

**`models.py`** — Defines 6 SQLAlchemy models plus 3 association tables. All
primary keys are UUID strings.

- **`User`** — username, email, `listening_streak` (int), `last_listened_at`
  (datetime). Has a self-referential many-to-many `friends` relationship through
  the `friendships` table.
- **`Song`** — title, artist, album, genre, `shared_by` (FK → the user who
  shared it), `shared_at`, `share_note`. Many-to-many with `Tag`.
- **`Tag`** — just a name; linked to songs via the `song_tags` join table.
- **`ListeningEvent`** — one row per listen (`user_id`, `song_id`,
  `listened_at`). This is the raw event log that both streaks and the feed are
  built from.
- **`Rating`** — a user's 1–5 score for a song. Has a `UniqueConstraint` on
  `(user_id, song_id)`, so a user can rate a song at most once (re-rating
  updates the existing row rather than inserting a new one).
- **`Playlist`** — name, `created_by`, `is_collaborative`. Songs are attached
  through the **`playlist_entries`** association table, which is the interesting
  one: it carries extra columns `position` (explicit ordering — songs have a
  defined order, not just insertion order), `added_by`, and `added_at`.
- **`Notification`** — `user_id` (recipient), `notification_type`, `body`,
  `read` flag. Generated when friends interact with a user's shared songs.

**`routes/`** — Four thin blueprints. Each function reads query params / JSON
body, calls exactly one service function, wraps the result in `jsonify`, and
maps a raised `ValueError` to an HTTP 404/400. **No business logic lives here.**
- `songs.py` — `/songs/search`, `/songs/<id>`, `/songs/<id>/rate` (POST),
  `/songs/<id>/listen` (POST)
- `playlists.py` — `/playlists/` (POST), `/playlists/<id>`,
  `/playlists/<id>/songs` (GET + POST)
- `users.py` — `/users/<id>`, `/users/<id>/streak`,
  `/users/<id>/notifications`, `/users/notifications/<id>/read` (POST)
- `feed.py` — `/feed/<user_id>/listening-now`, `/feed/<user_id>/activity`

**`services/`** — Where all the logic (and all five bugs) live:
- `streak_service.py` — records listening events and updates the consecutive-day
  streak (`record_listening_event`, `update_listening_streak`, `get_streak`).
- `feed_service.py` — builds the "Friends Listening Now" feed
  (`get_friends_listening_now`, using a 24h `RECENT_THRESHOLD`) and the general
  `get_activity_feed`.
- `search_service.py` — `search_songs` (title/artist `ILIKE` match, left-joined
  to tags) and `get_song`.
- `notification_service.py` — `create_notification`, `add_to_playlist` (adds a
  song **and** notifies the sharer), `rate_song`, `get_notifications`,
  `mark_as_read`.
- `playlist_service.py` — `create_playlist`, `get_playlist_songs` (ordered by
  `position`), `get_playlist`, `get_user_playlists`.

**`seed_data.py`** — Populates the DB with test users, friendships, songs, tags,
listening events, playlists, and ratings so the endpoints have data to return.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`.

### Data flow trace — user rates a song

This is the call chain the README points at, traced end to end:

1. Client sends `POST /songs/<song_id>/rate` with a JSON body
   `{"user_id": ..., "score": ...}`.
2. **`routes/songs.py::rate()`** parses `user_id` and `score`, returns 400 if
   either is missing, then calls `rate_song(user_id, song_id, int(score))`.
3. **`notification_service.py::rate_song()`** validates the score is 1–5, loads
   the `Song` and the rater `User` (raising `ValueError` → 404/400 if missing),
   then checks for an existing `Rating` on `(user_id, song_id)`. If one exists it
   updates the score; otherwise it inserts a new `Rating`. It commits and returns
   the `Rating`.
4. The route serializes `rating.to_dict()` and returns it with HTTP 201.

> **Observation while tracing:** unlike `add_to_playlist` — which, after adding
> the song, calls `create_notification(...)` for the song's original sharer —
> `rate_song` never creates a notification. That maps directly onto Issue #4
> ("notified when a friend added my song to a playlist but not when they rated
> it"). Good example of why tracing the *whole* chain matters: the fix isn't in
> the notification code itself, it's the missing call from `rate_song`.

### Data flow trace — sharing/adding a song triggers a notification

1. `POST /playlists/<playlist_id>/songs` with `{"song_id", "added_by"}`.
2. `routes/playlists.py::add_song()` → `add_to_playlist(playlist_id, song_id, added_by)`.
3. `notification_service.py::add_to_playlist()` loads the song, adder, and
   playlist; appends the song to `playlist.songs` if not already present; then
   **if the adder isn't the original sharer** (`song.shared_by != added_by`),
   calls `create_notification(user_id=song.shared_by, type="song_added_to_playlist", ...)`.
4. `create_notification` inserts a `Notification` row for the sharer and commits.
   Later, the sharer sees it via `GET /users/<id>/notifications`.

### Patterns I noticed

- **Strict layer separation.** Every route delegates to exactly one service
  function and does nothing but I/O parsing + response shaping. All logic is in
  `services/`. This is why the README says "the bugs live in the services layer"
  — the routes are too thin to hide a bug.
- **`ValueError` as the error protocol.** Services raise `ValueError` for
  not-found / bad-input; routes catch it and translate to 404 or 400. There is
  no custom exception hierarchy.
- **The event log is the source of truth.** Streaks and both feeds are all
  derived from `ListeningEvent` rows rather than from denormalized counters
  (except `listening_streak`, which is cached on `User` and updated on each
  listen).
- **Association tables carry data.** `playlist_entries` isn't a plain join — it
  stores `position`, `added_by`, `added_at`. Ordering a playlist means ordering
  by `position`, not by insert order.
- **Timezone handling is inconsistent.** Datetimes are created timezone-aware
  (`datetime.now(timezone.utc)`) but SQLite stores them naive, so
  `update_listening_streak` has to defensively re-attach `tzinfo`. This is a
  smell worth watching in the streak and feed bugs.
- **Cross-service imports are done lazily** inside functions (e.g.
  `add_to_playlist` imports `playlist_service` inside the body) to avoid
  circular imports.

---

## The Five Open Issues — notes and rough plan

Read all five before choosing. Based on the titles and an initial read of each
service, here's where each one points and my confidence:

| # | Issue | Service | Where I'd look first |
|---|-------|---------|----------------------|
| 1 | Listening streak keeps resetting | `streak_service.py` | `update_listening_streak` — the consecutive-day / calendar-boundary condition looks suspicious |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | The `RECENT_THRESHOLD` window and the aware-vs-naive datetime comparison in the cutoff filter |
| 3 | Same song shows up twice in search | `search_service.py` | `search_songs` outer-joins `song_tags` but never de-duplicates — a song with N tags returns N rows |
| 4 | Not notified when a friend rated my song | `notification_service.py` | `rate_song` never calls `create_notification` (confirmed while tracing above) |
| 5 | Last song in a playlist never shows up | `playlist_service.py` | `get_playlist_songs` returns `songs[:-1]`, which drops the final element |

**Rough plan — the three I'll tackle first:** Issues **#3 (search dup)**,
**#4 (missing rate notification)**, and **#5 (playlist last song)** are the
clearest to reproduce and reason about (a duplicated join row, a missing
function call, and an off-by-one slice). #1 and #5 already have test files
(`test_streaks.py`, `test_playlists.py`) I can lean on. I'll keep #1 (streak
calendar logic) as a strong backup if I want a fourth, since the calendar-day
edge cases are the most interesting root-cause work.

_(These are orientation hypotheses, not fixes — no service code has been changed
in this milestone.)_
