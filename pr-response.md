# PR Response Doc — CineLog Watchlist Feature

## AI Usage

<!-- Fill in at the end — how you used AI tools during this project -->

AI was used to understand the implementation of functions like add_to_collection and to evaluate design decisions like making watchlists private vs public. I asked about the pros and cons of public vs private watchlists and while the AI suggested to do private, I chose to make it public by default. However, the note that the public version doesn't match with the style of the collection was noteworthy.

## Comment 1 — Rename

**What I did:** I renamed the function save_to_watchlist() to add_to_watchlist() to match the standardized naming convention. I also changed all references to this function accordingly.

**How I verified:** I used the IDE to integrated rename tool and used both AI and the manual search to ensure all references changed accordingly. I also tested fucntionality to ensure nothing broke.

## Comment 2 — Deduplication

**What I did:** I added dedeuplication logic to the add_to_watchlist() and followed the same conventions as the deduplication logic found in the add_to_collection() function. The function raises a AlreadyInWatchlistError if the film is already present.
**How I verified:** I tested by adding two of the same film to the watchlist and checking if there was a duplicate entry.

## Comment 3 — Missing test

**What I did:** I added a new test file for watchlist and added a test to ensure that an error is raised when adding a nonexistent film. This is modeled after the similar function found for collection testing (test_add_to_collection_nonexistent_film_raises).
**How I verified:** I verified this test by running the test and checking the output matched the expected behavior (raising FilmNotFoundError).

## Comment 4 — Default visibility

**My position:** I made the default `public=True` for the watchlist. Watchlist entries should be public by default, and users should opt out to sharing rather than opt in.

**Reasoning:**
Since CineLog is framed as a community app, a public default maximizes discoverability and the social features like seeing what others plan to watch is the main point. Defaulting to private means those features start empty until users opt in.

**Tradeoff acknowledged:**
A watchlist reveals details about a user which they may not necessarily want as public information. Since many users will just keep the default, there are benefits to keeping informatin private unless explicilty allowed by the user. Additionally, keeping `WatchlistEntry` as public is not consistent with `CollectionEntry`, which has no public sharing at all.

## Comment 5 — Sort order

**My position:** I agree that the ordering should be date-added descending order.
**Reasoning:** Most users will likely want to see thing in reverse chronological order so that more recent films show up at the top of their watchlist. However, there is a case of large lists being easier to search through alphabetically. This could be handled on the frontend to keep the behavior consistent.
**Engagement with reviewer's point:** Overall, I agree that the watchlist should not be alphabetical as it is currently and should instead ordered based on date added.

## Comment 6 — Rebase

**What conflicted:** git did not report a textual merge conflict. Main's UUID refactor (`refactor: migrate film IDs from integer to UUID`) rewrote `models.py` — switching all IDs to UUIDs and, in the process, removing the `WatchlistEntry` model. My branch had never committed a change to `models.py` (the watchlist work lived in `routes/` and `services/`), so during the rebase there was no branch-side change to `models.py` to replay. Git silently kept main's version, which meant `WatchlistEntry` was dropped and the app could no longer import it.

**How I resolved it:** I caught the problem because the app failed to start with `ImportError: cannot import name 'WatchlistEntry' from 'models'`. I re-added the `WatchlistEntry` model to `models.py`, but updated `film_id` to a UUID foreign key (`db.String(36)`) so it matches the post-refactor schema used by `Film` and `CollectionEntry`. I also updated the stale integer references the migration exposed: the `add_to_watchlist()` docstring (film_id is now a UUID string) and the watchlist test (now uses a UUID string, `"00000000-0000-0000-0000-000000000000"`, instead of the integer `999999`).

**How I verified no conflict remains:** `git merge-base main feature/watchlist` resolves to the tip of `main`, so my branch is cleanly rebased on top of it, and `git log --merges origin/main..HEAD` returns nothing (no merge commits — history is linear on top of main). `git status` reports the branch in sync with `origin/feature/watchlist` (no ahead/behind) after the force-push. `python -c "from app import create_app; create_app()"` runs clean and registers the watchlist blueprint, and `pytest tests/` passes all 5 tests.

