# Project 5 — Mixtape Bug Hunt — Submission

## Codebase Map

### Main files and what each does

| File | Responsibility |
|------|---------------|
| `app.py` | Flask app factory (`create_app`) + SQLAlchemy `db` setup. Registers the route blueprints. |
| `models.py` | Defines the database entities and association tables. |
| `seed_data.py` | Populates the DB with test users, songs, friendships, playlists, and listening events. |
| `routes/songs.py` | Endpoints for sharing, searching, and rating songs. |
| `routes/playlists.py` | Endpoints for creating playlists and adding/listing songs. |
| `routes/users.py` | Endpoints for user profiles, listening streaks, and notifications. |
| `routes/feed.py` | Endpoints for "Friends Listening Now" and the activity feed. |
| `services/streak_service.py` | Records listening events and updates the consecutive-day streak. |
| `services/feed_service.py` | Builds the "Friends Listening Now" feed and the activity feed. |
| `services/search_service.py` | Song search by title/artist. |
| `services/notification_service.py` | Creates and retrieves notifications; handles add-to-playlist and rating. |
| `services/playlist_service.py` | Creates playlists and returns their ordered songs. |
| `tests/` | Pytest suites for streaks, search, and playlists. |

### The data model (`models.py`)

- **User** — stores `listening_streak` and `last_listened_at` directly on the row. `friends` is a self-referential many-to-many via the `friendships` table.
- **Song** — has a `shared_by` foreign key pointing at the original sharer (this drives who gets notified).
- **ListeningEvent** — one row per play, timestamped `listened_at`. Drives streaks and the feed.
- **Rating** — its own table, one row per (user, song), enforced by a unique constraint (not a column on Song).
- **Playlist** — songs are linked through the `playlist_entries` association table, which carries an explicit `position` column plus `added_by` and `added_at`.
- **Notification** — `user_id` is the recipient, with a `notification_type`, `body`, and `read` flag.
- **Tag** — linked to songs through the `song_tags` association table.

### Data flow — sharing a song triggers a notification

When a friend adds your song to a playlist:

1. Client calls `POST /playlists/<playlist_id>/songs`.
2. `routes/playlists.py` parses the request and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)`.
3. The service loads the `Song`, the adding `User`, and the `Playlist`.
4. It appends the song to `playlist.songs` (writing a `playlist_entries` row) and commits.
5. It reads `song.shared_by`. If the sharer is not the person who added the song, it calls `create_notification(user_id=song.shared_by, ...)`, which inserts a `Notification` row addressed to the original sharer and commits.

So notifications are always addressed to `song.shared_by`, and are only created when the actor is someone other than the sharer.

### Patterns I noticed

- **Thin routes, logic in services.** Every route immediately delegates to a service function; all business logic lives in `services/`. To debug an endpoint, go straight to the service it calls.
- **Everything is UTC.** Timestamps default to `datetime.now(timezone.utc)`, so any date-boundary logic reasons in UTC.
- **Association tables carry data.** `playlist_entries` isn't a plain join — it stores `position`, `added_by`, and `added_at`, so ordering and attribution depend on it.
- **Some read state is denormalized.** Streaks live on the User row while events and ratings live in their own tables.

_AI disclosure: I used Claude Code to help read the service files and confirm the call chain above. The conclusions reflect my own reading of the code._

---

## Root Cause Analyses

### Issue #1 — My listening streak keeps resetting

**How I reproduced it**

The bug only triggers when the day you listen is a Sunday, and today is a Tuesday (2026-07-07), so hitting the live `POST /songs/<id>/listen` endpoint wouldn't show it. Instead I drove the underlying service function directly, since `update_listening_streak(user, now)` accepts `now` as a parameter and lets me simulate any date:

1. Set kenji to `last_listened_at = 2026-07-04` (a Saturday) and `listening_streak = 12`, matching the report.
2. Called `update_listening_streak(kenji, 2026-07-05 09:00)` (a Sunday, a true 1-day gap).
3. Observed the streak drop to **1** instead of advancing to 13.
4. Ran two controls — Friday→Saturday and Sunday→Monday, both genuine consecutive-day listens — and both correctly incremented to 13. That isolated Sunday as the sole trigger.

All of this ran inside `db.session.rollback()`, so no persisted data was modified.

**How I found the root cause**

I traced top-down from the route: `POST /songs/<id>/listen` ([routes/songs.py](routes/songs.py)) → `record_listening_event()` → `update_listening_streak(user, now)` in [services/streak_service.py](services/streak_service.py). Reading `update_listening_streak`, the increment branch stood out:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

The moment I was confident: `datetime.weekday()` returns 6 for Sunday. So on a Sunday the `today.weekday() != 6` sub-condition is `False`, which makes the whole `elif` `False`, dropping a legitimate consecutive-day listen into the `else` (reset) branch. My controlled reproduction (Sunday resets, Saturday/Monday don't) confirmed this was the actual cause, not just a suspicious line.

**The root cause**

`datetime.weekday()` returns 6 for Sunday. The consecutive-day increment was gated on `days_since_last == 1 and today.weekday() != 6`. The day of the week is irrelevant to a listening streak — the only thing that matters is whether the previous listen was exactly one day earlier. That extra `and today.weekday() != 6` clause meant every consecutive-day listen that happened to land on a Sunday failed the condition and fell through to `else`, which resets the streak to 1. That is exactly kenji's report: "both times it was a Sunday."

**My fix and side-effect check**

I removed the spurious weekday clause, leaving:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

This restores the intended rule (increment on a one-day gap regardless of weekday) and changes nothing else. Side-effect check on both sides of the boundary:

- Sunday consecutive listen → 13 (fixed); Saturday and Monday consecutive → still 13 (working path unbroken).
- Same-day listen → no change (12 → 12); gap ≥ 2 days → still resets to 1.
- `pytest tests/test_streaks.py` → all 5 pass, including `test_streak_increments_on_sunday`, the test that directly covers this case.

**AI usage:** I used Claude Code to help trace the route→service call chain and to confirm what `datetime.weekday()` returns for Sunday. I verified the diagnosis myself by running the reproduction with controlled inputs before changing any code.
