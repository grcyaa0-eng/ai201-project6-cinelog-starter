# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude (AI assistant) throughout this project in a few specific ways:
- **Codebase orientation:** Before implementing the deduplication check for
  Comment 2, I reviewed `add_to_collection()`'s existing dedup pattern with
  Claude's help to understand what it checks and what it returns on a
  duplicate, then wrote my own `AlreadyInWatchlistError` class and dedup
  query for the watchlist, matching that pattern.
- **Comment 4 and 5 drafting:** I gave Claude my initial positions and
  reasoning (public-by-default because watchlists spark conversation
  between users, for Comment 4; date-added over alphabetical because
  alphabetical order is arbitrary relative to what's actually relevant to a
  user, for Comment 5) and asked it to help me expand those into fuller
  written arguments. The positions and core reasoning are mine; Claude
  helped with structuring and phrasing the final write-up.
- **Git/terminal troubleshooting:** I used Claude to help debug a rebase
  conflict on `.gitignore` and to sanity-check terminal commands throughout
  the project (renames, running tests, verifying file state).

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` and updated both the import and call site in
`routes/watchlist/watchlist.py`.

**How I verified:** Ran `grep -rn "save_to_watchlist" --exclude-dir=.git .`
to confirm no references to the old name remained anywhere in the codebase,
then ran the full test suite (`pytest tests/ -v`) to confirm nothing broke.

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()`, following
the same pattern as `add_to_collection()` in `services/collection_service.py`:
query `WatchlistEntry` by `user_id` and `film_id`, and if a match exists,
raise a new `AlreadyInWatchlistError` (defined in `watchlist_service.py`,
mirroring the style of `AlreadyInCollectionError`) instead of creating a
duplicate entry.

**How I verified:** Ran the full test suite (`pytest tests/ -v`, 4/5 passing
at this point) and manually tested by starting the app and POSTing the same
film_id to `/watchlist/<user_id>/add` twice, confirming the second call
raised the error instead of creating a duplicate row.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and wrote
`test_add_to_watchlist_nonexistent_film_raises`, modeled directly on
`test_add_to_collection_nonexistent_film_raises` in `test_collection.py` —
same fixture structure (`app`, `sample_user`, `sample_film`) and same
assertion pattern (`pytest.raises(FilmNotFoundError)`).

**How I verified:** Ran `pytest tests/test_watchlist.py -v` (passed), then
the full suite `pytest tests/ -v` (5/5 passing).

## Comment 4 — Default visibility
**My position:** `add_to_watchlist()` should default `public` to `True`.

**Reasoning:** Most users would likely prefer their watchlist activity to
be visible by default. A watchlist naturally sparks conversation between
users — seeing what someone else is planning to watch is a low-friction way
to start a discussion, get a recommendation, or find someone to watch
something with. Defaulting to public keeps that social layer active rather
than requiring users to manually opt in every time, which would mean most
watchlists stay hidden simply because people forget to toggle visibility,
not because they actually want privacy.

**Tradeoff acknowledged:** The cost of defaulting to public is that some
users may not want their viewing intentions visible to others — someone
might be watching something out of a personal interest they're not ready to
share, or simply doesn't want their activity broadcast by default. Defaulting
public means that information is exposed unless a user actively knows to
change it, which puts the burden of privacy on the user rather than the
platform.

## Comment 5 — Sort order
**My position:** Switch `get_watchlist()` to sort by `date_added` (most
recent first), matching the maintainer's suggestion.

**Reasoning:** Alphabetical order is arbitrary with respect to what a user
actually cares about — it sorts by a film's title, which has nothing to do
with why that film is on the list or how relevant it is to the user right
now. Sorting by `date_added` instead surfaces what the user most recently
decided was worth watching, which is a better proxy for relevance than
title happens to be. A film added yesterday because a friend recommended it
is more likely to be "top of mind" than one added months ago that just
happens to start with an early letter of the alphabet — alphabetical order
buries that signal.

**Engagement with reviewer's point:** The maintainer's argument was that
date-added better reflects how users actually think about their watchlist,
and I agree with the underlying reasoning: relevance to the user should
drive the ordering, not an arbitrary property like title. Alphabetical
sort optimizes for something a watchlist doesn't really need — quickly
locating one specific title, the way you'd scan a shelf — over something
it does need, which is surfacing what's current and top of mind. I don't
think alphabetical is without merit in general (it's more scannable if
you're hunting for a specific film), but for a watchlist specifically,
recency is the more useful default.


## Comment 6 — Rebase
**What conflicted:** I fetched and rebased `feature/watchlist` onto the
updated `origin/main` with `git fetch origin` and `git rebase origin/main`.
The rebase hit one conflict: both my branch and `main` had independently
added a `.gitignore` file (an add/add conflict), each with a slightly
different set of ignored paths (my version didn't include `.pytest_cache/`,
which main's version did).

**How I resolved it:** I opened the conflicted `.gitignore`, combined the
entries from both versions rather than choosing one over the other, and
removed the conflict markers, so nothing either branch had intended to
ignore was lost.

**How I verified no conflict remains:** After staging the resolved file
with `git add .gitignore` and running `git rebase --continue`, the rebase
completed successfully. I confirmed with `git status` (clean working tree,
no rebase in progress) and `git log --oneline --graph` that my commits sit
in a straight line on top of `origin/main` with no merge commits introduced
by my rebase. I then ran the full test suite (`pytest tests/ -v`) to
confirm everything still passes.

## PR Description
**What this feature does:**
Adds a watchlist feature to CineLog that lets users save films they intend
to watch later, separate from their collection of films they've already
watched. Users can add a film to their watchlist and view their full
watchlist, with duplicate entries and nonexistent films handled explicitly.

**Design decisions:**
- **Default visibility:** New watchlist entries default to `public=True`.
  Watchlists are social by nature — visibility by default supports
  discovery and conversation between users, with the tradeoff that some
  users may not want their viewing intentions shared unless they actively
  opt out.
- **Sort order:** `get_watchlist()` sorts by `date_added`, most recent
  first, rather than alphabetically. This better reflects what's currently
  relevant to the user, rather than an arbitrary ordering by title.

**How to test manually:**
1. Start the app: `python app.py`
2. Add a film to the watchlist:
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
3. View the watchlist: `curl http://127.0.0.1:5000/watchlist/<user_id>`
   — confirm the most recently added film appears first.
4. Confirm duplicate prevention — repeat step 2 with the same `film_id`,
   confirm it raises `AlreadyInWatchlistError` instead of creating a second
   entry.
5. Confirm nonexistent-film handling — repeat step 2 with a fake `film_id`,
   confirm it raises `FilmNotFoundError`.
6. Run the automated test suite for full confirmation: `pytest tests/ -v`

## git log Screenshot
<!-- Paste your `git log --oneline` screenshot here before submitting. -->

