# How to make a multiplayer game with ChatGPT

![Make Your ChatGPT Game Multiplayer](docs/images/card.jpg)

A worked example you can follow step by step: **ChatGPT** writes a single-player browser game as
one HTML file, then **Edgegap** takes it online with one authoritative server per match, plus a
matchmaker if you want a Find match button. Turf Painters, a 4-painter arena game where every tile
you roll over takes your color and bumping a rival splashes paint around it, is the game built
here.

This repo is a recipe, not a live demo. Nothing in it runs on its own: you create your own Edgegap
app version (and matchmaker), play, and remove them when you're done. Each server stops itself when
its match ends, so you only pay while people play.

```
chatgpt.com ──writes──> artifact/turf-painters.html          (one HTML file, solo game vs bots,
                          │  rules between                     plays in ChatGPT's file preview)
                          │  /* RULES:BEGIN */ ... /* RULES:END */
                          │
     tools/split-artifact.mjs ──copies the block──> client/rules.js (ES module, generated)
                                                       │
your page ──Find match──> Edgegap matchmaker ──starts──> one Node.js server per match
     <──────────── wss://<id>.pr.edgegap.net:<port> ──  runs rules.js at 60 Hz,
                                                        fills seats with bots,
                                                        stops itself after the match
```

Verified on Edgegap on 2026-11-20, both ways in: two browser tabs joined one server directly
(`?server=`) and played a full 60-second round, and the server stopped itself afterwards; then two
headless players and two browser tabs clicked **Find match**, were matched onto one server in about
15 seconds, and the server stopped itself 11 seconds after the round (and within about 30 seconds
of both tabs closing). Cost of a match: one 0.25 vCPU server for about 90 seconds.

## What you need

- A ChatGPT account (chatgpt.com). The game here was made on the Free plan with the default model.
- An Edgegap account (app.edgegap.com). The free tier is enough: one matchmaker that stops about
  3 hours after each start, one server at a time.
- A GitHub repo. GitHub Actions builds the server image, so you don't need Docker on your computer.
- Node.js 22 to run the split script and the local tests.
- Optional but recommended: an AI coding agent with the Edgegap MCP server connected
  (`https://mcp.edgegap.dev`). Its `edgegap-chat-canvas` skill holds the prompt, the split script, the
  server templates, the matchmaker config and the tests used here.

## Values you will fill in

| Placeholder | Where it comes from | Where it goes |
|---|---|---|
| `<registry-project>` | Edgegap dashboard > Container Registry (looks like `edgegap-xxxxxxxx`) | GitHub variable `EDGEGAP_REGISTRY_PROJECT` |
| registry username / token | Container Registry > Credentials | GitHub secrets `EDGEGAP_REGISTRY_USERNAME`, `EDGEGAP_REGISTRY_TOKEN` (never in the code) |
| `<image-tag>` | The build summary of the Server image workflow (`YYYYMMDD-HHMM`) | App version tag and name, `matchmaker/turf-painters.json` |
| `<server-url>` | Deployments > your deployment: `wss://<fqdn>:<external port>` | `?server=` on the page URL (step 6) |
| `<page-origin>` | Where you serve `client/` (`http://localhost:8080`, `https://<org>.github.io`...) | `allowed_cors_origins` in `matchmaker/turf-painters.json` |
| `<matchmaker-url>`, `<matchmaker-token>` | Matchmaker page > API URL and Auth Token | `client/config.js` |

The matchmaker Auth Token is a client token: it can only ask for matches and is made to ship in
game pages. Never put your Edgegap API token in the page, the repo or the image.

## Step 1: Ask ChatGPT for a game that can go online

ChatGPT writes the game as one HTML file and runs it in a preview next to the chat. The server needs
the rules on their own, so ask ChatGPT to keep every rule between two exact marker lines, with
nothing from the browser inside. The file stays one file that plays in the preview, and a script
copies the block out for the server (step 2).

The prompt used for Turf Painters, verbatim, in a new chat:

> Use canvas. Build a small browser game as a single HTML file called "Turf Painters" (plain
> Canvas 2D, no frameworks, no external libraries) and show it in canvas so I can play it in the
> preview. Gameplay: 4 round painters in a square arena (800x800 units, a 20x20 grid of floor
> tiles) move with WASD/arrow keys with drift (velocity + friction). Every tile a painter rolls
> over turns its color. Bumping another painter at speed splashes paint: the 3x3 tiles around the
> bumped painter turn to the bumper's color. A round lasts 60 seconds; the painter with the most
> tiles wins, then a results screen and restart. The human plays against 3 simple bots. VERY
> IMPORTANT architecture: put ALL game rules in one block between the exact lines
> /* RULES:BEGIN */ and /* RULES:END */, with NO window, document, canvas, audio,
> requestAnimationFrame or timers inside, and no export keyword. The world is plain data
> (objects, arrays, numbers, strings; no classes or Maps). The block defines createWorld(),
> addPlayer(world, id, name, isBot), removePlayer(world, id), setInput(world, id,
> {up,down,left,right}), step(world, dt) returning an array of events ({type:'bump', a, b} and
> {type:'win', winner}), and botInput(world, id) using only positions and tiles. Everything else
> (rendering, keyboard input, calling step() at 60 Hz) stays outside the block.

For your own game, change the gameplay sentences and keep the architecture part as it is.

Why the prompt is shaped this way:

| Prompt detail | Why it matters online |
|---|---|
| Rules between `/* RULES:BEGIN */` and `/* RULES:END */`, no `window`/`document`/canvas/timers, no `export` | The file still plays in ChatGPT's preview, and the block runs unchanged in Node.js on the server. No port, no drift between solo and online. |
| The world is plain data | The server sends its world over the network as JSON (here the 400 tiles as owner ids), and the page draws it with ChatGPT's own `draw()`. |
| `setInput(world, id, {up,down,left,right})` | That object is exactly what the browser sends: four booleans. |
| `step(world, dt)` returns events | The server forwards bumps to every player and ends the round on the win. |
| `botInput(world, id)` reads only positions and tiles | The server fills empty seats with the same bots, and the tests drive headless players with it. |
| Drift, bumping, painting | The game still feels right with 50 to 100 ms of network delay. Pick slippery movement, pushing, slow shots or turn-based play; avoid precise platforming, rhythm or instant-hit aiming. |

**Play it before you save it.** ChatGPT's first file played, but the bots never moved: its
`setInput` ignored bot players while the page fed the bots through `setInput`. Its `createWorld()`
also added the four painters itself and put every new one in the middle, which a server that adds
its own players can't use. One follow-up fixed both:

> Two fixes, keep everything else and the RULES markers the same. 1) The bots never move: setInput
> ignores bot players, but the game loop feeds bots through setInput. Let setInput work for every
> player. 2) createWorld() should return a world with no players; the page then adds the 4
> painters with addPlayer, and addPlayer places each new painter at its own free start spot (one
> per corner area), so any code can add up to 4 players with any ids. Give me the full updated
> HTML file.

Check the same things in your game: bots move in the preview, `createWorld()` returns no players,
and `addPlayer()` gives each new player its own start spot.

**Get the file out.** ChatGPT shows the file as a card: **Open** shows the preview, and the preview
has a download button. That worked for the first file. For the fixed one the download did nothing,
and ChatGPT no longer had the file when asked to paste it. Uploading the first file back with
"apply the two fixes and paste the complete updated HTML file in one code block" worked: **Copy
code** on that block gave the exact file. Ask for the code block from the start and you skip this.
Save it unchanged as `artifact/turf-painters.html`. That commit is tag `before` here; the commit
before it is ChatGPT's first file. `PROMPTS.md` has every prompt.

**Done when:** the file plays solo with moving bots, and it contains both marker lines.

## Step 2: Split out the rules

```
node tools/split-artifact.mjs artifact/turf-painters.html --page
```

- Copies the block into `client/rules.js`, adds `export { createWorld, addPlayer, ... }` for every
  top-level name, and marks the file as generated.
