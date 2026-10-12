# Changelog

All notable changes to the V4V Music Link Resolver Pro plugin are documented here.
Format based on [Keep a Changelog](https://keepachangelog.com/); versions follow [Semantic Versioning](https://semver.org/). The standard plugin lives in the sibling repository `ghmattf/v4v-musiclink`.

## Resolver Pro (v4v-musiclink-pro) — [1.20.36] - 2026-10-12

### Fixed

- **Tracks Fountain's search index misses now resolve from the album page** — live-confirmed 2026-10-12 on a real phafe.com share link whose track guid Fountain's `searchContent` action does not index (19 hits unfiltered, 20 with a `["TRACK"]` filter, target guid in neither) even though the track is live on Fountain and appears in the album page's own tracklist. When the track-level search returns a readable "no match" for the track guid (`last_fountain_hits_parsed`, i.e. the action worked and the guid simply isn't in the index — never on a transport failure or stale next-action), `render()` now resolves the album guid (normally a 30-day cache hit from the album-level view), fetches that album's Fountain page with the existing 8 s / no-redirect settings, and reads the track's id straight out of the embedded `\"tracks\"` list via the new `fountain_track_id_from_album_page()` parser (backslash-quote RSC payload, `_id` charset guard, id + guid + `_type` required together). A hit produces the real deep link `https://fountain.fm/track/<id>` and is stored on the pair row exactly like a search hit; every failure (album unresolvable, page fetched without the track, over the per-IP resolve budget) falls through to the existing, never-stored Fountain search fallback. The album-page answer is negative-cached per album|track pair (`fountain_albumpage_pro_v1_*`, definitive-miss TTL when fetched-but-absent, soft TTL on transport failure) and charged to the same `resolve` budget as every other visitor-driven upstream attempt; resolution stays synchronous — the user declined async rendering. `test_stored_links` grows 195 → 213 with render pins S7–S10 (deep-link rendered + stored + link-half-only upsert + Music Index half untouched, missing-track negative-cache with definitive TTL, transport failure leaves the search fallback and sets no album GET, cached album-page miss short-circuits the GET), gated on Pro; `test_fountain_path` grows 34 → 37 with parser pins against an escaped-quote fixture. Pro-only: the standard plugin has no `?album=`/`?track=` intake and never resolves.


## Resolver Pro (v4v-musiclink-pro) — [1.20.35] - 2026-10-11

### Security

- **Per-IP budget for visitor-driven upstream resolution** — the share page's `?album=`/`?track=` query args are visitor-controlled, and every cache miss used to cost an uncapped blocking outbound request: one Music Index GET (8 s ceiling) inside the page render plus, once a title resolved, one Fountain POST — repeatable without limit as an unauthenticated client churned distinct guids (security review 2026-10-06, finding O25; availability/abuse only, no data exposure). Each actual upstream attempt from the page-render path or the og:/document-title computation (`render()`, `compute_preview_fields()`) now spends a per-IP budget of 60/minute (`RESOLVE_RATE_MAX`, one shared `resolve` bucket through the existing `is_rate_limited()`, whose window semantics mean a rejected request never extends the block and no client can spend another's budget). A rejection is deliberately forgettable: no HTTP, no negative-cache entry — nothing is poisoned beyond the window, a retry resolves once the budget frees, and the page falls back exactly like an unresolvable guid (search view / Fountain search URL). Cached serves and remembered misses cost nothing, so a shared link still resolves once and then serves every visitor and og scraper from the cache row with zero upstream calls — viral traffic is unaffected. Exempt by design: the search relay and the log beacons (each already capped by its own budget) and the background Music Index refresh (`run_queued_mi_refresh()` clears the budget flag on entry, so one attempt per stale row per minute keeps running while the site is busy); the Fountain self-heal crawl keeps its own 15-minute throttle behind the same gate. `test_log_limits` grows 93 → 124: 31 `R*` pins covering the exempt path, budget-spent-only-on-real-attempts, no miss-entry poisoning, counter semantics, POST and self-heal-crawl suppression, the background exemption, and both flag setters.


## Resolver Pro (v4v-musiclink-pro) — [1.20.34] - 2026-10-06

### Security

- **Stored-link validation now fails closed for domainless apps** — identical to Resolver 1.22.30, mirrored into Pro's `is_valid_stored_url()`: a registry entry without a `domain` key (or an unknown app slug) now rejects the URL instead of accepting any https host, closing the fail-open branch the same-day security review flagged. The same three pins run against Pro in `test_stored_links`.


## Resolver Pro (v4v-musiclink-pro) — [1.20.33] - 2026-10-06

### Added

- **Per-app icon background colour** — identical to Resolver 1.22.29, mirrored into Pro: the new **Background** column in the Music apps table, `v4v_musiclink_pro_apps[$slug]['icon_bg']` persistence with the "None (transparent)" tickbox winning on save, the render-side `app_icon_bg()` hex guard, and `style="background-color: #hex"` on the `v4v-musiclink-pro-icon` `<img>` — which also shows in the Pro settings preview and the share-page render. README settings bullets updated; the same 13 pins run against Pro in `test_stored_links`.


## Resolver Pro (v4v-musiclink-pro) — [1.20.32] - 2026-10-05

### Changed

