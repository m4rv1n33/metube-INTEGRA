# Agent Guidelines

This is the INTEGRA fork of MeTube. It is not upstream and it does not send
pull requests upstream. It runs on one box, at `yt.m4rv1n.dev`, behind the
INTEGRA access gate, shared by a handful of people holding access codes.
Deployment facts (image, port, the retention rule the download volume needs)
live in `DEPLOY.md`.

## What this fork changed, and what must not drift back

- **Dark only.** `data-bs-theme="dark"` is pinned in `ui/src/index.html`. The
  theme picker, its cookie, the `prefers-color-scheme` listener and the `Theme`
  interface are gone. The gate in front of this is dark; a light MeTube behind
  it reads as a different product.
- **INTEGRA theme.** The zinc scale, Inter and IBM Plex Mono, and the `#eb0000`
  accent, matching the access gate, the management panel and vert-integra. The
  fonts are the gate's own data-URI file at `ui/src/assets/fonts.css`, bundled
  through `angular.json`. Never swap in a webfont CDN: self-hosted instances
  make no third-party requests.
- **Subscriptions are hidden**, not deleted: the Subscribe button, the
  subscriptions table and the three subscription-only advanced settings are out
  of `app.html`, while `app/subscriptions.py`, `subscriptions.service.ts` and
  the component methods stay. An unattended feed downloads on behalf of whoever
  set it up, and on a shared instance nobody owns that queue.
- **The completed list is per browser session.** The server still keeps one
  instance-wide list; `sessionDoneKeys` in `ui/src/app/services/downloads.service.ts`
  filters it down to what this tab downloaded. This exists so visitors do not
  read each other's titles. It is a courtesy, not a boundary: anyone who can
  reach the port can reach the files.
- **No login, ever.** The gate is the only authentication. Do not add user
  accounts, per-user folders or session ownership to MeTube itself.
- **yt-dlp is unpinned at image build time.** See the last layer of the
  `Dockerfile`. `uv.lock` no longer decides which yt-dlp ships.

When pulling in upstream changes, re-check each of these: upstream owns the
files they live in and will happily reintroduce a theme switch.

## Project scope

The upstream line on scope still applies, because it is what keeps this
maintainable, even though the person deciding is now you:

MeTube's contract is: give it a URL, it runs yt-dlp well, and correct files
appear.

**In scope:**

- Features that make the file yt-dlp writes at download time come out more
  correct, using only data the extractor already provides.
- Surfacing functionality yt-dlp itself owns as first-class UI options (a
  SponsorBlock toggle that passes postprocessor params, for example).
- Download queue, output templates, and UI improvements to the download
  workflow.

**Out of scope:**

- Tag editors, metadata dialogs, or any workflow that rewrites files after the
  download finished.
- Lookups against external metadata services (iTunes, Deezer, MusicBrainz), and
  more broadly any new network egress from the instance beyond what yt-dlp
  itself performs.
- Library organization: moving or renaming existing files into Artist/Album
  layouts, watch-folder processing, media-manager features. Dedicated tools
  (beets, Picard, Lidarr) do this properly.
- Host-side cleanup of the download volume. That is real and required, but it
  is deploy configuration, not application code. See `DEPLOY.md`.

**Corollaries:**

- Site-specific intelligence (parsing playlist-ID prefixes, URL path
  conventions, other platform internals) is extractor work and belongs in
  yt-dlp. Re-implemented here it silently breaks when the platform changes.
- Prefer enriching yt-dlp's info dict and letting its pipeline (FFmpegMetadata
  etc.) do the writing, over custom per-format tag-writing code in MeTube.
- Supplemental processing must never fail a download that otherwise succeeded:
  warn and continue, do not raise.
- A hardcoded sensible default beats a configuration surface.

## Documentation

`README.md` is upstream's documentation, kept so the environment variables and
yt-dlp notes stay findable; only the fork banner at the top is ours. Fork-specific
documentation goes in `DEPLOY.md` instead of growing the README. Docker Hub's
25,000-character README limit no longer applies: this fork does not publish
there and the workflow that enforced it is gone.

## Tech stack

- **Backend:** Python 3.13+, aiohttp, python-socketio 5.x, yt-dlp
- **Frontend:** Angular 22, TypeScript, Bootstrap 5, SASS, ngx-socket-io
- **Package managers:** uv (Python), pnpm (frontend)
- **Container:** Multi-stage Docker (Node builder + Python runtime), amd64 only

## Build & test commands

```bash
# Frontend (run from ui/)
pnpm install --frozen-lockfile
pnpm run lint
pnpm run build
pnpm exec ng test --watch=false

# Backend (run from repo root)
uv sync --frozen --group dev
python -m compileall app
uv run pytest app/tests/
```

**These no longer run in CI.** The fork's only workflow is
`.github/workflows/build.yml`, which builds and pushes the image; upstream's
quality-check job went with the rest of its release machinery. A broken
frontend still fails the image build, because the build stage compiles it, but
nothing else is checked for you. Run lint and both test suites locally before
committing.

Gotchas:

- Backend tests must run **from the repo root**: `main.py` resolves the
  static-assets path relative to the cwd, and several test modules import
  `main`. Running from `app/` makes five test modules fail to import.
- The frontend must be **built before** running backend tests (same reason: the
  assets at `ui/dist/metube/browser` must exist). The command order above is
  load-bearing.
