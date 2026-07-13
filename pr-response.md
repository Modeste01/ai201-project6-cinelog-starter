# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used Claude Code to help orient myself in the codebase — specifically to understand how `add_to_collection()` handles deduplication and what patterns the test suite follows. I also used it to stress-test my design arguments for Comments 4 and 5 by asking "what counterargument would a reviewer raise?" For Comment 4, the AI pointed out that public-by-default could expose user intent without explicit consent, which I acknowledged in my tradeoff section. For Comment 5, it raised the point about pagination expectations, which helped me strengthen my argument for date-added ordering. All final reasoning is my own.

<img width="328" height="111" alt="image" src="https://github.com/user-attachments/assets/fc9d72b6-ba55-4ee0-94f0-383ee4a778f5" />

<img width="655" height="322" alt="image" src="https://github.com/user-attachments/assets/a25576c0-9d0f-416a-ab0c-3939e839f666" />

## Comment 1 — Rename

**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated the import and call site in `routes/watchlist/watchlist.py`. I ran a project-wide grep for `save_to_watchlist` to confirm no other references existed.

**How I verified:** `grep -r "save_to_watchlist" .` returned zero results after the change. All tests pass with `pytest tests/ -v`.

## Comment 2 — Deduplication

**What I did:** Added an `AlreadyInWatchlistError` exception class and a deduplication check to `add_to_watchlist()` that queries for an existing entry with the same `user_id` and `film_id` before inserting. This follows the exact same pattern used in `add_to_collection()` with its `AlreadyInCollectionError`.

**How I verified:** Wrote a test (`test_add_to_watchlist_duplicate_raises`) that confirms adding the same film twice raises `AlreadyInWatchlistError` and only one entry exists in the database. All tests pass.

## Comment 3 — Missing test

**What I did:** Created `tests/test_watchlist.py` with a test `test_add_to_watchlist_nonexistent_film_raises` that mirrors `test_add_to_collection_nonexistent_film_raises` from the collection tests. Uses the same fixture pattern (app, sample_user, sample_film) and asserts that passing a nonexistent film UUID raises `FilmNotFoundError`.

**How I verified:** `pytest tests/test_watchlist.py -v` — both watchlist tests pass. Full suite also passes.

## Comment 4 — Default visibility

**My position:** I'm keeping `public=True` as the default.

**Reasoning:** CineLog is a community film tracking app — the core value proposition is discovering what other people want to watch. A watchlist that defaults to private creates a "dark social" problem: the feature exists but produces no community value until users actively opt in to sharing. Most users never change defaults, so a private default means most watchlists are invisible. With `public=True`, the community feed is populated from day one. Users who want privacy can set `public=False` explicitly — they're making a conscious choice, which is actually a stronger signal of intent than passively accepting a default.

**Tradeoff acknowledged:** Public-by-default means a user's "want to watch" intent is visible before they've explicitly chosen to share it. This could feel surprising for users coming from platforms where lists are private by default. However, CineLog's existing collection feature (films you've watched) is also public — there's no privacy toggle on `CollectionEntry` at all. Making watchlists private by default would be inconsistent with the existing model and would set a precedent that new features default to hidden.

## Comment 5 — Sort order

**My position:** I'm switching to `date_added` descending (newest first), matching the collection service pattern.

**Reasoning:** The maintainer's point about consistency is well-taken — `get_collection()` sorts by `date_added.desc()` and users will expect the same mental model across both views. "What did I add recently?" is the more common question for a watchlist than "what's alphabetically first?" Additionally, date-added ordering works well with pagination: new items always appear at the top, so a user checking back doesn't need to re-scan the entire list to find what changed. Alphabetical sorting means a newly added film could land anywhere in the list, making it harder to notice what's new.

**Engagement with reviewer's point:** The reviewer argued for date-added ordering for consistency with the collection. I agree, and I'd add that alphabetical ordering actually breaks a user expectation — when I add "Zodiac" to my watchlist, I expect to see it at the top of my list, not buried at the bottom. The watchlist is a "to-do" list for movies, and to-do lists are naturally ordered by recency. Alphabetical would only make sense if the primary use case were "find a specific film I added" — but that's what search is for.

## Comment 6 — Rebase

**What conflicted:** The `models.py` file conflicted because the main branch refactor (`refactor: migrate film IDs from integer to UUID`) replaced the entire file — changing `Film.id` from `db.Integer` to `db.String(36)` with UUID generation, and updating `CollectionEntry.film_id` to `db.String(36)`. My branch had added `WatchlistEntry` to the old version of `models.py`, which was lost during the rebase. The `.gitignore` also had an add/add conflict since main merged a `.gitignore` PR while my branch had its own.

**How I resolved it:** After the rebase applied main's version of `models.py` (without `WatchlistEntry`), I re-added the `WatchlistEntry` model with `film_id` updated to `db.Column(db.String(36), db.ForeignKey("film.id"))` to match the UUID refactor. I also updated the docstring in `watchlist_service.py` to reflect `film_id (str): UUID of the film` and changed the test's `fake_film_id` from `99999` to a UUID string. For `.gitignore`, I merged both versions keeping all entries.

**How I verified no conflict remains:** `git log --oneline` shows no merge commits. `pytest tests/ -v` passes all 6 tests. `git diff origin/main -- models.py` confirms `WatchlistEntry` uses `String(36)` for `film_id`.

## PR Description

### What this PR does

Adds a watchlist feature to CineLog that lets users save films they want to watch later. The implementation includes:

- `WatchlistEntry` model with UUID-based `film_id` and a `public` visibility flag
- `add_to_watchlist()` service function with film existence validation and duplicate prevention
- `get_watchlist()` service function returning films sorted by date added (newest first)
- REST endpoints: `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add`

### Design decisions

1. **Default visibility: `public=True`** — Matches CineLog's community-first philosophy. Collections are already public; watchlists should follow the same pattern. Users can explicitly opt out.

2. **Sort order: date-added descending** — Consistent with `get_collection()` ordering. Watchlists are "to-do" lists where recency matters more than alphabetical order.

### How to test manually

```bash
# Start the app
python app.py

# Add a film to user's watchlist (use a valid user_id and film_id from your DB)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'

# View user's watchlist
curl http://127.0.0.1:5000/watchlist/<user_id>

# Try adding the same film again (should get deduplication error)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'

# Run the test suite
pytest tests/ -v
```

## Git Log Screenshot

```text
a05b796 docs: add pr-response.md with review responses and design decisions
89509c8 fix: change watchlist sort order to date_added descending for consistency
7565913 fix: update WatchlistEntry film_id to UUID after main branch refactor
b58056e test: add tests for nonexistent film and duplicate in add_to_watchlist
6948e40 fix: add deduplication check to prevent duplicate watchlist entries
d4de645 fix: rename save_to_watchlist to add_to_watchlist per naming convention
ff84191 feat: add watchlist model and endpoint with add_to_watchlist service
```
