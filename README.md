# V4V Music Link Resolver Pro

A WordPress plugin that renders direct links to an album/track in Podcasting 2.0 music player apps (Fountain.fm, v4vmusic.com, StableKraft.app, PodTunes.app) for music indexed by [musicindex.org](https://musicindex.org) — an alternative to pod.link.

This is the **Pro** version. The standard version lives in the sibling repository [`ghmattf/v4v-musiclink`](https://github.com/ghmattf/v4v-musiclink); the two plugins can run on the same WordPress instance without conflict (separate class, shortcode, option names, cache tables, and asset handles).

## Features

- Shortcode `[musiclink_pro]` with `album`/`track`/`header` attributes, rendered inline in any post or page:
  ```
  [musiclink_pro album="{album-guid}"]
  [musiclink_pro album="{album-guid}" track="{track-guid}"]
  [musiclink_pro album="{album-guid}" track="{track-guid}" header="false"]
  ```
- **URL-parameter mode:** guid arguments can also arrive in the URL when the shortcode omits them:
  ```
  https://example.com/musiclink?album={album-guid}&track={track-guid}
  ```
- **Front-end search:** with no parameters in the shortcode or the URL, the page shows a search UI against Music Index; picking a result renders the regular release page (shareable URL).
- **Fountain.fm resolution:** deep links are resolved via Fountain's unofficial Server Action endpoint, with automatic self-healing of the `next-action` fingerprint and a search fallback (`https://fountain.fm/search?q="…"`) when nothing resolves.
- **Launch/open logging:** one row when a visitor picks a search result, plus one row per app-button launch. Page renders (including shared links) are never logged and raw guids are never stored. Both beacons are per-IP rate-limited (30/min per endpoint by default, configurable); the log keeps only its newest rows (100,000 by default).
- **Admin tabs:** View Log (with Clear log), Reports (totals by artist/album/track or by music app, with date-range and artist filters), and a **Cache** tab listing cached release rows (date-only Music Index and Fountain.fm timestamps, tick/cross art status) with per-row artwork/Fountain URL editing, launch, delete, and clear-links actions.
- **Extras:** Open Graph / Twitter Card social previews, a share button, and canonical URL fixes for guid'd URLs.
- **Per-app controls** (Settings → V4V MusicLink Pro): enable/disable each app, custom icon URL, icon background colour (or transparent), button order arrows, and an optional random order per page load.
- **Music Index refresh:** cached metadata serves immediately and refreshes in the background once older than 12 hours (configurable 1–720 h; rows are never evicted, so a Music Index outage can't blank an already-seen release).

## Target apps

| App | Link style |
|---|---|
| StableKraft.app | Formulaic GUID-based URL, no lookup required |
| PodTunes.app | Formulaic GUID-based URL, no lookup required |
| v4vmusic.com | Formulaic GUID-based URL (`/lookup/{album}` or `/lookup/{album}/{track}`) |
| Fountain.fm | Resolved via the unofficial Server Action endpoint, with automatic self-healing of the `next-action` fingerprint; falls back to a pre-filled exact-phrase search on the release title (or `fountain.fm/discover` when no title is known) |

## Data source

Music metadata (titles, artist, artwork) comes from the [MusicIndex.org](https://musicindex.org) API (`https://api.musicindex.org/v1`). No API credentials are needed. Every album/track pair is stored as one row in the plugin's own `{prefix}v4v_musiclink_pro_cache` table, which also holds that pair's resolved Fountain.fm link. Failed lookups are remembered separately in WordPress transients (1 hour for definitive misses, 30 seconds for unreliable ones).

## Development

See [CONTRIBUTING.md](CONTRIBUTING.md) for the versioning/build workflow and [CHANGELOG.md](CHANGELOG.md) for release history. The plugin file is `v4v-musiclink-pro.php` at the repository root.

### PHP toolchain

Linting and testing use a **portable PHP CLI kept in `php/`** — there is no system PHP dependency and `php` is generally **not** on `PATH`:

```bash
./php/install.sh   # first run / after a fresh clone (binary is gitignored)
./php/php -l v4v-musiclink-pro.php
```

Run the whole regression suite with `for f in tests/test_*.php tests/*_test.php; do ./php/php "$f"; done` — see the [Regression tests section of CONTRIBUTING.md](CONTRIBUTING.md#regression-tests).

If you are an agent or new contributor: **before you can lint or run any PHP, run `./php/install.sh` once**, then always invoke `./php/php`.
