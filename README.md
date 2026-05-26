# Sampige Embedded Filesystem Internship

8-week internship project: build a flash-backed storage service for Sampige OS,
demoable on the Sipeed M1s board.

## Where to start

1. Set up your local environment — see **Setup** below.
2. Read [`INTERN_PLAN.md`](INTERN_PLAN.md) — the full plan: goals, architecture
   decisions, reporting cadence, risks.
3. Open [`weeks/week1/README.md`](weeks/week1/README.md) — your first week's
   detailed punch list.
4. Each Friday, fill in `weeks/week<N>/STATUS.md` and commit it.

## Setup

This repo holds only planning artifacts. The osmo-mmi codebase you will modify
is a separate repo that lives locally at `refs/osmocom-demo/` (not tracked
here — see `.gitignore`). Populate it once after cloning:

```bash
# 1. Clone the parent osmocom-demo repo and its submodules (osmo-mmi is one of them)
git clone https://gitlab.vayavyalabs.com:8000/sampigesemi/osmocom-demo.git refs/osmocom-demo
cd refs/osmocom-demo
git submodule update --init --recursive

# 2. Create your work branch inside osmo-mmi
cd osmo-mmi
git checkout -b feature/flash-storage origin/main

# 3. Verify the Docker SDL build works (Week 1 onboarding)
./osmo-mmi-demo.sh build
./osmo-mmi-demo.sh run      # or: ./osmo-mmi-demo.sh vnc  if headless
```

All your code commits go on the `feature/flash-storage` branch in the
**osmo-mmi** repo. This planning repo only receives weekly `STATUS.md` updates
and design docs under `docs/`.

## Repo layout

```
README.md                       This file
INTERN_PLAN.md                  Full 8-week plan (the master)
docs/                           Architectural documents you write
refs/
  SS2000_Memory_Arch_v7_1.pptx  Sizing target reference (4 MB flash budget)
weeks/
  week1/ … week8/               Per-week brief + status template
```

## Working agreement (summary — full version in INTERN_PLAN.md §7)

| Cadence | What |
|---|---|
| Daily | Async standup: yesterday / today / blockers (3 lines) |
| Wednesday | 1:1 with mentor, 30 min |
| Friday | Live demo (15 min) + `STATUS.md` committed |
| Week 4 | Mid-internship review (30 min, with sponsor) |
| Week 8 | Final demo + handoff (45 min, with sponsor) |