- Refuses a block that uses `window`, `document`, `canvas`, timers, `console`, `import`/`export` and
  other browser-only or module code, with the line, and refuses one that lacks any of the six
  functions.
- `--page` (first time only) writes `client/index.html`: the artifact with the block removed and its
  `<script>` turned into a module that imports the rules. This is the page you make online in step 3.

When you change the game later in ChatGPT, save the new file over `artifact/turf-painters.html`
and run `npm run split`. CI runs `npm run check:rules`, which fails when `client/rules.js` no longer
matches the artifact, so the server can never run old rules.

**Done when:** `npm run check:rules` prints `client/rules.js matches artifact/turf-painters.html`.

## Step 3: Add online play

The prompt for an AI coding agent with the Edgegap MCP:

> Make this ChatGPT game online multiplayer on Edgegap. Use the Edgegap MCP and its
> edgegap-chat-canvas skill. Keep the solo mode, add a Find match button, 2 to 4 players per match,
> bots fill empty seats. Ask me before you create anything in my Edgegap account.

What gets added (all of it is in this repo, ready to copy):

| File | What it does |
|---|---|
| `tools/split-artifact.mjs` | Step 2: the rules block becomes `client/rules.js`; `--check` keeps them in sync. |
| `server/main.js`, `room.js`, `edgegap.js`, `util.js` | Edgegap templates: connections, phases (waiting, countdown, playing, over), bots, the self-stop when the match ends. |
| `server/game.js` | About 80 lines that wrap `client/rules.js`: take the four buttons from players, feed the bots `botInput()` (ChatGPT's `step()` doesn't), run `step()`, count tiles as scores, send the world 20 times a second. The only server file you adapt to your game. |
| `client/online.js`, `client/net.js` | Find match through the Edgegap JS SDK, then the WebSocket to the match server, with retries while the server boots. |
| `client/index.html` | ChatGPT's page, plus about 60 lines: Find match / Local server / Cancel buttons, drawing the server's world with ChatGPT's `draw()`, easing painters between updates, waiting/countdown/results text in the side panel, closing the socket when the tab closes. Start / Restart still plays solo. |
| `client/config.js` | `<matchmaker-url>` and `<matchmaker-token>`, empty until step 7. |
| `server/Dockerfile` | Node 22 image with `server/` and `client/rules.js`. |
| `tests/local-match.mjs` | Two headless players play a full match against a local server. Runs in CI on every build. |
| `tests/e2e.mjs` | The same through the real matchmaker and a real Edgegap server, then checks the server stopped itself. |

