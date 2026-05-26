# Week 6 — KV Store on Hardware

**Goal**: All Week 3 + Week 4 tests pass on the M1s board.

## Context

You have a working HAL on hardware (Week 5) and a tested `nv_kv` on host
(Weeks 3–4). This week you cross-compile `nv_kv` for BL808 and prove the same
tests pass. Ideally `nv_kv.c` itself doesn't change — only the build wiring.

## Punch list

### Cross-compile nv_kv
- [ ] Build `nv_kv.c` + smoke test for BL808 target
- [ ] Flash and run on M1s
- [ ] Smoke test passes (mount → 20 puts → reboot → 20 gets)

### Physical power-loss tests
- [ ] Build a hardware torture test: 50–100 iterations of (put → physically unplug power → re-power → mount → verify valid state)
- [ ] **Compaction window**: fill bank, then power-cycle during the compaction window — verify valid state
- [ ] UART logging so failures are diagnosable without a debugger

### Measurements
- [ ] Mount latency at boot (cycles or µs from `hal_flash_init` to mount complete)
- [ ] Per-op timing: put / get / delete / compaction
- [ ] Actual flash usage for the realistic NV set from Week 4

### Demo prep
- [ ] UART log lines for compaction events ("compaction triggered", "bank swap complete")

## Deliverables
- `nv_kv` runs on M1s (no code changes vs Week 4 ideally — only build wiring)
- `STATUS.md` includes timing numbers and physical power-loss test results

## Exit criteria
- All Week 3 + Week 4 tests pass on hardware
- Physical power-yank test passes 50+ iterations
- Mount latency documented

## Reporting (Friday)
- Live demo: write 20 keys, **physically yank the USB power**, plug back in, watch the boot log show all 20 keys recovered
- Fill in `STATUS.md`

## Tips
- Physical power-cycle exposes bugs that `SIGKILL` on host cannot — the flash chip itself may have programmed partial words
- A USB cable with an inline switch is the best tool for power-cycling
- If a physical power test fails but `SIGKILL` test passed: suspect the flash chip's write-in-progress behaviour. The chip may need a settling delay after write before next op
- Keep the test harness lean — every UART character costs time on a slow board
- If you find an `nv_kv.c` bug here, fix it once and re-run the **host** torture test before re-flashing — much faster iteration
