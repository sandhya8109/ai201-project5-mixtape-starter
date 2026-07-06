# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude throughout this project for codebase navigation, hypothesis testing, and debugging discipline — not for blind code generation.

**Orientation (Milestone 1):** I pasted `models.py` and asked for help understanding the data model and relationships before touching any service code. Claude pointed out that `User.listening_streak` is a stored counter rather than something derived from `ListeningEvent` history, and that `playlist_entries` has an explicit `position` column — both of these turned out to be directly relevant to the bugs I later fixed.

**Issue #1 (streak resetting):** Claude spotted the suspicious `today.weekday() != 6` condition in `update_listening_streak()` by reading the code, but I verified it myself in a script before accepting it as the root cause — confirming `datetime.weekday()` returns 6 for Sunday, then running the actual function with a simulated user to confirm the streak reset to 1 instead of incrementing. The diagnosis held up under testing.

**Issue #3 (duplicate search results) — where AI got it wrong:** Claude's first hypothesis was that the `outerjoin` to `song_tags` in `search_songs()` caused row fan-out for songs with multiple tags. I tested this directly — raw SQL confirmed the join really does produce duplicate rows at the database level (3 rows for a 3-tag song), but the actual `search_songs()` function still returned only 1 result. The hypothesis was wrong; something in SQLAlchemy's ORM layer was already deduplicating the rows. I never found the real trigger for this bug and did not fix it — I'm noting this as an honest account of an AI-suggested theory that didn't hold up under verification, and moved on to a different issue given time constraints.

**Issue #2 (feed showing stale listeners):** Claude's first two theories (rolling-window vs calendar-day confusion, then a naive/aware datetime mismatch in the DB) were also tested and ruled out with real data before we found the actual cause — the `RECENT_THRESHOLD = timedelta(hours=24)` constant was simply too large for a feature meant to represent "listening right now." I verified this by testing against real seeded data (a friend with a 2+ hour old event incorrectly showing as "listening now") and confirmed the fix using both a too-old case (correctly excluded) and a genuinely recent case (correctly included).

**Issue #5 (last playlist song missing):** This one was found almost entirely through the existing test suite rather than AI code-reading — `pytest tests/test_playlists.py` was already failing with a comment in the test itself (`# Bug causes this to return 4`). Claude read `get_playlist_songs()` and spotted the `songs[:-1]` slice immediately. I verified by running the failing test before the fix and the passing test after.

