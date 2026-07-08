# Mixtape Bug Hunt — Submission

## Codebase Map

### Main files and their roles

- `app.py` — Flask app factory and DB setup. Entry point for the app.
- `models.py` — Defines 7 SQLAlchemy models: User, Tag, Song, ListeningEvent, Rating,
  Playlist, Notification. Also defines 3 association tables:
  - `friendships` — many-to-many between users (User.friends)
  - `song_tags` — many-to-many between songs and tags
  - `playlist_entries` — many-to-many between playlists and songs, but with extra
    columns: `position` (explicit ordering), `added_by`, `added_at`. Songs in a
    playlist have an explicit position, not just insertion order.
- `routes/` — Thin HTTP layer. Each file (songs.py, playlists.py, users.py, feed.py)
  parses request input, calls a service function, and formats the JSON response.
  Routes contain no business logic themselves.
- `services/` — All business logic lives here. Five files: streak_service,
  feed_service, search_service, notification_service, playlist_service.

### Pattern noticed

Routes delegate immediately to services. Interestingly, some actions are handled by
a service file you wouldn't expect from the route's own file — e.g. adding a song
to a playlist (`routes/playlists.py`) calls `notification_service.add_to_playlist()`,
not a function in `playlist_service.py`. This is because adding a song to a playlist
always triggers a notification to the song's original sharer, so that side-effect
logic is bundled together in notification_service.

### Data flow #1 — retrieving a playlist's songs

`GET /playlists/<id>/songs` in `routes/playlists.py` calls
`playlist_service.get_playlist_songs(playlist_id)`. That function queries all Song
rows joined through the `playlist_entries` table, ordered ascending by `position`,
then returns `song.to_dict()` for each song. The route wraps this in
`{"songs": songs, "count": len(songs)}`.

Note: the docstring for `get_playlist_songs()` says "This function returns all
songs in the playlist," but the actual return statement is
`[song.to_dict() for song in songs[:-1]]` — a Python slice that drops the last
element of the ordered list. This means the function under-returns by one song,
and the route's `count` field reflects the truncated list, not the true total.
(Flagged for investigation — Issue #5. Not yet formally reproduced or fixed.)

### Data flow #2 — adding a song to a playlist (and the notification pattern)

`POST /playlists/<id>/songs` in `routes/playlists.py` calls
`notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)`.
That function: looks up the song, adder, and playlist; appends the song to
`playlist.songs` if not already present; then, if the adder is not the song's
original sharer, calls `create_notification()` to notify the sharer.

Compared to `rate_song()` in the same file: rate_song() saves a Rating record
(creating or updating) but has no equivalent notification block at all — no
self-check, no `create_notification()` call. The notification pattern that exists
for playlist-adds is simply absent for ratings.
(Flagged for investigation — Issue #4. Not yet formally reproduced or fixed.)

## Root Cause Analysis Entries

### Issue #5 — Last song in a playlist never shows up

**How I reproduced it:** Created a test playlist and inserted 4 songs into
`playlist_entries` with explicit positions 0–3 via `flask shell`. Called
`get_playlist_songs()` and confirmed it returned only 3 songs (missing the
4th, "Block Party," at position 3).

**How I found the root cause:** Started at `routes/playlists.py`'s
`get_songs()` endpoint, which calls `playlist_service.get_playlist_songs()`.
Reading that function, the query itself correctly joins `playlist_entries`
and orders by `position` ascending. The return statement was
`[song.to_dict() for song in songs[:-1]]` — noticed the `[:-1]` slice
directly contradicted the function's own docstring, which claims it
"returns all songs in the playlist."

**The root cause:** `songs[:-1]` is a Python slice that returns all
elements of a list except the last one. Since `songs` is already the
correctly-ordered full list of playlist songs, this slice silently drops
the last song in the list every time, regardless of playlist size. A
1-song playlist would show 0 songs; a 4-song playlist shows 3.

**My fix and side-effect check:** Changed the return statement to
`[song.to_dict() for song in songs]`, removing the slice entirely. I
checked `routes/playlists.py`, which uses this function's return value
for both the `songs` list and the `count` field in its JSON response —
both are now correct automatically, since they both derive from the same
(now-complete) list. No other code in the codebase calls this function.