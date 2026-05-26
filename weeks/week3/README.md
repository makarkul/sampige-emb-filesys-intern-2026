# Week 3 — KV Store on Host

**Goal**: `nv_kv` compiles, mounts, and persists data across "reboot" (process restart) against the SDL emulator.

## Context

You have the HAL (Week 2) and the on-flash format (Week 2). This week you write
the code that puts and gets keys using that format, against the SDL emulator.
Everything runs on your laptop — no hardware yet.

## Punch list

### Implement nv_kv
- [ ] `services/storage/nv_kv.h` — public API:
  ```c
  int nv_kv_mount   (void);                                                /* scan banks, pick active */
  int nv_kv_unmount (void);
  int nv_kv_get     (const char *key, void *buf, uint32_t max, uint32_t *out_len);
  int nv_kv_put     (const char *key, const void *val, uint32_t len);
  int nv_kv_delete  (const char *key);
  int nv_kv_format  (void);                                                /* wipe both banks, fresh start */
  ```
- [ ] `services/storage/nv_kv.c` implementing the format from `docs/nv_kv_format.md`:
  - **Mount**: scan both banks, pick the one with highest seq number and valid CRC
  - **Put**: append entry to active bank; older entries with same key become shadowed
  - **Get**: scan active bank from end backwards (newest wins); return first match
  - **Delete**: append a tombstone entry
  - **Compaction**: when active bank ≥ 75% full, copy live entries to the other bank, seal new bank header with `seq+1`, mark old bank dead
- [ ] CRC over every entry; corrupted entries skipped at scan time

### Smoke test
- [ ] `tests/test_nv_kv_smoke.c`:
  - mount → put 20 keys → unmount → mount → get all 20 → verify
  - put same key twice → second value wins
  - delete a key → get returns "not found"
  - put 100 keys to trigger compaction → verify all keys still readable after compaction

### Build integration
- [ ] Wire `nv_kv.c` into `fs/CMakeLists.txt` (currently just a placeholder)
- [ ] Wire smoke test into a `make test` target

## Deliverables
- `services/storage/nv_kv.{h,c}` (committed)
- `tests/test_nv_kv_smoke.c` (committed, passing)
- `fs/CMakeLists.txt` updated

## Exit criteria
- Smoke test passes
- Compaction visibly happens at the 75% threshold (add a log line so it's visible during the demo)
- Re-mount after every test reads back exactly what was written

## Reporting (Friday)
- Live demo: write 20 keys, `kill -9` the process mid-write, restart, prove data intact
- Fill in `STATUS.md`

## Tips
- "Scan from end backwards" is the simplest way to make latest-wins semantics work
- Hash the key with a fast hash (FNV-1a 32-bit is fine) to compare keys cheaply
- Keep the active bank's free pointer in RAM after mount — do not re-scan on every put
- After compaction, the new bank's seq number must be `> old bank's seq` so re-mount picks it
- For the kill-and-restart demo: `kill -9 $(pgrep test_nv_kv_smoke)` mid-write, then re-run and verify
- Use Valgrind on the smoke test once it passes — uninitialised reads here become silent corruption later
