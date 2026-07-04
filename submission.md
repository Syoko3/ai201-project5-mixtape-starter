# ai201-project5-mixtape-starter - submission.md

Screenshot of git log --oneline:


---

## AI Usage

- *How you used AI tools during codebase navigation and debugging:* 
- *What they helped you understand:* 
- *Where you verified or overrode their output:* 

---

## Codebase Map

**Main files & What each one does:**

- app.py: Creates the Flask app, configures SQLAlchemy, registers all route blueprints, and creates database tables.
- models.py: Defines the database schema and model relationships for users, songs, tags, listening events, ratings, playlists, playlist entries, friendships, and notifications.
- routes/songs.py: Handles song search, song detail lookup, song rating, and recording when a user listens to a song.
- routes/playlists.py: Handles playlist creation, playlist detail lookup, retrieving playlist songs, and adding songs to playlists.
- routes/users.py: Handles user profile lookup, listening streak lookup, notification retrieval, and marking notifications as read.
- routes/feed.py: Handles feed endpoints for "Friends Listening Now" and general friend activity.
- services/search_service.py: Contains song search logic and single-song lookup logic.
- services/streak_service.py: Records listening events and updates user listening streaks.
- services/feed_service.py: Builds feed responses from friends' ListeningEvent records.
- services/playlist_service.py: Creates playlists and retrieves playlist metadata or ordered playlist songs.
- services/notification_service.py: Creates notifications, handles playlist-add notification behavior, saves song ratings, retrieves notifications, and marks notifications as read.
- seed_data.py: Populates the database with sample users, friendships, songs, tags, playlists, ratings, listening events, and notifications.
- tests/: Contains pytest tests for streak, search, and playlist behavior.

**Data flow example: adding a song to a playlist triggers a notification**

1. A client sends POST /playlists/<playlist_id>/songs with song_id and added_by.
2. routes/playlists.py runs the add_song(playlist_id) route.
3. The route validates that song_id and added_by were provided.
4. The route calls add_to_playlist(playlist_id, song_id, added_by) from services/notification_service.py.
5. add_to_playlist() looks up the Song, the user who added it, and the Playlist.
6. If the song is not already in the playlist, it appends the song to playlist.songs and commits the database change.
7. If the person adding the song is not the original sharer, add_to_playlist() calls create_notification().
8. create_notification() creates a Notification row for the original song sharer and commits it.
9. Later, when the user visits GET /users/<user_id>/notifications, routes/users.py calls get_notifications() from services/notification_service.py.
10. get_notifications() queries the user's notifications, orders them newest first, converts them to dictionaries, and returns them to the route.

**Organization Patterns:**

- Routes are thin controller layers. They read request data, validate required fields, call service functions, and turn results or errors into JSON responses.
- Services contain most of the app logic. They query models, enforce feature rules, update records, and commit database changes.
- Models define both tables and serialization through to_dict() methods.
- Errors are usually raised as ValueError in the service layer and caught in the route layer.
- The app is organized by feature area: songs, playlists, users, feed, search, streaks, and notifications.
- Database access is done through the shared db object from app.py.
- Many features are connected through shared models. For example, ListeningEvent powers the feed and streaks, while Song, Playlist, and Notification connect playlist activity to user notifications.

---

## Root Cause Analysis



---

## Notes

**File summary:**
<!--- Give the AI the contents of a service file and ask "What is this module responsible for? What are its main functions and what does each one do?" --->
playlist_service.py handles playlist creation and retrieval. It validates users and playlists, creates new playlists, gets playlist metadata, lists playlists created by a user, and retrieves songs in playlist order. The main bug is in get_playlist_songs(), where songs[:-1] accidentally removes the final song from the returned list.

**Data flow trace:**
<!--- Ask "Given this services/ directory, trace how a song gets added to a user's feed — which functions are called and in what order?" (paste the relevant files as context) --->
A song gets added to a user's feed through the listening-event flow.

Relevant files:
- routes/songs.py
- services/streak_service.py
- models.py
- routes/feed.py
- services/feed_service.py

Function call order:

1. POST /songs/<song_id>/listen:
   - Defined in routes/songs.py as the listen(song_id) route.
   - Reads user_id from the request body.
   - Calls record_listening_event(user_id, song_id).

2. record_listening_event(user_id, song_id):
   - Defined in services/streak_service.py.
   - Looks up the user with db.session.get(User, user_id).
   - Creates a new ListeningEvent with the user ID, song ID, and current timestamp.
   - Adds that event to the database session.
   - Calls update_listening_streak(user, now).
   - Commits the database transaction.
   - Returns the new ListeningEvent.

3. update_listening_streak(user, now):
   - Defined in services/streak_service.py.
   - Updates the user's listening streak based on their previous last_listened_at date.
   - This does not directly affect the feed, but it happens during the same listen action.

4. ListeningEvent:
   - Defined in models.py.
   - This is the database record that represents "user listened to song."
   - Feed pages use these records to decide what songs appear.

5. GET /feed/<user_id>/listening-now:
   - Defined in routes/feed.py as listening_now(user_id).
   - Calls get_friends_listening_now(user_id).

6. get_friends_listening_now(user_id):
   - Defined in services/feed_service.py.
   - Gets the current user.
   - Finds their friends.
   - Queries recent ListeningEvent records from those friends.
   - Orders them from newest to oldest.
   - Keeps only the most recent song per friend.
   - Returns feed items with friend, song, and listened_at.

7. GET /feed/<user_id>/activity:
   - Also defined in routes/feed.py.
   - Calls get_activity_feed(user_id).

8. get_activity_feed(user_id):
   - Defined in services/feed_service.py.
   - Gets the current user's friends.
   - Queries their recent listening events.
   - Returns up to 20 events ordered by most recent first.

Summary:
The feed is powered by ListeningEvent records. When a user listens to a song, routes/songs.py calls record_listening_event(), which creates a ListeningEvent. Later, feed_service.py reads those ListeningEvent records to display songs in friends' feeds.

**Function explanation:**
<!--- Give the AI a function you don't understand and ask "Walk me through what this function does step by step, including what it returns and what could cause it to return an unexpected value." --->
