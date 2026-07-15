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

