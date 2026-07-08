# AI usage

First, for trying to reproduce the bugs, I asked Copilot how to run these endpoints. I was struggling to get the <user_id> and asked it to help me find it. It gave me this command to run on my seeded data: -c "import sqlite3; conn=sqlite3.connect('instance/mixtape.db'); print(conn.execute('select id, username from user').fetchall())" - which had worked in showing me the 5 users and their user id's.

# Codebase Map

## Main Files

**app.py**
The application entry point and setup layer. It creates the Flask app, configures the database connection and secret key, initializes SQLAlchemy, registers the route blueprints for songs, playlists, users, and feed, and creates the tables when the app starts. If the user runs the file directly, it starts the development server.

**models.py**
Defines the database schema. It contains the SQLAlchemy models for users, songs, tags, listening events, ratings, playlists, and notifications, plus the association tables that handle friendships, song tags, and playlist entries. It also includes each model’s to_dict() method, which turns database rows into JSON-friendly data for the API.

**seed_data.py**
Is the data bootstrap script. It drops and recreates the database, then inserts realistic sample data: users, friendships, tags, songs, listening events, playlists, and a notification. The seeded listening events are especially useful because they drive the feed and streak behavior when the user tests the app.

## Data Flow

When a user shares a song, the app stores that relationship on the Song record through the shared_by field in models.py. The shared song is then visible to other users through the normal song and playlist routes. The notification is not created at share time; it is created later when another user interacts with that shared song by adding it to a playlist. In routes/playlists.py, the playlist route calls services.notification_service.add_to_playlist(), which first loads the Song, User, and Playlist records, appends the song to the playlist, commits the change, and then checks whether the person adding the song is different from song.shared_by. If so, create_notification() inserts a new Notification row for the original sharer with the message that their song was added to a playlist.

## Patterns

One pattern is the route/service split. The route files stay thin: they read request data, validate required fields, call a service function, and convert the result into JSON. The service files do the actual database work, like creating playlists, recording listening events, or creating notifications. A second pattern is consistent serialization with to_dict(). The models define to_dict() methods, and the services and routes reuse them when returning users, songs, playlists, listening events, and notifications. That keeps the API responses consistent and avoids duplicating formatting logic. Another repeated pattern is centralized error handling through ValueError. Service functions raise ValueError when a record is missing or input is invalid, and the route handlers catch those errors and return a JSON error message with the right HTTP status code. Finally, the last pattern is the use of db.session.add() followed by db.session.commit() whenever the app changes data. That pattern appears in playlist creation, listening event recording, notification creation, and the seed script, which makes writes easy to follow and keeps the database changes explicit.

# Fixes - Root Cause Analysis Format

## Analysis 1

**1. Issue number and title**
Issue #1 — My listening streak keeps resetting

**2. How you reproduced it**
How I reproduced it: I reproduced it by first sending in the "/users/<user_id>/streak" using Kenji's generated user_id, at the end of the url. But then I realized I had to set my program to Sunday. To do this, I wrote a test that mocks datetime.now(timezone.utc) to a sunday with Kenji listening to a song the previous day, and repopulated my data with it. Afterwards, I used POSTMAN to first send a POST request that would set the day to Sunday with: POST http://127.0.0.1:5000/songs/<user_id>/listen; with a header of Content-Type: application/json and the raw body JSON of Kenji's user_id. After this, I then reran the GET request as the state of the program was on a Sunday.

**3.How you found the root cause**
Right away, I narrowed down the files needed to look at the be streak_service.py, this would make the most sense to check first because this is an issue with the streak. If the bug was not in the file, I would then start to look more into the users routes to see if it could be a data collection issue with each user. My navigation path was first check the streak_service, then check record_listening_event(), then check update_listening_streak(). I am confident I found it in the right place due to how the streak updates, a suspicious line being in a function that would make a lot of sense for a bug like this sets my confident high.

**4. The root cause**.
What was wrong was that there is an else statement in update_listening_streak() for when the streak count goes up one. This line is: elif days_since_last == 1 and today.weekday() != 6. There is one key flaw to this condition, and that is that on Python's weeday scale, Sunday is set to value 6 as an int. Because of this, if the user had listened the day before, passing the first condition, and the current day is sunday, the loop would go to the default result, which is the streak being set back to 1.

**5. Your fix and side-effect check**
My fix was the remove the second condition on this else check all together. If the feature of streak is to go up once a day regardless of the day, then "days_since_last == 1" as a condition will suffice for the result of increasing the streak count as one. "today.weekday() != 6" is a harmful condition and is completely unnecessary for this code to run. After this was done, I reran the test that would set the program to a state of being sunday WITH kenji listening to a song on saturday, before retesting in postman with both my POST and GET requests, getting the intended result, a streak that was the number it was meant to be and not 1.

## Analysis 2

**1. Issue number and title**
Issue #2 — Friends Listening Now shows people from yesterday

**2. How you reproduced it**
How I reproduced it: I reproduced it by first updating the seeded data so that Nova is the viewer, darius is the firend whose old listen should still show up, and Darius's listening events had one update to it yesterday at 11pm. Using a command like this: python -c "import sqlite3; from datetime import datetime, timedelta; db=sqlite3.connect(r'.../project5-mixtape-starter/instance/mixtape.db'); cur=db.cursor(); cur.execute(\"update listening_event set listened_at = ? where user_id = ?\", ((datetime.now()-timedelta(days=1)).replace(hour=23, minute=0, second=0, microsecond=0).isoformat(sep=' '), '<user_id>')); db.commit(); print('updated')". I then ran a get request to the url: http://127.0.0.1:5000/feed/<user_id>/listening-now in POSTMAN, and got a JSON return without darius in it.

**3.How you found the root cause**

**4. The root cause**.

**5. Your fix and side-effect check**

## Analysis 3

**1. Issue number and title**
Issue #3 — The same song keeps showing up twice in search

**2. How you reproduced it**
How I reproduced it: I reproduced this bug by first running a GET request with postman on ttp://127.0.0.1:5000/playlists/<darius_user_id>/songs and it showed 6 songs when it was meant to show 7. To run a quick fix, I tried making a post request with link to darius's songs and raw JSOn body with the song_id and added_by (darius). When I tried that, I got an error about the content type, so when I tried to run the GET again, there still was only 6 songs.

**3.How you found the root cause**

**4. The root cause**.

**5. Your fix and side-effect check**
