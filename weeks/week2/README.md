# Week 2 — HAL Interface & On-Flash Format Design

**Goal**: Lock the two contracts that everything else depends on. **Design first — no production code yet.**

## Context

This week defines two interfaces:

1. The **HAL flash interface** (`hal_flash.h`) — what every driver implements
2. The **on-flash KV format** (`nv_kv_format.md`) — the byte layout in NOR

Both must be reviewed and signed off before Week 3 starts coding. Getting the
format wrong here means re-flashing data through every later week. Cheap to
fix now, expensive to fix later.

## Punch list

### HAL interface (the abstract contract)
- [ ] Write `platform/hal/hal_flash.h` exposing:
  ```c
  int      hal_flash_init(void);
  int      hal_flash_read   (uint32_t addr, void *buf, uint32_t len);
  int      hal_flash_program(uint32_t addr, const void *buf, uint32_t len);
  int      hal_flash_erase  (uint32_t addr, uint32_t len);
  uint32_t hal_flash_sector_size(void);
  uint32_t hal_flash_total_size(void);
  ```
- [ ] **Header MUST be platform-blind** — no `#ifdef BL808`, `#ifdef SDL`, no SDK includes
- [ ] Document error codes (e.g., `HAL_FLASH_OK`, `HAL_FLASH_EALIGN`, `HAL_FLASH_EIO`)

### Flash geometry (M1s board fact-finding)
- [ ] Read the BL808 datasheet / M1s board schematic to confirm:
  - Sector size (typically 4 KB on SPI-NOR)
  - Page program size (typically 256 B)
  - Total flash size
- [ ] Record findings in `docs/m1s_flash_map.md` (geometry section only — partition map comes in Week 5)

### SDL emulator driver (the first implementation)
- [ ] Implement `platform/drivers/sdl/flash.c` against the HAL header
- [ ] Backing store: a single `flash_image.bin` file in `/workspace/data/<instance>/`
- [ ] **Enforce NOR semantics correctly**:
  - Writes can flip `1→0` only — any `0→1` transition must fail
  - Erase fills the whole sector with `0xFF`
  - Cross-sector erase must erase each sector independently (don't allow partial)
- [ ] CMake target for SDL build links this driver

### Unit test
- [ ] `tests/test_hal_flash.c` covering:
  - Write-then-read returns the same bytes
  - Erase-then-read returns all `0xFF`
  - Writing `0x00` over `0xFF` succeeds; writing `0xFF` over `0x00` fails
  - Cross-page program (e.g., 300 bytes spanning a 256-byte page boundary) works
  - Address alignment violations return `HAL_FLASH_EALIGN`

### On-flash format spec
- [ ] Draft `docs/nv_kv_format.md` covering:
  - **Bank header**: magic, version, sequence number, CRC
  - **Entry layout**: `{key_hash, format, len, payload, crc}`
  - **Active-bank selection rule**: highest sequence number with passing CRC
  - **Atomic update protocol**: write new entry → seal new entry's CRC → old entry is shadowed by key collision
  - **Compaction trigger**: active bank ≥ 75% full
  - **Compaction protocol**: copy live entries to other bank, seal new bank header (with seq+1), mark old bank dead
- [ ] **Mentor walks through `nv_kv_format.md` on paper. No nv_kv code is written until this is signed off.**

## Deliverables
- `platform/hal/hal_flash.h` (committed)
- `platform/drivers/sdl/flash.c` (committed)
- `tests/test_hal_flash.c` (committed, passing)
- `docs/m1s_flash_map.md` (geometry section)
- `docs/nv_kv_format.md` (signed off by mentor)

## Exit criteria
- All five deliverables committed on the feature branch
- Unit test passes via `make test_hal_flash` (or whatever target you wire)
- Mentor signature on the format spec — record in `STATUS.md`

## Reporting (Friday)
- Live demo: run the unit test; walk through the format spec
- Fill in `STATUS.md`

## Tips
- **NOR rule**: bits only go `1→0` on write. To set a bit back to `1`, you must erase the whole sector
- Keep the bank header small (16–32 bytes is plenty). It's read on every mount
- **Do not store absolute pointers** in the format — use offsets relative to the bank base. Banks may swap addresses
- CRC: pick CRC-32 IEEE (well-tested; zlib has one). Do not roll your own
- The unit test will run again on hardware in Week 5 — write it **portably** (no SDL-specific includes)
- Your SDL driver must enforce NOR semantics or it hides bugs that will bite on real flash
