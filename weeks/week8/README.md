# Week 8 — M1s Demo, Polish, Handoff

**Goal**: Demoable end-to-end. Documentation complete. Next intern can pick it up.

## Context

The system works. This week is about making it provably work, making it
maintainable, and shipping it. **Do not add new features this week.**

## Punch list

### Acceptance test (the demo)
- [ ] `tests/m1s_acceptance.sh`:
  - Boot M1s
  - Add 5 contacts in Phonebook
  - Compose 1 SMS and save as draft (exercises SMS OUTBOX via the RECORD format — no GSM stack required)
  - Change language in Settings
  - Power cycle
  - Verify: 5 contacts present, 1 draft SMS present, language preserved
  - **Note**: SMS-received and call-log persistence use the same storage code paths, so we don't include them in the acceptance — they'd require GSM connectivity for zero extra storage coverage
- [ ] **FOTA-safety check**:
  - Erase the application region of flash (simulating FOTA write)
  - Reboot with a fresh app
  - Verify: NV partition still intact (contacts, SMS, language still there)
- [ ] `factory_reset` path: explicit op that erases the NV partition + formats on next boot

### Documentation
- [ ] `fs/README.md`:
  - Partition layout (cross-reference `docs/m1s_flash_map.md`)
  - How to add a new FS-backed store
  - How to debug a corrupt mount
  - Footprint accounting (from Week 4 + Week 6 measurements)
- [ ] `docs/storage_arch.md` — update with final implementation (was draft in Week 1)
- [ ] `docs/nv_kv_format.md` — update with any spec changes that happened during implementation
- [ ] Add a "How to extend to SD card" section in `fs/README.md` — pointer for the next intern

### Final review
- [ ] Mentor code review session (block 2 hours)
- [ ] Address all review comments
- [ ] Tag release: `git tag intern-flash-storage-v1.0`

### Stretch (only if everything above is done — do NOT promise these in advance)
- [ ] SD card backend (`DEV_SD`) — same `nv_kv` code, different HAL driver
- [ ] Wear-leveling stats exposed via engineer-menu screen
- [ ] Secondary CRC over NVM payloads (defense-in-depth on top of per-entry CRC)

## Deliverables
- `tests/m1s_acceptance.sh` (committed, passing)
- All four docs (`fs/README.md`, `docs/storage_arch.md`, `docs/m1s_flash_map.md`, `docs/nv_kv_format.md`) up to date
- Release tag `intern-flash-storage-v1.0`
- Final handoff document in `STATUS.md`

## Exit criteria
- Acceptance script passes
- FOTA-safety test passes
- All docs reviewed and committed
- Tag pushed

## Reporting (Friday — Final Demo)
- 45-min final demo to mentor + project sponsor
- Live: acceptance script runs end-to-end on M1s
- Show: docs, code, test coverage
- Final `STATUS.md` is the handoff doc — **write it for the next person who picks this up**

## Tips
- Resist the urge to add features. Polish > novelty in Week 8
- Run the acceptance script ONCE EARLY in the week to find surprise issues
- For the FOTA-safety test, write a known pattern (e.g., `0xDEADBEEF` repeated) into the app region before erasing — proves the erase actually happened and the NV region was outside it
- Final `STATUS.md` should answer: what shipped, what slipped, what's open, what would I do differently. Be honest — it's more valuable than a polished sales pitch
