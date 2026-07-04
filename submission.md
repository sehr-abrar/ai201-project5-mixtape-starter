# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

I used an AI coding assistant (Claude Code) throughout this project, and it did
a lot of the hands-on work with me.

- **Codebase navigation:** I had it read through the README, `models.py`, and
  all of the route and service files, and help me build the codebase map —
  summarizing what each file is responsible for and tracing the data flows
  (route → service → model).
- **Reproducing the bugs:** it helped me pull real IDs out of the seeded
  database and drive the endpoints / service functions to trigger each reported
  bug before any code was changed, including proving why Bug #3 couldn't be
  reproduced (the ORM collapses the duplicate rows) so I could swap it for #1.
- **Implementing the fixes:** it helped me implement all three fixes —
  the playlist slice (#5), the missing rating notification (#4), and the Sunday
  streak-reset condition (#1) — and wrote the verification checks that confirmed
  each fix worked on both sides of the boundary without breaking related
  functionality or the test suite.
- **Writing this document:** it helped me draft the codebase map and the root
  cause analysis entries.

Where I verified things: after each fix I confirmed the behavior no longer
reproduced and re-ran the full `pytest` suite (ending at 13/13 passing), and I
checked library semantics like `datetime.weekday()` returning 6 for Sunday
rather than taking the explanation at face value.

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

**Chosen three (after Milestone 2 reproduction):** **#1 (streak reset)**,
**#4 (missing rate notification)**, and **#5 (playlist last song)**.

> **Why not #3?** I originally planned #3 but could not reproduce it (see
> Milestone 2 below) — the missing-`.distinct()` bug is real in the code but
> latent, because the legacy `session.query(Song)` identity map collapses the
> duplicate join rows before they reach the client. Per the milestone guidance
> ("if you can't reproduce it, try a different one"), I swapped in #1, which
> reproduces deterministically.

---

## Milestone 2: Reproducing the Bugs

App run on port **5001** (macOS AirPlay owns 5000). No code was changed in this
milestone — the streak repro drives the pure function directly and rolls back.

### Bug #5 — Last song in a playlist never shows up  ✅ reproduced

- **Root-cause location:** `playlist_service.py::get_playlist_songs`, final line
  `return [song.to_dict() for song in songs[:-1]]` — the `[:-1]` slice drops the
  last element.
- **How I reproduced it:** Playlist *"Late Night Vibes"*
  (`f9ba6334-40de-4894-a912-39bbe5d9e6b8`) has **7** entries in
  `playlist_entries` (confirmed by querying the join table directly).
  `GET /playlists/f9ba6334-.../songs` returns **`count: 6`** — the 7th
  (highest-`position`) song is missing every time. Deterministic for any
  non-empty playlist.

### Bug #4 — Rating a friend's song sends no notification  ✅ reproduced

- **Root-cause location:** `notification_service.py::rate_song` saves the
  `Rating` and commits but never calls `create_notification` — unlike its
  sibling `add_to_playlist`, which does notify the sharer.
- **How I reproduced it:** Song *"Midnight Drive"*
  (`227b2616-...`) was shared by user `f1ddabcf-...`. That sharer starts with
  **1** notification (a `song_added_to_playlist`). A *different* user, `darius`
  (`5348871b-...`), rated the song 5/5 via
  `POST /songs/227b2616-.../rate`. The rating saved successfully (HTTP 201), but
  the sharer's notification count stayed at **1** — no `song_rated` notification
  was ever created. Expected: count should rise to 2.

### Bug #1 — Listening streak resets (only on Sundays)  ✅ reproduced

- **Root-cause location:** `streak_service.py::update_listening_streak`, the
  branch `elif days_since_last == 1 and today.weekday() != 6:`. The
  `today.weekday() != 6` clause means that when the "today" of a consecutive
  listen falls on a **Sunday** (weekday 6), the increment branch is skipped and
  execution falls through to `else`, resetting the streak to 1.
- **State needed:** the *second* listen must land on a calendar Sunday, exactly
  one day after the previous listen (Saturday).
- **How I reproduced it:** Drove `update_listening_streak(user, now)` directly
  with controlled dates (it takes `now` as a parameter, so no clock mocking
  needed):
  - `last_listened = Sat 2026-07-04`, `now = Sun 2026-07-05`, starting streak 5
    → streak became **1** ❌ (should be 6).
  - Controls that behaved correctly: Sun→Mon gave 6 ✅, Mon→Tue gave 6 ✅.
    Only the transition *into* a Sunday resets, which matches the "keeps
    resetting" report for users who listen daily.

### Bug #3 — Duplicate search results  ⚠️ attempted, could not reproduce

- **Why it doesn't surface:** `search_songs` does
  `outerjoin(song_tags, ...)` with no `.distinct()`. For *"Crown Heights Anthem"*
  (3 tags) the SQL join genuinely emits **3 rows** (I confirmed `query.count()`
  == 3). But `.all()` on a legacy single-entity `session.query(Song)` runs the
  ORM identity-map uniquing, collapsing them back to **1** entity. A broad
  `?q=a` search returned 13 rows / 13 unique IDs — zero duplicates.
- **Conclusion:** the bug is latent (it would surface if the query were rewritten
  with `session.execute(select(...))`, or if rows were returned as tuples), but
  not triggerable through the current endpoint — so I set it aside per the
  milestone's fallback guidance and chose #1 instead.

_(Checkpoint: all three chosen bugs — #1, #4, #5 — can be triggered on demand.
No service code has been changed.)_

---

## Milestone 3: Root Cause Analyses

### RCA — Bug #5: The last song in a playlist never shows up

**How I reproduced it:** Queried the `playlist_entries` join table directly and
confirmed playlist *"Late Night Vibes"* has 7 rows. Called
`GET /playlists/<id>/songs` and got `count: 6` — the highest-`position` song was
always absent. Reproducible for every non-empty playlist.

**How I found the root cause:** Started at the route
`routes/playlists.py::get_songs`, which delegates to
`playlist_service.get_playlist_songs`. Read that function top to bottom. The
SQL query itself is correct — it joins `playlist_entries`, filters by playlist,
and orders by `position asc`, returning all 7 `Song` rows. The moment of
certainty was the very last line: the query result `songs` was sliced with
`songs[:-1]` in the return statement. That slice, not the query, is what drops a
row — and it drops exactly one, the last, which matches the symptom precisely.

**The root cause:** `get_playlist_songs` ended with
`return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice returns every
element *except the last*, so the final song (the one with the highest
`position`) was silently discarded on every call. The database, the join, and
the ordering were all correct; the bug was purely the truncating slice in the
serialization step.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs` — the smallest
possible fix, one token. Side-effect checks: (1) all three seeded playlists now
return API count == DB entry count (7 == 7). (2) Ordering by `position` is
unchanged, so songs still come back in playlist order. (3) Empty-playlist edge
case: created a fresh empty playlist and confirmed it returns `[]` rather than
erroring. (4) Ran `pytest tests/test_playlists.py` — 3/3 pass.
_(AI usage: none needed for this one — the slice was self-evident on reading.)_

### RCA — Bug #4: Notified when a friend adds my song to a playlist, but not when they rate it

**How I reproduced it:** Song *"Midnight Drive"* was shared by user
`f1ddabcf-...`, who started with 1 notification. A different user (`darius`)
rated the song via `POST /songs/<id>/rate` with `{score: 5}`. The rating saved
(HTTP 201), but re-fetching `GET /users/f1ddabcf-.../notifications` still showed
count 1 — no `song_rated` notification appeared.

**How I found the root cause:** The report itself was the clue — playlist-add
notifications work, rating notifications don't, so the two code paths must
diverge. Both live in `notification_service.py`. I read `add_to_playlist` and
`rate_song` side by side. `add_to_playlist` ends with a guarded call to
`create_notification(... type="song_added_to_playlist" ...)`. `rate_song`, by
contrast, saves the `Rating`, commits, and returns — with **no**
`create_notification` call anywhere in the function. That structural difference
between the two sibling functions was the moment of certainty: the notification
step wasn't broken, it was simply never written for the rating path.

**The root cause:** `rate_song` created/updated the `Rating` row and committed,
but never invoked `create_notification`. The notification-on-interaction feature
was only half-implemented — wired up for playlist adds but omitted for ratings —
so the song's original sharer was never told when someone rated their song.

**My fix and side-effect check:** After the commit in `rate_song`, I added a
guarded `create_notification` call that mirrors the exact pattern already used in
`add_to_playlist`: notify `song.shared_by` with type `"song_rated"`, and only if
`song.shared_by != user_id` so a user rating their own shared song doesn't
notify themselves. Side-effect checks: (1) a friend rating a song raises the
sharer's notification count by exactly 1, of type `song_rated`. (2) A user
rating their **own** shared song produces no notification (the guard works).
(3) Re-rating an already-rated song still works and notifies again — acceptable,
since a changed rating is genuinely new information for the sharer, and it
matches the "smallest fix" goal without adding dedup logic the issue didn't ask
for. (4) Full test suite: 12 pass; the single failure
(`test_streak_increments_on_sunday`) is the still-unfixed Bug #1 in a different
module, not a regression from this change.
_(AI usage: used AI to sanity-check the "compare the two sibling functions"
navigation strategy; confirmed the missing call myself by reading both.)_

### RCA — Bug #1: Listening streak keeps resetting (only on Sundays)

**How I reproduced it:** `update_listening_streak(user, now)` takes `now` as a
parameter, so I drove it directly with controlled dates instead of mocking the
clock. Starting streak 5, `last_listened = Sat 2026-07-04`, `now = Sun
2026-07-05` (a consecutive day) → the streak dropped to **1** instead of rising
to 6. Control transitions on other days (Sun→Mon, Mon→Tue) all correctly gave 6.
The failing unit test `tests/test_streaks.py::test_streak_increments_on_sunday`
independently confirmed it.

**How I found the root cause:** Traced from `POST /songs/<id>/listen` →
`routes/songs.py::listen` → `streak_service.record_listening_event` →
`update_listening_streak`. Read the branch structure. The docstring states the
rule plainly: "If the user listened yesterday: streak increments by 1." The code
was `elif days_since_last == 1 and today.weekday() != 6:`. The
`days_since_last == 1` half correctly detects "listened yesterday," but the
extra `and today.weekday() != 6` clause contradicts the documented rule — there
is no calendar reason a streak should behave differently on one weekday. That
mismatch between the docstring and the added condition was the moment I was
confident this was the cause, not just a suspicious area.

**The root cause:** Python's `datetime.weekday()` returns **6 for Sunday**. The
increment branch required `today.weekday() != 6`, so whenever a user's
consecutive-day listen fell on a Sunday, the `days_since_last == 1` branch was
skipped and execution fell through to the `else`, which resets the streak to 1.
In effect, any daily listener's streak was wiped out every Sunday — regardless
of the fact that they *had* listened the day before. The `weekday() != 6`
condition was spurious logic that never belonged in a "did they listen
yesterday?" check.

**My fix and side-effect check:** Removed the ` and today.weekday() != 6`
clause, leaving `elif days_since_last == 1:`. Now a listen exactly one calendar
day after the previous one always increments, on every weekday. Because this is
a boundary-condition bug, I verified **both sides of the boundary**:
`days_since_last == 1` on a Sunday now increments (5 → 6); a skipped day
(`days_since_last == 2`, Sat→Mon) still correctly **resets** to 1; and a
same-day repeat (`days_since_last == 0`) still correctly **no-ops** at 5. Full
test suite went from 12-pass/1-fail to **13/13 pass**, with
`test_streak_increments_on_sunday` now green.
_(AI usage: confirmed with AI that `datetime.weekday()` returns 6 for Sunday —
Mon=0 … Sun=6 — vs. `isoweekday()` where Sun=7; this ruled out any "off-by-one
weekday convention" interpretation and confirmed the clause was simply spurious
rather than a wrong constant.)_

---

## Summary

Three bugs fixed, each a separate commit, each with a one-line root cause:

| # | Bug | Root cause | Fix |
|---|-----|-----------|-----|
| 5 | Last playlist song missing | `songs[:-1]` slice truncated the result | `songs[:-1]` → `songs` |
| 4 | No notification on rating | `rate_song` never called `create_notification` | added a guarded `create_notification` mirroring `add_to_playlist` |
| 1 | Streak resets on Sundays | spurious `and today.weekday() != 6` in the "listened yesterday" branch (`weekday()` returns 6 for Sunday) | removed the clause |

Bug #3 (search duplicates) was investigated but is latent — the ORM identity map
collapses the duplicate join rows before they reach the client, so it can't be
triggered through the current endpoint.

_(Full AI usage is documented in the **AI Usage** section at the top of this
document.)_
