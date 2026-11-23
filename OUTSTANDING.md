# Outstanding items & findings

The single running list of open work for the V4V Music Link Resolver Pro plugin, so any
future session can pick up exactly where the last one stopped.

**How this list is maintained** is codified in `AGENTS.md`
("Outstanding items (running list)"). Short version: anything discussed but
not implemented in the same turn gets added here the same turn, with enough
context to continue without re-deriving it; an item leaves the list once it
is **implemented and released** (`CHANGELOG.md` + git history then carry it)
or **explicitly declined**. Implemented history is never here - it is in
`CHANGELOG.md`.

**Status key:** **Planned** = user picked it, plan below, implement only on
request. **On hold** = user deferred. **Open** = identified, awaiting triage.

Line numbers below predate Resolver 1.22.25 / Pro 1.20.31 (2026-10-05)
unless a note says otherwise; they will drift - re-locate by symbol name.

**Repo split (2026-10-11):** this repository now hosts the Pro plugin only; standard-side items live in `ghmattf/v4v-musiclink`. Closed decisions are kept intact in both repositories for context.
---

## O3. View Log timestamps are shifted (`current_time` vs `get_date_from_gmt`)

**Status: On hold** (user deferred 2026-09-25).

**What:** `log_view()` stores `current_time( 'mysql' )` - site-**local**
time (Pro ~line 324) - but the View Log table prints
`get_date_from_gmt( $row->logged_at, 'Y-m-d H:i:s' )` (~line 1000), which
treats that string as GMT and converts it to local again. On any site not
at UTC the View Log shows times shifted by the site's GMT offset. Reports
are already consistent (`get_report_date_bounds()` ~line 716 computes
bounds with `wp_timezone()` = local, compared against local-stored rows),
so this is display-only.

**Plan sketch:** minimal fix is the View Log cell - print `logged_at`
directly (it is already site-local) instead of `get_date_from_gmt()`. Do
**not** switch storage to GMT: that would misread every existing row and
desync the report bounds. Also check View Log header/notes for wording
about displayed timezones. Probe first if site timezone != UTC (phafe.com
is currently UTC, which hides the bug).
---

## O5. Settings readout: last Fountain attempt

**Status: On hold** (user deferred 2026-09-25).

**What:** Fountain "next-action" self-heal state is invisible to admins.
Today there is only the in-memory `last_fountain_status` property (Pro
~line 186, standard ~line 116), the `fountain_autodetect_throttle(_pro)`
transient (Pro ~line 2757), and the stored `..._fountain_next_action`
option. If the background self-heal keeps failing, nobody can see that it
ran, when, or with what result.

**Plan sketch:** persist an attempt record - option
`v4v_musiclink(_pro)_fountain_last_attempt` = timestamp + HTTP status -
whenever `fountain_autodetect_next_actions()` runs (standard ~line 1030,
Pro ~line 2756); render a read-only line next to the "next-action" field
on each settings page (standard ~line 472, Pro ~line 1508): "Last attempt:
2026-09-25 12:00 - status 404" or "never attempted". Both plugins. A
manual "Test now" button would be a scope extension - only if the user
asks.
---

## O7. Fountain self-heal is slow to recover (or never fires) - diagnosis

**Status: On hold** (user deferred 2026-10-05 - diagnosis kept, nothing
picked; the review notes below are current as of that date).

**What:** why, once self-heal is required, the button takes far too long
to start working again. Causes found this turn in `fountain_resolve()` /
`fountain_autodetect_next_actions()` (standard `fountain_resolve()` 1847,
`fountain_try_resolve()` 1914, `fountain_autodetect_next_actions()` 2049,
`fountain_scan_chunks_for_actions()` 2138; Pro 3569 / 3636 / 3771 / 3860),
roughly ranked:

1. Positive cache keyed on the hash - see O6 (also makes *whether*
   self-heal runs depend on which guid renders first).
2. The 15-minute throttle is armed **before** the attempt, unconditionally
   (standard 2050-2054, Pro 3771-3776), so any outcome -
   no candidates, unreachable seed page, aborted crawl - still costs a
   full 15 minutes of no effort, and consecutive failures stack.
3. The crawl runs synchronously inside the visitor's page render: 2 seed
   pages x 10s timeout + up to 20 delta chunks x 8s (plus up to 20
   fallback chunks x 8s when the delta is empty - the two-phase gate
   shipped with 1.22.24), plus the stale POST (8s) and retry (8s) ~ the
   theoretical worst is still ~6 minutes in one request, but a routine
   self-heal measured **~10s** on 2026-10-05 (the pre-gate always-walk was
   34s measured / ~5.5 min worst). If a
   proxy/PHP/client timeout kills it before `update_option()`, the work
   is discarded while the already-committed throttle transient blocks the
   next 15 minutes.
4. ~~`update_option()` only runs `if ( $result )`, so a *fresh* hash whose
   readable payload lacks this guid is never persisted - indistinguishable
   from "still stale" - producing a detect/fail loop.~~ **Fixed 2026-10-05
   (Resolver 1.22.24 / Pro 1.20.30):** a candidate that answers with a
   readable hits payload is now persisted even when our guid is absent.
5. ~~Deterministic first-candidate short-circuit: if the first
   near-"search" hex isn't the right action, detection returns the same
   wrong value forever until Fountain's chunk order changes.~~ **Fixed
   2026-10-05 (Resolver 1.22.24 / Pro 1.20.30):** detection returns every
   candidate (`fountain_autodetect_next_actions()`) and `fountain_resolve()`
   probes them in order - this was O11, released today.