## PR Description

### What this feature does

This PR adds a **watchlist** to CineLog, a per-user list of films someone wants to watch later, separate from the existing collection (which is for films they've already watched). Users can add a film to their watchlist, view everything on it, and the list is kept free of duplicates. It mirrors the shape of the existing collection feature so the codebase stays consistent.

Two endpoints are added under the `/watchlist` prefix:

| Method | Endpoint                   | Description                                                                                                                                                                                                                            |
| ------ | -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET`  | `/watchlist/<user_id>`     | Returns the user's watchlist as a list of films, newest addition first. Each film includes its `date_added` and `public` flag.                                                                                                         |
| `POST` | `/watchlist/<user_id>/add` | Adds a film to the user's watchlist. Body: `{ "film_id": "<uuid>" }`. Returns the created entry with `201`. Returns `400` if `film_id` is missing, `404` if the film doesn't exist, and `409` if the film is already on the watchlist. |

Under the hood this adds a `WatchlistEntry` model (linking a user to a film, with a `date_added` timestamp and a `public` visibility flag) and a `watchlist_service` holding the business logic, following the same layering as the collection feature.

### Design decisions

- **Default visibility: `public=True`.** New watchlist entries are public by default because CineLog is a community app, so the social value (seeing what others plan to watch) only exists if lists are discoverable without an opt-in step. _(Tradeoff noted in Comment 4: this exposes user data by default and is inconsistent with `CollectionEntry`, which has no sharing.)_
- **Sort order: `date_added` descending.** The watchlist returns the most recently added films first, since users are most interested in what they just queued up rather than an alphabetical listing.

### How to test it manually

1. **Get a film UUID:** `GET http://localhost:5000/films/` and copy the `id` of any film from the response. Grab a second film's `id` too, for step 6.
2. **Get a user UUID:** use the `id` of an existing seeded user (e.g. from `GET /collection/<user_id>` responses you already have, or query the `user` table). Call it `<USER>` below.
3. **Add a film:** `POST http://localhost:5000/watchlist/<USER>/add` with JSON body `{ "film_id": "<FILM_UUID>" }`. Expect `201` and a JSON entry echoing `user_id`, `film_id`, `date_added`, and `"public": true`.
4. **View the watchlist:** `GET http://localhost:5000/watchlist/<USER>`. Expect a list containing that film.
5. **Confirm newest-first ordering:** add the second film (repeat step 4 with the other UUID), then `GET /watchlist/<USER>` again and confirm the film you added _most recently_ appears first.
6. **Confirm deduplication:** re-add the first film (repeat step 4 with the same UUID). Expect `409 Conflict` with an "already in watchlist" error, and confirm `GET /watchlist/<USER>` still shows only one copy.
7. **Confirm error handling:** `POST` with a made-up UUID like `{ "film_id": "00000000-0000-0000-0000-000000000000" }` → expect `404`. `POST` with an empty body `{}` → expect `400`.

Example (curl):

```bash
curl http://localhost:5000/films/
curl -X POST http://localhost:5000/watchlist/<USER>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<FILM_UUID>"}'
curl http://localhost:5000/watchlist/<USER>
```

## Git Log

23b1104 (HEAD -> feature/watchlist, origin/feature/watchlist) fix: resolve uuid conflict by modifying all film_id references in watchlist from integers to uuid
dad635b fix: show watchlist in descending date_added order
702a792 test: include test to check raising error for adding non-existent film to watchlist
79f57a0 feat: add deduplication logic to add film to watchlist route
7ff00e3 fix: rename save_to_watchlist() to add_to_watchlist() to follow naming convention
c605e42 fix: update film retrieval method to use db.session.get in collection and watchlist services