- **Fountain failure now lands on a Fountain search for the release** — identical to Resolver 1.22.26, including the modal's extended fallback matcher.


## Resolver Pro (v4v-musiclink-pro) — [1.20.31] - 2026-10-05

### Added

- **Random music app button order** — identical to Resolver 1.22.25, with the option named `v4v_musiclink_pro_random_app_order` and the same client-side, cache-friendly shuffle behaviour.

### Removed

- **Legacy DB-migration code** — identical to Resolver 1.22.25 (Pro keeps its `maybe_create_log_table()` create + alter-in-place dbDelta and all log/row shapes); `mi_take_legacy_positive()` and its four call sites removed alongside the standard plugin's.


## Resolver Pro (v4v-musiclink-pro) — [1.20.30] - 2026-10-05

### Fixed
- **Identical to Resolver 1.22.24 — Fountain buttons deep-link again instead of falling back to `fountain.fm/discover`**: `fountain_autodetect_next_action()` returned only the first near-"search" hex id while Fountain publishes several (rotating them per deploy), so a dead first id was POSTed, produced a 200 with no `hits`, was correctly never persisted, and was re-derived every 15 minutes forever. The Pro mirror now exposes `fountain_autodetect_next_actions()` (every candidate, ordered and de-duplicated) and `fountain_resolve()` probes them in order, persisting the first id that answers with a readable hits payload - never a dead one - while a live id whose payload merely lacks our guid is persisted (and stops the probe) so the crawl does not repeat; `last_fountain_hits_parsed` is reset per attempt so the persist decision always describes the current response. The chunk walk keeps its old cost profile with the same two-phase rule (search-only delta walked in full, homepage's first 20 only when the delta found nothing: 34 s/1.4 MB full-window versus 10 s gated). Verified live: `fountain_live_test.php` **20/20** (0/20 before) against ids that rotated twice during the run, the working id being the second candidate both times; `fountain_fix_test.php` rewritten (104 -> 156 assertions) and the stale `tmp/fountain_probe.php` retired.


## Resolver Pro (v4v-musiclink-pro) — [1.20.29] - 2026-10-04

### Fixed
- **Identical to Resolver 1.22.23 — a Music Index lookup left over from before the cache-table migration is no longer read as a remembered miss, so a release renders its track again instead of falling back to the album.** The `_pro_` keys (`mi_track_pro_v4_…` / `mi_feed_pro_v4_…`) held positives until 1.20.27 and hold only remembered misses since; a positive still live from <= 1.20.26 made presence look like "not found", so lookups returned null, the page rendered the album alone and `queue_mi_refresh()` refused to run. The new `mi_take_legacy_positive()` returns and deletes a legacy payload (non-empty array with `title`) while leaving genuine misses (`array()`) to short-circuit exactly as before, and clears the key before the same presence tests in `queue_mi_refresh()` / `run_queued_mi_refresh()`.


## Resolver Pro (v4v-musiclink-pro) — [1.20.28] - 2026-10-04

### Added
- **Identical to Resolver 1.22.22 — artwork is editable per row in the Cache tab**: the Art cell is a green tick (an `image_url` is stored) or a red cross (none), and opens one shared dialog (preview image, editable Artwork URL, Update/Cancel) posting to `admin_post_v4v_musiclink_pro_update_cache_row` with `link_field=art`; `handle_update_cache_row()` whitelists it against the new `CACHE_EDIT_COLUMNS` (`fountain` → `fountain_url`, `art` → `image_url`), so links keep their https-on-own-domain rule while artwork accepts any http(s) URL or blank to drop it. Art edits stamp `mi_updated_at` and leave `fountain_updated_at` alone, link edits stamp only `fountain_updated_at` as before, and the new `sl_art_saved` / `sl_art_error` notices report the outcome ("Artwork URL updated." / "That artwork URL was not saved: it must be an http(s) URL, or blank to remove the artwork.").

### Changed
- **Identical to Resolver 1.22.22 — the Cache tab's columns and cells**: "MI updated" is renamed **"Metadata Updated" and moved to the first column** (then Artist, Album title, Track title, Album GUID, Track GUID, Art, Fountain.fm link, Fountain updated, Actions — still ten columns); both timestamp columns print the date only; the Fountain.fm link cell no longer prints the URL but shows **[Edit]** plus **[Launch]** (new tab) when a link is stored, or the red cross when not, with **[Remove]** alone left in the Actions column; the inline editor's help text and the Clear-cache docblock lose the stale read-only / "empty - will re-search" wording.
- **Docs: the README's Cache tab bullet describes the tick/cross art status, the date-only timestamps and the artwork/link editing it now supports.**


## Resolver Pro (v4v-musiclink-pro) — [1.20.27] - 2026-10-04

### Added
- **Identical to Resolver 1.22.21 — the stored-links table becomes the unified cache table `{prefix}v4v_musiclink_pro_cache` (schema 1) holding the Music Index half (titles, artist, new `image_url` art, new `mi_updated_at`) and the permanent Fountain half (`fountain_url`, `fountain_updated_at`) on one row keyed on `sha1(album_guid|track_guid)`**, with the admin tab relabeled **Cache**: columns Art, MI updated and Fountain updated added (Fountain URL remains the only editable field), row delete and both Clear scopes unchanged in behaviour, and every symbol moved to cache vocabulary (`maybe_create_cache_table()`, `cache_row_hash()`, `cache_get_row()`, `get_cached_fountain_url()`, `store_fountain_link()`, `render_cache_tab_content()`, `handle_update_cache_row()` / `handle_delete_cache_row()` / `handle_clear_cache()`, actions `admin_post_v4v_musiclink_pro_{update,delete}_cache_row` / `..._clear_cache`, option `v4v_musiclink_pro_cache_table_version`, `tab=cache`). Existing rows copy across on the next load with the resolved link preserved and `mi_updated_at` NULL, so the upgrade costs no blocking fetch, and the old `v4v_musiclink_pro_stored_links` table is dropped only after the copy.
- **The Settings tab gains "Music Index refresh"** (`v4v_musiclink_pro_mi_refresh_hours`, default 12, clamp 1–720, refresh-only — cached data is served at any age and never cleared by the setting).

### Changed
- **Identical to Resolver 1.22.21 — serve-stale Music Index refresh in the background**: stale rows render immediately and queue a pull executed on `shutdown` (no cron, `ignore_user_abort()`, `fastcgi_finish_request()` only when available); an unseen pair still fills blocking on first render; only `render()` may queue a refresh, never `ajax_search`, `ajax_log_open`, `ajax_log_app` or the og: preview (their fills of absent rows keep the old transient-miss traffic); positive Music Index transients are gone, negatives keep the existing TTL policy and keys — including the fix that checks the miss marker for presence instead of truthiness, so a remembered failure is no longer retried on every render.
- **Docs: the README's data-source paragraph now describes the cache table and the configurable serve-stale refresh**, and the Pro features paragraph lists the Cache tab.


## Resolver Pro (v4v-musiclink-pro) — [1.20.26] - 2026-10-01

### Changed
- **Identical to Resolver 1.22.20 — the unresolved Fountain fallback is now `https://fountain.fm/discover` rather than the bare homepage**: the render fallback, the modal's click intercept (with launch logging untouched — a click that only opens the modal still logs nothing, and confirming still logs the launch against the new URL), the settings help text and the surrounding comments all follow. Resolved deep links, stored links and the never-persist-the-fallback rule are unchanged.


## Resolver Pro (v4v-musiclink-pro) — [1.20.25] - 2026-09-26

### Added
- **The Report tab's "Date range" gains "This month" and "Last month"** — calendar months rather than rolling windows, computed in the site's own timezone like every existing preset: from the 1st at 00:00 up to *but not including* the 1st of the following month, so "Last month" covers a whole previous month whatever its length and "This month" ends exactly where the next begins. Both sit in the picker between "Today" and the rolling windows, are in the accepted-range whitelist, and bind into the same `logged_at >= %s` / `logged_at < %s` predicates the other presets use.

### Changed
- **Identical to Resolver 1.22.19 — no App field, one row per release with a per-app link column, and the reordered columns/filters**: the App column and filter are gone (the "Fountain.fm link" column identifies the app), the table is at schema version 2 keyed on `sha1(album_guid|track_guid)` with `app`/`url` replaced by `fountain_url`, existing rows are migrated in place with the resolved link copied across and the hash rebuilt, and the edit form's field name is whitelisted through `LINK_COLUMNS`. Columns and filters both read Artist, Album title, Track title, Album GUID, Track GUID, Fountain.fm link, Updated from one shared list, and the tab description is now "Music apps that do not support direct access via album/track GUID can be saved here."


## Resolver Pro (v4v-musiclink-pro) — [1.20.24] - 2026-09-26

### Changed
- **Identical to Resolver 1.22.18 — the link field is labelled "Fountain.fm link", and the two clear scopes do different things**: the per-app clear blanks only that app's link column (guids, titles, artist and the entry itself survive, `updated_at` is stamped, and the link re-resolves on the release's next view) while "Clear all stored links" still deletes whole rows; the notices, confirmation text and tab description each say which of the two just happened. The edit label, column header and filter label all read "Fountain.fm link".


