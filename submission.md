# Submission — Project 5: Mixtape Bug Hunt

## AI Usage

Claude was used throughout this project for codebase navigation, code explanation, and debugging assistance. GitHub Copilot was also used briefly during code investigation to confirm specific function behaviors.

During codebase orientation, I provided service files to Claude and asked it to summarize what each module was responsible for and what its main functions did. For example, I asked Claude to explain what `feed_service.py` was responsible for and walk through its two functions, `get_friends_listening_now()` and `get_activity_feed()`, before looking at any bugs. Claude also helped confirm specific Python behavior; for example, after noticing the `today.weekday() != 6` condition in `streak_service.py`, I asked Claude what `datetime.weekday()` returns for Sunday, and Claude confirmed it returns 6, which validated my hypothesis about Bug 1.

For Bug 3, I initially suspected `db.or_()` was causing duplicate search results since the query matched against both title and artist fields. I asked Claude whether `db.or_()` could cause a single song to appear twice, and Claude explained that SQLAlchemy deduplicates matches on the same row regardless of how many fields match. This ruled out that hypothesis and redirected me toward the outerjoin. I then traced the outerjoin issue myself by going back to `models.py` and confirming the `tags` relationship made the join redundant. I also used GitHub Copilot briefly at this stage to confirm that `lazy="subquery"` in the `tags` relationship would load tags automatically without requiring an explicit join in the query, which matched what I had reasoned independently.

For bug reproduction, I described the conditions needed to trigger some of the bugs to Claude, and Claude helped construct the flask shell commands and curl requests needed to reproduce them. For example, for Bug 1, I described that the bug only triggered on Sundays with consecutive listening days, and Claude helped build the controlled datetime simulation using Friday, Saturday, and Sunday timestamps. I then ran the commands myself and verified the outputs matched the expected buggy behavior before touching any code.

Claude also helped draft and refine some of the written sections of the submission documentation after the bugs were already identified and fixed. In all cases, I verified Claude's explanations by reading the relevant code myself before accepting them as correct. For Bug 2, Claude initially suggested a threshold value of 30 minutes for `RECENT_THRESHOLD`, but I overrode that with 1 hour since it kept the same `timedelta(hours=...)` format as the original value and was a more defensible and minimal change.

## Codebase Map

app.py creates a database instance that gets populated by the tables defined in models.py using SQLAlchemy. They are defined as classes, and there are 7 of them: User, Tag, Song, ListeningEvent, Rating, Playlist, and Notification. Additionally, the relationships between these tables are described in association/join tables; there are 3 of them, all many-to-many: friendships (between users), song_tags (between songs and tags), and playlist_entries (between playlists and songs, which also includes a position and added time).

Data flow — a user listens to a song using `POST /songs/<song_id>/listen` in songs.py, which is received by the listen() route handler and calls record_listening_event(user_id, song_id). This records that a user listened to a song and updates their streak by making two calls: first, it creates a listening event using ListeningEvent(user_id=user_id, song_id=song_id, listened_at=now) from models.py, then updates the user's streak by calling update_listening_streak(user, now), both defined in streak_service.py.

Patterns I noticed: Every route in routes/ delegates immediately to a service function; the route handles input parsing and JSON formatting while all business logic lives in services/. Additionally, the ListeningEvent table is connected to many services and might be a dependency for several of the bugs.

## Bug Fixes

### Reproduction Setup

To reproduce bugs involving service logic directly, I used the flask shell (`FLASK_APP=app:create_app flask shell`) to call service functions with controlled inputs, bypassing the HTTP layer. This was faster and more precise than firing curl requests. Imports and session setup were done at the start of each shell session:

```python
from datetime import datetime, timezone
from models import User, Song
from services.streak_service import update_listening_streak
from app import db
```

To obtain user and song IDs, I queried the database at the start of each session:

```python
for u in User.query.all():
    print(u.id, u.username)
for s in Song.query.all():
    print(s.id, s.title)
```

### Bug 1 — Streak resets on Sunday

**How I reproduced it:**

During codebase orientation, I opened `streak_service.py` and noticed an inconsistency between the docstring and the logic of `update_listening_streak(user, now)`. The docstring states the only condition for resetting the streak is "if more than one day has passed," but the code includes an additional condition not mentioned in the docstring: `today.weekday() != 6`. Using AI, I confirmed that `weekday() == 6` corresponds to Sunday in Python's datetime convention. This told me the condition to reproduce the bug was: a user must listen on consecutive days where the final day lands on a Sunday.

