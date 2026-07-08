# Project 5: Mixtape Bug Hunt Submission

## AI Usage

I used AI tools during codebase navigation and debugging, not just to generate code. I asked AI to help explain unfamiliar service files, trace route-to-service call chains, and compare broken logic against similar working code. I verified the AI explanations myself by running the tests, reading the actual functions involved, and checking that each fix matched the specific bug behavior.

Specific ways I used AI:

1. I used AI to help summarize the purpose of the main route and service files so I could understand how the app was organized before fixing bugs.
2. I used AI to help trace bugs from the failing behavior into the correct service function, especially for the streak and playlist bugs.
3. I used AI to compare the working notification pattern in `add_to_playlist()` with the missing notification behavior in `rate_song()`.

I did not rely on AI blindly. For example, before fixing the Sunday streak bug, I first ran `pytest tests/test_streaks.py` and confirmed the failing test showed that Saturday to Sunday should increment the streak but was resetting instead. I also verified every fix by rerunning the tests after changing the code.

---

## Codebase Map

### Main Files and Roles

- `app.py`: Creates the Flask application, configures the SQLite database, initializes SQLAlchemy, and registers the route blueprints.
- `models.py`: Defines the database models used by the app, including `User`, `Song`, `Playlist`, `Notification`, `Rating`, friendships, playlist entries, and listening events.
- `routes/songs.py`: Handles song-related API routes. It receives requests for song search, song details, listening events, and song ratings, then calls the matching service functions.
- `routes/playlists.py`: Handles playlist-related API routes. It calls playlist service functions to create playlists, get playlist metadata, and retrieve songs in a playlist.
- `routes/users.py`: Handles user-related API routes, including profile, streak, and notification endpoints.
- `routes/feed.py`: Handles feed-related routes for friend listening activity.
- `services/streak_service.py`: Contains the listening streak logic. It records listening events and updates a user’s streak depending on the date of their last listen.
- `services/playlist_service.py`: Contains playlist creation and playlist retrieval logic. It queries playlist songs through the playlist join table and returns them in position order.
- `services/search_service.py`: Contains song search logic.
- `services/notification_service.py`: Contains notification logic. It creates notifications and handles song interaction features such as adding a song to a playlist and rating a song.
- `seed_data.py`: Recreates and populates the local database with sample users, songs, playlists, tags, friendships, notifications, and listening events for testing.

### Data Flow Example: Rating a Song

When a user rates a song, the request reaches the song rating route in `routes/songs.py`. That route reads the user ID, song ID, and score from the request and calls `rate_song()` in `services/notification_service.py`.

The service validates the score, loads the song and rater from the database, checks whether the user already rated that song, then either updates the existing rating or creates a new `Rating` record. After the rating is saved, the service should also notify the original sharer of the song if the rater is a different user.

### Pattern I Noticed

The app separates route handling from business logic. The `routes/` files mainly receive requests and return JSON responses, while the `services/` files contain the logic that actually changes or reads data. Because of that, the best debugging strategy was to reproduce the bug first, identify the route connected to the broken feature, then follow the call into the matching service file and inspect the specific condition, query, or missing function call.

---

## Issue #1: Streak resets instead of incrementing on Sunday

### How I reproduced it

I reproduced this bug by running:

```bash
pytest tests/test_streaks.py
```

The failing test was `test_streak_increments_on_sunday`. It created a listening event on Saturday and then another listening event on Sunday. The expected result was that the user’s streak would increase from 1 to 2 because Saturday and Sunday are consecutive days. Instead, the streak stayed at 1, which showed that the code was treating Sunday as a reset case.

### How I found the root cause

I started with the failing test in `tests/test_streaks.py`, which pointed directly to the streak update behavior. From there, I opened `services/streak_service.py` and inspected `update_listening_streak()`, because that is the function responsible for deciding whether a streak should stay the same, increment, or reset.

The key moment was finding the condition:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

That condition matched the failing behavior exactly because Python’s `weekday()` returns `6` for Sunday.

### The root cause

The root cause was an incorrect Sunday-specific condition in `update_listening_streak()`. The code only incremented the streak when `days_since_last == 1` and `today.weekday() != 6`.

Since Sunday has a weekday value of `6`, the condition failed on Sundays even when the previous listening date was Saturday. That caused the code to fall into the `else` block and reset the streak to 1.

The bug was not that the app could not calculate consecutive days. The specific issue was that it added an unnecessary exception for Sundays, even though Saturday to Sunday is still a normal one-day difference.

