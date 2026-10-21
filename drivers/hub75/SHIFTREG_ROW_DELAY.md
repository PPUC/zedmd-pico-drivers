# Serial row decoder delay (SM5266P / SM5368P)

Panels with an SM5266P or SM5368P row decoder are stepped from the CPU in
`Hub75::step_shiftreg_row()`, which runs in the DMA interrupt for every row:

1. B (BK) high and C (row data) set: wait
2. A (row clock) high: wait
3. A low: wait
4. BK and row data low

If the waits are too short, the row clock skips or steps twice. On camera this
shows up as a waving or rolling image.

## Findings

We measured each wait with SysTick (counts `clk_sys` cycles) at
`clk_sys` = 266 MHz:

| Version | Requested | Measured (edge 1 / 2 / 3) | Result |
|---|---|---|---|
| `shiftreg_delay_cycles()` called for every wait (11ad2bed) | 18 | 344 / 343 / 342 | works |
| Delay cached in `start()` | 18 | 23 / 20 / 27 | too short |
| Cached, 340 cycles, `busy_wait_at_least_cycles()` | 340 | 344 / 343 / 460 | uneven |

What we learned:

- **The working version only worked because of overhead.** It asked for
  18 cycles, but calling `shiftreg_delay_cycles()` on every wait (a
  `clock_get_hz()` call from flash and a 64-bit divide) added about 325
  cycles. The panel needs much more than 18 cycles. Caching the value removed
  the overhead and exposed the real, too short delay.
- **Inlined busy-waits do not take the same time.**
  `busy_wait_at_least_cycles()` is an inline `subs #3` / `bcs` loop, so each
  call site gets its own copy. One copy was misaligned, which costs an extra
  instruction fetch per loop pass (4 cycles instead of 3). With 340 cycles
  that is about 113 passes, so about 115 extra cycles (460 instead of 344).

## Current method

- `HUB75_SHIFTREG_DELAY_CYCLES` (default 170) sets the hold time per edge
  **in cycles at 266 MHz**, about 0.64 µs. `shiftreg_delay_cycles()` scales it
  to the actual `clk_sys`, so the time stays the same at any clock speed.
  The minimum is 8 cycles.
- `start()` caches the value in `shiftreg_delay`, which keeps clock queries
  out of the ISR.
- All three edges call `shiftreg_wait()`, a single RAM function that is never
  inlined. Its loop starts with `.balign 4`, so every pass takes 3 cycles and
  all three waits take the same time (requested cycles plus a small fixed
  call overhead).

## Tuning

Override the delay at build time, for example in `platformio.ini`:

```ini
build_flags = -D HUB75_SHIFTREG_DELAY_CYCLES=200
```

The real minimum is somewhere between about 20 and 340 cycles at 266 MHz.
Lowering it shortens the time each row is blanked. If the image starts to roll,
raise it again. 170 cycles (about 0.64 µs per edge) gave the best result, so it
is the default.