Using the setup described above, I selected user nova (`c5885cc1-2b3d-4aae-8e8c-57aa54e17dda`) and song Midnight Drive (`87e531a2-6355-4e7e-b40f-6909a6ca844b`). I called `update_listening_streak()` directly with controlled datetimes simulating three consecutive listening days ending on Sunday:

```python
friday = datetime(2026, 7, 3, 12, 0, 0, tzinfo=timezone.utc)
update_listening_streak(user, friday)
db.session.commit()
print(f"After Friday: streak = {user.listening_streak}")
# After Friday: streak = 1

saturday = datetime(2026, 7, 4, 12, 0, 0, tzinfo=timezone.utc)
update_listening_streak(user, saturday)
db.session.commit()
print(f"After Saturday: streak = {user.listening_streak}")
# After Saturday: streak = 2

sunday = datetime(2026, 7, 5, 12, 0, 0, tzinfo=timezone.utc)
update_listening_streak(user, sunday)
db.session.commit()
print(f"After Sunday: streak = {user.listening_streak}")
# After Sunday: streak = 1
```

Expected streak after Sunday: 3. Actual: 1. The streak reset to 1 instead of incrementing despite three consecutive days of listening.

**How I found the root cause:**

The inconsistency between the docstring and the code in `update_listening_streak()` was the first signal. The docstring makes no mention of any day-of-week condition, but the code has `days_since_last == 1 and today.weekday() != 6` as the condition for incrementing the streak. Tracing this logic: if `days_since_last == 1` but `today.weekday() == 6` (Sunday), the condition evaluates to False and the `else` branch fires, resetting the streak to 1. This is the specific line that causes the bug, and the reproduction output confirmed it: three consecutive days ending on Sunday produced a streak of 1 instead of 3.

**Root cause:**

The `update_listening_streak()` function in `streak_service.py` contains an extra condition `today.weekday() != 6` in the `elif` branch that handles consecutive-day listening. Python's `datetime.weekday()` returns 6 for Sunday, so on any Sunday where `days_since_last == 1`, the condition evaluates to False and falls through to the `else` branch, which resets the streak to 1. This contradicts the docstring, which states the only reset condition is "if more than one day has passed." The Sunday check has no logical justification and causes any streak built through Saturday to silently reset on Sunday despite consecutive listening.

**Fix and side-effect check:**

The fix was removing `and today.weekday() != 6` from the `elif` condition in `update_listening_streak()`, leaving it as `elif days_since_last == 1:`. This makes consecutive-day listening always increment the streak regardless of the day of the week, which matches the docstring's stated behavior.

To verify the fix works on both sides of the boundary, I ran two tests in the flask shell. First, three consecutive days ending on Sunday produced streaks of 1, 2, 3 confirming Sunday no longer resets the streak. Second, skipping a day on Sunday (Friday to Sunday) and skipping a day on Tuesday (Sunday to Tuesday) both correctly reset the streak to 1 each time, confirming the skip-day reset behavior was not affected by the change. No other functions in `streak_service.py` reference `weekday()` or the day-of-week condition, so no related functionality was affected.

### Bug 2 — Friends Listening Now shows stale data

**How I reproduced it:**

Based on the bug description and the README identifying `feed_service.py` as the affected service, I opened it and read the docstring for `get_friends_listening_now(user_id)`, which references "recently" without defining a specific time window. Since the bug report mentions friends from yesterday appearing, I looked for where the time boundary is enforced and found a `cutoff` variable calculated as `datetime.now(timezone.utc) - RECENT_THRESHOLD`, where `RECENT_THRESHOLD = timedelta(hours=24)`. This told me the condition to reproduce the bug was: create a listening event for a friend between 1 and 24 hours ago and confirm it appears in the feed.

I first queried the friendship graph to identify suitable users. I selected nova (`c5885cc1-2b3d-4aae-8e8c-57aa54e17dda`) as the viewing user, kenji (`87300af4-4574-4516-8d05-6706c2dc6481`) as a friend who would listen now, and simone (`734d40ce-88a5-4d50-bccc-0286818d2eda`) as a friend who would listen at stale timestamps:

