# Prompts

## ChatGPT Canvas: the solo game

Sent on 2026-11-20 in a new chat on chatgpt.com (Free plan, default model). ChatGPT answered with
an HTML file it built in its workspace, playable in the file preview next to the chat ("Open Turf
Painters HTML", with a download button). It said it can't launch a playable Canvas preview in that
chat, so on this plan the preview is the file pane, not a canvas.

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

The first answer is commit 1 of this repo. In it the bots never move: `setInput` ignored bot
players while the page fed the bots through `setInput`. Its `createWorld()` also added the four
painters itself and `addPlayer` put every new painter in the middle, which a server that adds its
own players can't use.

## Follow-up 1: two fixes

> Two fixes, keep everything else and the RULES markers the same. 1) The bots never move: setInput
> ignores bot players, but the game loop feeds bots through setInput. Let setInput work for every
> player. 2) createWorld() should return a world with no players; the page then adds the 4
> painters with addPlayer, and addPlayer places each new painter at its own free start spot (one
> per corner area), so any code can add up to 4 players with any ids. Give me the full updated
> HTML file.

ChatGPT made both changes (and moved the page to string ids), but its "Download updated Turf
Painters.html" button did nothing, and the next answer said it no longer had the file.

## Follow-up 2: getting the file out

> Please paste the complete updated HTML file here in the chat, as one code block, so I can copy it.

ChatGPT asked for the file back. The first file was uploaded with:

> Here is the file. Apply the two fixes and paste the complete updated HTML file in one code block.

Its code block (copied with **Copy code**) is `artifact/turf-painters.html`, tag `before`: the
first answer with exactly the two fixes.
