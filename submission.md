# Mixtape — Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for a social music-sharing app: users share songs, build collaborative playlists, rate and listen to songs,and see what friends are listening to.

## AI usage

I used AI mainly for **navigation and debugging**

- **Tracing call chains.** I asked the AI to follow each symptom from its route down to the service it calls — e.g. `POST /songs/<id>/listen` → `record_listening_event()` → `update_listening_streak()`, and `GET /playlists/<id>/songs` → `get_playlist_songs()`. This let me land in the right file quickly instead of reading all five services top to bottom.
- **Explaining suspicious code.** Once I'd located a function, I had the AI explain edge cases — for example what `datetime.weekday()` returns for each day, which confirmed that `6` is Sunday and that the `weekday() != 6` check was the streak bug.
- **Confirming bugs before fixing.** I used it to run the existing test suite and small reproduction snippets so I could see each bug fail *before* editing anything, per the reproduce-first discipline.
- **Where I verified / overrode it.** I read and confirmed every diagnosis in the source myself before accepting a fix. One concrete catch: the project brief's example RCA describes the streak bug as `weekday() == 0`, but the actual code was `weekday() != 6`. Reading the real source kept me from documenting the wrong condition. The AI is good at explaining code I'd already found, but I treated its "the bug is probably here" guesses as leads to verify, not conclusions.

## Architecture

Requests flow through three layers:

```
HTTP request
   ↓
routes/*.py        ← Flask blueprints: parse input, format JSON, map errors → status codes
   ↓
services/*.py      ← all business logic + DB queries live here
   ↓
models.py          ← SQLAlchemy models + association tables (the schema)
   ↓
mixtape.db (SQLite)
```

`app.py` wires it together with an application factory, and `seed_data.py` populates a realistic database for testing.

## Main files and what each does

### `app.py`
The application factory. It defines the shared `db = SQLAlchemy()` object that every other module imports, and a `create_app()` function that configures SQLite, registers the four route blueprints under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and creates the tables. Running the file directly starts the dev server.

### `models.py`
Defines the whole database schema: 7 models (`User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`) and 3 association tables (`friendships`, `song_tags`, `playlist_entries`). `friends` is a self-referential many-to-many on `User`, and `playlist_entries` is special because it carries extra columns (`position`, `added_by`, `added_at`) so playlist songs have an explicit order and provenance rather than just insertion order. All IDs are UUID strings, and every model has a `to_dict()` method used for JSON serialization.

### `routes/`
Four thin Flask blueprints — `songs.py`, `playlists.py`, `users.py`, `feed.py` — that make up the HTTP layer. Each route parses the request body/query params, calls a single service function, and returns `jsonify(...)`. They contain no business logic or DB queries themselves; when a service raises `ValueError`, the route catches it and returns the JSON error.

### `services/`
Where all the business logic and database queries live, split into 5 modules. `search_service.py` searches songs by title/artist; `playlist_service.py` creates playlists and returns their ordered songs; `notification_service.py` handles adding songs to playlists, rating, and creating/reading notifications; `streak_service.py` records listening events and updates day-streaks; `feed_service.py` builds the "friends listening now" and activity feeds. Routes depend on these functions, and the functions validate their inputs and raise `ValueError` when something isn't found.

### `seed_data.py`
A standalone script (`python seed_data.py`) that drops and recreates all tables, then loads realistic test data: 5 users with friendships, 10 tags, 13 songs, listening events, 3 ordered playlists, and a sample notification. 

## Data flow — adding a song to a playlist triggers a notification

Example: darius adds nova's shared song to a playlist.