```python
for u in User.query.all():
    friends = u.friends.all()
    if friends:
        print(f"{u.username}: {[f.username for f in friends]}")
# nova: ['simone', 'kenji', 'darius']
```

I created listening events at three different timestamps and called `get_friends_listening_now` to confirm the boundary behavior:

- kenji listened now — expected to appear
- simone listened 23 hours ago (yesterday) — should NOT appear but does
- simone listened 49 hours ago (two days back) — expected to be excluded

```python
# Fresh event for kenji (now)
now = datetime.now(timezone.utc)
# Result: kenji 2026-07-04T17:18:15.949516 -> appears correctly

# Stale event for simone (23 hours ago, yesterday)
twenty_three_hours_ago = datetime.now(timezone.utc) - timedelta(hours=23)
# Result: simone 2026-07-03T18:27:56.917487 -> appears, this is the bug

# Older event for simone (49 hours ago, two days back)
two_days_ago = datetime.now(timezone.utc) - timedelta(hours=49)
# Result: simone does not appear -> correctly excluded
```

The filter correctly excludes events beyond 24 hours but incorrectly includes events from yesterday. Someone who listened 23 hours ago is not currently listening.

**How I found the root cause:**

Reading `get_friends_listening_now()` in `feed_service.py`, the specific cause became clear at the `RECENT_THRESHOLD` constant defined at the top of the file as `timedelta(hours=24)`. The cutoff is calculated as `datetime.now(timezone.utc) - RECENT_THRESHOLD`, and the filter passes any event where `listened_at >= cutoff`. A 24-hour window is too broad for a feature named "Friends Listening Now"; it includes events from yesterday that are clearly not "now." The reproduction confirmed this: simone's 23-hour-old event passed the filter and appeared in the feed alongside kenji's fresh event, while simone's 49-hour-old event was correctly excluded.

**Root cause:**

The `get_friends_listening_now()` function in `feed_service.py` uses a module-level constant `RECENT_THRESHOLD = timedelta(hours=24)` to calculate the cutoff time for filtering listening events. The cutoff is calculated as `datetime.now(timezone.utc) - RECENT_THRESHOLD`, and any event where `listened_at >= cutoff` passes the filter. A 24-hour window is too broad for a feature named "Friends Listening Now"; it includes events from yesterday that are clearly not current. Someone who listened 23 hours ago is not currently listening, but their event passes the filter and appears in the feed alongside genuinely recent events.

**Fix and side-effect check:**

The fix was changing `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(hours=1)` in `feed_service.py`. A 1-hour window is a defensible definition of "now" for a listening feed, keeping the same format as the original value while making the threshold meaningfully tighter.

To verify the fix works on both sides of the boundary, I ran tests in the flask shell creating listening events at four different timestamps and calling `get_friends_listening_now()` for nova. Kenji (now) and darius (30 minutes ago) appeared correctly, while simone (2 hours ago and 23 hours ago) was correctly excluded in both cases, confirming the tighter threshold works on both sides of the boundary. The `get_activity_feed()` function in the same file was not affected since it has no time filter and is intentionally unbounded.

### Bug 3 — Duplicate songs in search results

**How I reproduced it:**

Based on the bug description and the README identifying `search_service.py` as the affected service, I opened it and read the docstring for `search_songs()`. The docstring seemed fine on initial inspection. I first suspected `db.or_()` might cause a song to match twice when both title and artist fields contain the query string. However, after checking this intuition with Claude, I learned that `db.or_()` doesn't work that way; SQLAlchemy deduplicates matches on the same row regardless of how many fields match.

Still thinking in terms of what operations combine or multiply rows, I looked for the next candidate in the query. Joins, like `or`, are combination operators that can produce multiple rows from a single source row depending on how the tables relate. I found `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` in the query, which seemed suspicious since joining on `song_tags` could potentially produce one row per tag for each song. This reminded me of the codebase map I had written earlier, where I noted that `song_tags` is already an association table between songs and tags, raising the question of whether the outerjoin was redundant. I went back to `models.py` and confirmed that `Song` already defines a `tags` relationship: `tags = db.relationship("Tag", secondary=song_tags, lazy="subquery")`, which loads automatically when `Song.to_dict()` is called since `to_dict()` directly accesses `self.tags`. This suggested the outerjoin was unnecessary and could be the source of the duplicates.

