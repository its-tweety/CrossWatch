# Floppy provider notes

Technical notes on two integration issues between CrossWatch's Floppy module and
the Floppy API, found while running a SIMKL ↔ Floppy sync setup.

- **Issue A (fixed in this fork)**: rating sync creates duplicate consumption
  entries on Floppy.
- **Issue B (documented, not fixed)**: anime items synced to Floppy land in the
  TV Shows library instead of the Anime library.

---

## Issue A — Duplicate consumption entries on rating sync (FIXED)

### Symptom

For every rated movie or TV show synced from another provider (e.g. SIMKL) to
Floppy, Floppy ends up with two consumption rows for the same item:

1. one with `end_date` set and status `completed` (written by the history
   feature), and
2. one **without a date**, status `planning`, carrying the score — written by
   the ratings feature.

The second row is spurious. It pollutes history views and cannot be removed
safely by the user: deleting it drops the rating, and the next sync recreates
it (the write is idempotent per rating value, so the duplicate comes back on
the next diff that sees the score missing).

### Root cause

The Floppy ratings writer used `track_media()`:

```python
# providers/sync/floppy/_common.py — track_media()
try:
    api_post(adapter, f"media/{media_type}", json=body)   # POST /media/{type}
    return
except FloppyAuthError as exc:
    if getattr(exc, "status_code", None) not in {400, 409}:
        raise
    # ... fall back to PATCH
```

The strategy assumes `POST /media/{type}` returns a conflict (400/409) when the
item is already tracked, and falls back to `PATCH` in that case.

Floppy's `POST /api/v1/media/{type}/` (`MediaListView.post`) does not behave
that way for provider-sourced items: it is **append-oriented by design**. When
the referenced `Item` already exists, it does not conflict — it creates a **new
consumption row** (the media model instance) for the same item and returns 201.
The 409 path exists only for `source=manual` (an `IntegrityError` on the
manual form). The endpoint docstring states this explicitly:

> "This append-oriented endpoint keeps its historical default: omitted status
> means Planning. Clients updating an existing play should first read its
> consumption_id and use the exact history route instead."

Consequences for the ratings feature:

- The POST always succeeds (201) for already-tracked media, so the PATCH
  fallback never triggers.
- Every rating write on an already-tracked item appends a new consumption row
  with `{"status": planning, "score": <rating>}` — the spurious second entry.
- The history feature is unaffected only because it legitimately appends a
  dated consumption when the item is first tracked, and subsequent runs skip
  items whose history already matches.

Note the asymmetry that was already present in the module: the *clear* path
(removing a rating) used the correct primitive all along —
`api_patch(adapter, f"media/{typ}/tmdb/{tmdb_id}", json={"score": None})` —
while the *set* path did not.

### The fix

`providers/sync/floppy/_ratings.py`, `_write()`:

```python
try:
    if clear:
        api_patch(adapter, f"media/{typ}/tmdb/{tmdb_id}", json={"score": None})
    else:
        # PATCH the score on the already-tracked media row. Floppy's
        # POST /media/{type} is append-oriented: for provider-sourced
        # items it always creates a new consumption row (201, never
        # 409), so a POST-first strategy duplicates consumption entries.
        # Fall back to the creating POST only when the media is not
        # tracked yet (404).
        try:
            api_patch(adapter, f"media/{typ}/tmdb/{tmdb_id}", json={"score": rating})
        except FloppyAuthError as exc:
            if getattr(exc, "status_code", None) != 404:
                raise
            track_media(adapter, typ, tmdb_id, payload={"status": PLANNING, "score": rating}, patch_payload={"score": rating})
except Exception as exc:
    ...
```

Behavior matrix:

| Media state on Floppy | Action taken |
|---|---|
| Already tracked | `PATCH /media/{type}/tmdb/{id}` with `{"score": rating}` — no new consumption row |
| Not tracked (`404` on PATCH) | `POST /media/{type}` with `{"status": planning, "score": rating}` — creates the item and its first consumption row, as before |
| Any other PATCH error | Raised, item marked unresolved with the HTTP reason — no behavior change |

The 404 detection is reliable because `api_request()`
(`providers/sync/floppy/_common.py`) raises `FloppyAuthError` with a
`status_code` attribute for every non-2xx response; the same pattern is already
used in `_watchlist.py` (404 → `ensure_media` retry).

### Tests updated

`tests/test_floppy_sync.py`:

- `test_floppy_ratings_read_and_write_native_scale` — now asserts the write is
  a single `PATCH` with `{"score": ...}`.
- `test_floppy_ratings_fallback_patch_existing_and_skip_unsupported_scopes` →
  renamed `test_floppy_ratings_patch_tracked_media_and_skip_unsupported_scopes`
  (the "POST fails → PATCH fallback" scenario no longer exists; the test now
  asserts PATCH-first).
- `test_floppy_ratings_create_tracked_items` — now asserts
  `["PATCH", "POST"]`: the PATCH 404s (untracked), then the POST creates the
  item.
