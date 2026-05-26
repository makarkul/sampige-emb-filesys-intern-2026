# Week 4 — Power-Loss Robustness & Mid-Internship Review

**Goal**: Convince ourselves `nv_kv` survives every plausible interrupt point.
Mid-internship review with mentor + sponsor at end of week.

## Context

LittleFS is hammer-tested over 8 years. Our `nv_kv` is not. This week you build
the test harness that gives us confidence the custom code is correct.

This is the natural **off-ramp** for the internship. If Weeks 1–4 went rough,
this is where we agree on a smaller scope for Weeks 5–8.

## Punch list

### Torture harness
- [ ] `tests/test_nv_kv_torture.c`:
  - Loop 10,000 iterations:
    - Pick a random op: put (70%), delete (15%), get (15%)
    - Pick a random key from a small pool (say 30 keys)
    - For put: random value 4–256 bytes
    - Spawn child process to execute the op against the shared flash image
    - At a random microsecond into the op, `SIGKILL` the child
    - Re-mount in parent; verify the store is in a **valid state** (either pre-op or post-op, never garbage)
- [ ] Cover the **compaction window explicitly** — wait until bank is ≥ 70% full, then trigger one put with kill timing biased to mid-compaction
- [ ] **Deterministic mode**: seed the RNG, log the seed; failures must be reproducible

### Recovery paths
- [ ] `nv_kv_format()` handles "both banks unreadable" (virgin flash, dual corruption)
- [ ] Add an `nv_kv_fsck()` pass at mount that reports: bytes in use, dead bytes (shadowed), CRC failures

### Footprint accounting
- [ ] Measure actual flash usage for a realistic NV set: 50 contacts + 20 SMS + 10 settings + 5 NVM keys
- [ ] Document in `STATUS.md`: total bytes used, bytes wasted on shadows, compaction overhead

### Mid-internship review (Wednesday or Thursday)
- [ ] 30-min meeting with mentor + project sponsor
- [ ] Code review of HAL + `nv_kv` together
- [ ] Decide: stay on schedule / descope / extend? Document the decision in `STATUS.md`

## Deliverables
- `tests/test_nv_kv_torture.c` (committed, passing 10,000 iterations)
- Recovery `format()` path implemented and tested
- `STATUS.md` includes footprint numbers and review decision

## Exit criteria
- 10,000 torture iterations with **zero data-loss failures** (a "not in valid state after kill" failure is the gate)
- Mid-internship review complete; decisions logged
- Footprint within the 64 KB design target (or documented overage with justification)

## Reporting (Friday)
- Live: run a 1,000-iteration torture loop in front of mentor
- Fill in `STATUS.md`
- **Mid-internship review** is this week — discuss scope adjustments openly here, not at Week 8

## Tips
- The hardest bug to find is **compaction-mid-swap** where the new bank has data but isn't sealed yet — your active-bank selection must NOT pick that one
- Run the torture test under Valgrind too — memory bugs hide here
- Save the failing seed when a test fails — replay it deterministically while debugging
- If 10,000 iterations is too slow, parallelise — but only after you have a single-threaded version working
- A failure that "only happens sometimes" usually means RNG-dependent timing — log enough state per iteration to reconstruct what happened