The likely reproduction condition was therefore: search for a song that has multiple tags. I first listed all songs and their tags to identify suitable test cases:

```python
from models import Song
for s in Song.query.all():
    print(s.title, [t.name for t in s.tags])
# Midnight Drive []
# Block Party ['hip-hop']
# Crown Heights Anthem ['rap', 'hip-hop', 'boom bap']
```

The seed data had no 2-tag songs, so I added a second tag to Block Party (`5267e668-50a6-41f6-8c34-748d4a96a1e4`) by appending the existing `rap` tag, which is contextually appropriate given the song's genre:

```python
block_party = db.session.get(Song, '5267e668-50a6-41f6-8c34-748d4a96a1e4')
rap_tag = Tag.query.filter_by(name='rap').first()
block_party.tags.append(rap_tag)
db.session.commit()
# After: ['hip-hop', 'rap']
```

Testing via curl returned only 1 result per song in all cases, suggesting SQLAlchemy's identity map was masking the duplicates at the ORM level. To bypass this, I ran raw SQL queries directly against the database to confirm the bug at the SQL level:

```python
from sqlalchemy import text

# 0 tags: Midnight Drive
raw = db.session.execute(text("""
    SELECT song.id, song.title FROM song
    LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
    WHERE lower(song.title) LIKE lower('%Midnight Drive%')
""")).fetchall()
print(f"0 tags: {len(raw)} row(s)")
# 0 tags: 1 row(s)
# ('87e531a2-6355-4e7e-b40f-6909a6ca844b', 'Midnight Drive')

# 1 tag: Block Party (before adding second tag)
raw = db.session.execute(text("""
    SELECT song.id, song.title FROM song
    LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
    WHERE lower(song.title) LIKE lower('%Block Party%')
""")).fetchall()
print(f"1 tag: {len(raw)} row(s)")
# 1 tag: 1 row(s)
# ('5267e668-50a6-41f6-8c34-748d4a96a1e4', 'Block Party')

# 2 tags: Block Party (after adding rap tag)
raw = db.session.execute(text("""
    SELECT song.id, song.title FROM song
    LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
    WHERE lower(song.title) LIKE lower('%Block Party%')
""")).fetchall()
print(f"2 tags: {len(raw)} row(s)")
# 2 tags: 2 row(s)
# ('5267e668-50a6-41f6-8c34-748d4a96a1e4', 'Block Party')
# ('5267e668-50a6-41f6-8c34-748d4a96a1e4', 'Block Party')

# 3 tags: Crown Heights Anthem
raw = db.session.execute(text("""
    SELECT song.id, song.title FROM song
    LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
    WHERE lower(song.title) LIKE lower('%Crown%')
""")).fetchall()
print(f"3 tags: {len(raw)} row(s)")
# 3 tags: 3 row(s)
# ('21215787-5cf4-40e1-8718-35e722b7e177', 'Crown Heights Anthem')
# ('21215787-5cf4-40e1-8718-35e722b7e177', 'Crown Heights Anthem')
# ('21215787-5cf4-40e1-8718-35e722b7e177', 'Crown Heights Anthem')
```

The raw SQL confirms the bug scales with tag count: 0 tags produces 1 row, 1 tag produces 1 row, 2 tags produces 2 duplicate rows, and 3 tags produces 3 duplicate rows. SQLAlchemy's identity map masks these duplicates at the ORM level under normal conditions, making the bug conditional and difficult to observe through the API directly.

**How I found the root cause:**

The moment of confidence came when I confirmed in `models.py` that `Song` already loads tags automatically through its `tags` relationship using `lazy="subquery"`, making the outerjoin in `search_songs()` completely redundant. The raw SQL test then confirmed what the outerjoin was doing at the database level: producing one row per tag entry per song, which would surface as duplicates in any context where SQLAlchemy's identity map deduplication is not active. The specific cause is the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` call in `search_songs()`, which multiplies rows for songs with multiple tags without contributing any information that `to_dict()` doesn't already retrieve on its own.

**Root cause:**

