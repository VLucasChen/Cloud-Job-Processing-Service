# Cloud Job-Processing Service — Design

Design report (≤ 5 pages) for a multi-cloud service that runs a 3-stage
wildlife-image pipeline (CPU prep → GPU species ID → CPU report).
Due **Fri 9 Oct 2026 23:55**. Design only, no implementation.

> Keep this repository **private** until after the deadline (grading is rank-based).

**Status:** final (5 content pages + cover, contents, references). Facts and links checked in `notes/verification.md`.

## Layout

| Path | Purpose |
|---|---|
| `report/main.tex` → `report/main.pdf` | The report (LaTeX, Overleaf-compatible) |
| `diagrams/architecture.tex` → `.pdf` | Architecture diagram (TikZ, vector) |
| `notes/assumptions.md` | Sizing calculations and stated assumptions (bandwidth, pricing, egress, storage tiers, third parties) |
| `notes/decisions.md` | Design decision log: options, recommendation, trade-off, and my decision |
| `notes/verification.md` | Source and status for every external fact and price used |

## Build

```bash
cd diagrams && latexmk -pdf architecture.tex   # only after editing the diagram
cd ../report && latexmk -pdf main.tex
```

On Overleaf: upload `report/` and `diagrams/` keeping the folder structure, set `report/main.tex` as the main file.

## Plan

| Day | Work |
|---|---|
| until Tue 6 Oct | A1 interview |
| Wed 7 Oct | Read draft v1, decide D1–D11, mark disagreements |
| Thu 8 Oct | Rewrite key sections in my own words; failure scenario + multi-cloud deep pass; fit to 5 pages |
| Fri 9 Oct | Final proofread, export PDF, submit by afternoon |
