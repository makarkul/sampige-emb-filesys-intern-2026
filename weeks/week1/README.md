# Week 1 — Onboarding & Read-Only Exploration

**Goal**: Understand the layer cake before touching it.

## Context

Before changing anything, build a working mental model of:
- How the existing StorageSvc API is shaped
- How services in osmo-mmi initialise and talk to each other
- Why the SS2000 4 MB flash budget matters

**This week is read-only.** Do not modify code in `refs/osmocom-demo/osmo-mmi/`.

## Punch list

### Get the dev environment running (all via Docker — do NOT install SDL2 / cmake on host)
- [ ] Install Docker + docker-compose on your machine
- [ ] `cd refs/osmocom-demo/osmo-mmi`
- [ ] `./osmo-mmi-demo.sh build` — builds the Docker image (Ubuntu 22.04 + SDL2 + toolchain) and compiles the SDL target
- [ ] `./osmo-mmi-demo.sh run` — launches `featurephone-ui` (X11 forwarded; use `vnc` subcommand if headless)
- [ ] `./osmo-mmi-demo.sh shell` — opens an interactive shell inside the build container if you need to poke around
- [ ] Open Phonebook → add a contact; open Messages; change a setting in Settings
- [ ] Inspect `data/<instance>/` in the osmo-mmi directory (bind-mounted from container's `/workspace/data/`) — see how POSIX-file storage looks today

### Required reading (in order)
- [ ] `services/storage/StorageSvc.h` — the API contract you will implement against (227 lines)
- [ ] `services/storage/StorageSvc.c` — current SDL POSIX-file implementation (454 lines)
- [ ] `services/CLAUDE.md` — service threading rules (read twice; this matters in Week 7)
- [ ] `services/framework/sfw.h` + `sfw_lifecycle.c` — how services register and initialise
- [ ] `services/phb/PhbService.c` — find `phb_load_contacts` and `phb_save_contact` (the stubs you will wire up in Week 7)
- [ ] `refs/SS2000_Memory_Arch_v7_1.pptx` slides 2, 4, 11, 12 — flash budget context
- [ ] [LittleFS DESIGN.md](https://github.com/littlefs-project/littlefs/blob/master/DESIGN.md) — failure-mode awareness (we are *not* using LittleFS, but its design rationale informs ours)

### Deliverable
- [ ] Create `docs/storage_arch.md` with a one-page diagram showing:
  - How `srv_phb_add_contact()` flows down to storage today
  - The three formats (RECORD / FILE / BINARY) and the four devices (FLASH / SIM1 / SIM2 / SD)
  - Where the flash backend will plug in

## Exit criteria
- SDL build runs cleanly on your machine
- `docs/storage_arch.md` committed to your feature branch
- You can verbally walk a mentor through the storage layer in under 3 minutes

## Reporting (Friday)
- Live demo: open the diagram, walk through it
- Fill in `STATUS.md` in this folder and commit

## Tips
- All builds and runs go through `./osmo-mmi-demo.sh` — do not chase native SDL2 install issues on your host
- The "instance" in `data/<instance>/` is set by the `SAMPIGE_INSTANCE` env var inside the container
- LVGL renders inside a UI shell — see the "UI Shell" section of `osmo-mmi/CLAUDE.md`
- If you need to peek at build details, `How_TO_BUILD.md` documents the underlying CMake flags
- The osmo-mmi CLAUDE.md hardware constraints table is your single best reference for the production target