1. **Request:** `POST /playlists/<playlist_id>/songs` with body `{"song_id": ..., "added_by": <darius_id>}`.
2. **Route** — `add_song()` in [routes/playlists.py](routes/playlists.py) reads `song_id` and `added_by`, returns 400 if either is missing, then calls `add_to_playlist(playlist_id, song_id, added_by)`.
3. **Service** — `add_to_playlist()` in [services/notification_service.py](services/notification_service.py) loads and validates the song, the adding user, and the playlist, then appends the song to the playlist (writing a row into `playlist_entries`) and commits.
4. **Notification trigger:** in the same function, if `song.shared_by != added_by_user_id` (you didn't add your own song), it calls `create_notification()` aimed at `song.shared_by` — the **original sharer**, nova — with a message like *"darius added your song 'X' to the playlist 'Y'."*
5. **Read side:** nova later hits `GET /users/<id>/notifications`, which calls `get_notifications()` and returns her notifications newest-first.

The key detail: the notification goes to the song's **sharer**, not the playlist owner, and adding your own song is intentionally silent.

## Patterns I noticed

- **Strict route → service → model layering.** Routes only parse input and format JSON; every bit of logic and every DB query lives in `services/`. Services signal "not found" by raising `ValueError`, and routes uniformly turn that into a 4xx response.
- **`to_dict()` is the serialization contract.** Every model has one, and it's selective — e.g. `User.to_dict()` deliberately leaves out email.
- **Event-sourced feeds and streaks.** There's no stored "feed" table; the feed and streak features are computed on read from the append-only `ListeningEvent` log.
- **UUID string primary keys everywhere**, defaulting to `generate_uuid()` instead of autoincrement integers.

---


# Bug Hunt

I fixed **Issue #1 (streak resets), Issue #4 (no rating notification), and Issue #5 (last playlist song missing)**.



## Issue #1 — My listening streak keeps resetting

- **How I reproduced it:** Ran `pytest tests/test_streaks.py::test_streak_increments_on_sunday` — it fails. A Saturday listen sets the streak to 1, then the next day (a Sunday) it stays 1 instead of going to 2.
- **How I found the root cause:** Traced `POST /songs/<id>/listen` → `record_listening_event()` → `update_listening_streak()` in [services/streak_service.py](services/streak_service.py). The only weekday check in the function was on the "listened yesterday" branch, and Sunday was exactly the failing day.
- **The root cause:** `datetime.weekday()` returns **6 for Sunday**. The branch was `elif days_since_last == 1 and today.weekday() != 6:`, which means "only increment if it's *not* Sunday." So any consecutive listen on a Sunday failed the check and fell to the `else`, resetting the streak to 1.
- **Fix and side-effect check:** Removed the `and today.weekday() != 6` clause so it's just `elif days_since_last == 1:`. I left the same-day ("no change") and gap > 1 day ("reset to 1") branches untouched, and re-ran the full suite (13/13 pass) — specifically confirming the same-day and multi-day-gap streak tests still pass, so the fix only affects the consecutive-day case.


## Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it

- **How I reproduced it:** Rated another user's song with `rate_song(nova, simone's song, 5)` and checked `get_notifications(simone)` — the count was 0 before and 0 after, so the sharer got no notification.
- **How I found the root cause:** Compared the two write paths in [services/notification_service.py](services/notification_service.py). `add_to_playlist()` ends by calling `create_notification(...)`, but `rate_song()` saved the rating and returned with no notification step at all.
- **The root cause:** `rate_song()` was missing the notification entirely. It saves the `Rating` correctly but never mirrors the "notify the original sharer" behavior that `add_to_playlist` already has.
- **Fix and side-effect check:** After the rating commit, added a `create_notification(user_id=song.shared_by, notification_type="song_rated", body=...)` guarded by `if song.shared_by != user_id`, the same self-action guard `add_to_playlist` uses. Rating someone else's song now creates one `song_rated` notification; rating your own creates none.



## Issue #5 — The last song in a playlist never shows up

- **How I reproduced it:** Ran `pytest tests/test_playlists.py` — `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` both fail. A playlist with N songs returns N−1, and the missing one is always the last.
- **How I found the root cause:** Traced `GET /playlists/<id>/songs` → `get_playlist_songs()` in [services/playlist_service.py](services/playlist_service.py). The query orders every entry by `position` correctly, so the loss had to be in the return line: `return [song.to_dict() for song in songs[:-1]]`.
- **The root cause:** The comprehension iterated over `songs[:-1]` — "all songs except the last." Since the query is ordered by `position` ascending, this always dropped the highest-position song from the response.
- **Fix and side-effect check:** Changed `songs[:-1]` to `songs` so the full ordered list is returned. Ordering is unchanged. Both playlist tests pass.

## AI usage

I used the AI assistant to trace the route → service call chains, run the test suite, and confirm each bug with a small reproduction before changing code. I read and confirmed each diagnosis in the source myself before accepting a fix.

## Screenshot of git log --oneline
![alt text](/images/screenshot.png)