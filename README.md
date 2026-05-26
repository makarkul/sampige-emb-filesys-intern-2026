# Sampige Embedded Filesystem Internship

8-week internship project: build a flash-backed storage service for Sampige OS,
demoable on the Sipeed M1s board.

## Where to start

1. Read [`INTERN_PLAN.md`](INTERN_PLAN.md) — the full plan: goals, architecture
   decisions, reporting cadence, risks.
2. Open [`weeks/week1/README.md`](weeks/week1/README.md) — your first week's
   detailed punch list.
3. Each Friday, fill in `weeks/week<N>/STATUS.md` and commit it.

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

The actual osmo-mmi codebase you will modify lives at
`refs/osmocom-demo/osmo-mmi/` — that is a local working clone, not tracked in
this planning repo. Your code commits go on the `feature/flash-storage` branch
of the osmo-mmi repo.

## Working agreement (summary — full version in INTERN_PLAN.md §7)

| Cadence | What |
|---|---|
| Daily | Async standup: yesterday / today / blockers (3 lines) |
| Wednesday | 1:1 with mentor, 30 min |
| Friday | Live demo (15 min) + `STATUS.md` committed |
| Week 4 | Mid-internship review (30 min, with sponsor) |
| Week 8 | Final demo + handoff (45 min, with sponsor) |
