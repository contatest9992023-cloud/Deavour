# DEVOUR — "Start small. Consume everything."

Original hole-eating arcade game: solo campaign (4 maps, endless procedural levels) + real-time multiplayer **Arena**
(online with a server, or offline against bots). 12 languages (incl. RTL Arabic), pt-BR / English first.

## Run locally
```bash
npm start            # http://localhost:8080  (static site + WebSocket server on /ws)
npm test             # world determinism, balance, arena sim, server integration, anti-cheat checks
```
Requires Node 18+. No npm dependencies (the WebSocket server is implemented in `server/ws.js`).
Open the game through `http://` (ES modules do not load from `file://`).

## Project layout
```
public/                      static site (what you host)
  index.html, style.css, config.js
  js/core/                   PURE game logic shared by browser AND server (no DOM)
    data.js                  objects, maps, skins, events  <- add content here
    growth.js                progression curve (smooth, 30 levels)
    worldgen.js movers.js    deterministic world + animated cars/people/boats (functions of time)
    objects.js               pooled objects + spatial grid
    arena.js                 authoritative multiplayer simulation (also runs offline with bots)
    guard.js                 anti-exploit helpers
  js/engine/                 rendering, particles, absorb animation, audio, input, solo + arena game views
  js/net/sources.js          LocalSource (bots) / NetSource (WebSocket, interpolation, prediction)
  js/i18n/                   index.js + langs/*.js (add a language = one file + one line)
  js/save.js economy.js levels.js   persistence (signed), rewards, missions, achievements
  js/ui/ui.js                screens
server/                      server.js (HTTP + WS), rooms.js (matchmaking/rooms), ws.js (RFC6455)
test/                        headless tests
```

## Balance changes (v2)
* 30 hole levels, radius +10 % per level (was +20 %, 15 levels). Level-ups every ~4–15 s in a normal run.
* Energy per object is sub-linear in score and **at most one level per object**, so giant objects never cause jumps.
* Clearing a whole big map ends around level 21–25; level 30 ("Black Hole") needs several maps/arena play.
* Rewards are capped per match; coin payout from score was reduced.

## Anti-exploit
* **Arena is server-authoritative**: clients send only a movement direction (rate-limited, normalized, speed capped on the
  server). Score, growth, eating, kills and rewards are computed server-side. Anti-boosting: no reward when the same
  victim is farmed within 60 s; 3 s spawn protection; names sanitized; skin whitelist; message-size, flood, per-IP and
  AFK limits; Origin check (`ALLOWED_ORIGINS` env for cross-site hosting).
* **Solo**: results are validated (score/object rate, time vs real wall clock → speed-hack detection, impossible values),
  payouts are capped, rewarded ads have a cooldown + daily cap, save data is signed (hand-edited saves reset the economy),
  and game state is not exposed on `window`.
* Honest limit: a purely client-side game can always be tampered with by a determined user. Anything that must be fully
  trusted (global leaderboards, paid items) needs server accounts — the arena already shows the pattern.

## Deploying "as a site on Google"
Google Sites cannot run this game (no custom JS / WebSockets). Use Google Cloud instead:

**Option A – one service (simplest): Cloud Run serves the site AND the multiplayer server**
```bash
gcloud run deploy devour --source . --region southamerica-east1 --allow-unauthenticated \
  --session-affinity --timeout 3600 --min-instances 1 --max-instances 1
```
`max-instances 1` is required because rooms live in memory (scale by sharding rooms/regions later).

**Option B – Firebase Hosting for the static site + Cloud Run for the server**
1. Deploy the server as above. 2. Set `public/config.js`:
   `window.DEVOUR_CONFIG = { wsUrl: 'wss://YOUR-SERVICE.a.run.app/ws' };`
3. `firebase deploy --only hosting`. 4. On Cloud Run set `ALLOWED_ORIGINS=https://YOUR-SITE.web.app`.

If the server is unreachable the Arena automatically falls back to offline play against bots.

## Not done / next steps
Real ad SDK (the rewarded-ad button is a placeholder), accounts + global leaderboard, load testing with many rooms,
translation review by native speakers, PWA manifest/icons.