- `app/tests/test_ytdl_utils.py` stubs `yt_dlp` at import time. Run standalone,
  two tests fail with `AttributeError: <module 'yt_dlp'> does not have the
  attribute 'YoutubeDL'`; under the full suite the real module is imported
  first and they pass. Known quirk, not a bug in the code under test.
- The Angular CLI needs Node >= 24.15.0 (or 22.22.3, or 26). An older 24.x
  refuses to run `ng` at all, with a version message and exit code 3.

## Releases

Every push to `master` builds `ghcr.io/m4rv1n33/metube` and moves both `latest`
and a `sha-<commit>` tag. There is no staging branch and no dated GitHub
release: what is on master is what the box pulls. A weekly scheduled build
exists to refresh yt-dlp, so the image can change without any commit.

If a change needs verifying in a real container, build it locally
(`docker build .`) rather than pushing to see what happens.

## Commit messages

Explain the cause and the reasoning, not just the change. If the fork's issue
tracker is used, close the issue from the commit with a GitHub closing keyword
in parentheses at the end of the subject line:

```
fix: stop metadata probes from writing playlist sidecar files (closes #1040)
```

A bare `(#1040)` only references and reads as a pull-request number, so it does
not close anything.

Follow `.editorconfig`:
- Python: 4-space indent
- Everything else (TypeScript, YAML, JSON, HTML): 2-space indent
- UTF-8, LF line endings, trim trailing whitespace, final newline

Frontend additionally uses ESLint (`ui/eslint.config.js`) and Prettier (config
in `ui/package.json`: `printWidth=100`, `singleQuote=true`).

## Project structure

```
app/main.py          — HTTP server, Socket.IO events, REST API routes, Config class
app/ytdl.py          — Download queue logic, yt-dlp integration
app/subscriptions.py — Channel/playlist subscription manager (hidden in the UI)
app/state_store.py   — JSON-based persistent storage with atomic writes
app/dl_formats.py    — Video/audio codec/quality mapping
app/tests/           — pytest tests (asyncio_mode=auto)
ui/src/styles.sass   — the INTEGRA theme, mapped onto Bootstrap's variables
ui/src/assets/fonts.css — Inter and IBM Plex Mono as data URIs, bundled first
ui/src/app/          — Angular standalone components (no NgModules)
DEPLOY.md            — image, port, hostname, download-volume retention
```

## Key conventions

- Backend configuration lives in the `Config` class in `app/main.py` with
  env-var defaults in `_DEFAULTS`. New env vars go there.
- The app listens on 8081 inside the container, bound to `0.0.0.0`. Leave it
  there; the compose mapping and the Caddy block assume it.
- Real-time communication uses Socket.IO events, not REST polling.
- Frontend uses standalone Angular components with `inject()` for DI, RxJS
  Subjects for state, and `takeUntilDestroyed()` for cleanup.
- Frontend components use OnPush change detection: subscribe callbacks must
  call `cdr.markForCheck()`.
- Styling goes through the theme variables in `ui/src/styles.sass`
  (`--bg`, `--fg`, `--muted`, `--border`, `--surface`, `--accent`), not raw hex
  values and not Bootstrap's own palette classes. Bootstrap component variables
  (`--bs-btn-*`, `--bs-form-*`) are compiled from Sass upstream and do not
  follow `--bs-primary`, so each component that needs theming is restated by
  hand in that file.
- State is persisted as JSON files via `AtomicJsonStore` in
  `app/state_store.py`.
- Persisted state stays compact: the completed queue deliberately drops bulky
  entry data (see `_compact_persisted_entry` in `app/ytdl.py`).
- Custom yt-dlp postprocessors added to `ytdl_params['postprocessors']` run in
  **list order** within a stage. Mirror the ordering the yt-dlp CLI would
  produce (sponsor-segment removal before chapter splitting).
- No pre-commit hooks, and now no CI checks either. See the command list above.

## Checklist: adding a per-download option

New options on the download form (the `split_by_chapters` pattern) need **all**
of these:

1. `parse_download_options` in `app/main.py`.
2. A field on `DownloadInfo` in `app/ytdl.py`.
3. A `hasattr` backfill in `DownloadInfo.__setstate__` for old persisted
   records.
4. The safe-deserialization field list in `app/ytdl.py`.
5. UI form control + cookie persistence in `ui/src/app/app.ts` / `app.html`,
   and the payload in `downloads.service.ts` (plus its spec).
6. The redownload path in `app.ts`, so retries carry the option.

Threading it through `app/subscriptions.py` is no longer part of the checklist:
subscriptions are hidden here, so every option is a direct-download option. If
subscriptions are ever unhidden, that step comes back.

## Security invariants

There is no authentication in this application and none is wanted: the INTEGRA
access gate is the only thing between the internet and this queue. Never expose
the port without it, and never add an endpoint that assumes the caller is the
owner.

User input and extractor-provided metadata (titles, playlist names, URLs) are
untrusted. Use the existing guards instead of hand-rolling:

- User-submitted URLs go through the SSRF guard (see `test_url_guard.py` for
  the expected behavior).
- Anything that becomes a filesystem path goes through `_is_within_directory`
  and `_sanitize_path_component` in `app/ytdl.py`, including values that arrive
  via yt-dlp metadata, which sites can influence.