### My fix and side-effect check

I changed the condition from:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

to:

```python
elif days_since_last == 1:
```

This fixes the root cause because any consecutive calendar day should increment the streak, including Sunday.

To check for side effects, I reran:

```bash
pytest tests/test_streaks.py
```

All 5 streak tests passed. This confirmed that new streaks still start at 1, same-day listens do not double-count, skipped days still reset the streak, normal consecutive days still increment, and Sunday now increments correctly.

---

## Issue #5: Last song in playlist is missing

### How I reproduced it

I reproduced this bug by running the full test suite:

```bash
pytest
```

The failing tests were in `tests/test_playlists.py`. The test `test_playlist_returns_all_songs` expected 5 songs in the playlist, but the function returned only 4. The test `test_playlist_returns_songs_in_order` expected the titles `Track 1`, `Track 2`, `Track 3`, `Track 4`, and `Track 5`, but the returned list was missing `Track 5`.

### How I found the root cause

I started with the failing playlist tests and opened `services/playlist_service.py`, because the failure involved `get_playlist_songs()`. I followed the function that queries songs from the playlist join table and orders them by playlist position.

The query itself looked correct because it joined the playlist entries table and ordered by `position`. The moment I knew I had found the root cause was at the return line:

```python
return [song.to_dict() for song in songs[:-1]]
```

The query was getting the songs, but the return statement was slicing off the final item.

### The root cause

The root cause was the use of `songs[:-1]` in `get_playlist_songs()`. In Python, `songs[:-1]` means “return every item except the last one.”

Because of that, even when the database query correctly returned all playlist songs, the service removed the final song before sending the response back. The bug was not in the database query or the playlist ordering. The specific problem was the list slice in the return statement.

### My fix and side-effect check

I changed the return line from:

```python
return [song.to_dict() for song in songs[:-1]]
```

to:

```python
return [song.to_dict() for song in songs]
```

This fixes the root cause because the service now returns every song retrieved by the ordered query.

To check for side effects, I reran:

```bash
pytest tests/test_playlists.py
```

All 3 playlist tests passed. This confirmed that playlists return all songs, songs are still returned in position order, and empty playlists still return an empty list. I also reran the full test suite with `pytest`, and all 13 tests passed.

---

## Issue #4: Rating a song does not notify the original sharer

### How I reproduced it

I reproduced this bug by reading the issue description and then tracing the rating flow in the code. The route for rating a song calls `rate_song()` in `services/notification_service.py`.

I compared this with the working notification behavior in `add_to_playlist()`, where adding someone else’s song to a playlist creates a notification for the original sharer. In `rate_song()`, the rating was saved correctly, but there was no notification created afterward. This matched the reported behavior: the song rating action happened, but the original sharer was not notified.

### How I found the root cause

I started by opening `routes/songs.py` to see which function handles song rating. That led me to `rate_song()` in `services/notification_service.py`. Then I compared `rate_song()` with `add_to_playlist()` in the same file.

`add_to_playlist()` already had the correct pattern: after the action succeeds, it checks whether the actor is different from the original song sharer and then calls `create_notification()`.

The moment I knew I had found the root cause was when I saw that `rate_song()` ended with:

```python
db.session.commit()
return rating
```

There was no call to `create_notification()` after the rating was saved.

### The root cause

The root cause was a missing notification side effect in `rate_song()`. The function validated the score, found the song and rater, created or updated the `Rating`, committed the database change, and returned the rating. However, it never created a `Notification` for the original song sharer.

This was an architectural consistency bug, not a typo. Another song interaction, `add_to_playlist()`, already followed the notification pattern, but `rate_song()` did not.

### My fix and side-effect check

I added a notification after the rating is committed:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

This fixes the root cause because rating someone else’s shared song now creates a notification for the original sharer. The check `song.shared_by != user_id` prevents users from receiving notifications when they rate their own song.

To check for side effects, I reran the full test suite:

```bash
pytest
```

All 13 visible tests still passed. This confirmed that the notification change did not break playlist behavior, search behavior, or streak behavior. I also checked that the fix follows the same pattern already used by `add_to_playlist()`, so it is consistent with the existing notification design.

---

## Final Verification

I confirmed the app’s visible tests passed after the fixes:

```bash
pytest
```

Result:

```text
13 passed
```

I also confirmed the commit history on the `bugfix/mixtape` branch shows three separate bug fix commits:

```text
fix: notify song sharer when song is rated
fix: include final song in playlist results
fix: allow streaks to continue on Sunday
```
