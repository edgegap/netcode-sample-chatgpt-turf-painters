# Prompts

## ChatGPT Canvas: the solo game

Sent on 2026-11-20 in a new chat on chatgpt.com (Free plan, default model), one message, no
follow-ups. ChatGPT opened a canvas with the game; its HTML is saved unchanged as
`artifact/turf-painters.html` (git tag `before`).

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