## Resolver Pro (v4v-musiclink-pro) — [1.20.23] - 2026-09-26

### Added
- **Identical to Resolver 1.22.17 — stored Fountain links, managed from a fourth "Stored links" nav tab**: the Pro plugin now writes and reads `{$wpdb->prefix}v4v_musiclink_pro_stored_links` with the same schema and the same rules (exact album/track guid match used verbatim with no search, no transient, no self-heal and no write-back; miss or blanked link re-runs the search and upserts; album-only and track-level are separate rows; the plain fountain.fm homepage never stored; rows repopulate as pages are viewed), so a previously working link keeps working through a Fountain redeploy regardless of hash or transient state. The tab sits alongside Settings / Log / Reports with the same per-field filters, 50-row pagination, in-place link editing (blank = force a re-search), per-row removal, and the same Clear control offering "Clear all stored links" or one app at a time behind a confirmation. The table is created lazily behind `v4v_musiclink_pro_stored_links_table_version`, from the constructor on every load — alongside the existing log-table guard, for the same overwrite-the-file deploy reason.

### Security
- **Same three nonce- and capability-gated `admin_post` handlers as Resolver 1.22.17**: `check_admin_referer()` → `current_user_can( 'manage_options' )` → a row-scoped `UPDATE`/`DELETE` or an app-scoped `DELETE` (never `TRUNCATE`), with manually entered links required to be https on the app's own registry domain (subdomains allowed, lookalike hosts rejected) and blank accepted as "clear it".


