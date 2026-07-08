# Mixtape Bug Hunt — Submission

## AI Usage

Used Claude throughout for codebase navigation and debugging support, not code generation from scratch:

- Asked it to explain what `lazy="subquery"` does in SQLAlchemy relationships while reading `models.py`.
- Used it to reason through `datetime.weekday()` return values (Sunday = 6) while tracing the streak reset logic (Issue #1).
- Used it to confirm why joining `Song` to `song_tags` without `.distinct()` produces one row per tag rather than one row per song (Issue #3).
- Used it to reason through the difference between a rolling 24-hour window and a calendar-day boundary for the "Listening Now" feed (Issue #2).

In every case, I read the actual function myself before accepting a diagnosis, and verified the proposed root cause against the specific code rather than taking an explanation at face value.

## Codebase Map

**Main files:**
- `app.py` — Flask app factory, DB setup
- `models.py` — SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus join tables `friendships`, `song_tags`, `playlist_entries`
- `routes/` — HTTP layer (`songs.py`, `playlists.py`, `users.py`, `feed.py`); parses requests, calls services, formats JSON responses. Contains no business logic itself.
- `services/` — business logic layer (`streak_service.py`, `feed_service.py`, `search_service.py`, `notification_service.py`, `playlist_service.py`); this is where all 5 bugs live.
- `seed_data.py` — populates the DB with test users, songs, playlists, and tags.

**Patterns noticed:**
- Every route delegates immediately to a service function — routes only handle request parsing and response formatting.
- `Rating` and `Notification` have no foreign key link between them. Connecting "a rating happened" to "a notification should fire" is handled entirely in application code (an explicit function call), not enforced by the schema.
- `playlist_entries` is a many-to-many join table that carries extra columns (`position`, `added_by`, `added_at`) beyond a plain join — ordering within a playlist is explicit, not based on insertion order.
- `User.listening_streak` is a cached integer on the `User` row, not derived fresh from `ListeningEvent` history each time — it's incrementally updated by `streak_service.py` based on `last_listened_at`.

**Data flow — user rates a song:**
`POST /songs/<song_id>/rate` (`routes/songs.py`) → `notification_service.rate_song()`. This saves a `Rating` row (or updates one if the user already rated that song, enforced by a unique constraint on `user_id`+`song_id`), then commits. Compare to `add_to_playlist()` in the same file, which — after its core action — calls `create_notification()` to notify the song's original sharer. `rate_song()` originally had no equivalent call (see Issue #4 below).

## Bug Fixes

### Issue #1 — Listening streak resets on Sunday
- **Reproduced:** Traced the logic in `update_listening_streak()` — a user listening on consecutive days where the second day is a Sunday hits `days_since_last == 1`, but an extra `today.weekday() != 6` check evaluates false, sending it to the `else` branch instead of incrementing.
- **Root cause found via:** Read `services/streak_service.py` top to bottom, tracing `record_listening_event()` → `update_listening_streak()`. The `elif` condition combining `days_since_last == 1` with a weekday check stood out immediately as unrelated to the rules stated in the docstring.
- **Root cause:** Python's `datetime.weekday()` returns `6` for Sunday. The condition `days_since_last == 1 and today.weekday() != 6` requires the day to NOT be Sunday for the streak to increment — so any legitimate consecutive-day listen landing on a Sunday fails this check and falls into the `else` branch, which resets the streak to 1 instead of incrementing it.
- **Fix:** Removed the `and today.weekday() != 6` clause entirely. The streak now increments on any consecutive day, Sunday included, matching the documented rules.
- **Side-effect check:** Confirmed the `days_since_last == 0` (same-day, no change) and `days_since_last > 1` (reset) branches are untouched. Ran `pytest tests/test_streaks.py`.

### Issue #2 — Friends Listening Now shows stale entries
- **Reproduced:** Traced the logic against the report: a listen at 11pm one night would, under a 24-hour rolling cutoff, remain in the feed until 11pm the *next* night — matching the report of an 11pm listen still showing the following morning at 9am.
- **Root cause found via:** Read `get_friends_listening_now()` in `services/feed_service.py`. The line `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` stood out since `RECENT_THRESHOLD` is a flat `timedelta(hours=24)` — a rolling window, not a "today" check.
- **Root cause:** The cutoff for inclusion in the feed is "now minus 24 hours," a sliding window that always includes the last full day regardless of calendar boundaries. A listen at 11pm stays visible until 11pm the next day, not until midnight. The feature is meant to show "today's" listening activity, not "the last 24 hours," so events from the previous evening incorrectly persist into the next morning.
- **Fix:** Replaced the rolling `timedelta(hours=24)` cutoff with midnight UTC of the current day (`datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)`), so only events from the current calendar day are included.
- **Side-effect check:** Confirmed `get_activity_feed()` is unaffected — it doesn't filter by recency at all, it just takes the most recent N events regardless of time. Ran `pytest tests/test_streaks.py` and manually spot-checked timing near the midnight UTC boundary.

### Issue #3 — Duplicate search results
- **Reproduced:** Searched "Anthem" via `GET /songs/search?q=Anthem` — "Crown Heights Anthem" (which has 3 tags: rap, hip-hop, boom bap) returned 3 times, matching the report exactly.
- **Root cause found via:** Read `search_songs()` in `services/search_service.py`. Noticed the query joins `Song` to `song_tags` but the `.filter()` only references `title`/`artist` — the join isn't used for filtering at all, which was the tell that it didn't belong.
- **Root cause:** The query performs `.outerjoin(song_tags, ...)` before filtering. `song_tags` has one row per (song, tag) pair, so joining produces one result row per tag a song has. Since the query selects full `Song` objects with no `.distinct()`, a song with N tags is returned N times. Songs with 0–1 tags appear normal; songs with 2+ tags duplicate — exactly the "some show once, others two or three times" pattern reported.
- **Fix:** Removed the `.outerjoin(song_tags, ...)` call entirely, since it wasn't used for filtering and tags are already loaded separately via the `Song.tags` relationship inside `to_dict()`.
- **Side-effect check:** Confirmed tags still appear correctly in results (via the relationship, unaffected by removing the join). Ran `pytest tests/test_search.py`.

### Issue #4 — Missing rating notification
- **Reproduced:** Rated a song as a different user than the sharer via `POST /songs/<id>/rate`, then checked `GET /users/<sharer_id>/notifications` — no notification appeared, though the rating itself saved correctly.
- **Root cause found via:** Compared `add_to_playlist()` and `rate_song()` in `services/notification_service.py` line by line, per the README's hint that both routes go through this service.
- **Root cause:** `add_to_playlist()` calls `create_notification()` after committing the playlist change, but `rate_song()` saves the `Rating` and returns without ever calling `create_notification()`. The notification function exists and works correctly — it's just never invoked on the rating path.
- **Fix:** Added a `create_notification()` call at the end of `rate_song()`, mirroring the playlist pattern (skip self-notification when `song.shared_by == user_id`, otherwise notify `song.shared_by`).
- **Side-effect check:** Confirmed `add_to_playlist()` unaffected; ran `pytest tests/` to check nothing else broke.

### Issue #5 — Last song in playlist never shows up
- **Reproduced:** Fetched a playlist via `GET /playlists/<id>/songs` with 7 songs — only 6 returned. Added an 8th song and re-fetched: the previously-missing 7th song appeared, and the new 8th song was now missing instead.
- **Root cause found via:** Read `get_playlist_songs()` in `services/playlist_service.py`. The query itself was correct (joins `playlist_entries`, filters by playlist, orders by `position` ascending) — the docstring even claims "returns all songs in the playlist." The mismatch between that claim and the reported bug pointed straight at the return line.
- **Root cause:** The final line was `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element of the already-correctly-ordered list. Since `songs` is sorted ascending by `position`, the last element is always the most recently added song — so it's silently dropped every time, and adding a new song just shifts which song ends up last (and thus dropped).
- **Fix:** Removed the `[:-1]` slice so the full ordered list is returned.
- **Side-effect check:** Confirmed `get_playlist()` (metadata only) and `create_playlist()` are unaffected — they don't touch this function. Ran `pytest tests/test_playlists.py`.

## Commit History

`git log --oneline` on `bugfix/mixtape`:

```
92c6c18 (HEAD -> bugfix/mixtape, origin/bugfix/mixtape) fix: use calendar-day cutoff instead of rolling 24h window for listening-now feed
8d69ccf fix: remove slice that dropped the last song in a playlist
b01b10e fix: remove unnecessary join causing duplicate search results
0109e8a fix: remove erroneous Sunday check that reset streaks incorrectly
38fba7d docs: add codebase map and RCA for issue #4
49f47d7 fix: add missing notification call when a song is rated
2dfdeaa (origin/main, origin/HEAD, main) Add .gitignore file and update README with setup instructions
7b64551 initial commit
```