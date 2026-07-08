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

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it**

I used two methods:

1. *Controlled inputs (deterministic):* built an isolated scenario — a viewer and a friend whose only listening event was timestamped "yesterday 23:00" — then called `get_friends_listening_now(viewer)`. The friend appeared in the feed even though the listen was the previous evening. I ran this inside `db.session.rollback()` so nothing persisted.
2. *Seed data confirmation:* because the DB had been seeded the prior evening (~22:41 UTC) and I was now testing after 00:00 UTC the next day, all of nova's friends' most-recent seeded listens fell on the previous calendar day (~23:2x). Fetching nova's feed under the old rule showed simone, darius, and kenji as "listening now" — a direct recreation of the reported bug ("darius listening now to a song he played at 11pm last night"). I also confirmed this live via `GET /feed/<nova_id>/listening-now`.

Note the seed-data method is time-dependent: it only reproduces once the clock has crossed into a new UTC day relative to the seed timestamps. The controlled-input method reproduces at any time.

**How I found the root cause**

I traced from the route `GET /feed/<user_id>/listening-now` ([routes/feed.py](routes/feed.py)) → `get_friends_listening_now()` in [services/feed_service.py](services/feed_service.py). The two lines that defined "recent" stood out:

```python
RECENT_THRESHOLD = timedelta(hours=24)
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
```

The moment of confidence: the endpoint's contract is "friends listening *now* / today," but `now - 24h` is a rolling window, not a calendar boundary. I verified by printing each friend's last-listen timestamp alongside both an `old_cutoff = now - 24h` and a `new_cutoff = midnight today`, and confirmed the yesterday-evening events satisfied the old cutoff but not the calendar-day one.

**The root cause**

"Recent" was defined as a **rolling 24-hour window** (`datetime.now() - timedelta(hours=24)`) rather than the **current calendar day**. A listen at 11pm is only ~10 hours old at 9am the next morning, so it stays inside the 24-hour window and keeps appearing in "Listening Now" until it individually ages past 24 hours — i.e., until the same time the next day. The two concepts ("last 24 hours" vs "today") only coincide at midnight; at every other time the rolling window leaks in part of the previous day.

**My fix and side-effect check**

I replaced the rolling threshold with a calendar-day cutoff — midnight of the current UTC day:

```python
now = datetime.now(timezone.utc)
cutoff = now.replace(hour=0, minute=0, second=0, microsecond=0)
```

`.replace(...)` keeps the current date but zeroes the time fields, giving the start of today. The downstream filter `ListeningEvent.listened_at >= cutoff` then admits only events dated today. I also removed the now-unused `RECENT_THRESHOLD` constant and the `timedelta` import.

Side-effect check (both sides of the boundary, since this is a boundary bug):
- Friend who listened yesterday 23:00 → **not** shown (fixed).
- Friend who listened today 00:05 (just past the boundary) → **still** shown (didn't over-filter).
- `get_activity_feed`, which lives in the same file and reads the same events, was checked and is unaffected — it intentionally does not use `cutoff`.
- Live curl against a freshly restarted server: nova's feed returns `count: 0` (all friends listened yesterday), and after posting a fresh "today" listen for darius, darius correctly reappears.

There is no `test_feed.py` in the suite, so verification was via the reproduction scripts and live curl. This bug is a good candidate for the stretch regression test (a `test_feed.py` asserting the two boundary cases).

**Limitation noted:** "today" is defined in UTC, consistent with how the app stores all timestamps. For a user in a non-UTC timezone, the day boundary flips at 00:00 UTC rather than their local midnight. Making it match the user's local day would require per-user timezone handling, which is out of scope for this fix.

**AI usage:** I used Claude Code to help trace the route→service chain, to explain how `.replace()` produces a midnight timestamp, and to run the reproduction/boundary checks. I confirmed the rolling-window-vs-calendar-day diagnosis myself before editing.

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it**

I compared two independent counts of the same thing: how many songs are actually stored in a playlist versus how many the service returns. For each seeded playlist I queried the `playlist_entries` join table directly (the ground truth, which bypasses the buggy function) and also called `get_playlist_songs()`:

- Friday Energy: stored **7**, returned **6**
- Late Night Vibes: stored **7**, returned **6**
- Study Mode: stored **7**, returned **6**

Every playlist returned exactly one fewer song than it contained, matching darius's report that "Friday Energy says 7 songs but only 6 show." Because the two measurements are independent (raw table vs. service function), the mismatch proves the function is dropping a song.

**How I found the root cause**

I traced from the route `GET /playlists/<id>/songs` ([routes/playlists.py](routes/playlists.py)) → `get_playlist_songs()` in [services/playlist_service.py](services/playlist_service.py). The query itself was correct — it joins `playlist_entries`, filters by playlist, and orders ascending by `position`. The bug was in the very last line:

```python
return [song.to_dict() for song in songs[:-1]]
```

The moment of confidence: `songs[:-1]` is a Python slice meaning "every element except the last one." Since the results are ordered ascending by `position`, the last element is always the highest-position = most-recently-added song. The function's own docstring says "This function returns all songs in the playlist," which the code contradicts — confirming this line was the actual cause, not just a suspicious spot. It also explains darius's odd observation that adding a new song "frees" the previously-missing one: the old last song is no longer last, so it reappears, and the brand-new song becomes the one sliced off.

**The root cause**

`get_playlist_songs` correctly queried and ordered all playlist entries by `position`, but the return statement sliced the list with `songs[:-1]`, which discards the final element. Because the list is sorted ascending by position, the final element is always the most-recently-added song, so every call silently omitted exactly one song — the newest one.

**My fix and side-effect check**

I removed the slice so the comprehension iterates the full list:

```python
return [song.to_dict() for song in songs]
```

That is the entire change — one slice removed, nothing else touched. Side-effect check:
- Re-ran the reproduction: all three playlists now report stored=7, returned=7 (no song dropped).
- `pytest tests/test_playlists.py` → all pass, including the test that a newly added song is returned (which the `[:-1]` version failed).
- Boundary check: an empty playlist still returns `[]` — with the bug `[][:-1]` was also `[]`, so this case was unchanged and does not error.
- Ran the full suite (`pytest tests/`) to confirm the streak and feed fixes still pass alongside this one.

**AI usage:** I used Claude Code to help locate `get_playlist_songs` from the route and to explain Python's `[:-1]` slice semantics. I verified the off-by-one myself by comparing the raw-table count against the function's output before and after the change.
