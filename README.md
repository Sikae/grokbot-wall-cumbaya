# Grok Bot · Cumbayá

A Three.js welcome wall for Grok Bot meetups. Guests check in on Luma, a full-screen
animated Grok Bot greets each one by name on the venue projector, and every guest is
emailed a Cursor referral code (credits) the moment they check in.

**Daylight mode** (default): flat solid Grok Bot on a cream background, for a bright morning
room. Each check-in morphs to the next official silhouette and holds the welcome for about
9 seconds. The night-of cycle is only the preferred brand set: blob, pebble, bean, egg,
squircle, wedge, hex, cloud, teardrop, capsule. Body colors are the production hexes
(brown `#97683D`, red `#FF263C`, orange `#FF6700`, yellow `#FF9800`, green `#00C972`,
cyan `#00BCA6`, blue `#1084FE`, violet `#9159FE`, magenta `#FF309B`, gray `#777777`).
Idle starts on the blue blob. Flower, clover, and sharp triangle are not in the cycle.

**ASCII mode** (`?ascii`): the original terminal aesthetic — a WebGL character grid with
3D Grok Bots rendered through an ASCII shader, morphing silhouettes, drifting headline,
live check-in feed, and block-letter welcome overlays. Click or tap for an ASCII shockwave.

The server polls Luma (and accepts its webhook), allocates one code per guest, and emails
it via Resend. The wall page is a viewer. A hidden staff console handles lookups, resends,
manual check-ins, and a CSV export.

---

## Cumbayá Rehearsal Notes (Sat 3 Oct 2026)

**Projector setup:**
```sh
npm start                      # serves http://localhost:8787
# Open in browser: http://localhost:8787/?key=<your-token>
# Press F for fullscreen
```

**Demo the welcome flow:** Press `Space` to fake a check-in. Each press shows a welcome
animation with a new bot shape and colour.

**Staff laptop (desk mode):** Open `http://localhost:8787/?key=<token>&desk` on a separate
device. Press `D` to toggle the console — search guests, resend emails, manual check-ins.

**Docker:**
```sh
docker compose up -d --build   # wall on 127.0.0.1:8787
```

**URL parameters:**
| param | effect |
|-------|--------|
| `?key=<token>` | required when WALL_TOKEN is set |
| `?desk` | enable staff console (press D) |
| `?ascii` | use ASCII terminal mode instead of daylight |
| `?lang=es` | Spanish text ("Esperándote", "BIENVENIDO") |
| `?demo` | force demo mode with fake guests |

---

## Quick start (local, demo mode)

```sh
npm start                      # builds dist/index.html and serves http://localhost:8787
```

With no `.env` the wall runs a demo with fake guests and placeholder codes. Press `Space`
to fake a check-in, `F` for fullscreen.

## Running it for real

1. `cp .env.example .env` and fill in:
   - `EVENT_NAME` — the headline on the wall and the name in the email.
   - `LUMA_API_KEY` (Luma → Settings → Developer → API Keys) and `LUMA_EVENT_ID` (`evt-…`).
   - `RESEND_API_KEY` and `EMAIL_FROM` (the sender domain must be verified in Resend).
   - `WALL_TOKEN` — any secret; the wall is opened as `/?key=<token>`.
   - `PUBLIC_URL` — the public https URL once hosted (used for the email hero image and to
     self-register the Luma webhook).
2. Put your referral codes in `data/codes.json` (same shape as `data/codes.example.json`:
   two pools, each a list of `{code, url}`). Codes are handed out one per guest, pool A
   first; "one from each pool" is a setting in the staff console.
3. `npm start`, open `http://localhost:8787/?key=<token>` and press `F`.

Rehearsal switches in `.env`: `EMAIL_DRY_RUN=1` logs instead of sending; `EMAIL_TEST_TO`
redirects every guest email to you. Clear both before doors open. Restart after editing.

## Hosting (Docker on a box + Cloudflare tunnel)

```sh
git clone <this repo> && cd grokbot-wall
cp .env.example .env            # fill it in as above
cp data/codes.example.json data/codes.json   # then replace with your real codes
docker compose up -d --build    # wall on 127.0.0.1:8787
```

For a public https hostname without opening ports, use a Cloudflare named tunnel:
`cloudflared tunnel login`, `cloudflared tunnel create grokbot`,
`cloudflared tunnel route dns grokbot grokbot.example.com`, copy the credentials JSON into
`cloudflared/` next to a `config.yml` made from `config.example.yml`, set `PUBLIC_URL`, and
run `docker compose --profile tunnel up -d`. With `PUBLIC_URL` set the server registers a
Luma `guest.updated` webhook for itself (check-ins then show in about a second; polling
every 5 s is the fallback). `deploy.sh` wraps the compose commands.

`WALL_TOKEN` gates every route except the webhook and `/assets`. `data/state.json` holds
allocations and email status and survives restarts. The hosted HTML never embeds codes.

## Staff console

Open the wall with `&desk` in the URL and press `D`: search guests, see a guest's code and
QR, resend their email, release a code back to the pool, do a manual check-in for walk-ins,
export a CSV, change settings (event, pool mode, poll interval).

## How the email queue behaves

Sends are queued at 2/s with a timeout on each request, retries on rate limits and
transient failures, and a sweep every 20 s that re-queues anything stuck; a restart
re-queues every pending send. `GET /email/status`, `POST /email/sweep`, and `POST /email/send-now`
exist for emergencies. Guests with no email or no code left are recorded as skipped.

## Keys and URL params

| key | action |
| --- | --- |
| `F` | fullscreen |
| `Space` | demo only: fake a check-in |
| `D` / `M` / `Shift+E` | with `&desk`: console, manual check-in, export CSV |

| param | effect |
| --- | --- |
| `?key=<token>` | required when WALL_TOKEN is set |
| `?desk` | enable staff console (press D to open) |
| `?ascii` | ASCII terminal mode (dark theme) |
| `?dark` | alias for `?ascii` |
| `?lang=es` | Spanish UI text |
| `?demo` | force demo mode with fake guests |
| `?cell=22` | bigger text (ASCII mode) |
| `?fine=2` | chunkier backdrop glyphs (ASCII mode) |
| `?headline=…` | override the drifting headline |
| `?style=3&color=2&hold` | freeze bot form and colour |
| `?raw` | 3D scene without ASCII pass |

## Layout of the repo

```
src/index.html      the wall (three.js UMD + qrcode-generator are inlined by the build)
serve.mjs           server: static, Luma proxy + polling, allocation, Resend queue, webhook
build.mjs           → dist/index.html  (--no-codes for a public copy)
tunnel.mjs          laptop mode: cloudflared quick tunnel + webhook lifecycle
hook-relay.mjs      exposes only the webhook path for that tunnel
assets/hero-blue.jpg  email header, from the Grok Bot brand kit
data/codes.example.json  shape of the referral-code file
```

## Credits

three.js (MIT) and qrcode-generator (MIT) are vendored in `src/vendor/`. The email header
image is from the Grok Bot brand kit.
