## Comment 1 — Rename
**What I did:** Checked `services/watchlist_service.py` and confirmed the function is already named `add_to_watchlist()`, matching the project's `verb_to_noun` naming convention (consistent with `add_to_collection()` in `collection_service.py`). No rename was necessary.
**How I verified:** Ran `Get-ChildItem -Recurse -Filter *.py | Select-String -Pattern "save_to_watchlist|add_to_watchlist"` across the project. Found three matches, all referencing `add_to_watchlist` — the definition in `watchlist_service.py` and two references (import + call) in `routes/watchlist/watchlist.py`. No occurrences of the old `save_to_watchlist` name remain anywhere.


## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()` in `services/watchlist_service.py`, mirroring the pattern in `add_to_collection()` from `collection_service.py`. Added a new `AlreadyInWatchlistError` exception class and a query for an existing `WatchlistEntry` with the same `user_id` and `film_id` before creating a new entry.
**How I verified:** Ran `pytest tests/ -v` — all tests pass. [Add here if you also manually tested calling add_to_watchlist twice with the same ids.]

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on `tests/test_collection.py`'s fixture and assertion style. Added `test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises` — asserts that `add_to_watchlist()` raises `FilmNotFoundError` for a nonexistent film_id. Also added `test_add_to_watchlist_duplicate_raises` as a stronger verification of the Comment 2 dedup fix (the manual curl/PowerShell test I attempted earlier hit an unrelated dev-server config error, so this automated test gives real confidence the fix works).
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — both tests pass. Ran full `pytest tests/ -v` — all 6 tests pass with no regressions.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for new watchlist entries.

**Reasoning:** CineLog is explicitly a community film tracking app — the value of the platform comes from users being able to discover what others are watching or planning to watch. If watchlists defaulted to private, that discovery feature would effectively be dead for most users, since defaults are sticky and most people never go looking for a visibility toggle to turn sharing on. Defaulting to public keeps the feature aligned with the app's core purpose without adding friction for the majority of users who are fine sharing their watchlist.

**Tradeoff acknowledged:** A watchlist is different from a completed collection — it's aspirational and unfinished, and some users may not expect or want their "haven't watched yet" list to be visible by default (e.g., it can reveal viewing gaps or upcoming plans they didn't intend to broadcast). Rather than defaulting to private to avoid this, I think the better mitigation is transparency: clearly disclosing at signup or first use that watchlist entries are public by default, so users are making an informed choice rather than discovering it after the fact. A future visibility toggle (allowing users to mark individual entries private) would be a

## Comment 5 — Sort order
**My position:** Agreed with the maintainer — switched `get_watchlist()` to sort by `date_added` descending (most recently added first), instead of alphabetical by title.

**Reasoning:** A watchlist is inherently about "what am I planning to watch," and recency is usually more relevant here than alphabetical order — users are more likely to want to see what they just added rather than scroll to find it. This also brings the watchlist in line with `get_collection()`, which already sorts by `date_added` descending, so the two features now behave consistently.

**Bug found while testing this change:** While writing a test to confirm the new sort order (`test_get_watchlist_returns_newest_first`), I discovered that `WatchlistEntry` had no working `.film` relationship — calling `entry.film.to_dict()` in `get_watchlist()` raised `AttributeError: 'WatchlistEntry' object has no attribute 'film'`. This was a pre-existing bug: `Film` defines `collection_entries = db.relationship("CollectionEntry", backref="film", lazy=True)`, which is what gives `CollectionEntry` objects a working `.film` attribute, but there was no equivalent relationship declared for `WatchlistEntry`. Since `GET /watchlist/<user_id>` calls `get_watchlist()` directly, this bug would have broken that endpoint in production, not just in tests. I fixed it by adding `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to the `Film` model in `models.py`, mirroring the existing pattern for collections.

**How I verified:** Added `test_get_watchlist_returns_newest_first` to `tests/test_watchlist.py`, modeled on `test_get_collection_returns_newest_first`. After fixing the missing relationship, ran `pytest tests/ -v` — all 7 tests pass.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` and `git rebase origin/main`. The only explicit conflict git flagged was in `.gitignore` (both main and my branch added one independently) — resolved by merging both sets of ignore patterns into one file.

