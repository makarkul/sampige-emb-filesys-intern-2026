# Week 7 — StorageSvc Backend Integration

**Goal**: `StorageSvc.c` routes `DEV_FLASH` calls to `nv_kv` on M1s. PhbService writes survive reboot end-to-end.

## Context

You have working flash storage primitives. This week you wire them into the
existing service framework. **Thread safety matters here** — services have a
strict task-ownership model that you MUST follow.

## Punch list

### Re-read the contract (BEFORE coding)
- [ ] `services/CLAUDE.md` — full read, especially "Thread Safety Contract" and the per-service "Key Behaviours" sections
- [ ] Understand: which task calls into `StorageSvc`? Today it is any of: AT handler, LVGL task, SmsSrv, PhbSrv, BgWorker
- [ ] Design decision: serialise all `nv_kv` access via a single mutex (simplest correct answer)

### Storage backend wiring
- [ ] `services/storage/storage_flash.c` mapping the three formats to `nv_kv`:
  - `srv_storage_record_*` → key format `"rec:<store>"`, value is the packed record array
  - `srv_storage_file_*`   → key format `"file:<path>"`, value is the file blob
  - `srv_storage_nvm_*`    → key format `"nvm:<key>"`, value is the blob
- [ ] **Single mutex** around the `nv_kv` instance; taken on every public call
- [ ] In `StorageSvc.c`, for `device == DEV_FLASH`:
  - On BL808 target: route to `storage_flash.c`
  - On SDL target: keep current POSIX-file path (under `#ifdef SDL_HOST`)
- [ ] **Factory-region read-only path**: `srv_storage_nvm_is_readonly("imei")` returns `true`; reads come from a separate `factory_region_read()` helper

### PhbService activation
- [ ] Replace `phb_load_contacts()` stub with calls to `srv_storage_record_read_all(DEV_FLASH, "contacts", ...)`
- [ ] Replace `phb_save_contact()` stub with calls to `srv_storage_record_write_all(DEV_FLASH, "contacts", ...)`
- [ ] Test: add contact → reboot M1s → contact still there

### One more service for breadth
- [ ] Settings: persist language preference via `srv_storage_nvm_write("lang", ...)`
- [ ] Test: change language → reboot → language preserved

## Deliverables
- `services/storage/storage_flash.c` (committed)
- `StorageSvc.c` routes `DEV_FLASH` correctly
- PhbService stubs replaced
- Settings language preference persisted

## Exit criteria
- On M1s: add a contact in Phonebook → power cycle → contact present
- On M1s: change language in Settings → power cycle → language preserved
- **SDL Docker build still works** (`./osmo-mmi-demo.sh build && ./osmo-mmi-demo.sh run` — no regression for app developers)

## Reporting (Friday)
- Live demo: M1s side-by-side with SDL build. Add a contact on M1s, power-cycle, prove it persists. Show SDL build still works
- Fill in `STATUS.md`

## Tips
- The mutex around `nv_kv` is HELD across multi-step ops — if you put a key while holding the mutex, no other caller can read until you're done. That's fine; storage ops should complete in single-digit ms
- Watch out for the LVGL task calling synchronous storage — that can stall the UI for a flash erase (50–500 ms). If you see UI hangs, route through BgWorker (see `services/CLAUDE.md` "BgWorkerService — Key Behaviours")
- **Do not change `StorageSvc.h`**. If you find yourself wanting to, stop and discuss
- Keep changes to PhbService minimal — only the two stub functions should need updating
- Test the SDL Docker build (`./osmo-mmi-demo.sh build`) EARLY in the week — don't discover on Friday that desktop devs can't build
