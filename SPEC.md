# Pointer Tracer — Build Spec

A minimalist mobile mini-game for practicing C pointer logic through "code tracing" challenges. The player reads a short, randomized C snippet, predicts the final value of a variable, and gets instant feedback plus an animated memory map that steps through the code.

> Hand this file to Claude Code as the task brief (say "Build the app described in SPEC.md, starting with milestone 1").

---

## 1. Platform & stack (default — change if you prefer)

- **Android, Kotlin, Jetpack Compose**, single-activity, MVVM (`ViewModel` + `StateFlow`).
- No network, no accounts. Everything runs offline.
- Min SDK 26. Portrait-first, works one-handed.
- Unit tests with JUnit for the generator and interpreter (these are the core logic and must be trustworthy).

## 2. Core idea: generate + simulate, never hand-write answers

The correct answer must never be hard-coded. Each challenge is produced as a small **intermediate representation (IR)** of statements, which is:

1. **Rendered** to C source text for display, and
2. **Executed** by a tiny built-in interpreter over a simulated memory model to get the answer *and* a step-by-step trace for the visualizer.

This guarantees the code shown, the answer checked, and the animation are always consistent.

### 2.1 Memory model

- A flat simulated memory of 4-byte cells. Each variable/array element occupies one cell with a fake but plausible address (e.g. starting at `0x7ffc1000`, increasing by 4; arrays contiguous).
- Types supported: `int`, `int*`, `int**`, `int[N]` (N ≤ 5), function pointers `int (*)(int)`.
- Each cell stores: name (or `arr[i]`), address, type, value (int or address).
- The interpreter records a **snapshot after every statement** → `List<MemoryState>` plus which line executed and which cells changed.

### 2.2 IR (minimum set)

- `Declare(type, name, init)` — init is a literal, `&var`, `&arr[i]`, `arr`, or another expression
- `Assign(lvalue, expr)` — lvalue: `x`, `*p`, `**pp`, `arr[i]`, `*(p + k)`, `p[i]`
- `Expr`: literal, var, `&lvalue`, `*expr`, `expr + expr`, `expr - expr`, `expr * literal`, `arr[i]`, `p++` / `++p` / `p--` (as statement or in `*p++`), function call
- `FuncDef(name, params, body)` — only small helpers: `swap(int *a, int *b)`, `inc(int v)` (by value), `inc_ref(int *v)`, `double_it(int x)`, `add_one(int x)`
- `Call(func, args)`

Keep everything well-defined C: no out-of-bounds access, no uninitialized reads, no signed overflow, no sequence-point ambiguity (never modify and read the same object in one expression beyond `*p++`).

## 3. Challenge templates

Each template is a function `(Random, Difficulty) -> Challenge` that picks names, values, and variations. Minimum library:

| # | Template | Concept | Example twist |
|---|---|---|---|
| 1 | Basic deref | `int *p = &x; *p = 7;` | reassign `p` to another var mid-way |
| 2 | Swap via pointers | pass-by-reference | call `swap` then modify one var |
| 3 | By-value vs by-reference | `inc(x)` vs `inc_ref(&x)` | both calls in same snippet |
| 4 | Array + pointer arithmetic | `p = arr; p += 2; *p = ...` | `*(p+1)`, `p[-1]` |
| 5 | Increment traversal | `*p++ = v;` in a short loop (≤ 4 iters) | ask for a middle element |
| 6 | Double indirection | `int **pp = &p; **pp = ...; *pp = &y;` | redirect then write |
| 7 | Function pointer redirection | `int (*f)(int) = double_it; ... f = add_one;` | call before and after redirect |
| 8 | Aliasing | two pointers to the same var | write through one, read through other |

**Randomization:**
- Variable names from pools: `x, y, z, a, b, n, val, count`; pointers `p, q, ptr, ptr1, ptr2, pa`; double pointers `pp, pptr`; arrays `arr, nums, arr_a, data`. No duplicates within one snippet.
- Integer literals 0–20 (small, mental-math friendly).
- Snippet length 4–10 statements, depending on difficulty.
- Reject and regenerate if: the answer equals an initial literal *and* no write touched it (too trivial), or two templates produce the identical snippet in a row.

