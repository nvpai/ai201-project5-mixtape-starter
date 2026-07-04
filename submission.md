# Mixtape — Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for a social music-sharing app: users share songs, build collaborative playlists, rate and listen to songs,and see what friends are listening to.

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

Example: darius adds sara's shared song to a playlist.

1. **Request:** `POST /playlists/<playlist_id>/songs` with body `{"song_id": ..., "added_by": <darius_id>}`.
2. **Route** — `add_song()` in [routes/playlists.py](routes/playlists.py) reads `song_id` and `added_by`, returns 400 if either is missing, then calls `add_to_playlist(playlist_id, song_id, added_by)`.
3. **Service** — `add_to_playlist()` in [services/notification_service.py](services/notification_service.py) loads and validates the song, the adding user, and the playlist, then appends the song to the playlist (writing a row into `playlist_entries`) and commits.
4. **Notification trigger:** in the same function, if `song.shared_by != added_by_user_id` (you didn't add your own song), it calls `create_notification()` aimed at `song.shared_by` — the **original sharer**, sara — with a message like *"darius added your song 'X' to the playlist 'Y'."*
5. **Read side:** sara later hits `GET /users/<id>/notifications`, which calls `get_notifications()` and returns her notifications newest-first.

The key detail: the notification goes to the song's **sharer**, not the playlist owner, and adding your own song is intentionally silent.

## Patterns I noticed

- **Strict route → service → model layering.** Routes only parse input and format JSON; every bit of logic and every DB query lives in `services/`. Services signal "not found" by raising `ValueError`, and routes uniformly turn that into a 4xx response.
- **`to_dict()` is the serialization contract.** Every model has one, and it's selective — e.g. `User.to_dict()` deliberately leaves out email.
- **Event-sourced feeds and streaks.** There's no stored "feed" table; the feed and streak features are computed on read from the append-only `ListeningEvent` log.
- **UUID string primary keys everywhere**, defaulting to `generate_uuid()` instead of autoincrement integers.

---


# Bug Hunt



## Issue #1 — My listening streak keeps resetting

- **How I reproduced it:** Ran `pytest tests/test_streaks.py::test_streak_increments_on_sunday` — it fails. A Saturday listen sets the streak to 1, then the next day (a Sunday) it stays 1 instead of going to 2.
- **How I found the root cause:** Traced `POST /songs/<id>/listen` → `record_listening_event()` → `update_listening_streak()` in [services/streak_service.py](services/streak_service.py). The only weekday check in the function was on the "listened yesterday" branch, and Sunday was exactly the failing day.
- **The root cause:** `datetime.weekday()` returns **6 for Sunday**. The branch was `elif days_since_last == 1 and today.weekday() != 6:`, which means "only increment if it's *not* Sunday." So any consecutive listen on a Sunday failed the check and fell to the `else`, resetting the streak to 1.
- **Fix and side-effect check:** Removed the `and today.weekday() != 6` clause so it's just `elif days_since_last == 1:`. The same-day and multi-day-gap branches are unchanged.
