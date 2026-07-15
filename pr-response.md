# PR Response Doc, CineLog Watchlist Feature

My responses to the six review comments from @dev-lead. Comments 4 and 5 are design questions, so those are written answers instead of code.

## AI Usage

I used AI in a few small ways. Before I read the comments I had it summarize models.py, the collection service and the collection tests so I understood how the existing code was set up, and I checked what it said against the real code. When I got to the deduplication comment I asked it to walk me through the check in add_to_collection and what it raises on a duplicate, then I wrote my own version. After I drafted my answers for comments 4 and 5 I asked it to argue against me so I could see what I was missing, and I tightened both answers from there. The positions are mine.

## Comment 1, Rename

I renamed save_to_watchlist to add_to_watchlist in the watchlist service and updated the one place it was called in the watchlist route, both the import and the call. The collection service already uses add_to_collection, so this just keeps the naming consistent.

To make sure I caught every reference I opened the search panel in VSCode and searched the whole project for save_to_watchlist. The only results were the definition and the two lines in the route, so once I fixed those there was nothing left. I searched again after to confirm it was gone, and everything came back empty.

I ran the tests after the change and they still passed.

## Comment 2, Deduplication

I used the same approach the collection service uses. Before it creates the entry, add_to_watchlist now looks for an existing entry with the same user and film, and raises if it finds one. I added an AlreadyInWatchlistError class in the watchlist service, the same way AlreadyInCollectionError sits in the collection service. FilmNotFoundError is shared, so I imported it instead of making another copy.

I also updated the route to catch the new error and return a 409, and to return a 404 when the film does not exist. The collection route already does this, and without it a duplicate would come back as a 500.

To check it works I wrote a test that adds the same film twice, expects the error, and confirms only one row exists.

## Comment 3, Missing test

I made a new file, tests/test_watchlist.py, and followed the structure in test_collection.py, the same fixtures and the same in memory database. The one the comment asked for is the watchlist version of test_add_to_collection_nonexistent_film_raises. It passes a film id that does not exist and expects FilmNotFoundError.

I also added a basic happy path test and the duplicate test from comment 2, since they were quick to write and they lock in the behavior I changed. I ran pytest on the new file and then the full suite, seven passing.

## Comment 4, Default visibility

My take is that public should stay the default, as long as there's an easy way to make an entry private.

The thing is CineLog is a community app, and a watchlist is just the films someone wants to watch, so it's more about what they're planning than anything they'd want to hide. The whole reason a community app works is that you can see what the people around you are watching, and if every watchlist started private, none of that shows up until each user goes and flips it on. That basically kills the point of the feature. A watchlist is also less personal than a collection. A collection has your ratings and actual opinions on it, while a watchlist is really just "I might watch this at some point". So making the lighter one public by default sits fine with me.

I do get the other side though. Private by default is the safer call as a rule. Someone might add a film that gives away more than they realized before they clock that the list is public, and leaving people to turn privacy on themselves usually works out worse than having it on from the start. That's why my answer hangs on there being an obvious toggle. If we couldn't give people that, I'd switch the default to private.

## Comment 5, Sort order

I'd sort the watchlist by date added, but oldest first rather than newest first.

I'm with the reviewer that date is better than alphabetical, alphabetical order doesn't really tell you anything on a watchlist. The part I'd do differently is the direction. The collection is a log of what you've already watched, so newest first fits there because what you care about is your recent activity. A watchlist is the other way around, it's a queue of stuff you still want to get to. If I just matched the collection and went newest first, the film I added a minute ago ends up on top and the one I've been meaning to watch for months drops to the bottom, which is backwards for a queue. Oldest first pulls up the films that have been sitting there longest, and that's really what the watchlist is for.

The catch is the two lists wouldn't sort the same way anymore, so someone hitting both can't assume they match. I think that's ok since they're doing different jobs, but if the team would rather keep them consistent, then newest first on both is the fallback.

I've left the code alphabetical for now so the call stays open. If we go this way it's a one line change in get_watchlist, ordering by date_added ascending instead of the film title, plus a test along the lines of the collection sort one.

## Comment 6, Rebase

My branch was opened before the refactor that changed film ids from integers to UUIDs, and that refactor landed on main while my branch was open. I fetched origin and ran git rebase origin/main to move my work on top of it.

The interesting part is that only one file threw an actual conflict, the .gitignore, and it was an add/add conflict because main already added a .gitignore of its own while my branch added one too. Main's version already covered everything mine did plus one extra line, so I skipped my redundant commit and kept main's.

The bigger issue did not show up as a conflict at all. WatchlistEntry was defined back in the very first commit, and main's UUID refactor deleted it from models.py. Since my branch never touched that class again, the rebase treated it as main deleting something I had not changed, so it dropped WatchlistEntry silently with no conflict marker. The app still imported it in the watchlist service, so it would have broken on the next run. I caught it by grepping models.py for the class after the rebase finished. I fixed it by adding WatchlistEntry back with a String(36) UUID film_id so it lines up with the migrated Film, and I updated the leftover int mentions in the service docstring and the route body example to UUID.

To confirm it was clean I checked that git status had no conflicts, searched the project for leftover conflict markers, checked git log for merge commits and found none, and ran the tests again, seven passing.

## PR Description

What it does. This adds a watchlist to CineLog, a list of films a user wants to watch later, kept separate from their collection of films they have already watched. There are two endpoints. GET /watchlist/<user_id> lists a user's watchlist, and POST /watchlist/<user_id>/add adds a film with a body of { "film_id": "<uuid>" }. Add returns 201 on success, 404 if the film does not exist, and 409 if it is already on the watchlist. The service layer has add_to_watchlist with the deduplication check and get_watchlist, following the same conventions as the collection service.

Design decisions. First, watchlist entries default to public, to fit the community side of CineLog, on the condition that there is a clear way to make one private. Second, the watchlist sorts by date added, oldest first, so it reads like a queue, which is different from the collection sorting newest first. The reasoning for both is in comments 4 and 5.

How to test it. There is no endpoint to create films or users since films are seeded, so seed one of each from a shell first, then use the endpoints.

```bash
python - <<'PY'
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username="ada", email="ada@example.com")
    f = Film(title="Blade Runner", year=1982, genre="Sci-Fi")
    db.session.add_all([u, f]); db.session.commit()
    print("USER_ID =", u.id)
    print("FILM_ID =", f.id)
PY

python app.py   # runs at http://127.0.0.1:5000

# add the film, expect 201
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<FILM_ID>"}'

# add it again, expect 409
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<FILM_ID>"}'

# add a film that does not exist, expect 404
curl -s -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'

# view the watchlist, expect the film in a JSON array
curl -s http://127.0.0.1:5000/watchlist/<USER_ID>
```

For the automated tests, run pytest tests/ and you get seven passing, four for the collection and three for the watchlist.

## git log screenshot