**Difficulty** (auto-scaling): Easy = templates 1–3, one concept; Medium = 4–6, 5–8 lines; Hard = 6–8 combined, up to 10 lines, possibly two templates chained. Raise difficulty after 5 correct in a row, lower after 2 misses in a row.

## 4. Questions & input

- Question types: "What is the final value of `x`?", "What is `arr[1]`?", "What does `*p` evaluate to at the end?", "Which variable does `p` point to at the end?"
- **Number pad** for integer answers (0–9, backspace, minus sign, submit).
- **Choice chips** for "which variable does `p` point to?" style questions — 3–4 options generated from variables in the snippet, including the most likely wrong answer (e.g. where `p` pointed *before* the redirect).

## 5. Gameplay loop

1. Show snippet (syntax-highlighted, monospace, line numbers, horizontally scrollable if needed — never wrap code lines).
2. Show the question below it.
3. Player answers.
4. **Correct:** green flash + haptic tick, streak counter increments, "Next" button (or auto-advance after ~1s).
5. **Wrong:** red highlight, show the correct answer, and a one-to-three-sentence explanation generated from the trace — naming the line that decided the result (e.g. "Line 5 writes 12 through `p`, which points to `y` at that moment, not `x`.").
6. Either way, a **"Show memory"** button opens the visualizer.

Explanations are built from the trace: find the last write to the asked-about cell and describe the lvalue chain that reached it. For by-value questions, point out that the function modified its own copy.

## 6. Memory map visualizer (the key feature)

- A screen/bottom sheet showing the code on top (current line highlighted) and the memory map below.
- Each variable is a **box**: name above, value inside, address below in small muted text. Arrays are a row of adjacent boxes labelled `arr[0]…arr[n]`.
- Pointer values are drawn as **arrows** from the pointer's box to the target box (and the address shown as text too). Double pointers chain arrow → arrow. Function pointers point to a small labelled "function" box.
- Controls: ⏮ step back, ▶/⏸ auto-play, ⏭ step forward, and a scrubber over the statements.
- On each step: the changed box pulses; when a pointer is reassigned, its arrow **animates** smoothly from the old target to the new one. Pointer arithmetic slides the arrowhead along the array.
- Layout: variables in a column or grid, arrays in rows; keep arrows from crossing where practical. Must be readable on a ~380dp-wide screen.
- For function calls, show a separate small "stack frame" group for the callee's parameters while inside the call, removed on return — this is what makes pass-by-value click.

## 7. UI style

- Minimal: dark theme default (light supported), one accent color, monospace for code and values.
- Syntax highlighting colors: keywords, types, numbers, operators (`*`, `&`, `->`), comments — hand-rolled tokenizer is fine; no heavy dependency needed.
- Home screen: big "Play", difficulty selector (Auto / Easy / Medium / Hard), current streak and best streak.
- Persist best streak and stats (accuracy per template) with DataStore.

## 8. Milestones

1. **Core engine** — IR, memory model, interpreter with per-step snapshots, C renderer. Unit tests: every template, 1,000 random seeds each, asserting the interpreter never errors and the rendered code compiles in spirit (balanced braces, declared-before-use).
2. **Game screen** — snippet display with highlighting, question, number pad + choice chips, validation, feedback, explanation text.
3. **Memory visualizer** — boxes, arrows, stepping, animated pointer moves, stack frames for calls.
4. **Progression & polish** — auto difficulty, streaks, stats, haptics, theme.

Optional stretch: a debug toggle that exports the current snippet as a `.c` file with `printf` of the answer, so it can be verified with a real compiler.

## 9. Acceptance checks

- 50 consecutive generated challenges contain no duplicates and cover all 8 templates.
- For every generated snippet, the interpreter's answer matches what `gcc` would print (verify a sample of ~100 by hand or with the stretch export).
- The visualizer's final state always matches the answer checked by the game.
- Wrong-answer explanation always names a specific line number.
