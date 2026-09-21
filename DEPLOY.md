# Deploying this fork

Notes that belong to this fork specifically. Upstream's `README.md` still
documents every environment variable and yt-dlp option; nothing here replaces
it.

## Image

CI builds `ghcr.io/m4rv1n33/metube` on every push to `master`, on
`workflow_dispatch`, and weekly (Monday 04:00 UTC). Tags: `latest` and
`sha-<full commit sha>`. `linux/amd64` only, because INTEGRA is an x86_64 mini
PC.

The weekly build is not cosmetic. yt-dlp is installed unpinned at image build
time rather than from `uv.lock`, so a rebuild is the only thing that refreshes
extractors. If a site stops working, trigger a build before debugging anything
else.

## Listen port

The app listens on **8081** inside the container (`PORT` in `app/main.py`,
`EXPOSE 8081` in the `Dockerfile`), bound to `0.0.0.0`. Unchanged from
upstream, and nothing in the fork depends on it being anything else. Map it in
compose; do not rely on host networking.

The hostname is `yt.m4rv1n.dev`, and `yt` is also the maintenance flag name in
the INTEGRA management panel and the argument to `import maintenance` in the
host Caddy block. Those three spellings have to match exactly.

## Access

There is no login and no user accounts: the instance is reached only through
the INTEGRA access gate, and everyone who gets past it shares one queue. The UI
reflects that, subscriptions are hidden and the completed list only shows what
the current browser session downloaded. None of that is a security boundary.
Anything reachable at the container's port is reachable by anyone who can reach
that port, so it must never be exposed without the gate in front of it.

## Downloads directory needs a cleanup strategy

`DOWNLOAD_DIR` defaults to `/downloads`, with state in `/downloads/.metube`.
Every finished download stays on that volume forever: MeTube never deletes
media, and hiding the completed list from other visitors does not change what
is on disk. It only changes who sees the titles.

So the deployment has to answer this, not this repo:

- a retention rule for the download volume (for example a periodic sweep of
  files older than N days), and
- a size ceiling or an alert, since a full disk on INTEGRA affects every other
  service on the box, not just this one.

Deliberately not implemented here. Host-side retention is deploy configuration:
a cron job, a systemd timer, or a sidecar in the compose file that owns the
volume. Putting it in the application would mean MeTube deleting user files on
a timer, which is exactly the kind of after-the-download file management the
project keeps out of scope (see `AGENTS.md`).

`/downloads/.metube` holds the queue state files. A cleanup rule that walks the
whole volume must skip that directory, or the queue loses its history on the
next sweep.