## Resolver Pro (v4v-musiclink-pro) — [1.20.22] - 2026-09-26

### Changed
- **Comment-only docblock correction — no functional change**: `resolve_release_meta()`'s docblock pointed at "the social-preview meta tags below", but those producers (`compute_preview_fields()`, `output_social_preview_meta()`, `render_meta_tags()`) all sit above it, and the two log beacons are consumers too. The docblock now names the real consumers — on-page display, social-preview meta tags, and log beacons — which is why the resolution lives in one place. Matching fix in `v4v-musiclink` 1.22.16.


## Resolver Pro (v4v-musiclink-pro) — [1.20.21] - 2026-09-25

### Added
- **A "Log interactions" setting — enabled by default — now gates all log writing**: the new Settings → V4V MusicLink Pro checkbox (`v4v_musiclink_pro_log_enabled`) is honored by both beacons and by `log_view()` itself. When it is off, `ajax_log_open()`/`ajax_log_app()` answer `success` immediately: **no log row, no rate-limit budget consumed, and no Music Index lookup** (the client is fire-and-forget `sendBeacon`, and a deliberately disabled feature is not a failure), and `log_view()` refuses too as the write-path choke point for any future caller, so retention work can't run either. The nonce still authenticates first — the endpoints' auth contract is unchanged. The option is stored `'1'`/`'0'` and read as **absent = enabled**, so every existing site and every fresh install keeps logging exactly as before until an admin explicitly unticks the box; existing rows remain until Clear log or the row cap prunes them, and the rate/row-cap settings only apply while logging is on.

