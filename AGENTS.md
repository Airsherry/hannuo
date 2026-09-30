# AGENTS.md

## What this repo is

- A single-file static HTML game: `hannuo.html` (汉诺塔 / Tower of Hanoi, UI text in Chinese).
  All CSS and JS are inline; no `package.json`, no build/test/lint toolchain, no CI.
  Run it by opening the file directly in a browser — no dev server, no install step.
- `skill-不要重蹈覆辙-写文件.md` is a lessons-learned doc from a past failed session.
  Read it before making large edits; its rules are binding (see Workflow rules below).
- No other source files. Git repo exists but has **no commits yet** (everything untracked);
  `core.autocrlf=true`.

## Current state of `hannuo.html`

Complete and playable (5/6/7 discs, **press and drag** the top disc onto another peg to move it;
there is no click-to-pick-up / click-to-drop — a plain click only shows the "只能移动最上面的圆盘"
or "按住圆盘拖到另一根柱子上" hint. Pressing a disc that is not on top of its stack is refused).
Verified via `node --check` plus a DOM-stub simulation (full optimal solve at all three
difficulties, illegal moves, 3-miss game over, restart) and a headless-Chrome run of the
same scenarios.

### Difficulty: 5 / 6 / 7 discs (added on request)

`N` is a **`let`, not a `const`** — it starts at 5 (or `localStorage['hanoi-level']`) and is
changed by `setLevel(n)`, wired to the `.lvl button[data-n]` toolbar buttons (styled by the
`.lvl` / `.lvl button.on` CSS rules). `LEVELS=[5,6,7]`; `setLevel` ignores anything else.
Switching difficulty calls `newGame()`, which rebuilds the board and (via `stopDemo()`)
aborts any running demo.

**Everything derived from `N` must stay parameterised** — this is the easy thing to break:

- `wPct(i)` lerps the disc width from `W_OUT` (29%) down to `W_MIN` (8%), *not* the old
  hard-coded `29 - i*(20/4)`, which went **negative** for N=7.
- `build()` recomputes `STEP`/`DISC_H` as `min(6.6667, (STACK_TOP-BASE)/N)` so the tallest
  stack always fits under the rod, and pushes it into the `--discH` CSS variable that `.disc`
  reads (`height:var(--discH,6.6667%)`). For 5–7 discs the cap still wins, so nothing shrinks.
- `build()` also sets `hueStep = min(62, 360/N)`; the old fixed `i*62+8` made disc 0 (8°) and
  the last disc collide near red at N=7 (only 12° apart). The new step gives 62°/60°/51°.
- `demoSpeed()` returns 450/340/260 ms so a 127-move N=7 demo doesn't take a full minute.

### "放弃 · 看解法" demo (added on request)

The `#btnGiveUp` toolbar button first calls `newGame()` — so the demo **always replays the
canonical solution from the standard start**, never from the player's current state — then
animates `solve(N,0,2,1,[])` one step every `demoSpeed()` ms via `playDemo()`. While
`demo` is true the board is locked (`done=true`) and the button reads "停止演示"; clicking it
again, or pressing "新的一局", aborts via `stopDemo()` (which also clears the pending
`demoTimer`, so no queued step can leak in afterwards). When it finishes the board is left
solved with `done` still true and the button reading "再看一遍" (`playDemo` nulls
`demoTimer` there too).
The demo moves discs through `stepMove()` (no legality check, no `miss`) and never calls
`startTimer()`/`win()`, so it can neither start the clock nor write a `hanoi-best` record.

Game rules baked into the code — preserve them when editing:

- Disc ids `0..N-1`; **id 0 is the largest and starts at the bottom** of peg 0.
  Stack arrays are bottom-to-top, so the top disc is the *last* element.
  Move validity: `topOf(from) > topOf(to)` — a larger id is a smaller disc.
  Note the counter-intuitive consequence: id 2 is *bigger* than id 4, so `ok()` rightly
  refuses to put disc 2 on disc 4. Do not "fix" that.
- 3 illegal moves (`miss`) end the game; "新的一局" (`newGame()`) fully resets.
  Timer starts on the first *legal* move, stops on win/3-miss.
- Record: `localStorage['hanoi-best']`, keyed by disc count → `{m:moves,t:seconds}`,
  kept when moves improve (time breaks ties). Shown in `#record`.
- Sound is synthesized via WebAudio (`beep()`); `🔊` button toggles `soundOn`.
  It must be created lazily inside a user gesture or Chrome blocks it.

Geometry is percentage-based so the board scales with `#board` (design space
680×360): `pegX=[16,50,84]` are peg-center %, `BASE=15.5556` is the bottom disc's
offset and `STEP` is the per-disc offset (now derived from `N` in `build()`, capped
at 6.6667), `wPct()` is disc width in % of width. The CSS `.rod` values
(`bottom:15.5556%`, `height:69.4444%`) mirror `BASE`/`ROD_TOP` — **change them
together**. Rod centers come from `.pg:nth-child(n){left:…}` (0/34/68% + 32% width),
so keep `pegX` and those `left` values in sync.

## How to verify (there is no test suite)

Extract the script and syntax-check it:

```powershell
$c = Get-Content .\hannuo.html -Raw -Encoding UTF8
if ($c -match '(?s)<script>(.*?)</script>') { $Matches[1] | Out-File $env:TEMP\h.js -Encoding UTF8; node --check $env:TEMP\h.js }
```

Manual checklist (from the skill file — this repo's past failure was shipping a
truncated file):

- File ends with `</script></body></html>`, not cut off mid-function; no stray `...`,
  `TODO`, or "待写" markers left behind.
- Braces/brackets balanced, tags closed.
- Actually open it in a browser and play a move before reporting done.

## Workflow rules (from `skill-不要重蹈覆辙-写文件.md` — do not violate)

- **One step at a time, then stop.** Step 1 = minimal playable game; step 2 = visual polish;
  step 3 = extras (sound/animations) *only if asked*. After each step, report and wait for
  the user's go-ahead. Do not continue autonomously.
- **Delete and rewrite a broken file instead of patching it.** Appending fixes to a
  truncated file is what made it a mess last time.
- **Verify before claiming completion.** Never report "done" without the checks above.
- **Don't expand scope.** No extra features, effects, or refactors the user didn't ask for.

## Conventions & environment gotchas

- Files are UTF-8 with Chinese filenames/content. Windows PowerShell 5.1 renders Chinese
  output as mojibake by default — read files with the Read tool or `-Encoding UTF8`;
  quote Chinese paths with `-LiteralPath`.
- State lives in module-level globals (`N`, `P`, `sel`, `moves`, `BEST`); `localStorage`
  key `hanoi-best` holds the record. Keep that structure unless the user asks to refactor.
- UI strings (status messages, button labels) are Chinese — write new ones in Chinese to match.