6. Demand-driven only - no cron. Healing needs a render with a Fountain
   button that also gets a Music Index search text - the attempt is skipped
   entirely otherwise (standard `render()` 1747-1750; MI "not found" is
   negative-cached for 1 hour, `FAIL_CACHE_TTL` std 108), so on a
   quiet site attempts are naturally minutes-to-hours apart.
7. Unverified: site page caching may keep serving plain/stale HTML past
   the plugin's own heal.
8. **2026-10-05 observation (not a regression - baseline behaves the same):**
   `last_fountain_hits_parsed` is reset at the top of `fountain_try_resolve()`,
   so an answer served from the negative cache leaves it false. A release
   Fountain does not index therefore stops the crawl on the render that
   actually POSTed (the new "stored id is live, give up early" branch) but
   re-enters self-heal once that miss is being served from cache - i.e. the
   crawl can still repeat every 15 minutes for such releases. Making the
   negative cache remember *whether* it was parsed (`''` vs a marker) would
   close it; only worth it if the crawl cost ever matters.

**2026-10-05 review (Fountain working again - `fountain_live_test.php`
20/20 on the 1.22.24 / 1.20.30 release):** with causes 4 and 5 fixed and
stored links (1.22.17 / 1.20.23) covering every release that has rendered
once, this item now only concerns the *first* resolution after a redeploy.
Still live and unchanged: 1 (= O6), 2, 3 (typical cost only - see above),
6, 7, 8. Cause 3's timeout-discards-the-work hazard remains the sharpest
one, because the throttle is already committed by then (cause 2). No fix
picked yet - see the triage question below.