The server never trusts the browser: it only accepts four booleans, and decides movement, rams and
painting itself. Player names come from other players, so the page escapes them before showing them
(ChatGPT's scoreboard wrote them straight into the page, which was fine while every name was local).
The human's id was a constant (`"player"`) and the page labelled every non-bot as YOU: online the
id is the server's player id, and only your own painter says YOU.

**Done when:** `npm install && npm run test:local` prints `PASS`, and two tabs on the local page
(**Local server :7777**) see the same match.

## Step 4: Build the server image with GitHub Actions

`.github/workflows/server-image.yml` builds the server on every push to `main` (and on demand),
checks the rules match the artifact, runs the local match test, and pushes the image to the
Edgegap Container Registry as `registry.edgegap.com/<registry-project>/turf-painters:<image-tag>`.

One-time setup in the GitHub repo (Settings > Secrets and variables > Actions):

1. Variable `EDGEGAP_REGISTRY_PROJECT` = `<registry-project>`.
2. Secrets `EDGEGAP_REGISTRY_USERNAME` and `EDGEGAP_REGISTRY_TOKEN` from the Credentials box. Don't
   click Regenerate Token if other projects already push to that registry.

> **Pitfall:** with its default settings, `docker/build-push-action` adds provenance attestations,
> which wrap the image in a multi-manifest index. Edgegap then fails with "Unable to pull image from
> registry". Keep `provenance: false` and `sbom: false` in the workflow, as this repo does.

**Done when:** the workflow's summary shows `Pushed registry.edgegap.com/...:<image-tag>`.

## Step 5: Create the app version in the Edgegap dashboard

Applications > Create application (`turf-painters`), then the version form:

| Setting | Value | Why |
|---|---|---|
| Version name | `<image-tag>` | Easy to match builds to versions. |
| vCPU | 0.25 (512 MB RAM) | Plenty for a 4-player Node.js match. |
| Registry | Edgegap (registry.edgegap.com), repository `<registry-project>/turf-painters`, tag `<image-tag>` | The dashboard fills the pull credentials. |
| Max duration | 30 minutes | A backstop: the server normally stops itself much earlier. |
| Restart policy | Never | A crashed match should end, not restart half-played. |
| Port | 7777, protocol **WS**, name `gameport`, **TLS Upgrade on** | Pages on https can only open `wss://`. Edgegap adds TLS in front, so the server stays plain `ws`. |
| Env `EMPTY_TIMEOUT_S` | 15 | Stop 15 s after the last player leaves. |
| Env `FIRST_JOIN_TIMEOUT_S` | 60 | Stop if nobody connects. |
| Env `RESULTS_SECONDS` | 8 | Results screen length before the server stops. |

To ship a new server build later, change the tag on the version and save. Running matches are not
affected.

**Done when:** the version shows the port `7777 WS gameport` with TLS Upgrade `true` and the three
env vars.

## Step 6: Play online without a matchmaker

The quickest way online, and enough for playing with friends: start one server and share its
address.

1. Deployments > Create deployment, pick `turf-painters` and `<image-tag>`, and place it near the
   players (or let your coding agent call `edgegap_deploy` through the Edgegap MCP).
2. When it is Ready, copy the `gameport` address: `wss://<fqdn>:<external port>` is `<server-url>`.
3. Everyone opens `<page>?server=<server-url>`. The page joins that server straight away. Bots fill
   the empty seats, and the match starts 15 seconds after the first player joins.

**Done when:** two windows on that URL play the same match, and the deployment disappears from the
list by itself once the match is over and everyone has left.

## Step 7: Add the matchmaker (optional)

For a **Find match** button that groups strangers and starts a server for each match:

1. In `matchmaker/turf-painters.json`, set `application.version` to `<image-tag>` and list every
   `<page-origin>` in `allowed_cors_origins` (scheme + host + port; the file starts with
   `http://localhost:8080`).
2. Matchmakers > Create Matchmaker, upload the file, **Create and Start**. The dashboard may warn
   that the image "is not cached" or "not valid" when it can't check a private registry: continue,
   the end-to-end test proves it.
3. Copy the **API URL** and **Auth Token** into `client/config.js` (or a git-ignored
   `client/config.local.js` with the same shape, or `?mm=<url>&token=<token>` on the page URL).

The profile `turf-painters` matches 2 to 4 players. After 20 seconds in the queue, a lone player gets a
match on their own and bots fill the other seats, so nobody waits forever.

**Done when:** the matchmaker shows ONLINE and **Find match** on the page says "Searching for
players...".

## Step 8: Prove it end to end

1. Serve the page: `npm run client`, then http://localhost:8080 (or publish `client/` with
   `.github/workflows/pages.yml`, and add `https://<org>.github.io` to the CORS list).
2. Actions > **Matchmaker e2e** > Run workflow (it reads `client/config.js`). Two headless players
   queue, land on the same server, play a match, and the test checks that the server stopped
   itself. The passing run for this example:

   ```
   e2e-1: SEARCHING after 3s
   e2e-1: MATCH_FOUND after 13s
   e2e-1: HOST_ASSIGNED after 15s
   server: wss://<id>.pr.edgegap.net:<port>
   result: e2e-1 takes the turf!; scores: e2e-1 55, e2e-2 52, Rusty (bot) 16, Zed (bot) 12
   deployment stopped itself 11s after the match
   PASS
   ```

