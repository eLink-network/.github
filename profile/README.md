# Petrik eLink

Fleet operations system for battery-powered e-paper displays. Renders school
timetables and announcements to e-ink panels across campus, driven by ESP32
relay nodes over Wi-Fi — with an operations console for templates, schedules,
firmware and telemetry.

Petrik · internal project.

## Repos

| Repo | Stack | What it is |
|---|---|---|
| [`elink-web`](elink-web) | Next.js 16 · React 19 · Tailwind v4 · shadcn/ui | Operations console: fleet tables, template studio, topology, alerts |
| [`elink-server`](elink-server) | NestJS 11 · TypeORM · PostgreSQL · Socket.IO · Better Auth | REST API, realtime relay sockets, template rendering, cron scheduling, telemetry, stats |
| [`elink-relay-fw`](elink-relay-fw) | C/C++ · PlatformIO · Arduino (ESP32-S3) | Firmware for the relay nodes: websocket link to the server, NimBLE, panel communication |

A docker-compose file (this repo root) provides the PostgreSQL instance
shared by local development and deployment.

## Architecture

```
                 REST + Socket.IO
┌───────────┐  websocket   ┌──────────────┐  Wi-Fi / BLE  ┌────────────┐
│ elink-web │ ⇄──────────⇄ │ elink-server │ ────────────► │ elink      │
│ (console) │              │ (API, render,│               │ panels     │
└───────────┘              │  cron, stats)│               │ (e-ink)    │
                           └──────┬───────┘               └────────────┘
                                  │ TypeORM
                           ┌──────▼───────┐
                           │  PostgreSQL  │
                           └──────────────┘
```

How a push travels: a display's templates are bound per lesson state
(regular, free period, substituted, moved, moved out). The server composites
the active template into a bitmap, a cron expression or timetable change
triggers the push, and the parent relay node transfers the image to its
panels. Every send is recorded in push history with result and cause.

## Quickstart (development)

API (`elink-server`, :3002):

```bash
cd elink-server
npm install
cp .env.example .env 2>/dev/null || true   # configure before first run
npm run migration:run
npm run start:dev
```

Console (`elink-web`, :3000):

```bash
cd elink-web
npm install
npm run dev                 # set NEXT_PUBLIC_BACKEND_URL if not localhost:3002
```

Open http://localhost:3000 and sign in with GitHub. New accounts start as
`pending` until an administrator approves them in **Users**.

Firmware (`elink-relay-fw`, PlatformIO):

```bash
cd elink-relay-fw
pio run                     # build for esp32-s3-devkitc-1
pio run -t upload           # flash over USB
```

## Configuration

Common environment variables:

- `NEXT_PUBLIC_BACKEND_URL` — backend origin the console calls (default `http://localhost:3002`)
- `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` — backend OAuth (enables sign-in; without them login is disabled with a hint)
- `NEXT_PUBLIC_BATTERY_MIN_MV` / `NEXT_PUBLIC_BATTERY_MAX_MV` — battery mV → % mapping (defaults 2000–3000)
- `DISPLAY_ONLINE_WINDOW_MS` — how long after the last sighting a display still counts as online (default 10 min)
- `NEXT_PUBLIC_LOW_BATTERY_PCT` — low-battery alert threshold (default 20%)

Fleet alert defaults (all user-overridable per browser): low battery 20%,
offline 10 minutes, 3 failed pushes in 24 h.

## Deployment

- **Web**: deployable to Coolify / any Node host — see `elink-web/.env.coolify.example`. `NEXT_PUBLIC_BACKEND_URL` must point at the API origin and be set at build time.
- **Server**: `npm run build && npm run start:prod` behind the same origin as the console in production, or a dedicated API subdomain (then the session cookie's domain must cover both).
- **Database**: the compose service or a managed Postgres; run `migration:run` on upgrades.
- **Relays**: flashed over USB initially; later updates go through **Firmware OTA** in the console (staged, md5-verified, per-device results).

## Conventions

- Keyboard-first console: `⌘K` / `Ctrl+K` command palette, `g then d/s/r/t/a/l/p` for section jumps, `?` for the full list.
- All repos are TypeScript-first (`strict` on the web app); server migrations are named `TIMESTAMP-Name.ts` under `elink-server/src/migrations`.