### Changed
- **App-button favicons now come from each app's own domain — Google's favicon proxy is gone**: identical to Resolver 1.22.15 — every `google.com/s2` URL (which each page view resolved through Google, leaking the visitor's IP and page to a third party) is replaced by the app's own icon path (`stablekraft.app/favicon.ico`, `podtunes.app/favicon.png` — PodTunes serves a PNG and 404s the `.ico`, verified 2026-09-25 — `v4vmusic.com/favicon.ico`, `fountain.fm/favicon.ico`), all four verified live. `favicon_url()` remains the single source for both the rendered buttons and the settings-page icon previews; the og:image fallback (site logo/site icon) and admin-set icon overrides are unaffected.


## Resolver Pro (v4v-musiclink-pro) — [1.20.20] - 2026-09-25

### Changed
- **og:url (both social-preview modes) and the Share button now build from the sanitized canonical URL instead of the raw request URI**: previously all three emitted `home_url( add_query_arg( null, null ) )`, handing every attacker-supplied query arg to crawler-facing tags and shared links (and letting those three URLs disagree with the canonical tag, which was already rebuilt from album/track only). All four producers of this page's URL — the `get_canonical_url` filter, the `rel="canonical"` buffer rewrite, og:url in plugin-only *and* entire-site mode, and the Share button's `data-url` — now agree on `musiclink_canonical_url()`: request path + sanitized `album`/`track` args and nothing else. The helper also now returns the URL **unescaped**, with each output site escaping exactly once at its own echo: its value used to arrive pre-escaped and be `esc_url()`'d a second time by core's `rel_canonical()`/SEO plugins — harmless only because `esc_url()` is entity-idempotent, but the escaping count is now correct by construction instead of by luck.

### Security
- **Guids have a length cap at every intake point** (same contract as 1.22.14): the URL query args read by the render, social-preview, document-title and canonical paths, the shortcode attributes, and both log-beacon POST bodies now drop anything over 256 bytes as if the guid were absent — before any Music Index lookup, outbound app URL, or log write. Deliberately length-only: no charset whitelist, so free-form Music Index ids (e.g. `track_01-they're-right!`) pass through untouched end to end, from URL to render to beacon to log row.


## Resolver Pro (v4v-musiclink-pro) — [1.20.19] - 2026-09-24

### Changed
- **The log now records deliberate actions only - search-result opens and app launches - and never page renders**: renders used to be logged whenever a guid arrived in the URL, so refreshes, feed readers, previews and crawler hits all grew the table while saying little about actual usage - any request could add rows without bound. Page renders are no longer logged from any source (URL query string *or* hardcoded shortcode attribute); instead an "open" is logged at the moment a visitor **picks a result on the search page** - the click that generates the shareable URL - via a new `v4v_musiclink_pro_log_open` beacon (`wp_ajax` + `wp_ajax_nopriv`, `navigator.sendBeacon()` with a `fetch(…, {keepalive:true})` fallback, fired before the navigation). The render that follows, every later view of that URL, and shared links opened cold all deliberately log nothing, so each `app = ''` row now corresponds to someone actively choosing a release. As with launches, only guids travel to the server and titles are resolved from Music Index before anything is written (400 without an album guid, 404 when it no longer resolves), so the client never decides what lands in the log. The beacon reuses the existing `logNonce`/`v4v_musiclink_pro_log_app` nonce action, so already-cached pages keep verifying after upgrade. The reports keep their shape: the three per-release reports count opens (legacy view rows included) via `app = ''`, "Total by Music App" counts launches.

### Security
- **Both log-write beacons are rate-limited per client, and log growth is now bounded**: each beacon endpoint accepts at most **30 requests per minute per IP** by default before answering `429` - with no row written - and the check runs *before* any Music Index lookup, so a flood is never relayed upstream. Each endpoint has its own bucket, so launch floods can't starve search-result opens or vice versa, and the window semantics match the search limiter from 1.20.15: keyed on `REMOTE_ADDR` only (never the spoofable `X-Forwarded-For`), allowed requests reset the 60-second window, rejected requests never advance the counter, so a shared/NAT address can't be locked out permanently. Both budgets are admin-configurable (Settings → V4V MusicLink Pro; 1-600/min, blank/zero falls back to 30, and values written outside the settings UI are re-clamped so logging or the flood guard can't be silently disabled). The table itself is pruned back to its newest **100,000 rows** by default (also admin-configurable; 1,000-1,000,000), checked at most once per hour after a row is written - the size lookup walks the primary key past the cap, so paying it on every insert would be waste; the table may sit modestly above the cap between hourly checks but never grows without bound.


## Resolver Pro (v4v-musiclink-pro) — [1.20.18] - 2026-09-24

### Fixed
- **Fountain.fm buttons link directly to the track again (and one blip can no longer erase them for an hour)**: Fountain's 2026 restructure broke the unofficial search integration in two independent ways. (1) `searchContent` became a *page-bound* action: the plugin's POST to `https://fountain.fm/` - unchanged since the integration was written - now returns an empty `application/json {}` instead of results, while the byte-identical POST to `https://fountain.fm/search` returns the full hits payload (verified live: same `next-action` id, same body, only the path differs), so `fountain_try_resolve()` now targets `/search` with all other headers unchanged. (2) The action's registration id moved off the homepage into chunks only the `/search` page references - all 44 homepage chunks were verified to contain *zero* viable candidates - so `fountain_autodetect_next_action()`'s background self-heal could never find a fresh value after a redeploy; it now seeds `/search` first and scans the search-only chunk delta before the homepage's historic first 20 (the homepage remains as fallback coverage, the 15-minute throttle unchanged). Miss caching now follows the same policy as the Music Index fix in 1.20.16: only **definitive** answers (`404` for a stale next-action - whose retry uses a different cache key and is therefore never blocked - or a readable hits payload without our guid) keep `FAIL_CACHE_TTL`; transport failures, `5xx`, redirects and unreadable `200` bodies are cached for **30 seconds** (`SOFT_FAIL_CACHE_TTL`), so a single network blip can no longer pin the button to the plain fountain.fm link for a full hour. `extract_fountain_hit()` also refuses to treat an unparseable body as a trustworthy "no" any more. The settings page and plugin-header DevTools instructions now point at the POST to `fountain.fm/search` when capturing a next-action value by hand.


## Resolver Pro (v4v-musiclink-pro) — [1.20.17] - 2026-09-24

### Changed
- **v4vmusic.com button now uses guid lookup instead of title search**: v4vmusic.com added public URL-based resolution, so the button now points straight at `https://v4vmusic.com/lookup/{album}` (or `https://v4vmusic.com/lookup/{album}/{track}` when a track guid is present), built purely from the shortcode's guids exactly like the Stablekraft and PodTunes buttons. The button therefore no longer depends on Music Index at all: the previous implementation linked a title search (`/search?q=…&type=songs|albums`) and was *silently omitted* whenever title resolution didn't yield the right field - a release whose track lookup missed but album resolved lost the button entirely - and it now renders whenever the release itself renders.


## Resolver Pro (v4v-musiclink-pro) — [1.20.16] - 2026-09-24

### Fixed
- **An API hiccup no longer masquerades as "Album/Song not found" for an hour**: `mi_api_get()` used to collapse every failure - timeout, DNS error, 5xx, an SSRF-blocked redirect, an undecodable body *and* a true `404` - into one `null`, and both callers then recorded that `null` as a definitive "guid doesn't exist" for `FAIL_CACHE_TTL` (1 hour). One blip therefore poisoned the track and feed transients together, and every render for the next hour answered "Album/Song not found" straight from cache without a single network retry - even after Music Index recovered, and independently of the Pro keyspace, which is why one plugin could keep showing a stale success while the other showed a false failure. The status of each request is now recorded, and only a **definitive** miss (`404`, or a `200` whose payload simply lacks the title) earns the full hour; everything unconfirmed is negative-cached for **30 seconds** (`SOFT_FAIL_CACHE_TTL`) - enough to spare a struggling API a retry storm, little enough to self-heal almost immediately. An undecodable `200` is also demoted to unconfirmed, so a transient proxy error can never be remembered as a missing guid.


## Resolver Pro (v4v-musiclink-pro) — [1.20.15] - 2026-09-24

### Security
- **Search endpoint is rate-limited per client**: `ajax_search` now allows at most 30 requests per minute per IP before answering `429`, closing an amplification vector - the endpoint's nonce is printed on every page carrying the shortcode, so it only proved a request came from *a* page, and nothing previously bounded how many upstream Music Index batches (plus one detail lookup per result) a single client could trigger. The budget is a sliding window held in a transient keyed on `REMOTE_ADDR` only (never `X-Forwarded-For`, which a client can spoof to mint a fresh bucket): each allowed request resets the 60-second counter, while requests over the cap are rejected *without* advancing it, so hammering never extends the block - a quiet client gets its full budget back 60 s after its last allowed request and a shared/NAT address can never be locked out permanently. The guard runs after the empty-query early return, so no-op polls never spend budget, and the UI already surfaces a non-200 as "Search failed", so no script change was needed.
- **Fountain script discovery requires HTTPS**: `fountain_autodetect_next_action()` now rejects any `src` whose scheme is not `https` (previously the scheme was never checked - only the exact-host comparison). A cleartext `http://` response, modifiable in transit, can no longer supply a `next-action` value that would be persisted to the options table via `update_option()`.


## Resolver Pro (v4v-musiclink-pro) — [1.20.14] - 2026-09-23

### Added
- **App button launches are logged**: every click that actually takes the visitor to the app now writes its own row - the same date/artist/album/track a view carries, plus the app that opened, in a new `app` column on `{wp_prefix}v4v_musiclink_pro_log` (added in place by dbDelta, schema version 1 → 2; pre-existing rows default to `app = ''` and so stay views). The page fires a `navigator.sendBeacon()` beacon (falling back to `fetch(…, {keepalive:true})`) the instant the anchor is followed, and Fountain's "unable to generate a link" modal counts **only on OK** - Cancel, Escape and a backdrop click navigate nowhere, so they log nothing; a already-resolved Fountain URL logs on the click itself. The beacon carries the guids (read off `data-album`/`data-track` on the button container, which is what makes shortcode-embedded releases work - they have no query string to read) and the app slug; **titles are resolved server-side before anything is written**, so the client never decides what lands in the log, only registry slugs are accepted (400 otherwise, 404 for guids that no longer resolve), and it is guarded by its own `logNonce`, separate from the search nonce. Launches are logged from every placement, including releases hardcoded into a shortcode attribute - a click is a deliberate action, unlike a render.
- **New report: "Total by Music App"** - launches grouped by app under a "Total Launches" column, with the same artist filter and date-range presets as the existing reports. Slugs are displayed through the registry label, falling back to the slug itself so a row for an app that has since been retired still renders.

### Changed
- **View Log shows both kinds of row**: a new **App** column reads `—` for a page view and the app's name for a launch, and the count line now says "log entries" rather than "logged views" since it lists both. The **Clear log** button clears both kinds.
- **The artist/album/track reports keep counting views only** (`app = ''`): a launch repeats the artist/album/track of the click behind it, so including both would double numbers that have always meant "logged views". The scope of each is now explicit - those three reports = views, Music App = launches.
- **The button container emits `data-album`/`data-track`** (the beacon needs the guids). On a shareable page they were already in the URL; on a shortcode-embedded release they now appear in the page source too, where previously they existed only in the post editor.
- The empty-row `colspan` in the reports table is now computed from the columns actually shown (it was a hardcoded 4, which over-spans the 2-column Artist report).


## Resolver Pro (v4v-musiclink-pro) — [1.20.13] - 2026-09-23

### Added
- **A "Clear log" button on the View Log tab**: the view log could only ever grow - the only way to empty `{wp_prefix}v4v_musiclink_pro_log` was to drop the table by hand. The button now empties it through an `admin-post.php` action guarded by a nonce (so a crafted link or a bare `<img>` hit cannot wipe it) plus a `manage_options` capability check, asks for confirmation via `confirm()` first, and returns to the tab with a "View log cleared." notice. It only appears while the log actually has rows. Rows are deleted with `DELETE` rather than `TRUNCATE` so it also works on restricted WordPress database users.

### Changed
- **Releases hardcoded into a shortcode attribute are no longer logged as views**: `log_view()` fired on *every* successful render, including placements where the guid came from `[musiclink_pro album="…"]` in post content - so a single embedded button wrote a row on each render of that post (page views, feeds, previews and crawler hits alike) and the reports/View Log counted those as usage of the shareable `/musiclink?album=…&track=…` link. Only URL-driven views are logged now; whether a guid came from the URL is decided *before* the two sources are merged, since afterwards the value no longer says where it came from (when both are present the shortcode attribute supplies the render, and the view is skipped). What a row contains is unchanged - date plus the three titles Music Index returns. The raw `album`/`track` guids have never been stored, and no raw request params are written to any log anywhere in this plugin; that guarantee is now stated in the plugin header, the README, and asserted by the test suite.


## Resolver Pro (v4v-musiclink-pro) — [1.20.12] - 2026-09-23

### Added
- **Search results are paginated instead of stopping at 8**: the search box fetched Music Index's default batch, resolved the first 8 hits (`SEARCH_RESULT_LIMIT`) and discarded the rest, so only the top 8 were ever reachable - `q=love` alone has 885 unique results. Results now come 8 per page behind a «‹ Previous · Page N · Results X–Y · Next ›» control (the control hides when there is only one page). Ordering is exactly Music Index's relevance order - nothing is re-sorted, grouped or type-filtered.

### Changed
- **Search hits are accumulated per query behind a transient, and deeper pages chain Music Index's cursor**: `/v1/search` caps `limit` at 100 and exposes no offset or total (`page`/`offset`/`per_page` are silently ignored), so results beyond the first batch are reached by chaining opaque `pagination.cursor` values. The accumulated list only ever *appends*, so a hit keeps a stable index for the cache's lifetime - page boundaries never shift, navigating backwards costs no HTTP call, and detail lookups stay cached. Each request chains at most `SEARCH_MAX_BATCHES` (3) batches, and the cursor is kept so the next request continues from where it stopped. Because the endpoint never reports a total, the UI shows the page and its item range rather than "X of Y".
- **Duplicates are now removed across batches**: Music Index repeats hits (`q=love` returns 1358 rows for 885 unique; 56 rows for 30 unique on `q=Kurtisdrums`), so de-duplication is applied while accumulating - otherwise the same release would appear on two different pages. Keys are identity-based (`entity_type|entity_id|parent album guid`), never positional, so they can't disturb page boundaries. A track hit with no parent `feed_guid` is unresolvable and is dropped at the same point.
- **A no-match query can no longer chain forever**: Music Index returns **zero rows with `has_more: true`** for a query that matches nothing (verified live), so an empty batch now terminates the chain regardless of what the response claims; a cursor that comes back unchanged does the same. Cached 1 hour under `mi_search_hits_v2_`.


## Resolver Pro (v4v-musiclink-pro) — [1.20.11] - 2026-09-23

### Fixed
- **Fountain.fm button 404'd whenever Fountain had filed the release as a podcast**: the URL path was hardcoded to `/track/` or `/album/` purely from whether the shortcode carried a `track=` guid, but Fountain serves the *same* opaque id under four different prefixes (`/track/`, `/album/`, `/episode/`, `/show/`) according to how **it** has categorised the release - so any release Fountain ingested as a podcast rendered its client-side "Oops! That page can't be found" screen (always HTTP 200, so the status code cannot detect it). Fountain's own search response already carries the answer as a `_type` field on every hit, and that field was being parsed then thrown away. The extractor now returns the type alongside the id, and a new `fountain_path_segment()` maps `TRACK→track`, `ALBUM→album`, `EPISODE→episode`, `SHOW→show`, falling back to the old guid-based guess only when Fountain omits a recognised type. No extra HTTP request is made. Example: `?album=69c634ad…&track=016b464c…` now emits `/episode/3xnUxg38mNApJVgv4hXa` and `/show/trSnfDZ8VBGzsh6hjoS4` instead of the 404ing `/track/…` and `/album/…`.
- **Fountain resolution cache key bumped `fountain_id_pro_v4_` → `v5_`**: the cached value is now the whole hit (`id` + `type`) rather than the bare id, so entries written before this fix - which carry no type - are re-resolved once each rather than being served for their remaining 30-day lifetime.


## Resolver Pro (v4v-musiclink-pro) — [1.20.10] - 2026-09-23

### Security
- **Fountain script discovery now checks the exact host**: `fountain_autodetect_next_action()` accepted any script URL that merely *contained* the substrings `fountain.fm` and `_next`, so a page listing e.g. `https://evil.tld/fountain.fm/_next/x.js` would have made the server fetch an attacker-chosen URL. Sources are now normalised first (protocol-relative `//host/…`, root-relative `/…`) and only allowed when the parsed host is exactly `fountain.fm` or a subdomain of it - verified with 12 test cases covering the apex, subdomains, both relative forms, and the four substring/suffix/query bypass tricks.
- **No redirect hops on outbound requests (SSRF hardening)**: all four `wp_remote_*` calls (Music Index API, Fountain search POST, Fountain homepage, Fountain script chunks) now use `redirection => 0` instead of WordPress's default 5, so an open redirect on a remote host can no longer bounce a server-side request to arbitrary - including internal - addresses. Checked against the live hosts first: none of them issue redirects on these paths today. If one ever starts, Fountain degrades gracefully to its plain link and Music Index lookups fail soft into the existing "Album/Song not found" path.

### Changed
- **Canonical URL is now rebuilt from the path plus the sanitised `album`/`track` args only**: `fix_canonical_url()` and the output-buffer rewrite previously echoed the raw `REQUEST_URI` (`add_query_arg( null, null )`), handing every other - attacker-supplied - query arg to filter consumers that don't all escape on output. Both producers of the canonical tag now share one new `musiclink_canonical_url()` helper, so they also can't disagree with each other. *Behaviour change:* `…/musiclink/?album=X&utm_source=Y` now canonicalises to `…/musiclink?album=X` (junk args are dropped).


## Resolver Pro (v4v-musiclink-pro) — [1.20.9] - 2026-09-23

### Security
- **Stored XSS via the document title**: `override_document_title()` returned the release title (external Music Index API data) verbatim from the `pre_get_document_title` filter, and WordPress core echoes that value unescaped in `<title>` - a catalog entry containing `</title><script>…</script>` would close the element early and inject markup into the page. Affects the default "On (this plugin only)" mode; "Entire Site" mode already re-emitted the tag escaped via its output-buffer rewrite. The value is now passed through `esc_html()`.
- **Backreference injection in three `preg_replace()` replacement strings**: the canonical-tag rewrite (whose URL comes from the request's query string), the `<title>` rewrite, and the meta-tag rewriter all interpolated dynamic data into a `preg_replace()` replacement, where `$0` / `$1` / `$2` are expanded as backreferences. A crafted `?x=$2` URL corrupted/broke out of the rewritten canonical tag, and a release title containing `$0` spliced the raw old `<title>`/`<meta>` element back into the page. All three now use `preg_replace_callback()`, which does not parse `$` / `\`; output for benign inputs is byte-identical.


## Resolver Pro (v4v-musiclink-pro) — [1.20.8] - 2026-09-23

### Changed
- **Footer credit is now fully linked**: in "Generated by V4V MusicLink. Powered by MusicIndex.org", **MusicIndex.org hyperlinks to https://musicindex.org**, matching the standard plugin. The translatable string `'Powered by MusicIndex.org'` was split into `'Powered by'` plus a static `<a>` holding the brand name, so the brand is never translated; the existing `.v4v-musiclink-pro-footer a` rule already colours and underlines it, so no CSS change.

### Fixed
- **Footer "V4V MusicLink" link pointed at the visitor's own site**: it built `home_url( remove_query_arg( array( 'album', 'track' ) ) )` — the current page with its query args stripped — instead of the product page. It now links to https://phafe.com/musiclink, identical to the standard plugin. (Behaviour change: Pro no longer links back to the viewer's own musiclink page from the footer.)


## Resolver Pro (v4v-musiclink-pro) — [1.20.7] - 2026-09-23

### Added
- **Version number on the settings page** (Settings → V4V MusicLink Pro): the heading now reads "V4V MusicLink Pro Settings  Version 1.20.7", in small `description` styling next to the title, and appears above the tab bar so it shows on the Settings / View Log / Reports tabs alike. The version is read from the plugin file header at render time via `get_file_data()`, so `bump-version.sh`'s two maintained spots remain the only copies.

### Fixed
- **Settings page heading mislabelled**: it read "V4V MusicLink Settings" (the standard plugin's title) instead of "V4V MusicLink Pro Settings". The sidebar menu title was already correct, so menu and page header now match.


## Resolver Pro (v4v-musiclink-pro) — [1.20.6] - 2026-09-23

### Added
- **Admin-configurable Music apps order** (Settings → V4V MusicLink Pro): a new "Order" column in the Music apps table with ↑/↓ buttons moves each app up or down, and the `[musiclink_pro]` shortcode lists its buttons in that order. Positions are stored per app in the existing option (`v4v_musiclink_pro_apps[slug][order]`) and normalised to unique `0..n-1` on save, so re-saving cannot drift or collide. Options saved before this release carry no order yet and keep the current front-end order — no migration needed.

### Changed
- The **"Release header" row now appears above the "Music apps" row** on the settings page.
- The "Release header" **checkbox is now a Show/Hide drop-down**; the option name and stored values are unchanged (`v4v_musiclink_pro_show_header`), so existing settings keep working. Its label dropped the leading "Show", which is now the drop-down's first option.
- The Music apps description mentions the new Order column, and the file docblock documents both changes.


## Resolver Pro (v4v-musiclink-pro) — [1.20.5] - 2026-09-23

### Added
- **New admin setting "Release header"** (Settings → V4V MusicLink Pro): site-wide checkbox controlling whether the release header (title, artist, artwork) shows above the music app buttons. Default: shown (unchanged behaviour). Option name `v4v_musiclink_pro_show_header`, isolated from the standard plugin's option.
- **New optional shortcode attribute `header="true|false"`**: overrides the admin setting per placement, e.g. `[musiclink_pro album="…" header="false"]` hides the header for that instance. Accepted values: `true`/`false`, `1`/`0`, `yes`/`no`, `on`/`off` (case-insensitive); unrecognized or absent values fall back to the admin setting.
- Plugin docblock documents the new attribute.


## Resolver Pro (v4v-musiclink-pro) — [1.20.4] - 2026-09-23

### Fixed
- **Settings now save correctly**: three form fields posted the standard plugin's option names, which were not whitelisted in the Pro option group, so WordPress silently dropped them on save:
  - Per-app enable/icon fields now post `v4v_musiclink_pro_apps[…]`
  - Fountain next-action field now posts `v4v_musiclink_pro_fountain_next_action`
  - Social preview mode radios now post `v4v_musiclink_pro_social_preview_mode`
- `shortcode_atts()` filter name corrected from `'musiclink'` to `'musiclink_pro'`.

### Changed
- Namespaced all front-end CSS classes and the modal element ID with a `-pro` infix (`v4v-musiclink` → `v4v-musiclink-pro`, `v4v-musiclink-fountain-modal` → `v4v-musiclink-pro-fountain-modal`), so both plugins' shortcodes can appear on the same page without CSS selector or duplicate-ID collisions. **Breaks any custom CSS targeting Pro front-end classes.**
- Removed the temporary Fountain `debug_log()` helper and all its call sites.
- Moved a misplaced docblock onto `render_log_tab_content()`; updated the header comment to reference the View Log / Reports tabs.


## Earlier releases

Pre-changelog releases: Resolver 1.22.0 and earlier, Pro 1.20.3 and earlier. See git tags and the `archive/` subdirectories for those versions.