3. Open the page in two browser windows and click **Find match** in both within 20 seconds. Both
   land in the same match with two bots, on a server near you. `?autoplay=1` on the URL starts
   Find match on page load.
4. In the dashboard's Deployments list, nothing stays running after the match.

**Done when:** the e2e workflow prints `PASS` and the two windows played one match together.

## Step 9: Clean up

Nothing here should run non-stop:

- Each match server stops itself when the match ends (or 15 s after everyone leaves, or 60 s if
  nobody joins; 30 minutes at most). Stop a direct deployment from step 6 yourself if nobody joined
  it.
- **Stop or delete the matchmaker** when you're done testing. On the free tier it stops by itself
  about 3 hours after each start; Start or Quick Restart brings it back for the next session.
- If you published the page, players only get "Could not find a match" while the matchmaker is
  off; **Play vs bots** keeps working. Unpublish it (Settings > Pages) if you don't want it up.
- The app version, registry images and GitHub secrets cost nothing while idle.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `split-artifact: markers not found` | The file lost the marker lines. Ask ChatGPT to put back the exact lines `/* RULES:BEGIN */` and `/* RULES:END */`. |
| `split-artifact: the rules block uses browser-only or module code` | It names the line. Ask ChatGPT to move that code outside the block (drawing, sounds and timers belong to the page). |
| CI fails at "client/rules.js matches the artifact's rules block" | You saved a new file without running `npm run split`. Run it and commit `client/rules.js`. |
| Deployment log: "Unable to pull image from registry" | `provenance: false` / `sbom: false` in the build, check the registry secrets, then put the new tag on the version. |
| Find match fails immediately in the browser | The page's origin is missing from `allowed_cors_origins`, the matchmaker is stopped, or `client/config.js` has the wrong URL or token. |
| "Server starting, connecting (n/10)" | Normal for a few seconds after a server is assigned. If it never connects, check the port is WS with TLS Upgrade on. |
| Matched but the page shows nobody else | Both players must queue within 20 seconds of each other, or each gets their own match with bots. |
| Servers keep running | Check the env vars in step 5 and that restart policy is Never. The 30-minute max duration stops them anyway. |

## Make your own

1. Run the step 1 prompt with your own game idea, keeping the architecture part. Play it, then save
   the file as `artifact/<game>.html`.
2. Copy `tools/`, `server/`, `client/online.js`, `client/net.js`, `client/config.js`, `tests/`,
   `matchmaker/` and `.github/workflows/` from this repo, run the split with `--page`, then adapt
   `server/game.js` to your rules (mostly the snapshot fields), `playLikeABot()` in
   `tests/headless-player.mjs`, and the online part of `client/index.html`.
3. Follow steps 4 to 9 with your own placeholders.

## Repo layout

- `artifact/turf-painters.html`: ChatGPT's file, unchanged. The source of the rules.
- `PROMPTS.md`: every prompt, and what ChatGPT did.
- `tools/split-artifact.mjs`: copies the rules block into `client/rules.js`, checks it, keeps it in sync.
- `client/rules.js`: generated from the artifact. Imported by the page and by the server.
- `client/index.html`: ChatGPT's page with online play added; `client/online.js`, `net.js`, `config.js`: Edgegap JS SDK matchmaking, the WebSocket client, the matchmaker settings.
- `server/`: room, phases, bots, self-stop (Edgegap templates); `game.js` wraps the rules.
- `matchmaker/turf-painters.json`: matchmaker config to upload in the Edgegap dashboard.
- `tests/`: local match test and matchmaker end-to-end test.
- `.github/workflows/`: server image build, matchmaker e2e, optional GitHub Pages.

## Run locally

```
npm install
npm run server        # ws://localhost:7777
npm run client        # http://localhost:8080 -> "Local server :7777" (open two tabs)
npm run test:local    # rules in sync + two headless players play a full match
```

Credits: game design, rules and renderer by ChatGPT (chatgpt.com); Edgegap integration from the
Edgegap MCP skills (`edgegap-chat-canvas`, `edgegap-threejs`).