The `search_songs()` function in `search_service.py` includes an unnecessary `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` in its query. Because `song_tags` is a many-to-many association table, the join produces one row per tag for each matching song. A song with 2 tags produces 2 rows and a song with 3 tags produces 3 rows at the SQL level. The bug is conditional because SQLAlchemy's identity map deduplicates results by primary key at the ORM level in most cases, masking the duplicates from the API response. However, the outerjoin is entirely unnecessary since `Song.to_dict()` already loads tags automatically through the `tags = db.relationship("Tag", secondary=song_tags, lazy="subquery")` relationship defined in `models.py`, and the search filter only matches on title and artist, never on tags.

**Fix and side-effect check:**

The fix was removing the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line from the query in `search_songs()`, and cleaning up the now-unused `Tag` and `song_tags` imports from the import line, leaving it as `from models import Song`. This eliminates the source of the duplicate rows without affecting tag loading, since tags continue to be loaded automatically through the SQLAlchemy relationship.

To verify the fix, I confirmed at the SQL level that queries without the join return exactly 1 row per song regardless of tag count: Midnight Drive (0 tags), Block Party (2 tags), and Crown Heights Anthem (3 tags) all returned 1 row. Tag data was also confirmed to still load correctly through `to_dict()`, with all tags present in the results.

### Bug 4 — No notification when a song is rated

**How I reproduced it:**

For this reproduction I used nova (`c5885cc1-2b3d-4aae-8e8c-57aa54e17dda`) as the song sharer and simone (`734d40ce-88a5-4d50-bccc-0286818d2eda`) as the friend who rated the song. Nova shared Midnight Drive (`87e531a2-6355-4e7e-b40f-6909a6ca844b`). The seed data already includes one playlist notification for nova as a baseline, confirming the playlist notification path works correctly:

```bash
curl "http://127.0.0.1:5000/users/c5885cc1-2b3d-4aae-8e8c-57aa54e17dda/notifications"
# count: 1 — one existing playlist notification from darius (seeded)
```

Simone then rated Midnight Drive with a score of 5:

```bash
curl -X POST "http://127.0.0.1:5000/songs/87e531a2-6355-4e7e-b40f-6909a6ca844b/rate" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "734d40ce-88a5-4d50-bccc-0286818d2eda", "score": 5}'
# Response: rating saved successfully with score 5
```

Checking nova's notifications again:

```bash
curl "http://127.0.0.1:5000/users/c5885cc1-2b3d-4aae-8e8c-57aa54e17dda/notifications"
# count: 1 — still only the original seeded notification, no new rating notification
```

Expected: 2 notifications (playlist add + rating). Actual: 1 notification (playlist add only). The rating was saved but nova was never notified.

**How I found the root cause:**

Based on the bug description and the README identifying `notification_service.py` as the affected service, I first checked `routes/songs.py` to confirm the call chain. There I found that `POST /songs/<song_id>/rate` calls `rate_song()` imported directly from `notification_service.py`, confirming that `rate_song()` is the entry point for the rating action.

Opening `notification_service.py`, I focused on comparing `add_to_playlist()` with `rate_song()`, following the assignment hint that the root cause is architectural rather than a typo. Reading both docstrings, `add_to_playlist()` explicitly states it notifies the song's sharer, while `rate_song()` only mentions saving the rating. Comparing the two functions line-by-line, both share similar structural elements: input validation, database lookups, and guards. However, `add_to_playlist()` contains a dedicated block with the comment "Notify the person who originally shared the song (if it wasn't them who added it)" followed by a `create_notification()` call. That entire notification block is completely absent from `rate_song()`, which saves the rating and returns without ever notifying the song's original sharer. The moment of confidence was seeing that the guard condition (`song.shared_by != added_by_user_id`) and the `create_notification()` call pattern existed in `add_to_playlist()` but had no equivalent in `rate_song()`.

**Root cause:**

The `rate_song()` function in `notification_service.py` saves the rating correctly but never calls `create_notification()` to notify the song's original sharer. Comparing `rate_song()` line-by-line with `add_to_playlist()`, which handles a similar social interaction, reveals that `add_to_playlist()` contains a dedicated notification block guarded by `if song.shared_by != added_by_user_id` followed by a `create_notification()` call. That entire block is absent from `rate_song()`, meaning the song's sharer is never notified when someone rates their song, regardless of who the rater is.

**Fix and side-effect check:**

The fix was adding the missing notification block to `rate_song()` in `notification_service.py`, mirroring the pattern used in `add_to_playlist()`:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}'.",
    )
