# Mixtape Bug Hunt — Submission

## AI Usage
Used Claude to help navigate the codebase — asked it to explain what `lazy="subquery"` 
does in SQLAlchemy

## Codebase Map

**Main files:**
- `app.py` — Flask app factory, DB setup
- `models.py` — SQLAlchemy models: User, Tag, Song, ListeningEvent, Rating, Playlist, 
  Notification, plus join tables `friendships`, `song_tags`, `playlist_entries`
- `routes/` — HTTP layer (songs.py, playlists.py, users.py, feed.py); parses requests, 
  calls services, formats JSON responses
- `services/` — business logic (streak_service, feed_service, search_service, 
  notification_service, playlist_service); this is where all 5 bugs live

**Pattern noticed:** Routes never contain logic — they immediately delegate to a service 
function. `Rating` and `Notification` have no foreign key link between them; connecting 
"a rating happened" to "a notification should fire" is handled entirely in application 
code, not enforced by the schema.

**Data flow — user rates a song:**
`POST /songs/<song_id>/rate` (routes/songs.py) → `notification_service.rate_song()`. 
This saves a `Rating` row (or updates one if the user already rated that song, enforced 
by a unique constraint on user_id+song_id). Compare to `add_to_playlist()` in the same 
file, which — after its core action — calls `create_notification()` to notify the song's 
original sharer. `rate_song()` had no equivalent call (see Issue #4 below).

## Bug Fixes

### Issue #4 — Missing rating notification
- **Reproduced:** Rated a song as a different user than the sharer via 
  `POST /songs/<id>/rate`, then checked `GET /users/<sharer_id>/notifications` — no 
  notification appeared, though the rating itself saved correctly.
- **Root cause found via:** Compared `add_to_playlist()` and `rate_song()` in 
  `services/notification_service.py` line by line, per the README's hint that both 
  routes go through this service.
- **Root cause:** `add_to_playlist()` calls `create_notification()` after committing 
  the playlist change, but `rate_song()` saves the `Rating` and returns without ever 
  calling `create_notification()`. The notification function exists and works — it's 
  just never invoked on the rating path.
- **Fix:** Added a `create_notification()` call at the end of `rate_song()`, mirroring 
  the playlist pattern (skip self-notification, notify `song.shared_by`).
- **Side-effect check:** Confirmed `add_to_playlist()` unaffected; ran `pytest tests/` 
  to check nothing else broke.