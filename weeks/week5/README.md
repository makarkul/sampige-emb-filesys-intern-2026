# Week 5 — BL808 Flash Driver Bring-Up

**Goal**: Same HAL interface, now talking to real M1s SPI-NOR.

## Context

Everything you have built so far runs against the SDL emulator. This week you
bring up the real flash driver on the M1s board. If you did Week 2's HAL
design correctly, only ONE new file is needed: `platform/drivers/bl808/flash.c`.

The HAL header should NOT change. If it does, the abstraction was wrong somewhere.

## Punch list

### Toolchain
- [ ] Install BL808 RISC-V toolchain (see `targets/bl808/` README)
- [ ] Build and flash "blinky" from `external/M1s_BL808_example` to confirm the
      flashing flow works end-to-end on your specific board
- [ ] UART terminal connected (e.g., `minicom -D /dev/ttyUSB0 -b 2000000`)

### Partition map
- [ ] Find what flash regions the M1s SDK reserves (XIP region, partition table, etc.)
- [ ] Pick the NV storage region — must NOT overlap code, bootloader, or SDK reserved areas
- [ ] Document in `docs/m1s_flash_map.md` (partition section, appended to Week 2 geometry section)
- [ ] **Note**: BL808 flash is bigger than SS2000 target — give yourself headroom
      but **size the partition for SS2000 (64 KB)** so the design ports cleanly

### BL808 driver
- [ ] Create `platform/drivers/bl808/` directory (mirror layout of `platform/drivers/sdl/`)
- [ ] Implement `platform/drivers/bl808/flash.c` against the same `platform/hal/hal_flash.h` interface
- [ ] Use the BL808 SDK's flash primitives (`bl_flash.h` or equivalent — find in `external/M1s_BL808_SDK`)
- [ ] CMake: when target is BL808, link `platform/drivers/bl808/flash.c` instead of the SDL driver
- [ ] **The HAL header is unchanged from Week 2.** If you find yourself wanting to add
      a BL808-specific function to the header, **stop and discuss with mentor** — the
      abstraction is wrong somewhere

### Validation
- [ ] Cross-compile the Week 2 unit test for BL808
- [ ] Flash, run, capture UART log
- [ ] All Week 2 assertions pass on hardware

## Deliverables
- `platform/drivers/bl808/flash.c` (committed)
- `docs/m1s_flash_map.md` updated with partition section
- UART log of unit test on hardware (committed under `tests/logs/week5_hardware.log`)

## Exit criteria
- Week 2 unit test binary passes on M1s board (same test source, only the linked driver differs)
- HAL header is byte-for-byte identical to Week 2

## Reporting (Friday)
- Live demo: power on the M1s, watch the unit test pass over UART
- Fill in `STATUS.md`

## Tips
- Get blinky working first. Don't chase flash bugs while you're still fighting the flashing flow
- BL808 flash erase is slow (50–500 ms per sector) — **do not disable interrupts** around it
- The M1s SDK uses XIP for code execution. Be very sure your NV region does NOT overlap XIP, or you will brick your firmware on first write
- If a write returns success but the read does not match, suspect cache coherency: there may be an XIP cache that needs invalidating after a flash write
- **Backup your blinky image before flashing your driver test** — easy recovery if something goes wrong
- The BL808 SDK may expose flash ops as async (callback-based). If so, wrap them in a synchronous shim inside the driver — the HAL contract is synchronous