However, after the rebase reported success, I manually checked `models.py` rather than assuming it was correct, since the sort-order/relationship work in Comment 5 had already taught me not to trust that things "just work." I found that the rebase had silently dropped the entire `WatchlistEntry` class definition, even though `Film` still referenced it via `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)`. This happened because main's `models.py` never had a `WatchlistEntry` class (it only exists on my branch), and my commit that added the `watchlist_entries` relationship line applied cleanly against main's version without git flagging a conflict — even though the result was structurally broken (a relationship pointing to a nonexistent model).

**How I resolved it:** Rewrote `models.py` to restore the `WatchlistEntry` class, and updated its `film_id` column from `db.Integer` to `db.String(36)` to match the new UUID type on `Film.id` (the actual change called for in this comment). Verified against main's confirmed changes (`Film.id` UUID migration) using `git diff HEAD origin/main -- models.py` before the rebase, so the fix aligns with the intended refactor rather than guessing.

**How I verified no conflict remains:** Ran `Select-String -Path models.py -Pattern "class"` to confirm all four expected model classes (`User`, `Film`, `CollectionEntry`, `WatchlistEntry`) are present. Ran `pytest tests/ -v` — all tests pass, confirming `WatchlistEntry.film_id` as UUID works correctly end-to-end with the rest of the app. Ran `git log --oneline` to confirm the branch history is linear with no merge commits.


![alt text](image.png)

## AI Usage
I used AI throughout this project as a coding assistant, mainly for orientation, verification, and troubleshooting rather than generating my design decisions:

- **Codebase orientation:** Before touching any review comments, I had Claude help me read through `collection_service.py` and `test_collection.py` to understand the existing naming conventions and dedup pattern, which I then mirrored manually in `watchlist_service.py`.
- **Verification over assumption:** For Comment 1 (rename), I initially assumed a rename was needed, but used `grep`/`Select-String` to check first and found `add_to_watchlist` was already used everywhere — so I documented that no change was necessary instead of making an unnecessary edit.
- **Debugging a real bug:** While writing a test for Comment 5 (sort order), I hit an `AttributeError: 'WatchlistEntry' object has no attribute 'film'`. Claude helped me trace this to a missing `db.relationship` on the `Film` model (present for `CollectionEntry` but missing for `WatchlistEntry`), which I then fixed in `models.py`.
- **Catching a silent rebase issue:** After rebasing onto main (Comment 6), git reported success with no conflicts in `models.py`, but I manually checked the file rather than trusting that and discovered the rebase had silently dropped the entire `WatchlistEntry` class while a stale `Film.watchlist_entries` relationship still referenced it. Claude helped me understand why this happened (non-overlapping line changes don't trigger conflict markers) and I rewrote the model correctly, updating `film_id` to UUID to match the refactor.
- **Design decisions (Comments 4 and 5):** I made these decisions myself based on my own read of CineLog's purpose as a community app. I did not ask Claude to write these arguments; I used it only to help me articulate reasoning I had already decided on and to make sure my written response addressed the reviewer's actual point rather than just stating a preference.
- **Git/rebase mechanics:** Claude walked me through interactive rebase syntax (`reword`, `fixup`) and Vim keystrokes when I was cleaning up commit history, since I made a few mistakes with Vim's modal editing along the way (closing the editor without saving changes, `:wq` not applying edits). I eventually used `git commit --amend` directly to fix the one message that wasn't taking effect through the interactive rebase editor.




## PR Description

### What this feature does
Adds a watchlist feature to CineLog, letting users save films they want to watch later. Includes:
- `add_to_watchlist(user_id, film_id)` — adds a film to a user's watchlist, with duplicate prevention (raises `AlreadyInWatchlistError` if the film is already saved)
- `get_watchlist(user_id)` — returns a user's watchlist, sorted by most recently added
- `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add` endpoints

### Design decisions
**Default visibility (`public=True`):** Watchlist entries default to public. CineLog is a community film tracking app, and defaulting to public keeps the discovery/social value intact, since most users never change defaults. Tradeoff: some users may not expect an unfinished/aspirational list to be visible by default — mitigated by clear upfront disclosure rather than defaulting to private. See Comment 4 above for full reasoning.

**Sort order (date added, newest first):** Agreed with reviewer feedback to sort by `date_added` descending rather than alphabetically, matching the existing `get_collection()` pattern and how users naturally think about a "want to watch" list. See Comment 5 above for full reasoning.

### Manual testing
1. Start the app: `python app.py`
2. Add a film to a watchlist: