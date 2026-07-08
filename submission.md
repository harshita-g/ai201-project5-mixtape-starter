# AI Usage

I used AI to help me understand the structure of the unfamiliar codebase, plan a debugging workflow, and trace how the route files connect to the service files. I used AI support for codebase orientation and debugging guidance, but I verified behavior by running the Flask app locally, calling the API endpoints, reading the relevant files myself, and testing each fix before committing.

# Codebase Map

## Main files and responsibilities

- `app.py`: Creates the Flask application, initializes the database, and registers the route blueprints.
- `models.py`: Defines the SQLAlchemy database models used by the app, including users, songs, playlists, playlist-song relationships, notifications, and listening/streak-related data.
- `seed_data.py`: Seeds the local database with sample users, songs, playlists, listens, ratings, and notifications so the reported bugs can be reproduced.
- `routes/feed.py`: Defines feed-related API endpoints, including the "Friends Listening Now" endpoint.
- `routes/playlists.py`: Defines playlist-related API endpoints, including adding songs to playlists and fetching playlist songs.
- `routes/songs.py`: Defines song-related API endpoints, including song search and song rating.
- `routes/users.py`: Defines user-related API endpoints, including streak and notification routes.
- `services/feed_service.py`: Contains feed business logic, including deciding which friends appear in the listening-now feed.
- `services/notification_service.py`: Contains notification creation and retrieval logic.
- `services/playlist_service.py`: Contains playlist business logic, including adding and returning playlist songs.
- `services/search_service.py`: Contains song search logic.
- `services/streak_service.py`: Contains listening streak calculation/update logic.

## Data flow example: searching for a song

1. The client sends a request to `GET /songs/search?q=<query>`.
2. `routes/songs.py` receives the request and reads the search query from the URL.
3. The route calls the search-related service function in `services/search_service.py`.
4. The service searches matching songs from the database.
5. The route returns a response containing the result count and list of matching songs.

## Pattern noticed

The app is organized with a clear route/service split. Route files handle HTTP request parsing and response formatting, while service files contain the actual application logic. The reported bugs are likely in service-layer logic rather than route definitions.

## Issue #3 — The same song keeps showing up twice in search

### How I reproduced it

I tested the search endpoint by calling `GET /songs/search?q=Anthem` and inspected the response returned by the app. I also tested broader search queries to check whether the same song could appear more than once when the query matched songs that had multiple associated tags.

### How I found the root cause

I started from `routes/songs.py` and found that the `/songs/search` route reads the `q` parameter and calls `search_songs(query)` from `services/search_service.py`. In `search_service.py`, I noticed that the song query used an outer join with the `song_tags` table even though the search filter only checked the song title and artist.

### The root cause

The search query joined songs to the `song_tags` table. Since a single song can have multiple tags, the join could create multiple database rows for the same song. The search logic then converted the query results into dictionaries without needing that join, which could cause duplicate song entries in the API response.

### My fix and side-effect check

I removed the unnecessary outer join from the search query because the endpoint only searches by song title and artist. This keeps each matching song represented once while still allowing `song.to_dict()` to include the song’s tag data. After the change, I retested the search endpoint and confirmed that matching songs still appear and duplicate entries are not returned.