```

The block is placed after `db.session.commit()` so the rating is saved before the notification is created, matching the order used in `add_to_playlist()`.

To verify the fix, simone rated Midnight Drive and nova's notification count increased from 1 to 2, with a new `song_rated` notification reading "simone rated your song 'Midnight Drive'." To confirm the guard condition works correctly, nova rated her own song and the count stayed at 2, confirming no self-notification was created. The `get_notifications()` and `mark_as_read()` functions in the same file were not affected since they only read and update existing notifications.

### Bug 5 — Last song in a playlist never shows up

**How I reproduced it:**

My initial reproduction plan was to use curl to first check a playlist, note the last visible song, add a new song, and then check again, expecting the newly added song to be missing while the previously hidden last song would become visible. Using the seeded playlist "Late Night Vibes" (`74f50e06-1d57-4b7e-9b49-b2b1f9d1dcae`), I first called the API:

```bash
curl "http://127.0.0.1:5000/playlists/74f50e06-1d57-4b7e-9b49-b2b1f9d1dcae/songs"
# count: 6, last song shown: Golden Hour (position 6)
```

However, attempting to add a new song via `POST /playlists/<playlist_id>/songs` returned a 500 error: `NOT NULL constraint failed: playlist_entries.position`. This is because `add_to_playlist()` in `notification_service.py` uses `playlist.songs.append(song)` which does not set the required `position` column. This appeared to be a separate issue, so I fell back to comparing the API response against the raw database state directly:

```python
from models import playlist_entries, Song
from sqlalchemy import asc
from app import db

rows = db.session.execute(
    db.select(playlist_entries.c.position, Song.title)
    .join(Song, Song.id == playlist_entries.c.song_id)
    .where(playlist_entries.c.playlist_id == '74f50e06-1d57-4b7e-9b49-b2b1f9d1dcae')
    .order_by(asc(playlist_entries.c.position))
).fetchall()

for row in rows:
    print(row)
# (1, 'Midnight Drive')
# (2, 'Still Waters')
# (3, 'First Light')
# (4, 'Block Party')
# (5, 'Late Night Session')
# (6, 'Golden Hour')
# (7, 'Free Throws')
```

The database has 7 songs but the API returned only 6. Free Throws at position 7 is the last song and is missing from the API response, confirming the bug is specifically the last position song being dropped, not a random missing entry.

**How I found the root cause:**

Based on the bug description and the README identifying `playlist_service.py` as the affected service, I opened it and focused on `get_playlist_songs()` as the most directly related function. Reading the docstring, two things stood out: songs are returned in ascending order by position, and an explicit note states "This function returns all songs in the playlist." That note made the return statement immediately suspicious, since if all songs should be returned, any slicing would be a contradiction. Checking the return statement confirmed the specific cause: `return [song.to_dict() for song in songs[:-1]]`. In Python, `songs[:-1]` slices from the start up to but not including the last element, meaning the song at the last position is always excluded regardless of what it is. The comparison between the API response (6 songs) and the raw database state (7 songs) confirmed that Free Throws at position 7 was being dropped by this slice.

**Root cause:**

The `get_playlist_songs()` function in `playlist_service.py` uses `songs[:-1]` in its return statement: `return [song.to_dict() for song in songs[:-1]]`. In Python, `[:-1]` slices a list from the start up to but not including the last element, meaning the song at the highest position in the playlist is always excluded regardless of what it is. This directly contradicts the docstring, which explicitly states "This function returns all songs in the playlist." The bug affects every playlist consistently since the slice always drops the last element regardless of playlist size or content.

**Fix and side-effect check:**

The fix was removing `[:-1]` from the return statement in `get_playlist_songs()`, leaving it as `return [song.to_dict() for song in songs]`. This returns all songs in the playlist in ascending position order, which matches the docstring's stated behavior.

To verify the fix works on both sides of the boundary, I first confirmed via curl that all three seeded playlists now return 7 songs each with the last song correctly included: Late Night Vibes ends with Free Throws, Friday Energy ends with Harlem Renaissance, and Study Mode ends with Lagos to London. I then added a new song (Frequencies) to Late Night Vibes at position 8 directly via the flask shell and confirmed via curl that the count increased to 8 and Frequencies appeared correctly as the last song. No other functions in `playlist_service.py` reference the songs list or perform any slicing, so no related functionality was affected.