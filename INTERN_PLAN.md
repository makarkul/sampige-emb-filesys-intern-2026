# Sampige Embedded Filesystem Internship — 8-Week Plan

**Project**: Flash-backed Storage Service for Sampige OS
**Duration**: 8 weeks
**Intern**: 2nd-year CS, embedded/robotics background, new to RTOS service architecture
**Hardware**: Sipeed M1s (BL808 RISC-V SoC) for all experiments
**Sizing target**: SS2000 SoC (4 MB flash shared with code + FOTA + NV) — see `refs/SS2000_Memory_Arch_v7_1.pptx`
**Working tree**: `refs/osmocom-demo/osmo-mmi/` (osmo-mmi service layer)
**Branch**: `feature/flash-storage` (forked from `osmo-mmi/main`)

---

## 1. Goal

Implement a flash-backed `DEV_FLASH` backend for the existing `StorageSvc` API in `services/storage/StorageSvc.h`, so that on the M1s demo, Phonebook entries, SMS, settings, and NVM keys persist across power cycles.

The API contract (`StorageSvc.h`) is **fixed and not to be changed**. The intern is implementing the flash backend behind that interface; the SDL POSIX-file backend remains as the reference for desktop builds.

## 2. Non-Goals

- SD card backend (`DEV_SD`) — next-internship problem
- SIM card backend (`DEV_SIM1`, `DEV_SIM2`) — depends on AT+CPBR/CPBW work, separate track
- Any change to `StorageSvc.h` API
- Camera JPEG storage on internal flash (will go to SD on production hardware)
- FOTA implementation (SS2000 SBL team owns this — we only need to be FOTA-safe)

## 3. Architecture Decisions (locked Week 0, before kickoff)

| Decision | Choice | Rationale |
|---|---|---|
| FS approach | **Raw KV/log partition** (no LittleFS) | Bytes matter at 4 MB; ~60 KB saving vs LittleFS; no large-file use case on internal flash |
| Partition layout | Two-bank ping-pong | Atomic update; survives power loss; aligned to NOR sectors |
| Storage volume budget (SS2000 target) | 64 KB total (2 × 32 KB banks) | Fits SS2000 4 MB minus code/resources/FOTA scratch |
| Storage volume on BL808 (dev) | 256 KB (room for testing) | M1s flash is bigger; design still targets 64 KB footprint |
| Code organisation | `services/storage/nv_kv.c` + `platform/hal/hal_flash.h` (interface) + `platform/drivers/<target>/flash.c` (driver) | HAL is the **abstract contract** (platform-blind); per-target drivers live under `platform/drivers/` and are selected at link time. Do **not** put platform code inside the HAL layer. |
| Factory keys | Separate read-only region | IMEI / calibration / HW-rev — written once, never reformatted |

## 4. Layer Cake (what she builds, bottom-up)

```
┌──────────────────────────────────────────┐
│  StorageSvc DEV_FLASH backend            │  ← Week 7
│  (replaces POSIX file ops for FLASH)     │
├──────────────────────────────────────────┤
│  nv_kv.c — raw KV/log store              │  ← Weeks 3–4 (host) / Week 6 (M1s)
│  two-bank ping-pong, CRC, compaction     │
├──────────────────────────────────────────┤
│  platform/hal/hal_flash.h                │  ← Week 2
│  abstract interface: read/program/erase  │     (platform-blind contract)
├──────────────────────────────────────────┤
│  Driver — one of, selected at link time: │
│   platform/drivers/sdl/flash.c           │  ← Week 2  (host-file emulator)
│   platform/drivers/bl808/flash.c         │  ← Week 5  (real SPI-NOR via M1s SDK)
├──────────────────────────────────────────┤
│  BL808 SPI-NOR (M1s hardware) / host fs  │
└──────────────────────────────────────────┘
```

## 5. References

| Reference | Purpose |
|---|---|
| `refs/SS2000_Memory_Arch_v7_1.pptx` | Sizing target — 4 MB flash budget, single-bank FOTA model, NV partition footprint |
| `services/storage/StorageSvc.h` (in working tree) | Fixed API contract she implements behind |
| `services/CLAUDE.md` (in working tree) | Service threading rules — must read before Week 7 |
| `external/M1s_BL808_SDK` (working-tree submodule) | BL808 flash driver primitives for Week 5 |
| LittleFS DESIGN.md (https://github.com/littlefs-project/littlefs/blob/master/DESIGN.md) | Background reading — we are *not* using it, but the failure modes it documents inform our KV design |

## 6. Weekly Plan

### Week 1 — Onboarding & Read-Only Exploration

**Goal**: Understand the layer cake before touching it.

**Tasks**
- [ ] Install Docker; from `refs/osmocom-demo/osmo-mmi/` run `./osmo-mmi-demo.sh build && ./osmo-mmi-demo.sh run` (or `vnc` if headless). All SDL2/cmake/toolchain deps are inside the container — do NOT install them on host.
- [ ] Exercise Phonebook + Messages + Settings; observe the bind-mounted `data/<instance>/` directory filling up (this is the container's `/workspace/data/`)
- [ ] Read `services/storage/StorageSvc.h` — the API contract she'll implement
- [ ] Read `services/storage/StorageSvc.c` — the SDL reference behaviour
- [ ] Read `services/CLAUDE.md` — service threading rules
- [ ] Read `services/framework/sfw.h` + `sfw_lifecycle.c` — service registration & init
- [ ] Read `services/phb/PhbService.c` — concrete storage caller (see `phb_load_contacts` / `phb_save_contact` stubs)
- [ ] Read `refs/SS2000_Memory_Arch_v7_1.pptx` slides 2, 4, 11, 12 — the flash budget context
- [ ] Read LittleFS DESIGN.md — for failure-mode awareness
- [ ] Draft a one-page architecture diagram showing how `srv_phb_add_contact()` flows down to flash

**Exit criteria (outcome)**
- A committed `docs/storage_arch.md` with the layer diagram
- She can verbally walk a mentor through the three formats (RECORD / FILE / BINARY) and the four devices
- SDL build runs cleanly on her machine

**Reporting**: Friday demo — walk the diagram. Status note in `STATUS/week1.md`.

---

### Week 2 — HAL Interface & On-Flash Format Design

**Goal**: Lock the two contracts that everything else depends on. **No code that matters yet — design first.**

**Tasks**
- [ ] Write the HAL **interface** at `platform/hal/hal_flash.h`: `init`, `read`, `program`, `erase`, `sector_size`, `total_size` — **no platform-specific code in this header**
- [ ] Decide and document the BL808 SPI-NOR flash geometry (sector size, page program size, total size) — verify against M1s board datasheet, not assumed. Record in `docs/m1s_flash_map.md` (geometry only this week; partition map in Week 5)
- [ ] Implement the SDL **driver** at `platform/drivers/sdl/flash.c` — backs the HAL with a `flash_image.bin` file; **enforces NOR semantics** (1→0 writes only; erase resets sector to 0xFF)
- [ ] Wire CMake so the SDL target links `platform/drivers/sdl/flash.c` against the HAL header
- [ ] Write a small unit test: write-read, erase-read, 1→0 violation expected to fail
- [ ] Draft `docs/nv_kv_format.md` covering: bank header layout, entry layout `{key, format, len, payload, crc}`, atomic-update protocol, compaction trigger
- [ ] **Mentor sign-off on `nv_kv_format.md` on paper before any nv_kv code is written**

**Exit criteria (outcome)**
- `platform/hal/hal_flash.h` committed (interface only, no `#ifdef <target>`)
- `platform/drivers/sdl/flash.c` committed; unit test passes against SDL target
- `docs/nv_kv_format.md` committed and signed off by mentor

**Reporting**: Friday demo — walk through the format spec; mentor approval logged in `STATUS/week2.md`.

---

### Week 3 — KV Store on Host

**Goal**: Filesystem-level operations work end-to-end against the emulator. Survives "reboot" (process restart).

**Tasks**
- [ ] Implement `services/storage/nv_kv.c` against `hal_flash.h`
  - [ ] Bank scan + valid-bank selection at mount time
  - [ ] `nv_kv_get(key, buf, max, out_len)`
  - [ ] `nv_kv_put(key, val, len)` — append-only write
  - [ ] `nv_kv_delete(key)` — tombstone entry
  - [ ] Compaction: when active bank ≥ 75% full, copy live entries to other bank, swap
  - [ ] CRC over each entry; corrupted entries skipped at scan
- [ ] Smoke test on host: mount → write 20 keys → unmount → re-mount → verify all 20 readable
- [ ] Boundary test: fill bank to trigger compaction, confirm swap is atomic
- [ ] Wire into the CMake build (`fs/CMakeLists.txt` and `services/storage/CMakeLists.txt`)

**Exit criteria (outcome)**
- `nv_kv.c` passes smoke + compaction tests on SDL host
- Bank swap demonstrably atomic (kill process mid-swap, re-mount, no data loss)

**Reporting**: Friday demo — write 20 keys, kill process, restart, prove data intact. Status in `STATUS/week3.md`.

---

### Week 4 — Power-Loss Robustness & Mid-Internship Review

**Goal**: Convince ourselves the KV store survives every plausible interrupt point.

**Tasks**
- [ ] Build a torture-test harness: loop { random op (put/get/delete) → SIGKILL at random point } × 10,000 iterations
- [ ] After each kill, re-mount and verify the store is in *some* valid state (either pre-op or post-op, never garbage)
- [ ] Cover the compaction-mid-swap case explicitly (it's the worst-case window)
- [ ] Add `nv_kv_format()` recovery path for the "both banks unreadable" case (e.g., first boot on virgin flash)
- [ ] Document RAM/flash overhead actually observed (vs design budget)
- [ ] **Mid-internship review with mentor**: code review of HAL + nv_kv; agree on Week 5 hardware path

**Exit criteria (outcome)**
- Torture test runs 10,000 iterations with zero data-loss failures
- Mid-internship review passed; off-ramp decision made if Week 5 hardware is at risk

**Reporting**: 30-min mid-internship review meeting. Written code-review notes in `STATUS/week4.md`. **This is the natural checkpoint to extend, descope, or change tack.**

---

### Week 5 — BL808 Flash Driver Bring-Up

**Goal**: Same `hal_flash.h` API, now talking to actual M1s SPI-NOR.

**Tasks**
- [ ] BL808 toolchain installed; flash a "blinky" from `external/M1s_BL808_example` to confirm dev board + flashing flow
- [ ] Locate flash primitives in `external/M1s_BL808_SDK` (likely `bl_flash.h` within the SDK)
- [ ] Decide and document the flash partition map for NV storage on M1s — append to `docs/m1s_flash_map.md` from Week 2 — must not collide with code, bootloader, or BL808 SDK reserved regions
- [ ] Implement the BL808 **driver** at `platform/drivers/bl808/flash.c` against the same `platform/hal/hal_flash.h` interface (create `platform/drivers/bl808/` if it does not exist; mirror the layout of `platform/drivers/sdl/`)
- [ ] Wire CMake so the BL808 target links `platform/drivers/bl808/flash.c` (the HAL header is unchanged; only the driver swaps)
- [ ] Run the Week 2 unit test on hardware; tee UART log to file
- [ ] Validate erase / program / read across sector boundaries

**Exit criteria (outcome)**
- Week 2 unit test passes on M1s board (same test binary, only the linked driver differs)
- `docs/m1s_flash_map.md` updated with the partition map on actual hardware
- HAL header (`platform/hal/hal_flash.h`) is unchanged from Week 2 — the only new code is the BL808 driver. **If the HAL had to change for BL808, the abstraction was wrong; revisit Week 2.**

**Reporting**: Friday demo — UART log of unit test on hardware. `STATUS/week5.md`.

---

### Week 6 — KV Store on Hardware

**Goal**: Week 3–4 tests now pass on M1s.

**Tasks**
- [ ] Run host smoke test on hardware (mount → 20 writes → power cycle → re-mount → read)
- [ ] Run scaled-down torture test on hardware (physical power yanks, 50–100 iterations) — covers what SIGKILL on host cannot
- [ ] Measure mount latency at boot
- [ ] Measure flash usage per KB of NV data written
- [ ] Stress: 1000 put cycles, observe wear leveling across banks
- [ ] Add UART log statements for compaction events (for visibility during demo)

**Exit criteria (outcome)**
- Hardware behaves identically to host on all KV tests
- Mount latency and per-op timing documented in `STATUS/week6.md`

**Reporting**: Friday demo — power-cycle the M1s board live, show contacts persist. `STATUS/week6.md`.

---

### Week 7 — StorageSvc Backend Integration

**Goal**: `StorageSvc.c` routes `DEV_FLASH` calls to `nv_kv` on M1s; one real service (PhbService) writes survive reboot.

**Tasks**
- [ ] Re-read `services/CLAUDE.md` thread-safety contract — required before touching service code
- [ ] Add `services/storage/storage_flash.c` mapping `srv_storage_record_*`, `srv_storage_file_*`, `srv_storage_nvm_*` to `nv_kv_get/put/delete`
- [ ] Map the three formats into the unified KV store (key naming: `rec:<store>`, `file:<path>`, `nvm:<key>`)
- [ ] In `StorageSvc.c`, route `device == DEV_FLASH` to the new backend; keep POSIX path under `#ifdef SDL_HOST` for desktop build
- [ ] Add single mutex around the `nv_kv` instance (KV store is not thread-safe by itself)
- [ ] Replace `phb_load_contacts` / `phb_save_contact` stubs with real `srv_storage_record_*` calls
- [ ] Wire factory-region read-only path: `srv_storage_nvm_is_readonly()` returns `true` for IMEI / calibration / HW-rev keys

**Exit criteria (outcome)**
- On M1s: add a contact in Phonebook → power cycle → contact still there
- On M1s: change language in Settings → power cycle → language preserved
- SDL desktop build still works via `./osmo-mmi-demo.sh build` (no regression for app developers)

**Reporting**: Friday demo — live add-contact / power-cycle / verify on M1s. `STATUS/week7.md`.

---

### Week 8 — M1s Demo, Polish, Handoff

**Goal**: Demoable end-to-end. Documentation complete. Next intern can pick it up.

**Tasks**
- [ ] Run full M1s demo with new storage stack; acceptance script:
  - [ ] Add 5 contacts in Phonebook
  - [ ] Compose 1 SMS, save as draft (exercises OUTBOX via RECORD storage — no GSM stack required)
  - [ ] Change language in Settings
  - [ ] Power-cycle the board
  - [ ] Verify all three persisted
  - [ ] (SMS-received and call-log persistence use the same storage paths — excluded from acceptance because they'd require GSM connectivity for zero extra storage coverage)
- [ ] FOTA-safety test: erase the application region of flash (simulating FOTA write); verify NV partition is intact
- [ ] Wire up `factory_reset` path: erase NV partition + format on next boot
- [ ] Final documentation:
  - [ ] `fs/README.md` — partition layout, how to add an FS-backed store, how to debug a corrupt mount
  - [ ] `docs/storage_arch.md` — updated with final implementation
  - [ ] `docs/nv_kv_format.md` — finalised with any spec changes during implementation
- [ ] Final code review with mentor
- [ ] Tag release `intern-flash-storage-v1.0`

**Stretch (only if Weeks 1–8 ran clean — do NOT promise these)**
- [ ] SD card backend (`DEV_SD`) — same nv_kv code, different HAL
- [ ] Wear-leveling stats exposed via engineer-menu screen
- [ ] CRC over NVM blobs (defense-in-depth on top of per-entry CRC)

**Exit criteria (outcome)**
- M1s acceptance script passes
- FOTA-safety test passes
- All four documentation files committed and reviewed
- Release tagged

**Reporting**: Final demo + handoff document (`STATUS/final_report.md`) summarising what shipped, what slipped, what's open.

---

## 7. Weekly Reporting Mechanism

| Cadence | What | Where | Who |
|---|---|---|---|
| **Daily** | Async standup: yesterday / today / blockers | Slack / email, 3 lines | Intern → mentor |
| **Midweek (Wed)** | 1:1 unblock session, 30 min | Video call | Intern + mentor |
| **Friday** | Live demo (15 min) + written status (`STATUS/week<N>.md`, 1 page max) | Demo on her machine; status committed to repo | Intern leads; mentor reviews |
| **Week 4** | Mid-internship review, 30 min | Video call | Intern + mentor + project sponsor |
| **Week 8** | Final demo + handoff, 45 min | Live on M1s board | Intern + mentor + project sponsor |

**Status note template** (`STATUS/week<N>.md`):
```markdown
# Week N Status — <dates>

## Shipped
- <committed work, with commit hashes>

## Slipped
- <anything planned but not done — why, and new ETA>

## Blockers
- <things she needs help with>

## Next week
- <plan, lifted from INTERN_PLAN.md>

## Demo
- <one-line description of what was shown Friday>
```

Each week's status note is a **commit on the feature branch**, not a separate doc system — keeps everything in one place and discoverable via git log.

---

## 8. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Hardware bring-up (Week 5) slips | M | H | Weeks 2–4 are all on SDL host with flash emulator. If Week 5 slips, Week 6+ can continue against emulator and merge to hardware later. |
| Custom KV code has data-loss bugs | M | H | Week 4 torture test with 10,000 iterations is the gate; LittleFS DESIGN.md study informs what failure modes to test for. |
| BL808 SDK flash driver behaves differently than expected (timing, async, IRQ) | M | M | Week 5 scoped to driver bring-up alone, before higher layers touch it. |
| Scope creep on SIM / SD | M | M | Explicit non-goals (§2). Stretch goals in Week 8 only if everything else green. |
| `services/CLAUDE.md` threading rules violated → race conditions in Week 7 | L | H | Re-read mandated in Week 7; single mutex around nv_kv instance. |
| Partition map conflicts with M1s SDK reserved regions | L | M | Week 5 dedicates time to documenting the M1s flash map before coding. |

---

## 9. Definition of Done (project-level)

- [ ] M1s board boots Sampige OS with persistent storage
- [ ] Phonebook + SMS + Settings survive power cycle
- [ ] NVM keys (incl. read-only factory keys) accessible via `srv_storage_nvm_*`
- [ ] SDL desktop build still works unchanged
- [ ] Storage code design documented well enough that a new engineer can extend it (SD, SIM) without re-deriving the format
- [ ] On-flash footprint fits within the 64 KB SS2000 target (validated on hardware with `wc -c` style accounting)
- [ ] Release tag `intern-flash-storage-v1.0` on the feature branch
