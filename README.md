# SMOKEY: latest files (25 Nov 2026)

This folder holds the current version of everything that is built. Older dated snapshots are in `../archive/`.

| File | What it is |
|---|---|
| `smokey-demo.html` | The interactive demo, one file, 27 screens. Open it in a browser. Identical to `../deploy/index.html`. |
| `smokey-cortex-data-refresh-spec.md` | Spec for refreshing the demo page with actual data through Snowflake Cortex: page contract, data schema, source views, validation rules, the splice script and the agent instructions. Same file as `../specs/specs/013-cortex-data-refresh/spec.md`. |
| `demo-source.tgz` | Everything needed to rebuild the demo: `build_controls.py`, the six logic files, the base template and the Playwright tests. Unpack, then run `python3 build_controls.py` inside `demo-build/`; it writes `demo/smokey-demo.html`. |
| `design-canvas.tgz` | All 53 design boards and `canvas.json` as published, plus the four generator scripts. |

## Current elsewhere in the smokey folder
- `../deploy/`: Dockerfile, Kubernetes manifests and `index.html` (the demo).
- `../specs/`: Spec Kit specs 001 to 012, constitution, and `smokey-specs-v7-2026-11-25.tgz`.
- `../research/`: the two research reports and their notes.
- `../synthetic-portfolio/`: the 40-model synthetic data.
- `../control-plane-design-decisions.md`, `../stakeholder-control-review-2026-11-23.md`, `../model-board-coverage-2026-11-24.csv`.
- `../SMOKEY Control Plane Proposal.pdf` and `.pptx`: the proposal deck as exported on 22 Nov; the published deck has changed since.

## Published copies
- Demo: https://claude.ai/artifact/S6s6vBLkKScMbjAf5wxjNG
- Design canvas: https://claude.ai/artifact/G857d6ZwzK3fVtpcPUbGpW
- Proposal deck: https://claude.ai/artifact/QDs5afVN1jVe69Qi25ympn

## What moved to the archive
All dated snapshot folders (21 to 25 Nov), the two backup folders, `Archive.zip`, and spec archives v2 to v6. Nothing was deleted.

## Latest change (25 Nov)
AI agents are marked in purple (#6B2FA0 on #F3ECFA; lighter purple on dark) with an "AI agent" tag, in the demo and on 44 design boards. The previous `latest` files are in `../archive/latest-before-agent-colour-2026-11-25/`.

## Change on 25 Nov (Cortex refresh)
The demo keeps its data in one JSON block and labels each screen as actual, partly actual or illustrative once real data is loaded. Cycle time has a pinned scroll story. The previous `latest` files are in `../archive/latest-before-cortex-spec-2026-11-25/`.

## Change on 25 Nov (audit trail)
Audit trail screen added (28 screens): one model at a time, with a summary of it and a summary across all models. The Cortex spec gains an optional `trail` section and a source view for it. Previous files: `../archive/latest-before-audit-trail-2026-11-25/`.