**Where I overrode/verified rather than trusted AI:** In every case, I ran the actual code (via REPL scripts or pytest) before accepting a diagnosis, and two of three initial hypotheses (Issue #3's join theory, and an early Issue #2 theory about naive/aware datetime comparison) were disproven by testing and had to be discarded.

---

## Codebase Map

**app.py** — Flask app factory and DB setup via `create_app()`.

**models.py** — SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`. Notable design choices: `User.listening_streak` and `last_listened_at` are stored counters, not derived from `ListeningEvent` history each time. `playlist_entries` (the Playlist↔Song join table) has an explicit `position` column, so playlist order is intentional rather than insertion-order. `Notification` is generic — a `notification_type` string plus free-text `body`, with no polymorphic subtype per notification kind.

**routes/** — `songs.py`, `playlists.py`, `users.py`, `feed.py`. Every route does input parsing and response formatting only; all business logic is delegated immediately to a function in `services/`.

**services/** — `streak_service.py`, `feed_service.py`, `search_service.py`, `notification_service.py`, `playlist_service.py`. This is where all actual logic lives, and where all five bugs were reported to be.

**Data flow — a user listens to a song:**
`POST /songs/<id>/listen` (`routes/songs.py`) → `streak_service.record_listening_event()` → creates a `ListeningEvent` row, then calls `update_listening_streak()`, which compares `now` against `User.last_listened_at` to decide whether to increment, hold, or reset `listening_streak`.

**Pattern noticed:** Routes are thin — they only handle request parsing, calling one service function, and formatting the JSON response. All actual logic and all five bugs live in the `services/` layer, exactly as the README states.

---

## Root Cause Analysis

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** Isolated `update_listening_streak()` in a script rather than going through the HTTP API. Built a fake user object with `last_listened_at` set to Saturday, July 4, 2026, and called the function with `now` set to Sunday, July 5, 2026 — exactly one day later, which per the function's own documented rules should increment the streak. Instead the streak was set to 1.

**How I found the root cause:** Traced `GET /<user_id>/streak` in `routes/users.py` to `streak_service.get_streak()`, then to the actual mutation logic in `update_listening_streak()`. The docstring states the only reset condition is "more than one day has passed," but the code's `elif` branch was `days_since_last == 1 and today.weekday() != 6`. I confirmed in a separate check that `datetime.weekday()` returns 6 for Sunday, proving this condition silently excludes Sundays from the "one day passed" increment path.

**The root cause:** The increment branch required both `days_since_last == 1` and `today.weekday() != 6`. On any Sunday, the second condition evaluates to `False`, so a user who listened on consecutive days still fell into the `else` branch and had their streak reset to 1 instead of incremented. There's no rule in the docstring justifying a Sunday exclusion — it's an unintended artifact.

**My fix and side-effect check:** Removed `and today.weekday() != 6`, leaving `elif days_since_last == 1:`. Re-ran the reproduction script and confirmed the streak now correctly went to 6. Also tested the same-day case (streak unchanged, as expected) and the skipped-day case (streak reset to 1, as expected) to confirm neither of those branches was affected. Ran `pytest tests/test_streaks.py` — all 5 tests passed.

---

### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it:** Queried the real seeded database directly. User `kenji`'s only friend, `aaliya`, had a `ListeningEvent` from roughly 2+ hours earlier (per `seed_data.py`, explicitly commented as an event that "should NOT appear in listening now after fix"). Calling `get_friends_listening_now(kenji_id)` returned that event anyway.

**How I found the root cause:** Read `feed_service.py`'s `get_friends_listening_now()`. Two earlier theories were tested and ruled out: (1) that the 24-hour rolling window was somehow a calendar-day boundary bug — disproven, since the code correctly computes a rolling window, not a calendar day; (2) that a naive-vs-timezone-aware datetime mismatch broke the `>=` comparison — disproven by directly testing the filter with a manually constructed stale event and a recent event, which the query correctly separated. The actual issue was simpler: `RECENT_THRESHOLD = timedelta(hours=24)` is just too large a window for a feature meant to represent someone listening *right now*. The seed data itself defines "recent" as events within the past 30 minutes, in contrast to "older" events used to trigger this bug.

**The root cause:** `RECENT_THRESHOLD` was set to 24 hours, meaning any friend who listened at any point in the last full day was shown as "listening now" — which reads to users as showing people from a day ago, not genuinely current activity.

**My fix and side-effect check:** Changed `RECENT_THRESHOLD` to `timedelta(minutes=30)`. Verified against real data: `kenji`'s feed correctly excluded `aaliya` (2+ hour old event) after the fix, and `nova`'s feed (whose friends `darius`, `simone`, and `kenji` each had events from 10–20 minutes prior, per fresh seed data) still correctly included all three. Ran the full test suite — 13 passed (2 pre-existing failures in `test_playlists.py` were unrelated, tied to Issue #5, not caused by this change).

---

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Ran the existing test suite before making any changes. `pytest tests/test_playlists.py` failed on `test_playlist_returns_all_songs` (returned 4 songs instead of 5) and `test_playlist_returns_songs_in_order` (missing "Track 5" at the end).

**How I found the root cause:** Read `get_playlist_songs()` in `services/playlist_service.py` top to bottom. The SQL query correctly joins `Song` to `playlist_entries` and orders by `position` ascending — the ordering logic itself is correct. The bug is in the return statement: `return [song.to_dict() for song in songs[:-1]]`.

**The root cause:** `songs[:-1]` slices off the last element of any non-empty list. Since the query already returns songs in correct order, this always drops whichever song is positioned last in the playlist, regardless of playlist length. The function's own docstring ("This function returns all songs in the playlist") directly contradicts what the code does.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs`. Re-ran `pytest tests/test_playlists.py` — all 3 tests passed, including `test_empty_playlist_returns_empty_list`, which had passed even before the fix (an empty list sliced with `[:-1]` is still empty, so that test never exercised the bug). Ran the full suite — all 13 tests passed, confirming no regressions in search or streak logic.

**Regression test note:** `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` in `tests/test_playlists.py` already serve as regression tests for this bug — both would fail if the `[:-1]` slice were reintroduced.

---

## Issue #3 — Investigation Notes (not fixed)

I attempted Issue #3 (duplicate search results) but could not find the actual trigger within the time available. I tested the hypothesis that the `outerjoin` to `song_tags` in `search_songs()` caused duplicate rows for songs with multiple tags. Raw SQL confirmed the join produces genuine duplicate rows at the database level, but `search_songs()` itself returned only one result for the same song, meaning SQLAlchemy's ORM layer was already deduplicating before the response was built. I could not identify the actual second code path responsible for the reported duplication and moved on to fix a different issue given the three-bug minimum requirement.