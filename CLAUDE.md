# Bricks Breaker — CS 2606 Software Engineering

A Windows console Breakout clone in C++. Instructor base code by Chapman
(`reactivepixel`); the assignment is to complete seven numbered TODOs in
`Game.cpp` / `Game.h` while doing all source control through the Git CLI.

## Build & run

Open `Bricks.sln` in Visual Studio and build. Toolset is **v145**; there is no
`<LanguageStandard>` in `Bricks.vcxproj`, so MSVC defaults to **C++14** —
C++17 features (aggregate init with base classes, designated initializers,
`std::string_view`) are not available unless that property is added.
`ConformanceMode` is on.

`Main.cpp` enables CRT leak checking (`_CrtSetDbgFlag`), so any leak is
reported in the Output window at exit. This is a real constraint on design —
see "Value semantics" below.

### Controls

| Key | Action |
|---|---|
| ← → | Move paddle |
| Space | Launch / pause ball |
| R | Reset the round |
| Esc | Quit |
| ↑ ↓ | Speed up / slow down the frame rate at runtime |

## Architecture

```
BaseObject          position, color, visage, virtual Draw()
  ├── Ball          velocity, moving flag, Update() wall bounce
  └── Box           width/height/doubleThick, Draw(), Contains(x, y)

Console             static Win32 wrapper — colors, cursor, DrawBox, WordWrap
Game                owns ball, paddle, vector<Box> bricks, and game state
Main                fixed-timestep loop at FRAMES_PER_SECOND (15)
```

`Box` serves as both the paddle and the bricks.

Screen constants (`WINDOW_WIDTH` 80, `WINDOW_HEIGHT` 40, `FRAMES_PER_SECOND`)
live in `stdafx.h`.

## Conventions

**Update/render split.** `Update()` decides what is true; `Render()` decides
what it looks like. They never call each other — they communicate through
member variables on `Game`. The instructor pushed a commit specifically to
clarify this ("All rendering, including text, should occur in the Render
function"), so `CheckCollision()` must never print or call `Render()`.

**`Reset()` runs on every R press,** not just from the constructor. Anything
added to `Game` needs to be un-set there. Forgetting to `bricks.clear()`
silently stacks duplicate bricks on top of each other.

**Collision tests project one step ahead** — they ask whether the ball's *next*
position overlaps, not its current one.

## Where we landed, and why

These are choices, not requirements — recorded here so the reasoning survives
and so they can be revisited if something better turns up.

**We stored bricks by value.** `std::vector<Box>` rather than
`vector<Box*>`. TODO #1 asks for by-value, and going the pointer route would
have meant a manual `delete` in `Reset()`, another in the erase path, and a
destructor to write — with CRT leak checking on, any one we missed gets
reported at exit. Value semantics also sidestep the invalidation problem, which
we'd have hit anyway since TODO #5 erases from the vector mid-game. The
trade-off is copying a `Box` on every `push_back`, which at five bricks of a
vtable pointer and a few ints is not worth optimizing.

**We changed the starting brick color from `DarkGreen` to `DarkCyan`.** Hits
decrement the color by one and the brick dies at `Black` (0). The base code's
`DarkGreen` is `FOREGROUND_GREEN` = 2, so it reaches black in two hits, but
TODO #5 asks for three. `DarkCyan` is 3, which makes the count come out right.
This is a deliberate deviation from the base code, not an oversight.

**We test the paddle against the ball's current x, not its projected x.**
Found this chasing a bug where the ball would be lost in a corner with the
paddle sitting right under it. `Ball::Update()` reverses `x_velocity` at a wall
*without moving* `x_position`, so for one frame the ball sits at column 0 with
its velocity still pointing left — and projecting from there tests column −1,
off the board, where no paddle can be found. The vertical projection is still
correct and we kept it; only the horizontal one changed. Clamping the
projection into range would work equally well and is arguably more literal.

**We represent game state as an enum member rather than two bools.**
`CheckCollision()` sets it, `Render()` reads it. Two bools would work, but
`won && lost` is then a representable state that means nothing — the enum makes
that impossible to write. It also extends to a fourth state for free if the
game ever needs one.

## Gotchas

**`SetWindowSize` without `SetBufferSize`.** The window is the viewport; the
buffer is the grid behind it, and Windows defaults it to thousands of rows.
`Console::Clear()` fills `dwSize` — the *buffer* — so a mismatch means
blanking ~700,000 cells twice per frame to draw a playfield of 3,200. Set both.

**`Console::Lock()` may be a no-op.** It calls `LockWindowUpdate` on
`GetConsoleWindow()`. Under Windows Terminal (the Windows 11 default) that
returns a hidden conhost window, not the surface being painted, so the
anti-flicker mechanism does nothing. It also isn't what `LockWindowUpdate` is
for.

**`Console::Show` writes one character per call,** each with its own cursor
positioning — a full frame is ~150 syscalls.

## Known open items

- Flicker from clear-then-redraw plus per-character output. Proper fix is a
  `CHAR_INFO` back buffer presented with a single `WriteConsoleOutput`.
- Window is taller than the playfield; the paddle row is hardcoded rather than
  derived from `WINDOW_HEIGHT`.
- Space can restart the ball after a win or loss.

## Working with Claude on this repo

- **Pseudocode, not implementations.** Noah types all the real C++; the point
  is the practice. C++-flavored pseudocode with `<-` for assignment and
  `// ADD` / `// CHANGE` / `// DELETE` markers.
- **Answer the question asked.** No sweeping the file for unrelated issues, no
  full-function rewrites.
- **Never run Git.** Noah runs every branch, commit, and push himself — Git CLI
  fluency is an explicit goal of the course.
- Keep runtime traces visibly separate from code meant to be typed.
