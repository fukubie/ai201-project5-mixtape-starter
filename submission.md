# Project Submission: Mixtape Bug Hunt

## AI Usage
I utilized an AI assistant to accelerate codebase navigation, help summarize module responsibilities for the codebase map, and verify structural differences between database join operations during my root cause investigations. All code changes were reviewed and verified manually before committing.

---

## Codebase Map

### Main Files and Roles
* **`models.py`**: Defines the central SQLAlchemy database schemas (User, Song, Playlist, PlaylistSong, Notification). It establishes how entities relate to one another.
* **`routes/`**: Handles incoming HTTP requests, performs basic input parsing, and formats JSON responses. It acts as the entry point but contains no business logic; it immediately delegates tasks to the services layer.
* **`services/`**: The core business logic layer. Files like `streak_service.py`, `feed_service.py`, and `playlist_service.py` process data, apply conditional rules, and update the database. All five open bugs reside in this layer.
* **`app.py` & `seed_data.py`**: Handles the Flask application setup and seeds the local database with initial mock profiles, playlists, and histories for testing.

### Feature Data Flow Example (Rating a Song)
1. A user triggers a `POST` request to `/songs/<song_id>/rate` which lands in `routes/songs.py`.
2. The route extracts the song data and user information, then invokes the service layer via `rate_song()`.
3. The service layer handles updating the data metrics and calls `notification_service.py` to generate a new `Notification` record for the song's original owner.
4. The database is updated, and the route returns a success response back to the client.

### Architecture Patterns
The application strictly enforces a **Three-Tier Architecture** (Routes -> Services -> Models). Routes handle transport/HTTP, services handle business logic, and models handle data structure. Every endpoint immediately hands off execution to a dedicated service function.

---

## Root Cause Analyses

### Issue #5: The last song in a playlist never shows up

* **How you reproduced it:** Evaluated the data population schema in `seed_data.py`, which inserts 7 songs into the first playlist (`Late Night Vibes`). Cross-referenced this against the logic in `services/playlist_service.py` where a slice array of `[:-1]` is consistently applied to the query return list, ensuring that any payload served will mathematically be missing its final element.
* **How you found the root cause:** Traced the data layer execution from `routes/playlists.py` under the `/playlists/<playlist_id>/songs` endpoint down to the core service implementation `get_playlist_songs` inside `services/playlist_service.py`.
* **The root cause:** The `get_playlist_songs` function fetches the full ordered list of tracks using a SQLAlchemy query join on the `playlist_entries` table. However, the return statement reads `return [song.to_dict() for song in songs[:-1]]`. The Python slice `[:-1]` truncates the array by dropping the final element, resulting in an off-by-one data loss for the user.
* **Your fix and side-effect check:** Removed the `[:-1]` array slice constraint so the final line reads `return [song.to_dict() for song in songs]`. Checked surrounding functions (`get_playlist`, `create_playlist`) to ensure no other database queries or representations were modified or impacted.