- `test_floppy_ratings_shadow_covers_read_after_write_lag` and
  `test_floppy_ratings_remove_clears_score_with_null` — unchanged; still pass.

### Cleaning up existing duplicates

The fix prevents new duplicates. Rows already created before the fix can be
removed from Floppy **without** re-triggering the loop: delete the undated
consumption entry, then run a ratings sync — with the fix, the score is
re-applied via PATCH onto the surviving (dated) row, so the duplicate does not
return.

---

## Issue B — Anime land in TV Shows instead of the Anime library (KNOWN LIMITATION)

### Symptom

Anime watched or watchlisted on an anime-native provider (SIMKL) syncs to
Floppy correctly at the data level (right TMDb entry, right episodes, right
ratings), but the item appears in Floppy's **TV Shows** library rather than its
**Anime** library.

### How Floppy models anime ("grouped anime")

Floppy stores TV-shaped anime as a regular `Item` with
`media_type="tv"` plus `library_media_type="anime"` on the parent, seasons and
episodes. The Anime library is therefore a *bucket* over ordinary TV rows, not
a separate media type. The bucket is decided **once, when a show is first
tracked**, and routing is sticky afterwards. The shape depends on the user's
Anime Provider setting (TMDB/TVDB → grouped TV-shaped rows; MyAnimeList → flat
per-cour rows).

Classification is fail-closed (`docs/grouped_anime.md` in the Floppy repo): a
title is routed to grouped anime only when (1) TMDB reports the `Animation`
genre **and** (2) an exact TMDB/TVDB/IMDb id resolves in the pinned Kometa
Anime-IDs mapping with a MAL identity.

Crucially, this classifier runs on **webhooks (Plex/Jellyfin/Emby/Kodi/Stremio)
and internal importers only** (Trakt/Plex/Stremio/Simkl import). The REST API
path does not classify.

### Why CrossWatch cannot set the bucket today

- The external REST API accepts `library_media_type` only on
  `POST /media/{type}/` (read from the request body, see `MediaListView.post`
  in Floppy's `src/api/views.py`). Bucket-aware resolution exists elsewhere
  (`resolve_item_queryset` / `filter_item_bucket`), but the list-membership,
  season/episode history, watch and playback-progress routes used by
  CrossWatch do not expose the same field consistently, and some write bodies
  are validated against strict schemas.
- CrossWatch's wire format normalizes `anime` → `show`/`movie` for
  TMDb-native providers (`cw_platform/anime_mapping/service.py`,
  `mapped_or_default_media_type`), and the Floppy module declares only
  movies/shows in its manifest. No call site currently knows about
  `library_media_type`.
- Deciding "what is anime" from CrossWatch would mean duplicating Floppy's
  fail-closed classification with a different data source (AniBridge vs
  Kometa Anime-IDs). Divergence between the two would produce inconsistent
  buckets.
- Partial fixes are worse than none: if watchlist wrote the anime bucket but
  history did not (or vice versa), the same show would accrue rows in both
  buckets — the exact dual-library problem Floppy documents and repairs with
  its "Repair duplicated anime libraries" task. Floppy's API also excludes
  the anime bucket from tv/season/episode queries by default
  (`filter_item_bucket`), so a half-bucketed item becomes invisible to one
  side of the sync and risks re-creation as a duplicate.

### Do not work around it manually

Moving an already-synced anime into Floppy's Anime library by hand breaks the
sync: CrossWatch's reads and writes for tv/season/episode exclude the anime
bucket, so the next sync run would not find the item and would re-create it as
a TV row, duplicating the show across both libraries.

### What a proper fix requires

An upstream change, ideally coordinated across both projects:

1. CrossWatch: classify items as anime (it already has this via the anime ID
   mapping) and thread `library_media_type=anime` through **all** Floppy write
   paths (track, lists, season/episode history, playback progress, ratings)
   atomically for parent, seasons and episodes.
2. Floppy: expose bucket-aware writes (or classification on the API path) with
   a documented, schema-validated field.
3. A migration story for items already synced as plain TV rows.

Until then, anime synced to Floppy stays in TV Shows. The data (ids, episodes,
ratings, history) is correct — only the library view differs.

---

## References

CrossWatch:

- `providers/sync/floppy/_ratings.py` — ratings read/write (fixed here)
- `providers/sync/floppy/_common.py` — `track_media()`, `api_request()`,
  `FloppyAuthError` with `status_code`
- `providers/sync/floppy/_watchlist.py` — list membership + `ensure_media`
- `providers/sync/floppy/_history.py` — history writes (unchanged)
- `cw_platform/anime_mapping/service.py` — anime wire-format normalization

Floppy:

- `src/api/views.py` — `MediaListView.post` (append-oriented track endpoint,
  `library_media_type` from body)
- `src/api/helpers.py` — `resolve_item_queryset`, `filter_item_bucket`
  (default-excludes the anime bucket for tv/season/episode)
- `src/api/fork_views_playback.py` — `PlaybackProgressView.put`
- `docs/grouped_anime.md` — grouped anime model, classification policy,
  sticky routing, repair task