**Plan sketch (options identified, none picked):** ~~validate-on-200 and
persist regardless of our guid (1+4)~~ done 2026-10-05;
~~return a candidate list tried in order until one yields a readable hits
payload (5)~~ done 2026-10-05 (that was O11); arm the throttle
after a completed attempt with a 30-60s backoff on failure, keeping 15
minutes only after a conclusive scan (2); time-box the crawl and run it
via `fastcgi_finish_request()`/`shutdown` so a page timeout can't eat the
attempt (3); record every attempt - timestamp, stage reached, status,
candidate - which doubles as the O5 settings readout (6, diagnosis); a
cron-driven heal so recovery doesn't depend on traffic (6); positive
revalidation policy (7, shared with O6's open question).

**Mitigated (not fixed) by the stored-links work:** Resolver 1.22.17 /
Pro 1.20.23 make an already-resolved release immune to everything above -
its button URL now comes from the database, not from the hash, a
transient, or the self-heal - so the observed "homepage for 30+ minutes"
symptom can no longer recur for releases that have rendered once since
the upgrade. This item still matters for the *first* resolution after a
redeploy, which no stored row covers.

**Blocked on:** the user confirming which symptom is observed - plain
`fountain.fm` fallback vs a deep link Fountain rejects.
---

## O9. The live Fountain probes hardcode a rotated `next-action` token, so they now 404

**Status: On hold** (user deferred 2026-10-05 - fixture work not picked;
the plan below is current as of that date).

**What:** `tmp/fountain_candidate_test.php` and `tmp/fountain_case_test.php`
both POST to `https://fountain.fm/search` with a **hardcoded** `next-action`
header - `$SID` = `605af9c39c662ad04c72e52279d6db26f12973e250`
(`fountain_case_test.php` line 13) and a fixed candidate list passed to
`resolve_with()` (`fountain_candidate_test.php`). On the 2026-09-26 release
run both returned HTTP 404 with body `Server action not found.` on *every*
request (re-run once, identical), i.e. Fountain has rotated the action hash
those fixtures were captured against.

**This is a stale fixture, not a plugin bug.** `tmp/fountain_live_test.php`
autodetects the current next-action from the `/search-page` chunks exactly as
the shipped `fountain_autodetect_next_actions()` does, and it passed 20/20
against the same endpoint in the same run - so the production search path is
fine. Neither failing probe loads plugin code for its request either.

**Re-verified 2026-10-05 (Fountain working again - `fountain_live_test.php`
20/20 on the 1.22.24 / 1.20.30 release):** every hardcoded-token file still
404s on every POST, `Server action not found.` -
`fountain_case_test.php` (all 4 requests; its deliberate bogus-token
assertion at line 80 still passes as a 404),
`fountain_candidate_test.php` (both entries of the fixed list: `605af9c3…`
and `40654cc3…`), and a **third** live file this item did not know about:
`fountain_extract_verify.php` (`$SEARCH_ID` line 14, curl POST to
`/search` - 404 on both header variants it tests). The working ids that day
were `60f00dce…`, `60f643a9…`, `40089971…`, `60944205…` - rotation
continued, the fixtures did not. `fountain_search_matrix.php` and
`fountain_shape_matrix.php` also carry the old hex but make no network
calls (offline parsing fixtures) - they need no change.

**Plan sketch:** make the three live probes detect rather than embed - reuse
the chunk scan `fountain_live_test.php` already has (or call the plugin's
`fountain_autodetect_next_actions()` and feed one candidate from its list
into the `next-action` header - since 1.22.24 / 1.20.30 it returns a list,
and the plugin itself probes rather than trusts it);
keep the bogus-token request `fountain_case_test.php` already makes as the
deliberate stale-hash 404 assertion. Fold "the token must be detected, not
captured" into O4's `tests/` move so the fixtures cannot rot silently again.
---

## O10. Fountain now publishes an official API - the plugin still uses the internal `/search` server action

**Status: On hold** (user deferred 2026-09-26, after the unauthenticated
probe below; no code changes made, re-open only on request).

**Scope superseded by O28 (2026-11-21):** the user re-picked the
official-API migration with p20-link-resolver's no-key `/v1/search/lookup`
as the model. O28 absorbs this item when it ships - keep the probe record
below until then. (O28's Pro half is implemented 2026-11-23, pending
release; O10 goes at that release.)

**2026-10-05 note:** nothing is blocked on O10 - the O11 fix shipped
through the internal server-action path instead (probe every detected
candidate instead of trusting the first, Resolver 1.22.24 / Pro 1.20.30),
so the official API stays a "delete this whole redeploy-breaks-us class"
option rather than a dependency. The one remaining unknown is unchanged:
whether a real key returns album/track hits carrying `_guid` (one
authenticated call).

**What:** while reviewing the project against Fountain's Terms of Use
(`https://fountain.fm/terms-of-use`, last updated 2026-09-16) it turned out
Fountain now ships a documented public API that did not exist when this
plugin was written:

- `POST https://api.fountain.fm/v1/search/content` with
  `Authorization: Bearer $FOUNTAIN_API_KEY`, body `{"query":..., "types":["SHOW","EPISODE"]}`
  (docs: `https://fountain.fm/docs/endpoints/search`, markdown via
  `https://fountain.fm/llms.txt`; key/FAQ: `/faq/api-keys`; MCP at
  `https://api.fountain.fm/v1/mcp`).
- The 200 payload is `hits[]` carrying `_id`, `_type`, `_guid` and
  `info.title`/`subtitle`/`publisher` - i.e. **the same fields
  `extract_fountain_hit()` already parses** (standard ~line 1834, Pro ~3548).

The plugin instead POSTs to the **undocumented internal Next.js server
action** `https://fountain.fm/search` with a detected `next-action` hash,
`Origin`/`Referer` of fountain.fm and a spoofed Chrome 151 User-Agent
(`fountain_try_resolve()` standard ~1596, Pro ~3310), plus a self-heal crawl
of 2 seed pages and up to ~40 `/_next/static/*.js` chunks to rediscover that
hash (`fountain_autodetect_next_actions()` standard ~1714, Pro ~3428).

**Compliance read (not legal advice):** no clause of the ToS is plainly
breached - §10 bans "disrupt[ing] or exploit[ing] the platform" but says
nothing about automated access, `robots.txt` is `Allow: /`, the plugin is
unauthenticated (so §11's account/agent authorisation is not engaged), holds
no account to suspend (§3/§14), and request volume is low, cached (30-day
positive / 1-hour negative transients) and throttled (15-minute self-heal).
The honest description is **"not prohibited, but now off the sanctioned
path"**: the spoofed browser UA and the reliance on a reverse-engineered
build hash are the two soft spots (§10's undefined "exploit"; a UA-based
block later would be inconvenient to answer honestly).

**Probe results (2026-09-26, unauthenticated):**
- `POST https://api.fountain.fm/v1/search/content` with no key **and** with a
  bogus `Bearer` both return `403 {"ok":false,"error":{"code":"FORBIDDEN",
  "message":"Not authorized."}}` - a key is required; the two cases are
  indistinguishable. Cloudflare in front, `access-control-allow-origin: *`,
  **no rate-limit headers** on the 403.
- Keys: created from `fountain.fm/studio/projects`, prefix `fountain_`, sent
  as `Authorization: Bearer <key>`, base `https://api.fountain.fm/v1`.
- Cost/volume: FAQ `/faq/api-keys` - *"API and MCP calls are unlimited on
  every plan, so there is no metering on the calls themselves"*; pricing page
  - *"Drafts, searches, and API and MCP calls are free"* (no credits burnt).
  Plans are Lite $25/mo, Pro $50/mo, Custom - so a project sits behind a
  podcaster/artist account, not a free listener account (unconfirmed).
- Scope: `POST /search/content` body is only `query` / `types` / `parent` -
  **no project parameter**, and the docs describe it as "Search across
  Fountain: content, transcripts, and Nostr profiles", so the index read is
  global rather than limited to the caller's connected shows (needs one real
  call to confirm).
- `types` accepts `SHOW|EPISODE|ARTIST|ALBUM|TRACK|CLIP` - i.e. albums and
  tracks, exactly this plugin's domain.
- `ContentHit` = `_id` (`{type}:id:{id}`), `_type` (uppercase), `_guid`
  (**optional** - "Podcast GUID, when the content originates from an RSS
  feed"), `info.title`/`subtitle`/`publisher`/`audio`/`video`/`duration`,
  `links`.
- OAuth alternative for agent flows: `authorization_code` + PKCE (S256),
  public client (`token_endpoint_auth_methods: ["none"]`), endpoints
  `/v1/oauth/authorize` and `/v1/oauth/token`, protected resource `/v1/mcp`.

**Fit with the existing code (from reading, not yet executed):**
- `fountain_path_segment()` (std ~1877) already maps uppercase
  `TRACK`/`ALBUM`/`EPISODE`/`SHOW` - compatible with the official `_type`.
- `extract_fountain_hit()` matches on `_guid` (std ~1834) - but the official
  `_guid` is optional, so a hit without one would never match; needs a
  fallback (match on `info.title`/`subtitle`, or `_id` shape) plus a probe to
  see whether album/track hits actually carry `_guid`.
- The URL is built as `https://fountain.fm/{segment}/{_id}` (std ~1468-1471),
  but official `_id` is `episode:id:abc123` - the `{type}:id:` prefix must be
  stripped before use.

**Plan sketch:**
- ~~Probe the official endpoint~~ done 2026-09-26 (above); the remaining
  unknowns - does a real key return album/track hits with `_guid`, and does
  search return content the caller does not host - need one authenticated
  call, i.e. an actual Fountain project + API key.
- If usable: add an optional "Fountain API key" setting (server-side only,
  never printed into page markup), call the official endpoint from
  `fountain_resolve()`, and map `hits[]` into the same `_guid`/`_id`/`_type`
  shape so `extract_fountain_hit()` and `fountain_path_segment()` stay as-is.
- Keep the current server-action path as an automatic fallback when no key
  is configured, so existing installs don't regress; this also gives a
  second answer to O7 (self-heal churn disappears for keyed installs).
- Both plugins, unchanged per-plugin keyspaces. Remove this item when the
  API path ships (or when the user declines it).
---

## O13. "Flush Music Index data" action on the Cache tab

**Status: On hold** (user deferred 2026-10-03 - declined as part of the Applies to both plugins; also tracked in the sibling repository (`ghmattf/v4v-musiclink` for standard / `ghmattf/v4v-musiclink-pro` for Pro) since the 2026-10-11 repo split.
unified cache-table work of that day, listed at the user's request;
implement only if re-picked).

**What:** a control on the Cache tab to purge only the Music Index half
of rows - blank `image_url`, `album_title`, `track_title`, `artist` and
set `mi_updated_at = NULL` - while keeping `fountain_url` +
`fountain_updated_at`, because under the shipped cache model (Resolver
1.22.21 / Pro 1.20.27) the MI half is disposable cache and the link half
is data. Rows repopulate their MI half on the next view (the blocking
fill used for a pair with no cached MI data).

**Plan sketch:** a fourth scope beside the existing Clear controls
(std/Pro Cache tab), `admin_post` handler modelled on
`handle_clear_cache()`: `check_admin_referer()` ->
`current_user_can( 'manage_options' )` -> one scoped `UPDATE ... WHERE`
(never `TRUNCATE`); confirmation wording distinguishes it from
"Clear all" (which still deletes whole rows, links included); notice
reports the affected row count. Both plugins, full rename applied.
---

## O14. No matching cache row → ignore transient TTLs and refresh live

**Status: Planned** (user requested 2026-10-04; they asked for the design Applies to both plugins; also tracked in the sibling repository (`ghmattf/v4v-musiclink` for standard / `ghmattf/v4v-musiclink-pro` for Pro) since the 2026-10-11 repo split.
answer first - implement only on their go-ahead. Ordering is now settled:
O11 shipped first, on 2026-10-05, so O14 may follow whenever they pick it).

**What:** when `cache_get_row()` returns nothing for the pair, stop
consulting the transient caches and go straight to the live lookup, so a
Clear on the Cache tab means "every release re-resolves now" rather than
"delete the rows and hope".

**Why (the user's hit, 2026-10-04):** Clear removes rows only - the
transients outlive it - so after clearing every row on the site the page
still rendered a deep Fountain link from the 30-day hit transient, and
the tab showed no row for it. Same class of gap for MI: a remembered miss
stops the fetch for up to `FAIL_CACHE_TTL`.

**Sites (std; pro is the mirror at 1373 / 3606 / 3924 / 3947 / 3975 /
3996 / 4028 / 4139):**
- MI: `mi_get_feed()` 2199, `mi_get_track()` 2222 -> `mi_fetch_and_record()`
  2253 -> `mi_fetch_feed()` 2274 / `mi_fetch_track()` 2306. Skip
  `mi_take_legacy_positive()` (2416) and the `get_transient( $negative_key )`
  check (2285 / 2316) when the caller had no row.
- Fountain: `get_cached_fountain_url()` 869 -> `fountain_try_resolve()`
  1884. Skip the `fountain_id_v5_*` read (positive AND, if the strict
  version is chosen, the negative) when there is no row.
- Background refreshes of rows that DO exist (`queue_mi_refresh()` /
  `run_queued_mi_refresh()`) keep honouring negatives - that is not the
  no-row case.

**Open decision:** are remembered *failures* ignored too? TTLs are
`FAIL_CACHE_TTL` = 1 h and `SOFT_FAIL_CACHE_TTL` = 30 s
(`negative_cache_ttl()` 2596, `fountain_negative_cache_ttl()` 1970).
Recommendation put to the user: ignore positives only, keep honouring
failures - otherwise an outage turns every view of every row-less release
into blocking API calls plus a Fountain POST per view, on the unofficial
endpoint we are already careful with.

**Ordering warning (resolved 2026-10-05):** this originally said to do
O11 first, because ignoring the 30-day Fountain hit while self-heal was
broken would have turned the user's one working deep link into
`fountain.fm/discover` with nothing able to re-resolve it. O11 shipped as
Resolver 1.22.24 / Pro 1.20.30, so that hazard is gone and O14 no longer
has to wait on anything.

**Plan sketch:** thread a "no row" flag from `mi_get_*()` /
`get_cached_fountain_url()` into the fetch/resolve helpers; both plugins
(identical after `_pro` normalization); extend `tmp/test_stored_links.php`
with: no row + remembered MI miss => live fetch runs; no row + cached
Fountain hit => POST attempted; row present => both still honoured.
---

## O18. New shortcode parameter accepting a direct fountain.fm link (both versions) - attribute only, never a URL query param

**Status: On hold** (user deferred 2026-10-05 - scope and precedence Applies to both plugins; also tracked in the sibling repository (`ghmattf/v4v-musiclink` for standard / `ghmattf/v4v-musiclink-pro` for Pro) since the 2026-10-11 repo split.
decisions from that day are retained below).

**What:** a `fountain` attribute on BOTH shortcodes (`[musiclink]` and
`[musiclink_pro]`) taking a pre-known direct link, e.g.
`[musiclink album="..." track="..." fountain="https://fountain.fm/<segment>/<id>"]`,
used as-is for that placement - no Server-Action resolve, no self-heal, no
Cache-tab edit needed. **Decided 2026-10-05: the attribute OVERRIDES the
cache row** for that placement; precedence is attribute -> cache row ->
`fountain_resolve()`. (The original request said Pro only; scope widened to
both versions 2026-10-05.)

**Where (both; pro is the mirror):**
- Attribute intake: `shortcode_atts( array( 'album', 'track', 'header' ) )`
  std **1637-1641** / pro **3121-3125** - unknown keys are silently dropped
  by `shortcode_atts`, so adding one is cheap, **but standard also has the
  `$unexpected` whitelist at std 1636**: an attribute name not in its
  `array( 'album', 'track', 'header' )` causes ALL attributes to be
  ignored, so `fountain` must be added to both lists in std or the feature
  silently never applies there.
- Fountain block that consumes it: std **1727-1766** (`$fountain_url`
  default 1728, cache-row hit 1737-1740, resolve 1750, `store_fountain_link()`
  1764) / pro **3208-3248** (default 3210, resolve 3232, store 3246) -
  the attribute check goes in front of the cache-row branch per the
  precedence decision above.
- Buttons render through `esc_url( $link['url'] )` std **2751** / pro **4483**.
- Existing "give me a link by hand" mechanism is the Cache-tab inline
  `fountain_url` edit (`handle_update_cache_row`) - global per release;
  the new attribute is per placement and outranks it (decided).

**Why not a URL query param (the user's abuse concern):** Pro is the only
surface that reads the query string for content (`get_guid_arg()` 612-615,
`$_GET['album']`/`$_GET['track']` feeding 3130-3135 for the shareable
/musiclink page). If a `fountain` URL followed that pattern, **any
visitor** could publish `?fountain=https://evil.example/phish` and the page
would emit that as the button's href - reflected link injection, and with a
full-page cache an attacker's URL can be frozen into cached HTML for every
later visitor. The shortcode-attribute path is author-gated (post content
needs `edit_posts`+), which is the same trust level already trusted with
`album`/`track`/`header`. Recommendation baked into this item: **attribute
only; no `$_GET` twin.** Standard can never grow this surface by accident -
it has no URL-parameter mode at all (README:22) - so the rule binds Pro.
If a URL-driven flavor is ever demanded, it needs
the validation below plus an explicit sign-off that any visitor may set it.

**Validation requirements (attribute value):** parse as URL; scheme must be
`https`; host exactly `fountain.fm` or `*.fountain.fm` - the same trust rule
the chunk scan already uses (std 2087-2090 / pro mirror); reject anything
else (`javascript:`, `http:`, look-alike hosts, userinfo tricks like
`fountain.fm@evil.tld`) by falling back to the normal cache-row/resolve
path rather than rendering it; keep the 256-byte length cap pattern
(`guard_guid()` std 1625 / pro 605-607) for consistency; output stays on
the existing `esc_url()` line (std 2751 / pro 4483).

**Open decisions:**
1. Persistence: should a valid attribute value be written back via
   `store_fountain_link()` (making one placement's override global) or stay
   placement-local (recommended - a per-post attribute silently editing the
   shared cache row would surprise the Cache tab).
2. Does the confirm-first modal need to know about it? Today `isFallback`
   matches only `discover`/`search?`/bare homepage, so a validated deep
   link correctly does not intercept - confirm no change needed.

**Plan sketch:** add `'fountain' => ''` to `shortcode_atts` in both
plugins (std 1637 **and the std 1636 `$unexpected` whitelist** / pro 3121);
new `fountain_direct_url_or_reject()` sanitizer per the rules above,
mirrored; feed it into both Fountain blocks **ahead of the cache-row branch**
(decided precedence: attribute -> cache row -> resolve); leave
`fountain_resolve()` / self-heal / stored-links untouched for placements
that don't supply it; unit-test the sanitizer as a host/scheme/length
matrix plus render assertions per plugin (valid attr -> button href, attr
outranks a conflicting cache row, invalid attr -> falls back, attribute
never reaches `store_fountain_link()` unless open decision 1 says so);
README usage sections for both versions gain the attribute.

---

## O28. Fountain official API support (Pro only) - port p20-link-resolver's no-key lookup

**Status: Open** - implemented in this repo 2026-11-23, pending release;
remove this entry when the version ships (`CHANGELOG.md` + git carry it
from then on).

**Scope: Pro only** (user confirmed O28/O29 affect this plugin alone;
the "Applies to both plugins / also tracked in the sibling repository"
tracking from 2026-11-21 is dropped, and the standard-plugin halves
described below - render block std 2316-2356, `get_cached_fountain_url()`
864, its deliberate no-resolve design - are out of scope, untouched).

**What:** move Fountain deep-link resolution onto the documented public API
(`api.fountain.fm/v1`, docs `fountain.fm/docs.md`) as `p20-link-resolver`
already has, instead of the unofficial paths: Pro's Next.js server action +
self-healing `next-action` fingerprint (`fountain_try_resolve()` +
`fountain_autodetect_next_actions()` - breaks on every Fountain redeploy,
the O7/O9/O11 churn), and standard's no-resolve design at all (its button is
only cache row -> `fountain.fm/search?q="title"` -> `discover`; render block
std 2316-2356, deliberate per the comment there).

**The p20 implementation to port (verified live in that repo 2026-11-12/16):**
- `POST api.fountain.fm/v1/search/lookup` - guid -> opaque Fountain id, **no
  API key required** (`fountain_id()` / `fountain_lookup()`, p20 AGENTS
  Gotchas + README Fountain section).
- `GET api.fountain.fm/v1/content/{type}:id:{id}` - reverse: id ->
  `hit._guid` (+ parent guid for episodes).
- Miss discipline: definitive (200/404) vs unreliable (`~` sentinel outside
  the id alphabet) cached separately, 30-day hit TTL, derived URL remembered
  on the cache row (`fountain_url` / `fountain_date`, 30 days) so a fresh
  row serves with no Fountain call at all.
- p20's own OUTSTANDING item #7 records this port as the music repos' half.

**Music-specific unknowns (probe before coding):**
1. Does `/v1/search/lookup` accept Music Index album/track guids? Its `_guid`
   is documented as "when the content originates from an RSS feed" - music
   feeds are RSS so plausibly yes, **unverified for music**. Probe with one
   known album guid + one track guid from Matt's machine (the agent sandbox
   usually cannot reach fountain.fm / api.fountain.fm; p20's AGENTS records
   reachability from Matt's machine on 2026-11-12 - don't assume it holds).
2. If lookup doesn't fit: `POST /v1/search/content` with
   `types:["TRACK","ALBUM"]` - but that endpoint **requires a Bearer key**
   (O10's 2026-09-26 probe: 403 with none/bogus; keys `fountain_...` from
   `fountain.fm/studio/projects`, "API and MCP calls are unlimited on every
   plan" - free to call, but behind a paid plan). Hits carry `_guid`
   (optional) and `_id` = `{type}:id:{id}` - strip the prefix before
   building `https://fountain.fm/{segment}/{id}`
   (`fountain_path_segment()` already maps the uppercase types).

**Where:**
- Pro: `fountain_resolve()` 3666, `fountain_try_resolve()` 3743,
  `fountain_autodetect_next_actions()` 3889, `extract_fountain_hit()` 4028,
  `fountain_path_segment()` 4071, render block 3255-3352,
  `get_cached_fountain_url()` 1313, `store_fountain_link()` 1337.
- Standard: render block 2316-2356, `get_cached_fountain_url()` 864 - no
  resolver exists today (deliberate); this item gives standard an
  API-backed resolver too, per the user's request.

**Plan sketch:** probe unknowns 1-2 -> port Pro's `fountain_resolve()` to
the no-key lookup first, keeping the server-action as automatic fallback
(or drop it if the user prefers at implement time); give standard a
`fountain_id()`-style resolver feeding its render block; precedence stays
cache row (incl. hand-edited; O18 attribute when picked) -> API resolve ->
generic search/discover fallback; only real deep links are ever stored (the
never-store-fallbacks rule at Pro 3300-3304); copy p20's miss-caching
discipline and add a schema constant to invalidate remembered lookups; tests
in `tests/`, README + CHANGELOG; ship independently per repo.

**Implemented 2026-11-23 (Pro half only; standard half never picked):**
- New `const FOUNTAIN_API_SCHEMA = 1` and
  `fountain_api_resolve( $guid, $parent_guid = '' )`, called FIRST from
  `fountain_resolve( $guid, $search_text, $parent_guid = '' )` - render
  passes the album guid as the parent when the target is the track (the
  only context that holds both), so a track goes up as
  `podcast:item:guid:` + `parent` and a feed as `podcast:guid:` alone.
  Tri-state: array hit / `''` definitive miss / `false` unreliable /
  `null` not attempted (over budget - no HTTP, nothing remembered, so the
  `resolve` window rather than the cache bounds a burst, O25).
  Cache `fountain_api_pro_v{schema}_md5(guid|parent)`: hit 30 days,
  definitive `''` at `FAIL_CACHE_TTL`, unreliable under the `~` sentinel
  at `SOFT_FAIL_CACHE_TTL`.
- Legacy body extracted unchanged as `fountain_resolve_legacy()` and kept
  as automatic fallback for every non-hit outcome; a definitive API miss
  promotes `last_fountain_hits_parsed` after the legacy attempt so the
  render's album-page track fallback still fires when the legacy path
  could not parse an answer of its own.
- Probes resolved 2026-11-21 (from Matt's machine): feed guid ->
  `ALBUM 9aSjTQRfSX1vb3dStBWs`, track + parent -> `TRACK
  qkMEo45HgMB0uZ5tUlXE`, track without parent -> 400/empty, unknown guid
  -> 200 with empty `hits` (definitive), reverse needs the `:id:` form
  (`:guid:` -> 400, wrong type -> 500). Music unknown 1 above: answered
  - Music Index guids are accepted.
- Tests: new `tests/fountain_api_test.php` (35 pins: payload shapes,
  cache keys/TTLs, miss discipline, budget gate, fallback + flag
  promotion, `{type}:id:` stripping, id-alphabet refusal);
  `fountain_fix_test` section E now filters to legacy `fountain.fm/search`
  POSTs; `test_stored_links` warms both resolver caches; `fountain_live_test`
  gains an `api`/`legacy` per-case expectation (API cases live-green
  2026-11-23; the legacy case pins the API miss + that the fallback ran,
  and reports fountain.fm's dead server action as a NOTE - O9).

**Relationship to existing items:** absorbs O10 when it ships - remove O10
then; O10 keeps the 2026-09-26 probe record until that point. Also the
long-term fix for O7/O9/O11's redeploy churn (self-heal becomes unnecessary
for resolved releases). Supersedes the 2026-09-26 decline of "Fountain
official-API migration" under Closed decisions - re-requested by the user
2026-11-21 with p20's working no-key precedent.
---

## O29. Paste an app share link in search -> jump straight to that release (Pro only)

**Status: Open** - implemented in this repo 2026-11-23, pending release;
remove this entry when the version ships (`CHANGELOG.md` + git carry it
from then on).

**Scope: Pro only** (user confirmed O28/O29 affect this plugin alone;
the "Applies to both plugins / also tracked in the sibling repository"
tracking from 2026-11-21 is dropped - standard's `rest_search()` 2991 is
untouched).

**What:** today a pasted share link falls into `mi_search_hits()` (Music
Index title search) and finds nothing. Recognise a pasted share URL for one
of the target apps and jump straight to that release instead - the feature
`p20-link-resolver` ships (README "Search UI"; its `ajax_search()` detection
block + `search_direct_link()`).

**Links to recognise (the apps these plugins themselves emit, so reversal is
mostly pure parsing):**
- **StableKraft** `stablekraft.app/album/{album guid}[?track={track guid}]`
  (std 2278-2286 / pro 3207-3215) - guids in the URL, **no HTTP**.
- **PodTunes** `podtunes.app/g/{album guid}[/{track guid}]` (std 2290-2301
  / pro mirror just above its fountain block) - no HTTP.
- **v4vmusic.com** `v4vmusic.com/lookup/{album guid}[/{track guid}]` (std
  2303-2314 / pro mirror) - no HTTP.
- **Fountain** `fountain.fm/{segment}/{id}`: match the cache row's stored
  `fountain_url` first (no HTTP - both plugins carry the column), then
  reverse through `GET api.fountain.fm/v1/content/{type}:id:{id}` ->
  `hit._guid` (p20's `search_fountain_link()` -> `fountain_reverse()`,
  verified live there 2026-11-16). Music unknown: does that reversed RSS
  guid equal the Music Index guid? Same probe as O28 unknown 1.

**Contract to port** (p20 `search_direct_link()`, p20-link-resolver.php
~685, same null / array() / false everywhere):
- **null** = not a share link -> fall through to the normal MI title search
  (plain titles must never route through this - the detection regex returns
  null for non-app input).
- **array()** = recognised but definitively absent -> "no results"; must
  NOT fall through to title-searching the URL text.
- **false** = recognised but unreliable (transport/5xx) -> empty result,
  retried next time.
- direct payload = `array( 'album_guid' => ..., 'track_guid' => ..., ... )`
  so the UI jumps straight to the release, same result shape the existing
  MI hits produce.
- Paste hygiene: schemeless gets `https://` prepended, `www.` stripped,
  query strings ignored; every pasted guid through `guard_guid()`.

**Where:** std `rest_search()` 2991 (detection ahead of `mi_search_hits()`
2865, behind the block/admin search UI); Pro `ajax_search()` 3565 (nonce
already checked; decide placement vs `is_rate_limited( 'search', 30,
MINUTE_IN_SECONDS )` 3586 - pure-parse shapes spend no upstream budget, the
Fountain reverse does, so spend the budget only on upstream-dependent
shapes).

**Caching:** transient per reversed ref with p20's definitive-vs-soft split
(p20: `p20l_frv_*`, `FOUNTAIN_FAIL_TTL` / `SOFT_FAIL_TTL`); add a schema
constant so bumping it invalidates remembered reversals.

**Plan sketch:** detection helpers mirroring p20's split (Fountain first,
then a host switch for the three guid-carrying apps); wire into both search
endpoints; extend `tests/` (pure-parse shapes assert no HTTP; Fountain
reverse mocked; plain-title search unchanged); README + CHANGELOG; ship
independently per repo.

**Implemented 2026-11-23 (Pro half only; standard half never picked):**
- `search_direct_link( $q )` wired into `ajax_search()` right after the
  empty-query check and BEFORE `is_rate_limited( 'search', 30, MINUTE )`:
  `null` falls through to the title search, `array()` returns the empty
  payload (never title-searching the URL text), `false` sends
  `wp_send_json_error( null, 502 )`, a direct payload becomes one result
  with `page = 1` / `has_more = false` - the browser JS already navigates
  on `album_guid`/`track_guid`, so no front-end change.
- Helpers: `search_fountain_link()` (cache-row `fountain_url` match first
  - zero HTTP, then `fountain_reverse()` against
  `GET api.fountain.fm/v1/content/{kind}:id:{id}`), `search_app_link()`
  host switch (`stablekraft.app/album/{a}[?track=]`,
  `podtunes.app/g/{a}[/{t}]`, `v4vmusic.com/lookup/{a}[/{t}]`, scheme
  prepend, `www.` strip, `rawurldecode`, `guard_guid`, `~`-delimited
  regexes - `#` collides with the `#` inside `[^/?#]`), `direct_pair()`,
  `search_result_row()` (row shape extracted from the MI hit loop, shared
  by both paths), `row_has_cached_mi()`, `search_rate_limited()` (per-request
  memo: the single search unit is spent only when an upstream call will
  actually go out, and at most once per request).
- Caching: `fountain_frv_pro_v{FOUNTAIN_API_SCHEMA}_md5(kind|id)` with
  p20's definitive-vs-soft split (`''` at `FAIL_CACHE_TTL`, `~` at
  `SOFT_FAIL_CACHE_TTL`); the shared schema constant invalidates both
  O28 lookups and O29 reversals together.
- Music unknown (reversed RSS guid == Music Index guid): answered live
  2026-11-21 - `parent._guid` / `links: album:guid:` carry the feed guid.
- Tests: new `tests/test_direct_link.php` (55 pins: the four-link shapes,
  decode/guard handling, single-unit charging, cache-row shortcut with
  zero HTTP, definitive vs soft misses, budget 502/429, unusable and
  unrecognised input, untitlable direct result); `test_search_pagination`
  gained the `wp_parse_url()` stub the detection path needs.
---

## Closed decisions (context only - not work items; do not re-open or re-propose)

- **Standard-plugin security review** (2026-10-06): no critical or
  high-severity findings across every surface (shortcode intake, output
  escaping, admin-post nonces/caps, REST gate, prepared SQL, fixed-host
  outbound HTTP with `redirection => 0`, no dangerous sinks). Its four
  low findings are resolved below or shipped: stored-URL fail-closed fix
  released as Resolver 1.22.30 / Pro 1.20.34 (see CHANGELOG).
- **Pro-plugin security review** (2026-10-06): full read of
  `v4v-musiclink-pro.php` at 1.20.34 - every surface verified clean except
  one low availability finding (the unbudgeted visitor-driven resolve
  path), fixed as Pro 1.20.35 (the per-IP `resolve` budget, formerly
  recorded as O25 - shipped 2026-10-11). Checked: shortcode/`$_GET`
  intake (`sanitize_text_field` + length-only `guard_guid`), output
  escaping everywhere (render HTML, search UI, social meta, document
  title, Log/Reports/Cache tables, settings fields, `textContent`-only JS
  DOM building - `innerHTML` used solely to clear), all four admin-post
  handlers nonce + `manage_options`, all three AJAX endpoints nonce-gated
  (search 30/min, beacons per-IP budgets; public-page nonces by design and
  documented), no REST routes, every `$wpdb` call prepared or static with
  whitelisted columns, outbound HTTP fixed-host with `redirection => 0`
  (MI API; fountain.fm seeds/chunks exact-host + HTTPS at ~3816-3829, 15-min
  crawl throttle; next-action tokens hex-only or `sanitize_text_field`'d, so
  no header injection), no dangerous sinks (no eval/unserialize/extract/
  file/shell), canonical/og rebuild from sanitized args only, Reports dates
  via `createFromFormat('Y-m-d')`. Pro's artwork edit branch shares the
  declined http:// art rule (see below); its admin post handlers mirror
  standard's.
- **O20 live-harness 404s** (closed 2026-10-05): duplicate of O9 - the
  `fountain_candidate_test`/`fountain_case_test` 404s are stale hardcoded
  `next-action` fixtures against fountain.fm (the originally recorded
  `api.musicindex.org` host was wrong); O9's detect-don't-embed plan is
  the work. Re-open only to triage that plan.
- **http:// artwork accepted - known issue, no current fix, closed**
  (security review finding 2, decided 2026-10-06): tightening
  `handle_update_cache_row()`'s art branch to https-only was declined -
  upstream artwork URLs can be http-only and the site cannot force https.
  Accepted impacts: strict browsers may auto-upgrade or block the art
  `<img>` (worst case: missing artwork), and an on-path attacker on the
  http hop could substitute image bytes; no script execution or session
  leakage is possible from an image, and `esc_url()` still governs what
  can be stored. Do not re-propose the https-only art rule.
- **No rate limit on standard's REST search - known issue, closed**
  (security review finding 3, decided 2026-10-06): `rest_search()` and
  its siblings are `manage_options` + nonce gated with deliberately no
  per-IP limiter (documented in the `register_rest_routes()` docblock);
  worst case is an admin generating bounded upstream MI load (3 batches
  of 100 per query, 8 hits resolved per page). Revisit only if the routes
  ever become non-admin or public, where Pro's per-IP relay limiter would
  then apply.
- **Block builder `clean_value()` strips dangerous characters from GUIDs -
  known issue, closed** (security review finding 4, decided 2026-10-06):
  the block UI deletes `"'<>&` from album/track GUID fields (defence in
  depth for the saved shortcode attribute; the server-side
  `sanitize_text_field()` + `guard_guid()` remain the authority), which
  can mangle a guid containing e.g. an apostrophe. No plugin fix: GUIDs
  should not use dangerous characters in the first place, and such guids
  should be corrected on the Music Index feed itself. Do not re-propose a
  client-side relaxation without a feed-side fix.
- **Fountain self-heal stays inline** (security review finding #5):
  declined 2026-09-24 - no refactor of the inline self-heal.
- **Telegram stale preview** (investigated 2026-09-25): phafe.com serves
  fresh, correct og: tags; the stale image is Telegram's own per-URL cache
  (no purge API). No plugin change; `?tg=1` renders a fresh variant.
- **Production "not logging launches" report** (2026-09-24): false alarm -
  logging worked; no code change resulted.
- **Double-escape Info item**: closed as part of Pro 1.20.20 - the
  canonical helper returns an unescaped URL and every consumer escapes
  exactly once (see CHANGELOG 1.20.20).
- **Security review findings #1-#4, #6-#8**: all implemented and released
  through 1.22.14 / 1.20.20 (see CHANGELOG).
- **Fountain official-API migration + spoofed-UA transparency fix**
  (offered as O10 options 2026-09-26): declined that day - user chose to
  stop after the unauthenticated probe. O10 carries the recorded probe data;
  re-propose only on request. **Re-requested by the user 2026-11-21 - now
  O28**, with p20-link-resolver's working no-key lookup as the precedent.
- **O11 fix approach** (settled 2026-10-05): probe every detected
  `next-action` candidate rather than tighten the near-"search" heuristic,
  and skip O10's official API (Step 0) because O10 needs a paid key and is
  On hold - shipped as Resolver 1.22.24 / Pro 1.20.30 with a two-phase
  chunk walk (search-only delta in full, homepage's first 20 only when the
  delta is empty) so the crawl keeps its pre-fix cost. Re-propose neither
  the heuristic-only variant nor a full-window walk.
- **O6 guid-keyed Fountain positive cache** (planned 2026-09-26): removed
  from the work list 2026-10-05 on user instruction; re-propose only on
  request (full plan preserved in git history).
