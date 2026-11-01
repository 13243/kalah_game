# Oware (Mancala)

A two-player Oware/Awari game I wrote in C++ for a class. It uses Win32 for
the window and legacy OpenGL (fixed-function pipeline, no shaders) for
drawing. There's a computer opponent that plays with alpha-beta pruning at
three difficulty levels.

The game is Oware: 6 pits per player, 6 stones to start, 2 scoring kalahs.
Players alternate picking up a pit and dropping stones counter-clockwise,
with the standard capture rules — land your last stone in your own kalah,
get another turn; land in an empty pit on your side with stones opposite,
you capture both.

## What's in it

- Player vs Player and Player vs Computer modes
- Three AI difficulties (search depth 1, 3, or 5 plies)
- Clickable menu and pits (no keyboard control except Esc to quit)
- Win/lose/draw detection when a side empties
- Standard Oware capture and extra-turn rules

## Building

You need Visual Studio with C++ and Windows SDK. MSVC only — the code uses
Win32 (`windows.h`, `WinMain`) and links `opengl32.lib`.

Put `stb_easy_font.h` next to the `.cpp`. Get it from
https://github.com/nothings/stb (it's a single header, no dependencies).

The code has a hardcoded include at the top:

```cpp
#include <C:\Users\katle\OneDrive\Desktop\C Programs\CS_sem2\stb_easy_font.h>
```

Change that to a relative path, or move `stb_easy_font.h` to the same
folder and use:

```cpp
#include "stb_easy_font.h"
```

Then from a Developer Command Prompt:

```
cl /EHsc oware.cpp opengl32.lib gdi32.lib user32.lib
```

Or just open it in Visual Studio and build. There's a `#pragma comment` for
`opengl32.lib` already in the file, so linking usually just works from VS.

## How to play

Launch the exe. Menu:

- **Player** — two humans, hot-seat
- **Computer** — pick a difficulty, then play against the AI
- **Exit** — quits

In game, click a pit on your side. The green rectangles on the left and
right are the player's own scoring kalahs at the bottom and top. When a side
runs out of stones the remaining stones are swept into the opposite kalah
and the winner is printed in the middle of the board.

It's strictly turn-based with mouse only. Press Esc any time to quit.

## How it works

Board is a flat `int[14]`:

```
[0..5]   Player A's pits
[6]      Player A's kalah
[7..12]  Player B's pits
[13]     Player B's kalah
```

Stones are distributed counter-clockwise. The turn-skipping logic (skip the
opponent's kalah when wrapping around) is in `performTurn`, and the capture
check is the same function — both are slightly duplicated in
`checkSimulatedMove`, which is the version the AI calls on a copied board so
it doesn't touch the real state.

Rendering is stateless per frame: the whole board is rebuilt every loop
iteration from `initialBoard`, and `glClear` + redraw + `SwapBuffers`. No
dirty flags, no caching. It's fine at 60 fps for a board this size.

## AI

`determineBestTurn` tries each of the six legal moves, simulates it, and
scores the result. `pruneWithAlphaBeta` does the recursive search with
alpha-beta pruning. The evaluation function is:

```
(board[B_kalah] - board[A_kalah]) * 100 + stones_on_B - stones_on_A
```

so it weights the score difference heavily and the stone count as a
tiebreaker. If a move gives Player B another turn, the search doesn't
decrement depth for that ply — B keeps searching — otherwise it hands the
turn to A and decrements. That's the standard trick to avoid the AI
under-valuing extra turns.

Search depth is set by the difficulty enum:

```cpp
enum difficultyLevel { EASY = 1, MEDIUM = 3, HARD = 5 };
```

On modern hardware this is instant at EASY and MEDIUM, and roughly a second
at HARD. If you want it stronger, raise HARD to 7 — it gets noticeably
slower but not unreasonable.

## Known issues

These are problems in the code, not things I'm going to fix but worth
flagging if anyone reads this:

- **`displayError` is broken.** It does `*msg = "..."` which assigns a
  pointer to a `char`. That's technically a compile error in modern MSVC
  under `/W4` and nonsense at runtime. It's never called anyway — the
  console UI it was for got replaced by OpenGL, but I left it in.
- **`getMax` / `getMin` use `==` instead of `=`.** `currentTurn == 1;` is a
  comparison, not an assignment. Dead expression, compiler warns, doesn't
  break anything but it's sloppy. Should be `currentTurn = 1;` or just
  removed.
- **`checkEmptySideSingleTurn` doesn't return on all paths.** When both
  sides are non-empty it returns `1`, when a side is empty it distributes
  and falls off the end (undefined return value). In practice the caller
  only reads the return value in the `!checkEmptySideSingleTurn(...)`
  branch, so it works, but it's a real bug waiting to bite.
- **No bounds check on button arrays.** `WM_MOUSEMOVE` iterates
  `buttons[i]` up to 14 but `buttons[]` only has 3 elements and
  `gameButtons[]` has 12. So that loop reads past the end of the small
  array. UB, just happens not to crash on Windows because the memory
  nearby is the start of `gameButtons`. I'd fix this if I had to ship it.
- **Hardcoded file path in the include.** Already mentioned in Build.
- **Legacy OpenGL.** `glBegin`/`glEnd` era, no vertex buffers, no shaders.
  Won't run on macOS past 10.14, won't work with GLFW/Vulkan, etc. Fine for
  a Win32 app targeting Windows 7+.
- **Button hitboxes are for a fixed window.** They're in the ortho
  coordinates set by `glOrtho(0, windowWidth, windowHeight, 0, -1, 1)` in
  `WM_SIZE`, so resizing the window does scale them, but the layout assumes
  the default 1080×800. Smaller windows clip the board.
- **No unit tests.** The AI could be tested with fixture boards — I never
  did.

## License

MIT.
